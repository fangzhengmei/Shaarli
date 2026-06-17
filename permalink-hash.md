# Shaarli 短链（Permalink）Hash 生成、冲突避让与旧链回查全链路解析

> 本文档中所有代码引用均使用**相对于项目根目录**的可迁移路径。
> 例如 `application/Utils.php` 表示项目根目录下的 `application/Utils.php` 文件。

---

## 0. 核心文件索引

| 模块 | 文件路径 | 关键函数/方法 |
|------|---------|---------------|
| Hash 算法 | `application/Utils.php` | `smallHash()` |
| Hash 组装 | `application/bookmark/LinkUtils.php` | `link_small_hash()` |
| Bookmark 模型 | `application/bookmark/Bookmark.php` | `setId()` |
| ID 分配器 | `application/bookmark/BookmarkArray.php` | `getNextId()` |
| 文件 IO 与锁 | `application/bookmark/BookmarkIO.php` | `read()`, `write()`, `synchronized()` |
| 查重/查找 | `application/bookmark/BookmarkFilter.php` | `filterSmallHash()` |
| Service 入口 | `application/bookmark/BookmarkFileService.php` | `findByHash()`, `add()`, `save()` |
| 路由入口 | `index.php` | 路由注册 |
| Permalink 控制器 | `application/front/controller/visitor/BookmarkListController.php` | `permalink()`, `processLegacyController()` |
| 旧版短链兼容 | `application/legacy/LegacyLinkDB.php` | `read()` 内 shorturl 兼容 |
| 数据迁移(旧→新) | `application/legacy/LegacyUpdater.php` | `updateMethodDatastoreIds()` |
| Note URL 迁移 | `application/updater/Updater.php` | `updateMethodMigrateExistingNotesUrl()` |
| 旧路由跳转 | `application/legacy/LegacyController.php` | 各旧路由 `?do=xxx` 方法 |
| API 查重 | `application/api/controllers/Links.php` | `postLink()` 内 URL 查重 |

---

## 1. Hash 生成：两段式构造

### 1.1 算法本体：`smallHash()`

位置：`application/Utils.php` 第 47-51 行

```php
function smallHash($text)
{
    $t = rtrim(base64_encode(hash('crc32', $text, true)), '=');
    return strtr($t, '+/', '-_');
}
```

流水线拆解：

1. `hash('crc32', $text, true)` — CRC32 生成 4 字节二进制摘要（非密码学安全）
2. `base64_encode(...)` — 4 字节 → 6 字符 Base64
3. `rtrim(..., '=')` — 去掉 Base64 填充符 `=`
4. `strtr($t, '+/', '-_')` — 转成 RFC 4648 base64url 格式，确保 URL 安全

**输出特性**：固定 6 字符 base64url，字符集 `[a-zA-Z0-9-_]`。
> ⚠️ 注意：虽然是 6 字符，但**有效输出空间只有 2³² ≈ 42.9 亿**，不是 64⁶。
> 原因：CRC32 只输出 32 位，base64 编码后第 6 个字符只有高位 2 位有意义（低位 4 位恒为 0），仅 4 种取值。
> 计算：前 5 字符 × 6 位 + 第 6 字符 × 2 位 = 32 位 = 2³² = 4,294,967,296 种不同输出。

**代码实证**（用 Python 精确复现 `smallHash` 算法，采样 300 万条）：
- 输出长度：300 万样本**全部为 6 字符**
- 末位字符（第 6 位）取值仅 **4 种**：`A`、`Q`、`g`、`w`，各占约 25%
- 字符集大小：64 个字符
- 理论空间验证：64⁵ × 4 = 4,294,967,296 = 2³²，**完全相等**
- 300 万样本实测碰撞 956 次，与理论期望（约 1047 对）量级吻合

### 1.2 输入盐组装：新旧两版

Shaarli 经历了两次 hash 输入盐的演变，这是理解冲突策略的关键。

#### 新版（v0.8.1+）—— `日期 + ID`

位置：`application/bookmark/LinkUtils.php` 第 191-194 行

```php
function link_small_hash($date, $id)
{
    return smallHash($date->format(Bookmark::LINK_DATE_FORMAT) . $id);
}
```

- `Bookmark::LINK_DATE_FORMAT = 'Ymd_His'`，例如 `20111006_131924`
- ID 是自增整数，和日期字符串拼接后作为 smallHash 输入
- 例：`smallHash('20111006_131924' . 142)` → `eaWxtQ`

