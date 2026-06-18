# Shaarli 书签并发编辑保护机制分析

本文档梳理 Shaarli 项目中多人编辑同一书签时的并发保护机制，涵盖读写保护、版本检测、冲突提示与落盘顺序四个方面。

---

## 一、整体架构概览

书签数据的核心流转链路：

```
HTTP Request
    ↓
Controller (ShaarePublishController / Links API)
    ↓
BookmarkFileService 构造 → read() [锁1] → 加载到内存快照
    ↓
内存中修改书签（无锁）
    ↓
BookmarkFileService::save() → write() [锁2] → 写入磁盘
```

关键文件清单：

- [ContainerBuilder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/container/ContainerBuilder.php) — 依赖注入与 Mutex 初始化
- [BookmarkFileService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php) — 书签业务服务，含 `set()` / `add()` / `save()`
- [BookmarkIO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php) — 文件读写与互斥锁封装
- [Bookmark.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/Bookmark.php) — 书签实体，含 `updated` 时间戳
- [BookmarkArray.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkArray.php) — 内存中书签集合
- [init.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/init.php) — `SHAARLI_MUTEX_FILE` 常量定义
- [ShaarePublishController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/front/controller/admin/ShaarePublishController.php) — Web 表单保存逻辑
- [ShaareManageController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/front/controller/admin/ShaareManageController.php) — 批量操作逻辑

---

## 二、读写保护（互斥锁机制）

### 2.1 锁的选型与初始化

Shaarli 使用第三方库 `malkusch/lock` 提供的 **FlockMutex**（文件锁）作为全局互斥锁。

