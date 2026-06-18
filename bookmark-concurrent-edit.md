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
BookmarkFileService (业务逻辑层)
    ↓
BookmarkIO (文件读写层，带互斥锁)
    ↓
datastore.php (磁盘文件存储)
```

关键文件清单：

- [ContainerBuilder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/container/ContainerBuilder.php) — 依赖注入与 Mutex 初始化
- [BookmarkFileService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkFileService.php) — 书签业务服务，含 `set()` / `add()` / `save()`
- [BookmarkIO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php) — 文件读写与互斥锁封装
- [Bookmark.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/Bookmark.php) — 书签实体，含 `updated` 时间戳
- [BookmarkArray.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkArray.php) — 内存中书签集合
- [init.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/init.php) — `SHAARLI_MUTEX_FILE` 常量定义

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
        new FlockMutex(fopen(SHAARLI_MUTEX_FILE, 'r'), 2),  // 第二个参数 2 = LOCK_NB 非阻塞模式
        $container->loginManager->isLoggedIn()
    );
};
```

锁文件使用 `init.php` 自身作为锚点（[init.php#L63](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/init.php#L63)）：

```php
define('SHAARLI_MUTEX_FILE', __FILE__);
```

> **设计意图**：使用项目入口文件本身作为锁文件，避免额外创建锁文件，同时保证锁文件一定存在。

### 2.2 锁的作用范围

锁包裹的是 **整个数据存储文件的读写操作**，而非单条书签。也就是说：

- 任何一次书签的读（`read()`）或写（`write()`）都会持有全局锁
- 同一时刻只能有一个 PHP 请求对 datastore 文件进行 IO 操作
- 粒度是"整个数据库文件"，不是"单条书签"

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
    });

    // ... 反序列化解码 ...
}
```

注意：读操作也加锁，保证读写之间的互斥，避免读到半写入的损坏数据。

### 2.5 写操作的锁保护

[BookmarkIO.php#L114-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L114-L142) 中的 `write()` 方法：

```php
public function write($links)
{
    // ... 可写性检查、数据序列化编码 ...

    $this->synchronized(function () use ($data) {
        if (!$this->checkDiskSpace($data)) {
            throw new NotEnoughSpaceException();
        }
        file_put_contents($this->datastore, $data);
    });
}
```

在锁内还包含了磁盘空间检查，确保有足够空间（预留 500KB 余量）。

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

`updated` 字段的实际用途：
1. **展示用途**：在页面/Feed/API 中显示"最后编辑时间"
2. **调试追踪**：通过历史记录和时间戳判断书签是否被修改过
3. **隐式的 Last-Write-Wins**：因为文件锁是全局的，最后一个获得锁并写入的请求会覆盖之前的修改，`updated` 只是记录了谁是"最后一个"

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

实际行为是 **Last-Write-Wins（LWW）**：最后一个完成 `save()` 的请求会完全覆盖前面的修改。

---

## 五、落盘顺序

### 5.1 单次编辑的落盘流程

当用户提交编辑表单时，调用链如下：

```
ShaarePublishController::save()
  ├─ 1. checkToken()                        // CSRF 校验
  ├─ 2. bookmarkService->exists($id)        // 判断是新增还是编辑
  ├─ 3. bookmarkService->get($id)           // 读取当前书签（内存中）
  ├─ 4. 更新 bookmark 对象字段（title, description, tags, ...）
  ├─ 5. bookmarkService->addOrSet($bookmark, false)
  │     └─ bookmarkService->set($bookmark, false)
  │           ├─ validate()                 // 数据校验
  │           ├─ setUpdated(new DateTime()) // 刷新更新时间
  │           └─ 更新内存 bookmarks[$id]    // 仅内存，未落盘
  ├─ 6. 执行插件钩子 save_link
  ├─ 7. bookmarkService->set($bookmark)     // 再次 set，这次触发落盘
  │     ├─ setUpdated(new DateTime())       // 再次刷新 updated
  │     ├─ 更新内存 bookmarks[$id]
  │     └─ save()
  │           ├─ bookmarks->reorder()       // 重排内存顺序（按日期降序）
  │           ├─ bookmarksIO->write()       // 写磁盘（持锁）
  │           │     ├─ synchronized() {     // FlockMutex 加锁
  │           │     │   ├─ checkDiskSpace() // 检查磁盘空间
  │           │     │   └─ file_put_contents() // 原子写入
  │           │     }
  │           └─ pageCacheManager->invalidateCaches() // 清缓存
  └─ 8. history->updateLink($bookmark)      // 写入操作历史
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