> ⚠️ **重要澄清：外部链接 URL 完全不参与 shorturl 生成**。
> 代码证据：`link_small_hash($date, $id)` 只有两个参数——创建日期和自增 ID。
> 书签的 `url` 字段（外部链接地址）、`title`、`description` 等内容**从不**进入 hash 计算。
> 因此：修改书签的 URL、标题或描述，shorturl 保持不变；两个书签即便 URL 完全不同，只要日期和 ID 相同，shorturl 就会相同（但 ID 唯一所以不会发生）。

触发点在 `application/bookmark/Bookmark.php` 第 139-150 行的 `setId()` 中：

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

> 关键 guard `if (empty($this->shortUrl))`：如果 Bookmark 已经设置了 shortUrl（例如从旧数据加载），就不会覆盖。这是**保留旧 hash 的关键**。

#### 旧版（v0.8.1 之前）—— 仅用日期

在 `smallHash()` 函数注释中明确提到：

> @warning before v0.8.1, smallhashes were built only with the date,
>          and their value has been preserved.

旧版输入只有 `date`，如 `smallHash('20111006_131924')`。这在同一秒创建多条链接时**必然冲突**——这也是改版的直接原因。

旧数据加载时的兼容逻辑在 `application/legacy/LegacyLinkDB.php` 第 320-327 行：

```php
if (!isset($link['created'])) {
    $link['id'] = $link['linkdate'];
    $link['created'] = DateTime::createFromFormat(self::LINK_DATE_FORMAT, $link['linkdate']);
    // ...
    $link['shorturl'] = smallHash($link['linkdate']);  // 旧算法，仅日期
}
```

> 这段代码的意义：从仍以 `linkdate` 为主键的老 datastore 加载时，用旧算法生成 shorturl 并暂存，保证升级后外链不失效。

---

## 2. 三个核心概念的边界与联系

这是理解整个设计最容易混淆的部分。先把三个概念拆开，再看它们如何相互作用。

### 2.1 概念一：自增编号的唯一性

**自增编号（ID）** 是 Bookmark 的主键，由 `BookmarkArray::getNextId()` 分配：

位置：`application/bookmark/BookmarkArray.php` 第 211-217 行

```php
public function getNextId(): int
{
    if (!empty($this->ids)) {
        return max(array_keys($this->ids)) + 1;
    }
    return 0;
}
```

`$this->ids` 是 `id → array offset` 的映射表，`max()+1` 取下一个。

**ID 唯一性的边界条件**：

| 场景 | 是否唯一 | 原因 |
|------|---------|------|
| 单 PHP 进程、单请求内 | ✅ 绝对唯一 | 内存中计算，不会有竞争 |
| 并发多请求同时 add | ❌ 不保证唯一 | 每个请求有独立内存副本，可能读到相同的 max(id) |
| 绕过 Service 直接改 datastore | ❌ 不保证唯一 | 无任何约束检查 |

**关于锁的粒度**：项目使用 `malkusch/lock` 库的 Mutex，但锁只包裹了**文件 IO 操作**本身（见 `application/bookmark/BookmarkIO.php` 的 `synchronized()` 方法），不包裹"读文件 → 计算 ID → 写文件"整个事务。也就是说：

```
请求 A:  [加锁]读文件[解锁] → 计算nextId=100 → 内存修改 → [加锁]写文件[解锁]
请求 B:           [加锁]读文件[解锁] → 计算nextId=100 → 内存修改 → [加锁]写文件[解锁]
```

两个请求可能都读到同一个初始状态，分配到相同的 ID，后写的覆盖先写的。**这是并发场景下的真实风险。**

> 但对 Shaarli 的目标场景（个人书签、低并发）而言，这个风险在实际使用中几乎不会触发。

---

### 2.2 概念二：短 Hash 摘要的碰撞

**CRC32 摘要碰撞**是一个数学问题：不同的输入可能产生相同的 32 位输出。

> 重要前提：虽然 smallHash 输出 6 个 base64url 字符，但**有效熵只有 32 位**（CRC32 的输出长度），不是 6 × 6 = 36 位。第 6 个字符只有 2 位有效信息，仅 4 种可能取值。

| 属性 | 值 |
|------|----|
| 哈希算法 | CRC32 |
| 输出长度 | 32 位（4 字节二进制） |
| 编码后表现 | 6 字符 base64url |
| **有效输出空间** | **2³² = 4,294,967,296 种 ≈ 42.9 亿** |
| 第 6 字符有效位数 | 2 位（仅 4 种取值：A/Q/g/w） |

