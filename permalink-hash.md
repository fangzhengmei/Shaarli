# Shaarli 短链（Permalink）Hash 生成、冲突避让与旧链回查全链路解析

## 0. 核心文件索引

| 模块 | 文件 | 关键函数/方法 |
|------|------|---------------|
| Hash 算法 | [Utils.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/Utils.php) | `smallHash()` (L47-L51) |
| Hash 组装 | [LinkUtils.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/LinkUtils.php) | `link_small_hash()` (L191-L194) |
| Bookmark 模型 | [Bookmark.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/Bookmark.php) | `setId()` (L139-L150) |
| ID 分配器 | [BookmarkArray.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkArray.php) | `getNextId()` (L211-L217) |
| 查重/查找 | [BookmarkFilter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkFilter.php) | `filterSmallHash()` (L178-L188) |
| Service 入口 | [BookmarkFileService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkFileService.php) | `findByHash()` (L110-L124), `add()` (L222-L239) |
| 路由入口 | [index.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/index.php) | 路由注册 (L117-L118) |
| Permalink 控制器 | [BookmarkListController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/front/controller/visitor/BookmarkListController.php) | `permalink()` (L129-L160), `processLegacyController()` (L210-L242) |
| 旧版短链兼容 | [LegacyLinkDB.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/legacy/LegacyLinkDB.php) | `read()` 内 shorturl 兼容 (L320-L327) |
| 数据迁移(旧→新) | [LegacyUpdater.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/legacy/LegacyUpdater.php) | `updateMethodDatastoreIds()` (L247-L280) |
| Note URL 迁移 | [Updater.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/updater/Updater.php) | `updateMethodMigrateExistingNotesUrl()` (L151-L173) |
| 旧路由跳转 | [LegacyController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/legacy/LegacyController.php) | 各旧路由 `?do=xxx` 方法 |
| API 查重 | [Links.php](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/api/controllers/Links.php) | `postLink()` 内 URL 查重 (L126-L135) |

---

## 1. Hash 生成：两段式构造

### 1.1 算法本体：`smallHash()`