**注意**：重排只发生在落盘前，这意味着在多次内存修改期间，索引可能暂时不一致，但这不影响通过 ID 访问（因为 ID 访问走 `ids[]` 映射而不是数组下标顺序）。

#### Step 2 — 持锁写盘 `write()`

[BookmarkIO.php#L114-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/37-Shaarli/application/bookmark/BookmarkIO.php#L114-L142)：

数据序列化流程：
```
BookmarkArray
  → serialize()           // PHP 序列化
  → gzdeflate()           // 压缩
  → base64_encode()       // Base64 编码
  → 加 PHP 前缀后缀 "<?php /* ... */ ?>"
  → file_put_contents()   // 写入文件（在锁内）
```

前缀后缀的作用：防止直接通过 HTTP 请求访问 datastore 文件时泄露内容（PHP 标签会让服务器执行而非输出）。

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
- 多个修改作为一个"批次"原子性地（相对于文件锁）写入磁盘

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

## 六、并发场景下的实际行为推演

### 场景 1：两个用户同时编辑不同书签

- 用户 A 编辑书签 #1，用户 B 编辑书签 #2
- 文件锁保证 `write()` 不会交错执行
- 两个修改都会被持久化，互不影响
- 唯一开销：后执行的请求需要等待先执行的释放文件锁

### 场景 2：两个用户同时编辑同一书签（LWW）

```
T0: 用户 A 打开编辑页 → 读取数据（title="原始标题"）
T1: 用户 B 打开编辑页 → 读取数据（title="原始标题"）
T2: 用户 A 提交 → 改为 "标题A" → 获取锁 → 写入磁盘 → updated=T2
T3: 用户 B 提交 → 改为 "标题B" → 等待锁 → 获取锁 → 写入磁盘 → updated=T3
结果: 书签标题 = "标题B"，用户 A 的修改丢失，无任何提示
```

这就是 Last-Write-Wins 的语义。

### 场景 3：用户 A 编辑，用户 B 同时删除

- 删除也走 `bookmarkService->remove()` → `save()` → `write()` 持锁流程
- 如果删除先落盘，后续编辑的 `set()` 会因 `BookmarkNotFoundException` 失败
- 如果编辑先落盘，删除会覆盖（书签被删除）

---

## 七、设计局限与潜在风险

| 风险点 | 说明 |
|--------|------|
| **无细粒度锁** | 全局文件锁，任何书签修改都会阻塞所有其他书签的读写，高并发下性能瓶颈明显 |
| **无乐观锁版本校验** | `updated` 字段仅用于展示，不用于 CAS（Check-And-Set），并发编辑同一书签会静默丢失修改 |
| **锁获取失败静默降级** | `synchronized()` 在 `LockAcquireException` 时直接执行，不做任何告警或日志，可能在共享主机上完全丢失互斥保护 |
| **全量序列化写入** | 每次修改都需序列化整个书签集合并重新写盘，书签数量大时性能差 |
| **无冲突合并策略** | 不存在字段级的冲突检测或自动合并（如 Git 式的三路合并） |
| **无编辑会话锁** | 不存在"检出-编辑-检入"模式，无法在 UI 层提示用户"该文档正在被编辑" |

---

## 八、总结

Shaarli 的并发保护是一个 **轻量级、基于全局文件锁的 Last-Write-Wins 方案**：

1. **读写保护**：使用 `FlockMutex` 全局文件锁包裹 datastore 的所有 IO 操作，保证读写互斥、写写互斥；锁获取失败时静默降级。
2. **版本检测**：每个书签有 `updated` 时间戳，每次 `set()` 自动刷新，但仅用于展示和追踪，**不参与**保存时的冲突校验。
3. **冲突提示**：仅实现了"URL 重复"的 409 冲突检测；对同一书签的并发编辑无任何提示，后写入者覆盖先写入者。
4. **落盘顺序**：`reorder()`（内存重排） → `write()`（持锁序列化写盘） → `invalidateCaches()`（缓存失效）。批量操作时采用"内存多改 + 一次落盘"优化。