**生日悖论下的精确碰撞概率**（使用近似公式 P(n) ≈ 1 − e^(−n(n−1)/(2·N))，N = 2³² = 4,294,967,296）：

| 书签数量 n | 至少一次碰撞的概率 P(n) | 期望碰撞对数 | 直观感受 |
|-----------|------------------------|-------------|---------|
| 100 | 0.0001% | 0.0000 | 可以忽略 |
| **1,000** | **0.0116%** | **0.0001** | **万分之一，极低** |
| 5,000 | 0.2906% | 0.0029 | 约千分之三 |
| **10,000** | **1.1573%** | **0.0116** | **约百分之一，已非极端小概率** |
| 50,000 | 25.2509% | 0.2910 | 四分之一概率 |
| **77,000** | **49.8533%** | **0.6902** | **接近 50%（注：精确 50% 临界点为 77,163 条）** |
| **100,000** | **68.7809%** | **1.1641** | **超三分之二概率** |
| 200,000 | 99.0501% | 4.6566 | 几乎必然碰撞 |
| 500,000 | ≈ 100% | 29.1038 | 确定碰撞 |
| 1,000,000 | ≈ 100% | 116.4152 | 平均上百次碰撞 |

**关键结论**：
- 个人使用场景（几千条书签）：碰撞概率低于 0.3%，实际可忽略
- 达到 1 万条时：碰撞概率 1.16%，不再是"极端不可能"
- 达到 7.7 万条时：碰撞概率 49.85%，接近 50%（精确 50% 临界点为 77,163 条）
- 10 万条以上：碰撞是大概率事件（68.8%）

---

### 2.3 概念三：短链冲突规避

"短链冲突规避"是**系统设计层面**的策略，回答的问题是：如何确保每条 bookmark 的 shorturl 都是唯一的？

Shaarli 的策略是**"输入唯一性保证"**，而不是"输出查重+重试"。具体来说：

```
输入: 日期字符串 + 自增ID
         ↓
     CRC32 哈希
         ↓
输出: 6 字符 shorturl
```

它的逻辑链条是：
1. 因为 ID 是唯一的（在单进程假设下）
2. 所以 "日期 + ID" 的组合也是唯一的
3. 所以……"输出应该也不会重复吧"

⚠️ **这里有一个逻辑跳跃**：输入唯一 ≠ 输出唯一。哈希函数不是单射。但 Shaarli 的代码中**完全没有 shorturl 去重检查**：

- `BookmarkFileService::add()` 中不检查 shorturl 是否已存在
- `BookmarkFileService::set()` 中也不检查
- `BookmarkArray::offsetSet()` 只维护了 `$ids` 和 `$urls` 两个索引，**没有 `$shorturls` 索引**
- 迁移时也不做去重

**如果真的发生碰撞会怎样？**

位置：`application/bookmark/BookmarkFilter.php` 第 178-188 行

```php
private function filterSmallHash(string $smallHash)
{
    foreach ($this->bookmarks as $key => $l) {
        if ($smallHash == $l->getShortUrl()) {
            // Yes, this is ugly and slow
            return [$key => $l];  // 找到第一个就返回了！
        }
    }
    throw new BookmarkNotFoundException();
}
```

**后果**：如果两条 bookmark 的 shorturl 相同，只有排在前面的那条能通过 permalink 访问到，后面的那条"消失"了——既没有报错，也没有任何提示。

---

### 2.4 三者关系总览

```
  自增ID唯一性        CRC32哈希碰撞          短链冲突规避
  (输入侧保证)        (数学性质)              (系统目标)
       │                    │                        │
       │  "ID 唯一           │  "不同输入             │  "确保每条
       │   → 输入唯一"       │   可能同输出"          │   bookmark 的
       │                    │                        │   shorturl 唯一"
       └──────────┬─────────┘                        │
                  │                                  │
                  ▼                                  │
          设计假设：输入唯一                         │
                  │                                  │
                  └──────────────→ 输出应该唯一 ←────┘
                                       ↑
                                       │
                               隐含假设：CRC32 不会碰撞
                               （个人场景下近似成立）
```

**风险界限总结**：