位置：[Utils.php#L47-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/Utils.php#L47-L51)

```php
function smallHash($text)
{
    $t = rtrim(base64_encode(hash('crc32', $text, true)), '=');
    return strtr($t, '+/', '-_');
}
```

流水线拆解：

1. `hash('crc32', $text, true)` — 用 CRC32 生成 4 字节二进制摘要（非密码学安全，只是为了短+唯一）
2. `base64_encode(...)` — 4 字节 → 约 6 字符 Base64
3. `rtrim(..., '=')` — 去掉 Base64 的填充符 `=`
4. `strtr($t, '+/', '-_')` — 把 Base64 的 `+` 换成 `-`，`/` 换成 `_`，得到 **RFC 4648 base64url** 格式，确保 URL 安全

**输出特性**：固定 6 字符，字符集 `[a-zA-Z0-9-_@]`，非加密安全哈希。

### 1.2 输入盐组装：新旧两版

Shaarli 经历了两次 hash 输入盐的演变，这是冲突避让的核心：

#### 新版（v0.8.1+）—— `日期 + ID`

位置：[LinkUtils.php#L191-L194](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/LinkUtils.php#L191-L194)

```php
function link_small_hash($date, $id)
{
    return smallHash($date->format(Bookmark::LINK_DATE_FORMAT) . $id);
}
```

- `Bookmark::LINK_DATE_FORMAT = 'Ymd_His'`，例如 `20111006_131924`
- ID 是自增整数，和日期拼接后作为 smallHash 的输入
- 例：`smallHash('20111006_131924' . 142)` → `eaWxtQ`

触发点在 [Bookmark.php#L139-L150](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/Bookmark.php#L139-L150) 的 `setId()` 中：

```php
public function setId(?int $id): Bookmark
{
    $this->id = $id;
    if (empty($this->created)) {
        $this->created = new DateTime();
    }
    if (empty($this->shortUrl)) {
        $this->shortUrl = link_small_hash($this->created, $this->id);
    }
    return $this;
}
```

> 注意 guard `if (empty($this->shortUrl))`：如果 Bookmark 已经设置了 shortUrl（例如从旧数据加载），就不会覆盖，这是**保留旧 hash 的关键**。

#### 旧版（v0.8.1 之前）—— 仅用日期

在 `smallHash()` 函数注释中明确提到：

> @warning before v0.8.1, smallhashes were built only with the date,
>          and their value has been preserved.

即旧版输入只有 `date`，如 `smallHash('20111006_131924')`。这在同一秒创建多条链接时**必然冲突**，催生了改版。

旧数据加载时的兼容在 [LegacyLinkDB.php#L320-L327](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/legacy/LegacyLinkDB.php#L320-L327)：

```php
if (!isset($link['created'])) {
    $link['id'] = $link['linkdate'];
    $link['created'] = DateTime::createFromFormat(self::LINK_DATE_FORMAT, $link['linkdate']);
    // ...
    $link['shorturl'] = smallHash($link['linkdate']);  // 仅日期，与旧版一致
}
```

这段代码的意义：**从仍以 `linkdate` 为主键的老 datastore 加载时，用旧算法生成 shorturl 并暂存，保证升级后外链不失效。**

---

## 2. 冲突避让：从"事后查重"转向"事前唯一性设计"

Shaarli 并没有做传统意义上的"生成后查数据库看是否重复 + 重试"。它的策略是**让输入源本身就是全局唯一的**，从而在数学上消除冲突可能。

### 2.1 唯一性来源：自增整数 ID

位置：[BookmarkArray.php#L211-L217](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkArray.php#L211-L217)

```php
public function getNextId(): int
{
    if (!empty($this->ids)) {
        return max(array_keys($this->ids)) + 1;
    }
    return 0;
}
```

`$this->ids` 是 ID → array offset 的映射表，`array_keys($this->ids)` 取出所有已有 ID，`max()+1` 就是下一个。ID 全局单调递增。

### 2.2 生成链路闭环

位置：[BookmarkFileService.php#L222-L239](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkFileService.php#L222-L239)

```php
public function add(Bookmark $bookmark, bool $save = true): Bookmark
{
    // ...
    $bookmark->setId($this->bookmarks->getNextId());  // 1. 先拿到唯一 ID
    $bookmark->validate();                            // 2. 校验（含 shortUrl 非空）

    $this->bookmarks[$bookmark->getId()] = $bookmark; // 3. 写入
    // ...
}
```

而 `setId()` 内部会调用 `link_small_hash(created, id)`，因为 `created+id` 的组合**全局唯一**（id 唯一），所以输出的 CRC32 即使理论上可能碰撞，但输入源本身绝不重复——**这是一种设计层面的冲突避让，而非运行时查重**。

> 设计洞察：把 shortUrl 的"输入唯一性问题"降维为"自增 ID 唯一性问题"，而后者只需要一把互斥锁（项目用了 `malkusch/lock` 的 Mutex，见 BookmarkFileService 的构造函数）就可以保证。

### 2.3 URL 维度的查重（不是 hash 查重）

REST API 创建链接时有一层 URL 查重（409 Conflict），这是业务层面的"重复书签不重复存"，不是 shorturl hash 查重。

位置：[Links.php#L126-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/api/controllers/Links.php#L126-L135)

```php
if (
    ! empty($bookmark->getUrl())
    && ! empty($dup = $this->bookmarkService->findByUrl($bookmark->getUrl()))
) {
    return $response->withJson(ApiUtils::formatLink($dup, ...), 409, ...);
}
```

`findByUrl()` 走的是 [BookmarkArray.php#L224-L234](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkArray.php#L224-L234) 中维护的 `$urls` 哈希表（key=url, value=offset），O(1) 命中。

---

## 3. 查重/查找：从 URL 到 Bookmark 的路径

### 3.1 Permalink 页面路由

路由注册在 [index.php#L117-L118](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/index.php#L117-L118)：

```php
$this->get('/', '\Shaarli\Front\Controller\Visitor\BookmarkListController:index');
$this->get('/shaare/{hash}', '\Shaarli\Front\Controller\Visitor\BookmarkListController:permalink');
```

所以 Permalink 的形式是 `https://host/shaare/{6位hash}`。

### 3.2 控制器入口 → 查找

位置：[BookmarkListController.php#L129-L160](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/front/controller/visitor/BookmarkListController.php#L129-L160)

```php
public function permalink(Request $request, Response $response, array $args): Response
{
    $privateKey = $request->getParam('key');

    try {
        $bookmark = $this->container->bookmarkService->findByHash($args['hash'], $privateKey);
    } catch (BookmarkNotFoundException $e) {
        // 404
    }
    // 渲染单条 linklist 模板
}
```

注意 `?key=` 参数：私有链接（`isPrivate()` 为 true）在游客未登录时，需要附带该链接的一次性分享 key 才能访问。key 存储在 `additional_content['private_key']` 里（见 [BookmarkFileService.php#L110-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkFileService.php#L110-L124)）。

### 3.3 findByHash → filterSmallHash：线性扫描

位置：[BookmarkFileService.php#L110-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkFileService.php#L110-L124)

```php
public function findByHash(string $hash, string $privateKey = null): Bookmark
{
    $bookmark = $this->bookmarkFilter->filter(BookmarkFilter::$FILTER_HASH, $hash);
    $first = reset($bookmark);
    if (
        !$this->isLoggedIn
        && $first->isPrivate()
        && (empty($privateKey) || $privateKey !== $first->getAdditionalContentEntry('private_key'))
    ) {
        throw new BookmarkNotFoundException();
    }
    return $first;
}
```

真正的查找实现在 [BookmarkFilter.php#L178-L188](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/BookmarkFilter.php#L178-L188)：

```php
private function filterSmallHash(string $smallHash)
{
    foreach ($this->bookmarks as $key => $l) {
        if ($smallHash == $l->getShortUrl()) {
            // Yes, this is ugly and slow
            return [$key => $l];
        }
    }
    throw new BookmarkNotFoundException();
}
```

作者自己都标注了 `ugly and slow` —— 当前是**O(n) 线性扫描**全部 bookmarks，逐个比较 `getShortUrl()`。Shaarli 是纯文件存储、面向"个人几百几千条书签"的体量设计，所以这种实现是可接受的。若要优化，可以在 BookmarkArray 中追加一个 `shortUrl => offset` 的映射表。

---

## 4. 旧链兼容与回查策略（三层保护）

### 4.1 第一层：Query String 小 Hash 重定向

老版本 Shaarli 的 permalink 是挂在根 URL 上的，例如 `https://host/?abcdef`（6 位 hash 直接当 query string）。新版本改走 `/shaare/abcdef` 路由，所以在首页控制器里做了一次兼容检测。

位置：[BookmarkListController.php#L210-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/front/controller/visitor/BookmarkListController.php#L210-L242)

```php
protected function processLegacyController(Request $request, Response $response): ?Response
{
    // Legacy smallhash filter
    $queryString = $this->container->environment['QUERY_STRING'] ?? null;
    if (null !== $queryString && 1 === preg_match('/^([a-zA-Z0-9-_@]{6})($|&|#)/', $queryString, $match)) {
        return $this->redirect($response, '/shaare/' . $match[1]);
    }

    // Legacy controllers (mostly used for redirections)  -- ?do=xxx 等
    // ...
}
```

正则 `/^([a-zA-Z0-9-_@]{6})($|&|#)/` 匹配：
- 开头正好 6 个合法 hash 字符
- 后面要么结束（`$`），要么接 `&`（其他参数），要么接 `#`（锚点）

命中后做 HTTP 重定向到新路由 `/shaare/{hash}`。**这是第一层旧链兼容：把旧 URL 结构的访问引导到新结构。**

### 4.2 第二层：数据迁移时保留旧 shorturl 值

数据从老版本升级时，有两个连续的 update 方法确保旧 hash 不丢失：

#### 步骤 A：LegacyLinkDB 读取时临时生成 shorturl（仅日期旧算法）

见 [LegacyLinkDB.php#L320-L327](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/legacy/LegacyLinkDB.php#L320-L327)，对还没迁移的、主键仍是 `linkdate` 的老数据：

```php
$link['shorturl'] = smallHash($link['linkdate']);  // 旧算法，仅日期
```

生成后暂存在内存里的 link 数组里。

#### 步骤 B：updateMethodDatastoreIds —— 从日期主键迁移到整数 ID，**不重算 shorturl**

位置：[LegacyUpdater.php#L247-L280](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/legacy/LegacyUpdater.php#L247-L280)

```php
public function updateMethodDatastoreIds()
{
    // ...判断数据库是否已是整数 ID 为主键...

    $links = [];
    foreach ($this->linkDB as $offset => $value) {
        $links[] = $value;        // value['shorturl'] 是步骤 A 生成的旧值
        unset($this->linkDB[$offset]);
    }
    $links = array_reverse($links);
    $cpt = 0;
    foreach ($links as $l) {
        unset($l['linkdate']);   // 去掉旧主键
        $l['id'] = $cpt;         // 分配新的整数 ID（0,1,2...）
        $this->linkDB[$cpt++] = $l;  // ★ 注意： $l['shorturl'] 没有变化！
    }
    // 保存 + 重排序
}
```

关键点：整个迁移过程只替换主键字段，**不动 `shorturl`**，因此：
- 老用户用 `?abcdef` 访问 → 经第 4.1 节的重定向到 `/shaare/abcdef` → 查表找到 `shorturl=abcdef` 的那条 bookmark → 正确命中

#### 步骤 C：updateMethodMigrateDatabase —— 数组 → Bookmark 对象

位置：[LegacyUpdater.php#L581-L596](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/legacy/LegacyUpdater.php#L581-L596)

```php
foreach ($this->linkDB as $key => $link) {
    $linksArray[$key] = (new Bookmark())->fromArray($link, ...);
}
```

`Bookmark::fromArray()` 直接把 `$data['shorturl']` 赋值给 `$this->shortUrl`（见 [Bookmark.php#L69-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/bookmark/Bookmark.php#L69-L91)），不会触发 `setId()` 中的 shortUrl 重算逻辑。旧值原样保留。

### 4.3 第三层：Note URL 的 `?` → `/shaare/` 迁移

旧版本中，"笔记"类型（无外部 URL，纯站内文本）的 url 字段被写成 `?abcdef` 形式，即和 permalink 的 query string 一致。新路由改为 `/shaare/abcdef` 后，需要把已存在的这些旧 url 也改过来。

位置：[Updater.php#L151-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/18-Shaarli/application/updater/Updater.php#L151-L173)

```php
public function updateMethodMigrateExistingNotesUrl(): bool
{
    foreach ($this->bookmarkService->search()->getBookmarks() as $bookmark) {
        if (
            $bookmark->isNote()
            && startsWith($bookmark->getUrl(), '?')
            && 1 === preg_match('/^\?([a-zA-Z0-9-_@]{6})($|&|#)/', $bookmark->getUrl(), $match)
        ) {
            $updated = true;
            $bookmark = $bookmark->setUrl('/shaare/' . $match[1]);
            $this->bookmarkService->set($bookmark, false);
        }
    }
    if ($updated) {
        $this->bookmarkService->save();
    }
    return true;
}
```

正则 `^\?([a-zA-Z0-9-_@]{6})($|&|#)` 把旧 url `?PCRizQ` 提取出 `PCRizQ`，重写为 `/shaare/PCRizQ`。

> 三层兼容汇总：
> 1. **HTTP 层**：`?{hash}` → 302 → `/shaare/{hash}`
> 2. **数据层**：迁移时旧 shorturl 值（仅日期算法）一字不改带入新表
> 3. **字段层**：note 的 url 从 `?{hash}` 规范化为 `/shaare/{hash}`

---

## 5. 三段处理链路总览图

```
┌───────────────────────────────────────────────────────────────────────┐
│                        一、生成链（创建 Bookmark）                     │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   BookmarkFileService::add()                                          │
│        │                                                              │
│        ▼                                                              │
│   BookmarkArray::getNextId()    ← 取 max(ID)+1，保证 ID 全局唯一      │
│        │                                                              │
│        ▼                                                              │
│   Bookmark::setId(id)                                                 │
│     ├─ 若 created 为空 → new DateTime()                               │
│     └─ 若 shortUrl 为空 → link_small_hash(created, id)                │
│                                │                                      │
│                                ▼                                      │
│                         smallHash(date_str + id)                      │
│                           ├─ crc32 (raw binary)                       │
│                           ├─ base64_encode                            │
│                           ├─ rtrim '='                                │
│                           └─ strtr '+/ → -_'  (base64url)            │
│                                │                                      │
│                                ▼                                      │
│                   产出 6 字符 shortUrl 写入 Bookmark                  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────┐
│                        二、查重链（按 shortUrl 查找）                  │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   GET /shaare/{hash}                                                  │
│        │                                                              │
│        ▼                                                              │
│   BookmarkListController::permalink()                                 │
│        │                                                              │
│        ▼                                                              │
│   BookmarkFileService::findByHash(hash, privateKey?)                  │
│        │                                                              │
│        ▼                                                              │
│   BookmarkFilter::filter(FILTER_HASH, hash)                           │
│        │                                                              │
│        ▼                                                              │
│   filterSmallHash()  → O(n) 线性遍历 bookmarks，比较 getShortUrl()    │
│        │                                                              │
│        ├─ 找到 + 公开 → 返回                                          │
│        ├─ 找到 + 私有 + 登录 → 返回                                   │
│        ├─ 找到 + 私有 + 游客 + ?key= 匹配 → 返回                      │
│        └─ 未找到 → throw BookmarkNotFoundException → 404 页面        │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────┐
│                     三、外链跳转链（旧链兼容回查）                      │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   用户访问  ?abcdef   (v0.8 之前的旧格式)                              │
│        │                                                              │
│        ▼                                                              │
│   GET / → BookmarkListController::index()                             │
│        │                                                              │
│        ▼                                                              │
│   processLegacyController()                                           │
│     ├─ 正则匹配 QUERY_STRING 是否为 6 位合法 hash                     │
│     └─ 命中 → redirect('/shaare/abcdef')   (第 1 层兼容)              │
│        │                                                              │
│        ▼                                                              │
│   回到"查重链"                                                         │
│        │                                                              │
│        ▼                                                              │
│   查表时，老 bookmark 的 shorturl 字段                                 │
│     = 迁移时 smallHash(仅日期) 的旧值      (第 2 层兼容)              │
│        │                                                              │
│        └─ filterSmallHash 线性比较 → 命中正确条目 ✅                   │
│                                                                       │
│   ─────────────────────────────────────                               │
│   附加：数据迁移管线（Updater 系列）                                   │
│   1. LegacyLinkDB::read()       → 用旧算法 smallHash(linkdate) 做临时 │
│   2. updateMethodDatastoreIds() → 替换主键为 ID，不动 shorturl        │
│   3. updateMethodMigrateDatabase() → 数组转 Bookmark，仍不动 shorturl│
│   4. updateMethodMigrateExistingNotesUrl() → ?xxx → /shaare/xxx      │
│                                                 (第 3 层兼容)         │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 6. 设计权衡总结

| 维度 | 选择 | 代价 |
|------|------|------|
| 冲突避让 | 不做"生成后查重"，而是用"自增 ID + 日期"保证输入唯一 | 依赖文件锁保证 ID 分配原子性；若外部绕过 service 直接改 datastore 可能破功 |
| Hash 算法 | CRC32 + base64url | 非加密安全，可被伪造，不能用于任何安全授权 |
| 查找效率 | O(n) 线性扫描 `filterSmallHash` | 代码简单但对超大量 bookmark 有性能压力；可追加 `shortUrl => offset` 映射表优化 |
| 旧链兼容 | 三层防护：HTTP 重定向 + 迁移时保留 shorturl 原值 + Note URL 字段修正 | 迁移步骤多，但链路完整，老用户的分享链接不会失效 |