在 [ContainerBuilder.php#L100](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/container/ContainerBuilder.php#L95-L103)：

```php
$container['bookmarkService'] = function (ShaarliContainer $container): BookmarkServiceInterface {
    return new BookmarkFileService(
        $container->conf,
        $container->pluginManager,
        $container->history,
        new FlockMutex(fopen(SHAARLI_MUTEX_FILE, 'r'), 2),  // 第二个参数 2 = 超时时间 2 秒
        $container->loginManager->isLoggedIn()
    );
};
```

锁文件使用 `init.php` 自身作为锚点（[init.php#L63](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/init.php#L63)）：

```php
define('SHAARLI_MUTEX_FILE', __FILE__);
```

> **设计意图**：使用项目入口文件本身作为锁文件，避免额外创建锁文件，同时保证锁文件一定存在。

### 2.2 ⚠️ 锁的作用范围（最关键的理解点）

**锁只包裹单次 `read()` 或 `write()` 调用，不包裹整个"读-改-写"周期。**

这是理解并发行为的核心：

```
请求 A 的生命周期：
┌──────────────────────────────────────────────────────────┐
│ 构造 BookmarkFileService                                  │
│   → $this->bookmarksIO->read()                            │
│      → synchronized() {  [锁A获取]                        │
│            file_get_contents()  ← 读磁盘                  │
│          }  [锁A释放]                                      │
│                                                           │
│ 内存中修改书签（set()/add()/remove()）                     │
│  ← 这段时间完全没有锁！其他请求可以随意读写磁盘               │
│                                                           │
│ save()                                                    │
│   → $this->bookmarksIO->write()                           │
│      → synchronized() {  [锁B获取]                        │
│            checkDiskSpace()                               │
│            file_put_contents()  ← 写磁盘（基于旧快照！）    │
│          }  [锁B释放]                                      │
└──────────────────────────────────────────────────────────┘
```

**锁 A 和锁 B 是两次独立的锁获取，之间没有任何关联！**

这意味着：
- `read()` 有自己的锁，读完立即释放
- `write()` 有自己的锁，写前获取、写完释放
- "读→改→写"整个周期**没有被同一个锁保护**
- 每个请求有自己独立的内存快照（`$this->bookmarks`）

### 2.3 synchronized 封装与降级策略

在 [BookmarkIO.php#L152-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L152-L159) 中定义了锁的包装方法：

```php
protected function synchronized(callable $function): void
{
    try {
        $this->mutex->synchronized($function);
    } catch (LockAcquireException $exception) {
        // 获取锁失败时降级：直接执行，不做互斥保护
        $function();
    }
}
```

> **关键降级逻辑**：如果 `LockAcquireException` 被抛出（例如某些共享主机不支持文件锁），系统会静默降级为无锁执行。这是为了兼容性而做的妥协，但会丢失并发保护。

### 2.4 读操作的锁保护

[BookmarkIO.php#L75-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L75-L104) 中的 `read()` 方法：

```php
public function read()
{
    // ... 文件存在性和可写性检查 ...

    $content = null;
    $this->synchronized(function () use (&$content) {
        $content = file_get_contents($this->datastore);
    });  // ← 锁在这里释放了！

    // ... 反序列化解码（无锁） ...
}
```

读操作加锁的目的：保证不会读到半写入的损坏数据。但锁只保护 `file_get_contents` 这一行。

### 2.5 写操作的锁保护

[BookmarkIO.php#L114-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L114-L142) 中的 `write()` 方法：

```php
public function write($links)
{
    // ... 可写性检查、数据序列化编码（无锁） ...

    $this->synchronized(function () use ($data) {
        if (!$this->checkDiskSpace($data)) {
            throw new NotEnoughSpaceException();
        }
        file_put_contents($this->datastore, $data);
    });  // ← 锁只保护这部分
}
```

在锁内包含了磁盘空间检查和实际的文件写入。

---

## 三、版本检测（updated 时间戳）

### 3.1 版本字段

Bookmark 实体有两个时间字段：

| 字段 | 含义 | 设置时机 |
|------|------|----------|
| `created` | 创建时间 | 书签首次创建时 `setId()` 自动设置 |
| `updated` | 最后更新时间 | 每次 `BookmarkFileService::set()` 时自动刷新 |

在 [Bookmark.php#L49-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/Bookmark.php#L49-L52) 定义：

```php
/** @var DateTimeInterface Creation datetime */
protected $created;

/** @var DateTimeInterface datetime */
protected $updated;
```

### 3.2 updated 的自动刷新

核心逻辑在 [BookmarkFileService.php#L200-L217](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L200-L217) 的 `set()` 方法：

```php
public function set(Bookmark $bookmark, bool $save = true): Bookmark
{
    // ... 权限校验、存在性检查、validate ...

    $bookmark->setUpdated(new DateTime());  // ← 每次 set() 自动刷新更新时间
    $this->bookmarks[$bookmark->getId()] = $bookmark;

    if ($save === true) {
        $this->save();
        $this->history->updateLink($bookmark);
    }
    return $this->bookmarks[$bookmark->getId()];
}
```

### 3.3 "版本检测"的实际含义

⚠️ **重要澄清**：Shaarli **并没有**实现传统意义上的"乐观锁"或"悲观锁"版本检测机制。具体来说：

- **没有**在保存时比较"用户提交时的版本"与"当前最新版本"
- **没有**使用 `updated` 字段做 `if_match` / `if_unmodified_since` 之类的前置校验
- 表单中 **没有** 隐藏的 `lf_updated` 字段（模板 [editlink.html](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/tpl/default/editlink.html) 中只有 `lf_id`）
- **没有**在 `write()` 时重新读取磁盘数据做对比

`updated` 字段的实际用途：
1. **展示用途**：在页面/Feed/API 中显示"最后编辑时间"
2. **调试追踪**：通过历史记录和时间戳判断书签是否被修改过
3. **记录 LWW 结果**：最后一个完成写入的请求，其 `updated` 就是最终值

---

## 四、冲突提示

### 4.1 已实现的冲突检测

Shaarli 实现了 **URL 重复冲突** 检测，但 **没有** 实现"同一书签并发编辑"的冲突提示。

#### 4.1.1 API 层的 URL 重复冲突（409 Conflict）

在 REST API 的 [Links.php#L117-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/api/controllers/Links.php#L117-L142) `postLink()` 中：

```php
// duplicate by URL, return 409 Conflict
if (
    ! empty($bookmark->getUrl())
    && ! empty($dup = $this->bookmarkService->findByUrl($bookmark->getUrl()))
) {
    return $response->withJson(
        ApiUtils::formatLink($dup, index_url($this->ci['environment'])),
        409,
        $this->jsonStyle
    );
}
```

同样的检测也出现在 `putLink()` [Links.php#L171-L181](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/api/controllers/Links.php#L171-L181)，但排除了自身 ID。

#### 4.1.2 Web 层的隐式冲突处理

在 Web 表单控制器 [ShaarePublishController.php#L182-L226](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/front/controller/admin/ShaarePublishController.php#L182-L226) 的 `buildLinkDataFromUrl()` 中：

```php
// Check if URL is not already in database (in this case, we will edit the existing link)
$bookmark = $this->container->bookmarkService->findByUrl($url);
if (null === $bookmark) {
    // 创建新书签 ...
} else {
    // 编辑已有书签 ...
    $link['linkIsNew'] = false;
}
```

即：如果用户尝试添加的 URL 已存在，系统会自动切到"编辑模式"而不是报错。

### 4.2 未实现的：同一书签的并发编辑冲突

对于"两个用户同时编辑同一个书签 ID"的场景：

1. **没有** UI 级别的提示（如"该书签正在被他人编辑"）
2. **没有** 保存时的版本校验（如"该书签已被他人修改，请刷新后重试"）
3. **没有** 合并逻辑（如 diff 合并字段冲突）
4. **没有** 写入前重新读取最新数据做对比

实际行为是 **Last-Write-Wins（LWW）**：最后一个完成 `write()` 的请求会用自己的**整个内存快照**覆盖磁盘内容。

---

## 五、落盘顺序

### 5.1 单次编辑的完整落盘流程

当用户提交编辑表单时，调用链如下（注意标注锁的位置）：

```
ShaarePublishController::save()
  ├─ 1. checkToken()                                  // CSRF 校验
  ├─ 2. bookmarkService->exists($id)                  // 检查内存中的快照
  ├─ 3. bookmarkService->get($id)                     // 从内存快照读取
  ├─ 4. 更新 bookmark 对象字段（title, description, tags, ...）
  ├─ 5. bookmarkService->addOrSet($bookmark, false)
  │     └─ bookmarkService->set($bookmark, false)
  │           ├─ validate()                           // 数据校验
  │           ├─ setUpdated(new DateTime())           // 刷新更新时间
  │           └─ 更新内存 bookmarks[$id]              // 仅内存，未落盘
  ├─ 6. 执行插件钩子 save_link                         // 插件可能修改数据
  ├─ 7. bookmarkService->set($bookmark)               // 再次 set，触发落盘
  │     ├─ setUpdated(new DateTime())                 // 再次刷新 updated
  │     ├─ 更新内存 bookmarks[$id]
  │     └─ save()
  │           ├─ bookmarks->reorder()                 // Step 1: 内存重排
  │           ├─ bookmarksIO->write()                 // Step 2: 写磁盘
  │           │     ├─ 序列化编码（无锁）              // serialize → gzdeflate → base64
  │           │     └─ synchronized() {               // [锁获取]
  │           │           ├─ checkDiskSpace()
  │           │           └─ file_put_contents()      // 原子写入整个快照
  │           │        }                              // [锁释放]
  │           └─ pageCacheManager->invalidateCaches() // Step 3: 清缓存
  └─ 8. history->updateLink($bookmark)                // 写入操作历史
```

### 5.2 save() 方法的三步落盘顺序

[BookmarkFileService.php#L309-L319](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L309-L319)：

```php
public function save(): void
{
    // 权限检查 ...

    $this->bookmarks->reorder();                        // Step 1: 内存重排
    $this->bookmarksIO->write($this->bookmarks);        // Step 2: 持锁写盘
    $this->pageCacheManager->invalidateCaches();        // Step 3: 失效缓存
}
```

#### Step 1 — 内存重排 `reorder()`

[BookmarkArray.php#L244-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkArray.php#L244-L263)：

- 按 `sticky` 置顶标记优先排序（置顶书签在前）
- 按 `created` 创建时间降序（最新在前）
- 重建 `urls[]` 和 `ids[]` 索引映射

#### Step 2 — 持锁写盘 `write()`

[BookmarkIO.php#L114-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L114-L142)：

数据序列化流程：
```
BookmarkArray（内存快照，可能已过时）
  → serialize()           // PHP 序列化
  → gzdeflate()           // 压缩
  → base64_encode()       // Base64 编码
  → 加 PHP 前缀后缀 "<?php /* ... */ ?>"
  → file_put_contents()   // 写入文件（在锁内）
```

前缀后缀的作用：防止直接通过 HTTP 请求访问 datastore 文件时泄露内容（PHP 标签会让服务器执行而非输出）。

⚠️ **关键**：`write()` 写入的是**整个内存快照**，而不是增量修改。

#### Step 3 — 缓存失效

落盘成功后立即调用 `invalidateCaches()` 清除页面缓存，保证后续请求能读到最新数据。

### 5.3 批量操作的落盘优化

对于批量操作（如批量改标签、批量改可见性），采用 **多次内存修改 + 一次落盘** 的模式：

以 [ShaareManageController.php#L212-L286](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/front/controller/admin/ShaareManageController.php#L212-L286) `addOrDeleteTags()` 为例：

```php
$count = 0;
foreach ($ids as $id) {
    $bookmark = $this->container->bookmarkService->get((int) $id);
    // ... 修改 bookmark ...
    $this->container->bookmarkService->set($bookmark, false);  // false = 不落盘
    ++$count;
}

if ($count > 0) {
    $this->container->bookmarkService->save();  // 统一落盘一次
}
```

这样做的好处：
- 减少磁盘 IO 次数
- 减少文件锁的持有次数
- 多个修改作为一个"批次"写入磁盘

但并发风险依然存在：如果在 `foreach` 循环期间，其他请求修改了磁盘数据，最终 `save()` 会用旧快照覆盖。

### 5.4 两次 set() 调用的微妙之处

观察 [ShaarePublishController.php#L98-L156](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/front/controller/admin/ShaarePublishController.php#L98-L156) 的 `save()` 方法，可以看到 `set()` 被调用了两次：

| 调用 | 参数 | 作用 |
|------|------|------|
| 第 1 次 `addOrSet($bookmark, false)` | `save=false` | 仅更新内存，**插件钩子前**的状态 |
| 第 2 次 `set($bookmark)` | 默认 `save=true` | 插件可能修改了数据，再次刷新 `updated`，并触发落盘 |

两次调用之间执行了插件钩子 `save_link`，插件可以修改书签数据。第二次 `set()` 确保：
1. 插件修改也被纳入 `updated` 时间戳
2. 插件修改后的数据最终落盘

---

## 六、并发场景下的实际行为推演（对照代码）

### 6.0 先明确两个基本事实

**事实 1**：每个请求有独立的内存快照。`BookmarkFileService` 是在容器中每次请求新建的，构造时调用 `read()` 加载完整数据。

**事实 2**：`write()` 写入的是**整个内存快照**，不是增量修改。即使你只改了一个书签的标题，写入的也是全部书签的数据。

---

### 场景 1：两个用户同时编辑**不同**书签（最反直觉！）

⚠️ **即使编辑不同书签，后写入的也会覆盖先写入的！**

```
磁盘初始状态：[书签#1: title="A", 书签#2: title="B"]

T0: 请求 A 到达 → 构造 BookmarkFileService
    → read() [锁1] → 内存快照_A = {#1:"A", #2:"B"} → [锁1释放]

T1: 请求 B 到达 → 构造 BookmarkFileService
    → read() [锁2] → 内存快照_B = {#1:"A", #2:"B"} → [锁2释放]

T2: 请求 A 修改书签#1 title="A2"
    → set(#1, false) → 内存快照_A = {#1:"A2", #2:"B"}
    → save()
        → write() [锁3获取]
            → file_put_contents(整个快照_A)
            → 磁盘变为：{#1:"A2", #2:"B"}
        → [锁3释放]

T3: 请求 B 修改书签#2 title="B2"
    → set(#2, false) → 内存快照_B = {#1:"A", #2:"B2"}  ← 书签#1 还是旧值！
    → save()
        → write() [锁4获取]
            → file_put_contents(整个快照_B)  ← 覆盖！
            → 磁盘变为：{#1:"A", #2:"B2"}  ← 请求 A 的修改丢失了！
        → [锁4释放]

最终结果：
  书签#1: title = "A"  （请求 A 的修改被覆盖丢失！）
  书签#2: title = "B2" （请求 B 的修改保留）
```

**代码依据**：
- `read()` 在 [BookmarkFileService.php#L80](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L80) 构造时调用，之后不再重新读取
- `write()` 在 [BookmarkIO.php#L137-L140](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L137-L140) 写入整个 `$this->bookmarks`

---

### 场景 2：两个用户同时编辑**同一**书签（经典 LWW）

```
磁盘初始状态：[书签#42: title="原始标题"]

T0: 用户 A 打开编辑页 → 构造 → read() → 快照_A = {#42:"原始标题"}
T1: 用户 B 打开编辑页 → 构造 → read() → 快照_B = {#42:"原始标题"}

T2: 用户 A 提交 → 改为 "标题A"
    → set(#42) → 快照_A = {#42:"标题A"}
    → save() → write() [锁A]
        → 磁盘变为：{#42:"标题A", updated=T2}

T3: 用户 B 提交 → 改为 "标题B"
    → set(#42) → 快照_B = {#42:"标题B"}
    → save() → write() [锁B]  ← 等待锁A释放
        → 磁盘变为：{#42:"标题B", updated=T3}

最终结果：
  书签#42: title = "标题B"，用户 A 的修改丢失，无任何提示
```

**代码依据**：
- 表单 [editlink.html](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/tpl/default/editlink.html) 中只有 `lf_id`，没有 `lf_updated` 版本字段
- `set()` 方法 [BookmarkFileService.php#L205-L207](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L205-L207) 只检查 ID 是否存在，不做版本校验

---

### 场景 3：用户 A 编辑，用户 B 同时删除（交叉操作）

#### 子场景 3a：删除先落盘

```
磁盘初始状态：[书签#42 存在]

T0: 请求 A（编辑）构造 → read() → 快照_A 包含 #42
T1: 请求 B（删除）构造 → read() → 快照_B 包含 #42

T2: 请求 B 执行 remove(#42, false) → 快照_B 不含 #42
    → save() → write() [锁B]
        → 磁盘变为：[不含 #42]

T3: 请求 A 执行 set(#42) → 检查 isset($this->bookmarks[42])
    ↓ 内存快照_A 中 #42 还在！检查通过！
    → save() → write() [锁A]
        → 磁盘变为：[包含 #42（修改后）]

结果：书签被删除后又"复活"了，且是修改后的版本！
```

**代码依据**：
- `set()` 的存在性检查 [BookmarkFileService.php#L205-L207](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L205-L207)：
  ```php
  if (! isset($this->bookmarks[$bookmark->getId()])) {
      throw new BookmarkNotFoundException();
  }
  ```
  这个检查只看**当前请求的内存快照**，不看磁盘真实状态。

#### 子场景 3b：编辑先落盘

```
T0: 请求 A（编辑）构造 → read() → 快照_A 包含 #42
T1: 请求 B（删除）构造 → read() → 快照_B 包含 #42

T2: 请求 A 执行 set(#42) → save() → write()
    → 磁盘：书签#42 已更新

T3: 请求 B 执行 remove(#42, false) → 快照_B 不含 #42
    → save() → write()
    → 磁盘：书签#42 被删除

结果：编辑的修改被删除覆盖，符合预期。
```

---

### 场景 4：三个请求同时操作（连环覆盖）

```
磁盘初始：{#1:"A", #2:"B", #3:"C"}

T0: 请求 X 构造 → 快照_X = {#1:"A", #2:"B", #3:"C"}
T1: 请求 Y 构造 → 快照_Y = {#1:"A", #2:"B", #3:"C"}
T2: 请求 Z 构造 → 快照_Z = {#1:"A", #2:"B", #3:"C"}

T3: 请求 X 修改 #1 → "A2" → save() → 磁盘={#1:"A2", #2:"B", #3:"C"}
T4: 请求 Y 修改 #2 → "B2" → save() → 磁盘={#1:"A", #2:"B2", #3:"C"}  ← X 丢失
T5: 请求 Z 修改 #3 → "C2" → save() → 磁盘={#1:"A", #2:"B", #3:"C2"}  ← X、Y 都丢失

最终结果：只有 Z 的修改保留，X 和 Y 的修改全部丢失！
```

---

## 七、设计局限与潜在风险

| 风险点 | 说明 | 代码位置 |
|--------|------|----------|
| **锁不保护读改写周期** | 锁只包裹单次 read/write，"读→改→写"之间无锁，旧快照会覆盖新数据 | [BookmarkFileService.php#L80](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L80) |
| **无细粒度锁** | 全局文件锁，任何书签修改都会阻塞所有其他书签的读写，高并发下性能瓶颈明显 | [ContainerBuilder.php#L100](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/container/ContainerBuilder.php#L100) |
| **无乐观锁版本校验** | `updated` 字段仅用于展示，不用于 CAS，并发编辑同一书签会静默丢失修改 | [editlink.html](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/tpl/default/editlink.html) |
| **锁获取失败静默降级** | `synchronized()` 在 `LockAcquireException` 时直接执行，不做任何告警或日志 | [BookmarkIO.php#L156-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L156-L158) |
| **全量序列化写入** | 每次修改都需序列化整个书签集合并重新写盘，书签数量大时性能差 | [BookmarkIO.php#L124](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L124) |
| **内存检查不看磁盘** | `set()` 的存在性检查只看当前内存快照，不反映磁盘真实状态 | [BookmarkFileService.php#L205](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L205) |
| **无冲突合并策略** | 不存在字段级的冲突检测或自动合并（如 Git 式的三路合并） | - |
| **无编辑会话锁** | 不存在"检出-编辑-检入"模式，无法在 UI 层提示用户"该文档正在被编辑" | - |

---

## 八、总结

Shaarli 的并发保护是一个 **轻量级、基于单次 IO 锁的 Last-Write-Wins 方案**，核心要点：

### 8.1 读写保护与锁等待
- 使用 `FlockMutex` 文件锁包裹单次 `file_get_contents()` 和 `file_put_contents()`，保证不会读到半写入的损坏数据
- **锁参数 `2` 是超时时间（秒）**，不是 `LOCK_NB`：锁被占用时等待最多 2 秒，超时则抛出 `LockAcquireException`
- **不保护整个"读-改-写"周期**：`read()` 和 `write()` 是两次独立的锁获取，中间修改内存时完全无锁
- **静默降级策略**：锁获取失败时不报错、不打日志，直接无锁执行，可能导致数据损坏

### 8.2 版本检测
- 每个书签有 `updated` 时间戳，每次 `set()` 自动刷新，但**仅用于展示和追踪**
- 不参与保存时的 CAS（Check-And-Set）校验，表单中无版本字段

### 8.3 冲突提示
- **仅实现了"URL 重复"的 409 冲突检测**，但检测基于内存快照，且检测与写入之间无锁保护（TOCTOU 竞态）
- **同 URL 并发新增时，两个请求都会通过检测**，后写入的用相同 ID 覆盖先写入的所有数据
- 对同一书签、甚至不同书签的并发编辑**无任何提示**，后写入者用自己的整个内存快照覆盖先写入者

### 8.4 落盘顺序与写入原子性
- 落盘顺序：`reorder()`（内存重排） → `write()`（持锁序列化整个快照写盘） → `invalidateCaches()`（缓存失效）
- **写入无原子性保障**：
  - `file_put_contents()` 返回值完全未检查，失败或半写入静默忽略
  - 无"临时文件 + rename"的事务性写入模式，中途中断会导致文件损坏
  - `checkDiskSpace()` 检查与实际写入之间存在 TOCTOU 竞态
- 批量操作时采用"内存多改 + 一次落盘"优化，但并发风险不变

### 8.5 最严重的问题
1. **即使编辑完全不同的书签，并发操作也会互相覆盖**：每次写入的是整个内存快照，而不是增量修改
2. **同 URL 并发新增会静默丢失数据**：`getNextId()` 基于旧快照分配相同 ID，后写入者完全覆盖先写入者
3. **锁超时后静默降级**：高并发下自动放弃互斥保护，无任何告警

---

## 九、附录：锁等待与写盘边界的关键代码事实

本附录集中列出三个边界问题的精确代码事实，每条均附行号与原始代码片段。

### 9.1 边界一：2 秒超时无锁降级

#### 事实 9.1.1 — 锁超时参数为 2 秒

[ContainerBuilder.php#L95-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/container/ContainerBuilder.php#L95-L102)：

```php
$container['bookmarkService'] = function (ShaarliContainer $container): BookmarkServiceInterface {
    return new BookmarkFileService(
        $container->conf,
        $container->pluginManager,
        $container->history,
        new FlockMutex(fopen(SHAARLI_MUTEX_FILE, 'r'), 2),  // ← 第二参数 2 = 超时 2 秒
        $container->loginManager->isLoggedIn()
    );
};
```

**关键事实**：
- `FlockMutex` 构造函数第二参数是 `$timeout`（秒），**不是** `LOCK_NB` 标志位
- 锁被占用时，malkusch/lock 会等待最多 2 秒（用 `ext-pcntl` 或忙等待重试）
- 2 秒后仍获取不到锁 → 抛出 `LockAcquireException`

#### 事实 9.1.2 — 异常捕获后静默降级为无锁执行

[BookmarkIO.php#L152-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L152-L159)：

```php
protected function synchronized(callable $function): void
{
    try {
        $this->mutex->synchronized($function);
    } catch (LockAcquireException $exception) {
        $function();   // ← 捕获异常后，直接无锁执行原函数！
    }
}
```

**关键事实**：
- 第 156 行捕获 `LockAcquireException`，第 157 行直接调用 `$function()`
- 降级后**没有日志、没有告警、没有给用户的任何提示**
- 注释（第 146 行）明确说明这是为兼容共享主机做的妥协：
  > `If the lock can't be acquired (e.g. some shared hosting provider), we execute the function without mutex.`
- 关联 issue：[https://github.com/shaarli/Shaarli/issues/1650](https://github.com/shaarli/Shaarli/issues/1650)（见第 148 行 `@see`）

#### 事实 9.1.3 — 降级的影响范围

降级同时影响 `read()` 和 `write()`，因为两者都通过 `synchronized()` 包装：

- `read()` 的锁包装：[BookmarkIO.php#L86-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L86-L88)
  ```php
  $this->synchronized(function () use (&$content) {
      $content = file_get_contents($this->datastore);
  });
  ```
- `write()` 的锁包装：[BookmarkIO.php#L132-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L132-L141)
  ```php
  $this->synchronized(function () use ($data) {
      if (!$this->checkDiskSpace($data)) {
          throw new NotEnoughSpaceException();
      }
      file_put_contents($this->datastore, $data);
  });
  ```

**后果**：一旦降级，读和写都不再有互斥保护，多个请求可能同时 `file_put_contents()` 同一文件，导致内容交错损坏。

---

### 9.2 边界二：同 URL 并发新增时重复检测失效 + 相同 ID 覆盖

#### 事实 9.2.1 — URL 重复检测在内存快照中进行，与写入之间无锁

[Links.php#L125-L137](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/api/controllers/Links.php#L125-L137)（API `postLink`）：

```php
// duplicate by URL, return 409 Conflict
if (
    ! empty($bookmark->getUrl())
    && ! empty($dup = $this->bookmarkService->findByUrl($bookmark->getUrl()))  // ← T1: 检查（内存快照）
) {
    return $response->withJson(
        ApiUtils::formatLink($dup, index_url($this->ci['environment'])),
        409,
        $this->jsonStyle
    );
}

$this->bookmarkService->add($bookmark);  // ← T2: 写入（中间无锁，且基于同一旧快照）
```

**关键事实**：
- 第 128 行 `findByUrl()` 查询的是**当前请求的内存快照**，不是磁盘实时状态
- 第 128 行（检查）与第 137 行（写入）之间**没有任何锁保护**
- 两个并发请求都读到"无此 URL"的旧快照 → 都通过检查 → 都执行 `add()`

#### 事实 9.2.2 — `findByUrl` 只查内存数组

[BookmarkFileService.php#L129-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L129-L132)：

```php
public function findByUrl(string $url): ?Bookmark
{
    return $this->bookmarks->getByUrl($url);  // ← 只查内存 $this->bookmarks
}
```

`getByUrl` 实现 [BookmarkArray.php#L224-L234](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkArray.php#L224-L234)：

```php
public function getByUrl(string $url): ?Bookmark
{
    if (
        ! empty($url)
        && isset($this->urls[$url])
        && isset($this->bookmarks[$this->urls[$url]])
    ) {
        return $this->bookmarks[$this->urls[$url]];
    }
    return null;
}
```

**关键事实**：第 228 行 `isset($this->urls[$url])` 只查内存中的 `urls[]` 映射，不访问磁盘。

#### 事实 9.2.3 — `getNextId` 基于内存快照，并发时分配相同 ID

[BookmarkArray.php#L211-L217](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkArray.php#L211-L217)：

```php
public function getNextId(): int
{
    if (!empty($this->ids)) {
        return max(array_keys($this->ids)) + 1;  // ← 基于内存 ids[] 计算
    }
    return 0;
}
```

**关键事实**：
- 第 214 行 `max(array_keys($this->ids)) + 1` 基于当前内存快照的最大 ID
- 两个并发请求的快照相同 → `getNextId()` 返回**相同的值**
- 例如快照中最大 ID 为 5，两个请求都会拿到 `nextId = 6`

#### 事实 9.2.4 — `add` 不做二次 URL 检查，直接用分配的 ID 写入

[BookmarkFileService.php#L222-L239](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php#L222-L239)：

```php
public function add(Bookmark $bookmark, bool $save = true): Bookmark
{
    if (true !== $this->isLoggedIn) {
        throw new Exception(t('You\'re not authorized to alter the datastore'));
    }
    if (!empty($bookmark->getId())) {
        throw new Exception(t('This bookmarks already exists'));
    }
    $bookmark->setId($this->bookmarks->getNextId());  // ← 第 230 行：分配 ID（可能与另一请求相同）
    $bookmark->validate();

    $this->bookmarks[$bookmark->getId()] = $bookmark;  // ← 第 233 行：写入内存快照
    if ($save === true) {
        $this->save();  // ← 第 235 行：整个快照落盘，覆盖磁盘
        $this->history->addLink($bookmark);
    }
    return $this->bookmarks[$bookmark->getId()];
}
```

**关键事实**：
- 第 230 行 `setId()` 用内存快照计算的 ID，并发时会重复
- 第 233 行 `$this->bookmarks[$bookmark->getId()]` 用相同 ID 覆盖内存中的条目
- 第 235 行 `save()` → `write()` 用整个内存快照覆盖磁盘
- **`add()` 内部没有重新检查 URL 是否已存在**，完全信任控制器层的检查结果

#### 事实 9.2.5 — 并发新增的完整时序

```
磁盘初始：{#5: url="old.com"}，最大 ID = 5

T0: 请求 A 构造 → read() → 快照_A = {#5:"old.com"}，nextId 计算 = 6
T1: 请求 B 构造 → read() → 快照_B = {#5:"old.com"}，nextId 计算 = 6（相同！）

T2: 请求 A 执行 findByUrl("new.com") → 快照_A 无 → 通过
T3: 请求 B 执行 findByUrl("new.com") → 快照_B 无 → 通过

T4: 请求 A 执行 add()：
    → setId(6)（第 230 行）
    → bookmarks[6] = bookmark_A（第 233 行）
    → save() → write() [锁] → 磁盘 = {#5:"old.com", #6:"bookmark_A"}

T5: 请求 B 执行 add()：
    → setId(6)（第 230 行，和 A 相同！）
    → bookmarks[6] = bookmark_B（第 233 行，覆盖内存中的 A）
    → save() → write() [锁] → 磁盘 = {#5:"old.com", #6:"bookmark_B"}  ← A 的数据丢失！

最终：只有一个 #6，内容是 B 的，A 完全丢失，无 409 提示
```

---

### 9.3 边界三：普通文件写入失败或半写入风险

#### 事实 9.3.1 — `file_put_contents` 返回值完全未检查

[BookmarkIO.php#L132-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L132-L141)：

```php
$this->synchronized(function () use ($data) {
    if (!$this->checkDiskSpace($data)) {
        throw new NotEnoughSpaceException();
    }

    file_put_contents(       // ← 第 137 行
        $this->datastore,    // ← 第 138 行
        $data                 // ← 第 139 行
    );                       // ← 第 140 行：返回值被完全丢弃！
});
```

**关键事实**：
- 第 137-140 行 `file_put_contents()` 的返回值没有被任何变量接收，也没有 `if` 判断
- `file_put_contents()` 的返回值语义：
  - 成功：返回写入字节数（`int`）
  - 失败：返回 `false`
  - 部分写入：返回实际写入字节数（可能小于 `strlen($data)`）
- 由于不检查返回值，以下情况全部**静默通过**：
  - 磁盘满 → 写入失败，datastore 保持旧内容，用户以为保存成功
  - 文件权限被改 → 写入失败，同上
  - 磁盘 IO 错误 → 写入失败或部分写入，同上
  - 部分写入 → datastore 文件损坏

#### 事实 9.3.2 — 无临时文件 + rename 的原子写入

整个 `write()` 方法 [BookmarkIO.php#L114-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L114-L142) 中，写入方式为：

```php
$data = base64_encode(gzdeflate(serialize($links)));     // 第 124 行：序列化
// ... 前缀后缀拼接 ...
$this->synchronized(function () use ($data) {
    if (!$this->checkDiskSpace($data)) {
        throw new NotEnoughSpaceException();
    }
    file_put_contents($this->datastore, $data);          // 第 137-140 行：直接覆盖原文件
});
```

**关键事实**：
- 第 137 行直接对 `$this->datastore`（原文件）执行 `file_put_contents`
- **没有**先写入临时文件（如 `datastore.php.tmp`）
- **没有**用 `rename()` 做原子替换
- `file_put_contents` 内部实现是 `open → truncate → write → close`，**truncate 之后、write 完成之前**，原文件内容已被清空

**半写入损坏场景**：
```
file_put_contents 执行过程：
  1. open(datastore.php, 'w')      → 打开文件
  2. truncate(datastore.php, 0)    → 清空原文件（此时原数据已丢失！）
  3. write(datastore.php, $data)   → 开始写入新数据
     ↓ 如果此时发生：
     ├─ PHP 进程被 kill（如 max_execution_time）
     ├─ 服务器断电
     ├─ 磁盘故障
     └─ 内存不足 OOM
     → 文件停留在"部分写入"状态
  4. close(datastore.php)          → 关闭文件
```

如果第 3 步中断，下次 `read()` 时 [BookmarkIO.php#L92-L94](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L92-L94) 会失败：

```php
$links = unserialize(gzinflate(base64_decode(
    substr($content, strlen(self::$phpPrefix), -strlen(self::$phpSuffix))
)));
```

`gzinflate()` 或 `unserialize()` 会抛出异常，导致整个应用无法加载书签数据。

#### 事实 9.3.3 — `checkDiskSpace` 的 TOCTOU 竞态与降级放行

[BookmarkIO.php#L169-L176](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L169-L176)：

```php
public function checkDiskSpace(string $data): bool
{
    if (function_exists('disk_free_space') === false) {
        return true;   // ← 第 172 行：无法检测时直接放行！
    }
    return disk_free_space(dirname($this->datastore)) > (strlen($data) + 1024 * 500);
}
```

**关键事实**：
- 第 171 行：如果 `disk_free_space` 函数不存在（某些受限环境），直接 `return true` 放行
- 第 175 行：检查通过后，到第 137 行 `file_put_contents` 执行之间，磁盘空间可能被其他进程消耗
- 这是一个经典的 TOCTOU（Time-of-Check to Time-of-Use）问题
- 即使检查在锁内，锁也无法阻止**操作系统其他进程**消耗磁盘空间

**场景**：
```
T0: checkDiskSpace() → 剩余 10MB > 所需 1MB + 500KB → 通过
T1: 其他系统进程写入 10MB 日志 → 磁盘满
T2: file_put_contents() → 磁盘满，写入失败
T3: 返回值未检查 → 静默"成功"
```

#### 事实 9.3.4 — 降级为无锁时多进程同时写入的交错损坏

当 [BookmarkIO.php#L156-L157](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L156-L157) 降级为无锁执行时：

```php
} catch (LockAcquireException $exception) {
    $function();   // ← 无锁执行 file_put_contents
}
```

两个请求同时执行 `file_put_contents($this->datastore, $data)`：

```
请求 A: open → truncate → write("AAAA...") → close
请求 B: open → truncate → write("BBBB...") → close

由于没有互斥锁，两个 write 可能交错：
  文件内容 = "AAAABBBBAAAABBBB..."（交错损坏）
  或 = "BBBB..."（B 完全覆盖，A 丢失）
  或 = "AAAA..."（A 完全覆盖，B 丢失）

无论哪种结果，unserialize/gzinflate 都无法解析
```

**关键事实**：`file_put_contents` 本身不保证原子性，PHP 文档明确说明它是 `fopen + fwrite + fclose` 的组合，多进程并发写同一文件会产生交错。