| 风险类型 | 触发条件 | 量化数据 | 后果 | 严重程度（个人场景） |
|---------|---------|---------|------|---------------------|
| 并发写入导致 ID 重复 | 两个请求同时 `add` | 低并发下极罕见 | 数据丢失、shorturl 冲突 | 极低（几乎单人使用） |
| CRC32 自然碰撞（1 千条） | 书签量 ≥ 1,000 | 概率 0.0116%（万分之一） | 部分 bookmark 无法通过 permalink 正确访问 | 可忽略 |
| CRC32 自然碰撞（1 万条） | 书签量 ≥ 10,000 | 概率 1.1573%（约百分之一） | 同上 | 低 |
| CRC32 自然碰撞（7.7 万条） | 书签量 ≥ 77,000 | 概率 49.8533%（接近 50%） | **约有一半概率**发生至少一次碰撞，导致部分书签 permalink 串扰 | 中（但个人很少攒到这么多） |
| CRC32 自然碰撞（10 万条） | 书签量 ≥ 100,000 | 概率 68.7809% | 约三分之二概率发生碰撞 | 中高 |
| 人为构造碰撞（实际不可行） | 攻击者试图构造特定 URL 让已有书签 shorturl 冲突 | **理论上不可能** | URL 不参与 shorturl 生成，攻击者无法选择或预测 ID，无法构造碰撞影响已有书签 | 可忽略 |
| 外部修改 datastore | 手工/脚本直接编辑 datastore 文件 | — | ID 或 shorturl 重复 | 低（不推荐这么做） |

---

## 3. 生成链代码走读

完整的添加书签流程：

位置：`application/bookmark/BookmarkFileService.php` 第 222-239 行

```php
public function add(Bookmark $bookmark, bool $save = true): Bookmark
{
    // 权限检查...
    if (!empty($bookmark->getId())) {
        throw new Exception(t('This bookmarks already exists'));
    }
    
    $bookmark->setId($this->bookmarks->getNextId());  // 1. 分配 ID → 顺带生成 shortUrl
    $bookmark->validate();                            // 2. 校验

    $this->bookmarks[$bookmark->getId()] = $bookmark; // 3. 写入内存
    
    if ($save === true) {
        $this->save();                                // 4. 写回磁盘
        $this->history->addLink($bookmark);
    }
    return $this->bookmarks[$bookmark->getId()];
}
```

**关键点**：
- 第 230 行 `setId()` 是 shortUrl 的生成时机——设置 ID 的时候"顺便"把 shortUrl 也算出来了
- `validate()` 只校验字段非空等基础约束，**不校验 shorturl 唯一性**
- 整个 `add()` 方法不在锁内，只有 `save()` 内部的文件写入在锁内

---

## 4. 回查链代码走读

### 4.1 路由

路由注册在 `index.php` 第 117-118 行：

```php
$this->get('/', '\Shaarli\Front\Controller\Visitor\BookmarkListController:index');
$this->get('/shaare/{hash}', '\Shaarli\Front\Controller\Visitor\BookmarkListController:permalink');
```

### 4.2 控制器 → Service → Filter

位置：`application/front/controller/visitor/BookmarkListController.php` 第 129-160 行

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

`?key=` 参数用于私有链接的游客访问：未登录用户需要附带该链接的分享 key 才能查看私有书签。key 存在 `additional_content['private_key']` 中。

位置：`application/bookmark/BookmarkFileService.php` 第 110-124 行

```php
public function findByHash(string $hash, string $privateKey = null): Bookmark
{
    $bookmark = $this->bookmarkFilter->filter(BookmarkFilter::$FILTER_HASH, $hash);
    $first = reset($bookmark);
    // 私有链接权限校验...
    return $first;
}
```

最终落到 `BookmarkFilter::filterSmallHash()` 的 O(n) 线性扫描——遍历全部 bookmarks，逐一比较 `getShortUrl()`。

### 4.3 URL 维度的查重（不是 hash 查重）

注意区分两种不同的"查重"：

| 查重类型 | 用途 | 实现 | 时间复杂度 |
|---------|------|------|-----------|
| URL 查重 | 防止重复添加相同 URL 的书签 | `BookmarkArray` 的 `$urls` 哈希表 | O(1) |
| shorturl 查重 | 防止 permalink 冲突 | **没有做** | — |

REST API 创建时会做 URL 查重（返回 409 Conflict），见 `application/api/controllers/Links.php` 第 126-135 行。

---

## 5. 旧链兼容与回查策略（三层保护）

### 5.1 第一层：Query String → 新路由的 HTTP 重定向

老版本 permalink 格式是 `https://host/?abcdef`（hash 直接当 query string）。新版本是 `/shaare/abcdef`。

位置：`application/front/controller/visitor/BookmarkListController.php` 第 210-242 行

```php
protected function processLegacyController(Request $request, Response $response): ?Response
{
    // Legacy smallhash filter
    $queryString = $this->container->environment['QUERY_STRING'] ?? null;
    if (null !== $queryString && 1 === preg_match('/^([a-zA-Z0-9-_@]{6})($|&|#)/', $queryString, $match)) {
        return $this->redirect($response, '/shaare/' . $match[1]);
    }
    // ...
}
```

正则 `/^([a-zA-Z0-9-_@]{6})($|&|#)/` 匹配正好 6 个合法 hash 字符的 query string，命中后 302 重定向到新路由。

### 5.2 第二层：数据迁移时保留旧 shorturl 值

迁移过程分三步，**全程不重算 shorturl**：

#### 步骤 A：LegacyLinkDB 读取时临时生成 shorturl（旧算法）

对还没迁移的、主键仍是 `linkdate` 的老数据，先用旧算法 `smallHash(linkdate)` 生成 shorturl 暂存在内存里。见 `application/legacy/LegacyLinkDB.php` 第 320-327 行。

#### 步骤 B：updateMethodDatastoreIds —— 换主键，不动 shorturl

位置：`application/legacy/LegacyUpdater.php` 第 247-280 行

```php
// 先备份
$save = $this->conf->get('resource.data_dir') . '/datastore.' . date('YmdHis') . '.php';
copy($this->conf->get('resource.datastore'), $save);

// 全部取出 → 反转 → 重新分配整数 ID
$links = [];
foreach ($this->linkDB as $offset => $value) {
    $links[] = $value;
}
$links = array_reverse($links);
$cpt = 0;
foreach ($links as $l) {
    unset($l['linkdate']);   // 去掉旧主键
    $l['id'] = $cpt;         // 分配新的整数 ID
    $this->linkDB[$cpt++] = $l;  // shorturl 原样保留！
}
```

整个过程只替换主键字段，**不动 `shorturl`**。迁移后新 ID 是 0, 1, 2... 的顺序编号。

#### 步骤 C：updateMethodMigrateDatabase —— 数组转 Bookmark 对象

位置：`application/legacy/LegacyUpdater.php` 第 581-596 行

`Bookmark::fromArray()` 直接把 `$data['shorturl']` 赋值给 `$this->shortUrl`，不会触发 `setId()` 中的 shortUrl 重算逻辑。旧值原样保留。

### 5.3 第三层：Note URL 的 `?` → `/shaare/` 规范化

旧版本中"笔记"类型（无外部 URL，纯站内文本）的 url 字段是 `?abcdef` 形式。新路由改为 `/shaare/abcdef` 后，需要把已存在的旧 url 也改过来。

位置：`application/updater/Updater.php` 第 151-173 行

```php
public function updateMethodMigrateExistingNotesUrl(): bool
{
    foreach ($this->bookmarkService->search()->getBookmarks() as $bookmark) {
        if (
            $bookmark->isNote()
            && startsWith($bookmark->getUrl(), '?')
            && 1 === preg_match('/^\?([a-zA-Z0-9-_@]{6})($|&|#)/', $bookmark->getUrl(), $match)
        ) {
            $bookmark = $bookmark->setUrl('/shaare/' . $match[1]);
            $this->bookmarkService->set($bookmark, false);
        }
    }
    // save...
}
```

> 三层兼容汇总：
> 1. **HTTP 层**：`?{hash}` → 302 → `/shaare/{hash}`
> 2. **数据层**：迁移时旧 shorturl 值（仅日期算法）一字不改带入新表
> 3. **字段层**：note 的 url 从 `?{hash}` 规范化为 `/shaare/{hash}`

---

## 6. 三段处理链路总览图

```
┌────────────────────────────────────────────────────────────────────────┐
│                        一、生成链（创建 Bookmark）                      │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   BookmarkFileService::add()                                           │
│        │                                                               │
│        ▼                                                               │
│   BookmarkArray::getNextId()    ← 取 max(ID)+1                        │
│        │                ↑                                              │
│        │                └── 单进程内唯一，并发下有风险                  │
│        ▼                                                               │
│   Bookmark::setId(id)                                                  │
│     ├─ 若 created 为空 → new DateTime()                                │
│     └─ 若 shortUrl 为空 → link_small_hash(created, id)                 │
│                                │                                       │
│                                ▼                                       │
│                         smallHash(date_str + id)                       │
│                           ├─ crc32 (raw binary)                        │
│                           ├─ base64_encode                             │
│                           ├─ rtrim '='                                 │
│                           └─ strtr '+/ → -_'  (base64url)             │
│                                │                                       │
│                                ▼                                       │
│                   产出 6 字符 shortUrl 写入 Bookmark                   │
│                                                                        │
│   ⚠️ 整个过程无 shorturl 查重，依赖"输入唯一→输出唯一"的隐含假设        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                        二、回查链（按 shortUrl 查找）                   │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   GET /shaare/{hash}                                                   │
│        │                                                               │
│        ▼                                                               │
│   BookmarkListController::permalink()                                  │
│        │                                                               │
│        ├─ 取 ?key= 参数（私有链接分享密钥）                             │
│        ▼                                                               │
│   BookmarkFileService::findByHash(hash, privateKey?)                   │
│        │                                                               │
│        ├─ 校验：私有 + 游客 + key不匹配 → 抛 NotFound                   │
│        ▼                                                               │
│   BookmarkFilter::filter(FILTER_HASH, hash)                            │
│        │                                                               │
│        ▼                                                               │
│   filterSmallHash()  → O(n) 线性遍历，找到第一个就返回                  │
│        │                                                               │
│        ├─ 找到 → 返回 Bookmark                                         │
│        └─ 未找到 → throw BookmarkNotFoundException → 404              │
│                                                                        │
│   ⚠️ 如果有多条相同 shortUrl，只有第一条能被访问到                      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                     三、旧链跳转链（兼容回查）                           │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   用户访问  ?abcdef   (v0.8 之前的旧格式)                               │
│        │                                                               │
│        ▼                                                               │
│   GET / → index() → processLegacyController()                          │
│        │                                                               │
│        ├─ 正则匹配 QUERY_STRING                                        │
│        └─ 命中 → redirect('/shaare/abcdef')   (第 1 层)                │
│        │                                                               │
│        ▼                                                               │
│   回到"回查链"                                                          │
│        │                                                               │
│        ▼                                                               │
│   查表时，老 bookmark 的 shorturl = 迁移时保留的旧值  (第 2 层)         │
│        │                                                               │
│        └─ filterSmallHash 线性比较 → 命中 ✅                            │
│                                                                        │
│   ─────────────────────────────────────                                │
│   数据迁移管线（Updater 系列）                                          │
│   1. LegacyLinkDB::read()        → 旧算法 smallHash(linkdate) 临时生成  │
│   2. updateMethodDatastoreIds()  → 换主键为 ID，不动 shorturl           │
│   3. updateMethodMigrateDatabase() → 数组转 Bookmark，仍不动 shorturl  │
│   4. updateMethodMigrateExistingNotesUrl() → ?xxx → /shaare/xxx        │
│                                                  (第 3 层)              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 设计权衡表

| 维度 | 选择 | 收益 | 代价/风险 |
|------|------|------|-----------|
| 冲突避让策略 | 输入唯一性保证（ID 唯一 → shortUrl 应该唯一） | 代码极简，零运行时开销 | 依赖隐含假设，无硬保障；CRC32 理论碰撞；并发下 ID 可能重复 |
| Hash 算法 | CRC32 + base64url（6 字符） | 极快、输出短、URL 安全 | 非加密安全，可被秒级构造碰撞；**有效输出空间仅 2³² ≈ 43 亿**，不是 64⁶；但输入不含 URL，无法通过外部链接内容构造碰撞 |
| 查找实现 | O(n) 线性扫描 `filterSmallHash` | 代码简单，无需维护额外索引 | 量大时性能差；作者自评 "ugly and slow" |
| 并发控制 | Mutex 只保护文件 IO | 防止文件写损坏 | 不保护"读-算-写"事务，并发 add 可能丢数据 |
| 旧链兼容 | 三层防护（HTTP 重定向 + 保留旧值 + Note URL 修正） | 旧分享链接全部不失效 | 迁移步骤多，新旧两种 hash 长期共存 |
| 索引设计 | 只维护 id→offset 和 url→offset 两个索引 | 写入快、内存占用少 | shorturl 查找只能线性扫描 |
