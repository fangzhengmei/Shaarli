# Shaarli 标签云与搜索过滤分析

## 一、整体架构概览

标签云与搜索过滤涉及的核心类和文件：

| 层次 | 类/文件 | 职责 |
|------|---------|------|
| 控制器 | [TagCloudController](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/visitor/TagCloudController.php) | 标签云/列表页渲染 |
| 控制器 | [TagController](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/visitor/TagController.php) | 标签增删重定向 |
| 控制器 | [BookmarkListController](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/visitor/BookmarkListController.php) | 书签列表搜索过滤 |
| 控制器 | [SessionFilterController](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/admin/SessionFilterController.php) | 登录用户的可见性会话过滤 |
| 控制器 | [PublicSessionFilterController](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/visitor/PublicSessionFilterController.php) | 访客的每页数量/无标签过滤 |
| 服务层 | [BookmarkFileService](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php) | 搜索入口 & 标签计数 |
| 过滤层 | [BookmarkFilter](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php) | 核心过滤逻辑（标签/全文/哈希） |
| 结果层 | [SearchResult](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/SearchResult.php) | 分页封装 |
| 模型 | [Bookmark](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/Bookmark.php) | 书签实体（含标签数组） |
| 格式化 | [BookmarkFormatter](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/formatter/BookmarkFormatter.php) | 标签列表格式化 & 隐私标签过滤 |
| 格式化 | [BookmarkDefaultFormatter](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/formatter/BookmarkDefaultFormatter.php) | 搜索高亮 & HTML转义 |
| 工具 | [LinkUtils](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/LinkUtils.php) | `tags_str2array` / `tags_array2str` / `tags_filter` / hashtag自动链接 |
| 工具 | [Utils](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/Utils.php) | `normalize_spaces` / `escape` / `alphabetical_sort` |
| 模板 | [tag.cloud.html](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/tpl/default/tag.cloud.html) | 标签云页面 |
| 模板 | [tag.list.html](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/tpl/default/tag.list.html) | 标签列表页面 |
| 模板 | [tag.sort.html](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/tpl/default/tag.sort.html) | 排序导航条 |
| 模板 | [linklist.html](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/tpl/default/linklist.html) | 书签列表（含搜索结果与标签过滤） |

---

## 二、标签解析

### 2.1 标签分隔符

系统支持通过配置项 `general.tags_separator` 自定义标签分隔符，默认为空格 `' '`。该配置贯穿所有标签解析逻辑。

### 2.2 字符串 → 数组：`tags_str2array`

定义于 [LinkUtils.php#L217-L223](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/LinkUtils.php#L217-L223)：

```php
function tags_str2array(?string $tags, string $separator): array
{
    $separator = str_replace([' ', '/'], ['\s', '\/'], $separator);
    return preg_split('/\s*' . $separator . '+\s*/', trim($tags ?? ''), -1, PREG_SPLIT_NO_EMPTY) ?: [];
}
```

关键行为：
- 空格分隔符会被转为 `\s` 正则字符类，因此 **各种 Unicode 空白字符（tab、换行等）均能作为分隔符**
- 使用 `\s*` 和 `+` 量词：标签前后和分隔符之间的多余空白被忽略
- `PREG_SPLIT_NO_EMPTY`：空字符串标签被丢弃
- 逗号在此函数中**不被特殊处理**（但在 `BookmarkFilter::tagsStrToArray` 中被替换为空格）

### 2.3 数组 → 字符串：`tags_array2str`

定义于 [LinkUtils.php#L234-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/LinkUtils.php#L234-L237)：

```php
function tags_array2str(?array $tags, string $separator): string
{
    return implode($separator, tags_filter($tags, $separator));
}
```

先通过 `tags_filter` 清洗，再 `implode`。

### 2.4 标签清洗：`tags_filter`

定义于 [LinkUtils.php#L247-L253](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/LinkUtils.php#L247-L253)：

```php
function tags_filter(?array $tags, string $separator): array
{
    $trimDefault = " \t\n\r\0\x0B";
    return array_values(array_filter(array_map(function (string $entry) use ($separator, $trimDefault): string {
        return trim($entry, $trimDefault . $separator);
    }, $tags ?? [])));
}
```

- `trim` 时同时去除标准空白字符和分隔符字符
- `array_filter` 移除清洗后的空条目
- `array_values` 重置数组索引

### 2.5 Bookmark 模型中的标签设置

[Bookmark::setTags](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/Bookmark.php#L353-L363) 对保存的标签做了额外处理：

```php
public function setTags(?array $tags): Bookmark
{
    $this->tags = array_map(
        function (string $tag): string {
            return $tag[0] === '-' ? substr($tag, 1) : $tag;
        },
        tags_filter($tags, ' ')
    );
    return $this;
}
```

- **去除前导 `-`**：保存标签时如果标签以 `-` 开头，会被去掉。这意味着 `-tag` 存储为 `tag`，`-` 前缀只在搜索过滤时有意义（表示排除），不作为标签名的一部分持久化

---

## 三、过滤条件

### 3.1 过滤入口：`BookmarkFileService::search`

[BookmarkFileService.php#L137-L171](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L137-L171)：

```php
public function search(
    array $request = [],
    string $visibility = null,
    bool $caseSensitive = false,
    bool $untaggedOnly = false,
    bool $ignoreSticky = false,
    array $pagination = []
): SearchResult {
    if ($visibility === null) {
        $visibility = $this->isLoggedIn ? BookmarkFilter::$ALL : BookmarkFilter::$PUBLIC;
    }
    $searchTags = isset($request['searchtags']) ? $request['searchtags'] : '';
    $searchTerm = isset($request['searchterm']) ? $request['searchterm'] : '';
    // ...
    $bookmarks = $this->bookmarkFilter->filter(
        BookmarkFilter::$FILTER_TAG | BookmarkFilter::$FILTER_TEXT,
        [$searchTags, $searchTerm],
        $caseSensitive,
        $visibility,
        $untaggedOnly
    );
    return SearchResult::getSearchResult(
        $bookmarks,
        $pagination['offset'] ?? 0,
        $pagination['limit'] ?? null,
        $pagination['allowOutOfBounds'] ?? false
    );
}
```

核心流程：
1. **可见性默认值**：未登录 → `public`；已登录 → `all`
2. **组合过滤**：使用 `FILTER_TAG | FILTER_TEXT`（位或运算值为 `"vuotext"`）同时应用标签和全文过滤
3. **分页封装**：过滤结果经 `SearchResult::getSearchResult` 切片

### 3.2 过滤路由：`BookmarkFilter::filter`

[BookmarkFilter.php#L86-L135](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L86-L135)：

**过滤类型**：

| 常量 | 值 | 说明 |
|------|----|------|
| `$FILTER_HASH` | `"permalink"` | 精确哈希匹配 |
| `$FILTER_TEXT` | `"fulltext"` | 全文搜索 |
| `$FILTER_TAG` | `"tags"` | 标签过滤 |
| `$DEFAULT` | `"NO_FILTER"` | 无过滤（仅可见性过滤） |
| `$FILTER_TAG \| $FILTER_TEXT` | `"vuotext"` | 标签 + 全文组合过滤 |

**组合过滤 (`vuotext`) 的执行顺序**：

1. 若无任何搜索请求（标签和全文都为空）→ 返回 `noFilter` 或 `filterUntagged`
2. 先确定候选集：若 `untaggedonly`，从无标签书签中筛选；否则从全部书签中筛选
3. **先标签过滤，后全文过滤**：标签过滤缩窄范围后，再在结果上做全文搜索，是渐进式过滤
4. 每步都创建新的 `BookmarkFilter` 实例，用上一步结果作为输入

### 3.3 标签过滤：`filterTags`

[BookmarkFilter.php#L317-L415](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L317-L415)：

#### 3.3.1 输入解析

```php
$inputTags = $tags;
if (!is_array($tags)) {
    $inputTags = tags_str2array($inputTags, $tagsSeparator);
}
```

标签字符串通过 `tags_str2array` 转为数组。

#### 3.3.2 公开可见性下隐藏标签过滤

```php
if ($visibility === self::$PUBLIC) {
    $inputTags = array_values(array_filter($inputTags, function ($tag) {
        return ! startsWith($tag, '.');
    }));
    if (empty($inputTags)) {
        return [];
    }
}
```

- 当可见性为 `public` 时，**以 `.` 开头的隐藏标签从搜索输入中移除**
- 如果移除后搜索标签为空，返回空结果（防止未登录用户通过隐藏标签搜索到私有内容）

#### 3.3.3 标签搜索中的描述内 hashtag

```php
$search = $bookmark->getTagsString($tagsSeparator);
if (strlen(trim($bookmark->getDescription())) && strpos($bookmark->getDescription(), '#') !== false) {
    $descTags = [];
    preg_match_all(
        '/(?<![' . self::$HASHTAG_CHARS . '])#([' . self::$HASHTAG_CHARS . ']+?)\b/sm',
        $bookmark->getDescription(),
        $descTags
    );
    if (count($descTags[1])) {
        $search .= $tagsSeparator . tags_array2str($descTags[1], $tagsSeparator);
    }
}
```

- 标签搜索不仅匹配书签的正式标签，**还匹配描述中的 `#hashtag`**
- `HASHTAG_CHARS = '\p{Pc}\p{N}\p{L}\p{Mn}'`：支持 Unicode 字母、数字、下划线、组合标记
- 使用负向后顾 `(?<![...])` 确保 `#` 前不是 hashtag 有效字符，避免匹配 URL 中的 `#`
- 这意味着描述中写的 `#linux` 可以被 `searchtags=linux` 搜索到

### 3.4 全文搜索：`filterFulltext`

[BookmarkFilter.php#L211-L303](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L211-L303)：

#### 搜索语法

| 语法 | 含义 | 示例 |
|------|------|------|
| 空格分隔的词 | AND 逻辑（所有词都必须出现） | `hello world` → 同时含 hello 和 world |
| `"exact phrase"` | 精确短语匹配 | `"hello world"` → 含完整短语 |
| `-word` | 排除（NOT 逻辑） | `hello -world` → 含 hello 但不含 world |
| `-` 前缀不作用于精确短语 | 精确短语始终是正向匹配 | `-"hello world"` → 不会排除 |

#### 处理流程

1. **大小写归一化**：`mb_convert_case(html_entity_decode($searchterms), MB_CASE_LOWER, 'UTF-8')`
   - 先 `html_entity_decode` 解码 HTML 实体
   - 再 `MB_CASE_LOWER` 转 UTF-8 小写，支持 Unicode（西里尔字母、希腊字母等）
2. **提取精确短语**：`/"([^"]+)"/` 正则匹配引号内容
3. **分离 AND 词和排除词**：以 `-` 开头的词进入 `$excludeSearch`，其余进入 `$andSearch`
4. **构建全文搜索字符串**：[buildFullTextSearchableLink](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L617-L634) 将 title、description、url、tags 用 `\` 分隔拼接并全部小写化
5. **匹配与高亮**：搜索命中后，将匹配位置存入 `search_highlight` 附加内容，供格式化层渲染高亮

### 3.5 无标签过滤：`filterUntagged`

[BookmarkFilter.php#L424-L451](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L424-L451)：

- 仅返回 `getTags()` 为空数组的书签
- 同样受可见性约束
- 受插件 `filterSearchEntry` 约束

---

## 四、多标签组合逻辑

### 4.1 AND 逻辑（默认）

标签过滤的核心是 **正则表达式**，通过 [tag2regex](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L481-L500) 将每个标签转为正则片段，然后**串联拼接**实现 AND 逻辑：

```php
$re_and = implode(array_map([$this, 'tag2regex'], $inputTags));
$re = '/^' . $re_and;
// ...
$re .= '.*$/';
```

每个标签被转为正向前瞻/负向前瞻断言：

```
tag1 → (?=.*(?:^| )tag1(?:$| ))
tag2 → (?=.*(?:^| )tag2(?:$| ))
组合 → /^(?=.*(?:^| )tag1(?:$| ))(?=.*(?:^| )tag2(?:$| )).*$/
```

这确保**所有标签都必须出现**在书签的标签字符串中。

### 4.2 排除标签（`-` 前缀）

[BookmarkFilter::tag2regex](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L481-L500) 中：

```php
if ($tag[0] === "-") {
    $tag = substr($tag, 1);
    $negate = true;
}
```

排除标签生成**负向前瞻**：

```
-tag1 → (?!.*(?:^| )tag1(?:$| ))
```

这意味着书签标签列表中**不能出现**该标签。

### 4.3 OR 逻辑（`~` 前缀）

[BookmarkFilter::filterTags](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L346-L354) 中：

```php
$orTags = array_filter(array_map(function ($tag) {
    return startsWith($tag, '~') ? substr($tag, 1) : null;
}, $inputTags));
$re_or = implode('|', array_map([$this, 'tag2matchterm'], $orTags));
if ($re_or) {
    $re_or = '(' . $re_or . ')';
    $re .= $this->term2match($re_or, false);
}
```

- 以 `~` 开头的标签被提取并移除 `~` 前缀
- 多个 OR 标签用 `|` 连接成一个子正则
- 整个子正则作为**一个正向前瞻**附加到主正则中
- OR 标签**不参与** AND 逻辑（在 `tag2regex` 中被跳过）

示例：搜索 `linux ~ubuntu ~debian`：
- `linux` 是 AND 条件（必须出现）
- `ubuntu` 或 `debian` 至少出现一个
- 正则：`/^(?=.*(?:^| )linux(?:$| ))(?=.*(?:^| )(ubuntu|debian)(?:$| )).*/i`

### 4.4 通配符（`*`）

[tag2matchterm](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L512-L540) 中：

```php
if ($tag[$i] === '*') {
    $term .= '[^' . $tagsSeparator . ']*?';
} else {
    // ...
    $term .= preg_quote(substr($tag, $i, $offset - $i + 1), '/');
}
```

- `*` 被替换为 `[^separator]*?`（非贪婪匹配除分隔符外的任意字符）
- 非 `*` 字符通过 `preg_quote` 转义，防止注入正则特殊字符
- 支持 `*` 在任意位置：`prog*`、`*gram`、`pro*am`

### 4.5 强制包含（`+` 前缀）

```php
if ($tag[0] === "+" && $tag[1]) {
    $tag = substr($tag, 1);
}
```

`+` 前缀显式标记为 AND 条件（默认行为），主要用于在 OR 标签存在时明确指定某个标签必须匹配。

### 4.6 组合查询总结

| 前缀 | 语义 | 正则技术 | 示例 |
|------|------|----------|------|
| 无前缀 | AND（必须包含） | 正向前瞻 `(?=...)` | `linux` |
| `+` | AND（显式包含） | 正向前瞻 `(?=...)` | `+linux` |
| `-` | NOT（必须排除） | 负向前瞻 `(?!...)` | `-windows` |
| `~` | OR（至少匹配一个） | 分组正向前瞻 `(?=...(a\|b)...)` | `~ubuntu` |
| `*` | 通配符 | `[^sep]*?` | `lin*` |

---

## 五、隐私边界

### 5.1 三层隐私机制

```
┌─────────────────────────────────────────────────┐
│  第1层：数据加载                                  │
│  BookmarkFileService 构造函数                     │
│  privacy.hide_public_links → 不加载任何书签       │
├─────────────────────────────────────────────────┤
│  第2层：可见性过滤                                │
│  BookmarkFilter::noFilter / filterTags / etc.    │
│  visibility = all | public | private             │
├─────────────────────────────────────────────────┤
│  第3层：隐藏标签                                  │
│  BookmarkFormatter::filterTagList                │
│  BookmarkFileService::bookmarksCountPerTag       │
│  以 '.' 开头的标签对未登录用户不可见               │
└─────────────────────────────────────────────────┘
```

### 5.2 第1层：数据加载阶段的隐私

[BookmarkFileService 构造函数](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L62-L105)：

```php
if (!$this->isLoggedIn && $this->conf->get('privacy.hide_public_links', false)) {
    $this->bookmarks = new BookmarkArray();
}
```

当配置 `privacy.hide_public_links` 为 `true` 且用户未登录时，**完全不加载数据文件**，书签列表为空。这是最高级别的隐私保护。

### 5.3 第2层：可见性过滤

#### 可见性来源

- **未登录用户**：`BookmarkFileService::search` 默认 `$visibility = BookmarkFilter::$PUBLIC`
- **已登录用户**：默认 `BookmarkFilter::$ALL`，可通过 [SessionFilterController::visibility](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/admin/SessionFilterController.php#L22-L47) 切换为 `public` 或 `private`
- 会话 key：`SessionManager::KEY_VISIBILITY`（即 `"visibility"`）

#### 过滤实现

所有过滤方法（`noFilter`、`filterTags`、`filterFulltext`、`filterUntagged`）中均包含：

```php
if ($visibility !== 'all') {
    if (!$bookmark->isPrivate() && $visibility === 'private') {
        continue;
    } elseif ($bookmark->isPrivate() && $visibility === 'public') {
        continue;
    }
}
```

| visibility | 私有书签 | 公开书签 |
|------------|---------|---------|
| `all` | ✅ 可见 | ✅ 可见 |
| `public` | ❌ 不可见 | ✅ 可见 |
| `private` | ✅ 可见 | ❌ 不可见 |

#### Permalink 的隐私

[BookmarkFileService::findByHash](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L110-L124)：

```php
if (
    !$this->isLoggedIn
    && $first->isPrivate()
    && (empty($privateKey) || $privateKey !== $first->getAdditionalContentEntry('private_key'))
) {
    throw new BookmarkNotFoundException();
}
```

私有书签的 permalink 仅在以下情况可访问：
- 用户已登录
- 或者提供了正确的 `private_key`（通过"分享私有链接"功能生成）

### 5.4 第3层：隐藏标签（`.` 前缀）

隐藏标签（以 `.` 开头）在多处被过滤：

#### 5.4.1 搜索输入过滤

`filterTags` 中，当 `visibility === 'public'` 时移除以 `.` 开头的搜索标签：

```php
if ($visibility === self::$PUBLIC) {
    $inputTags = array_values(array_filter($inputTags, function ($tag) {
        return ! startsWith($tag, '.');
    }));
    if (empty($inputTags)) {
        return [];
    }
}
```

#### 5.4.2 标签云/列表过滤

[BookmarkFileService::bookmarksCountPerTag](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L324-L363)：

```php
if (
    empty($tag)
    || (! $this->isLoggedIn && startsWith($tag, '.'))
    || $tag === BookmarkMarkdownFormatter::NO_MD_TAG
    || in_array($tag, $filteringTags, true)
) {
    continue;
}
```

未登录用户不会在标签云/列表中看到 `.` 开头的标签。`nomarkdown` 标签也始终被隐藏。

#### 5.4.3 格式化层过滤

[BookmarkFormatter::filterTagList](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/formatter/BookmarkFormatter.php#L373-L389)：

```php
protected function filterTagList(array $tags): array
{
    if ($this->isLoggedIn === true) {
        return $tags;
    }
    $out = [];
    foreach ($tags as $tag) {
        if (strpos($tag, '.') === 0) {
            continue;
        }
        $out[] = $tag;
    }
    return $out;
}
```

在模板渲染的书签标签列表中，未登录用户也看不到 `.` 前缀标签。

#### 5.4.4 隐私边界总结

| 场景 | `.` 前缀标签 | 私有书签 |
|------|-------------|---------|
| 未登录 + 搜索 | 从搜索输入中移除，无法作为过滤条件 | 不会出现在搜索结果中 |
| 未登录 + 标签云 | 不显示 | 不统计 |
| 未登录 + 书签列表标签 | 不显示 | 不显示 |
| 已登录 | 全部可见 | 全部可见 |

---

## 六、特殊字符处理

### 6.1 标签名中的特殊字符

| 字符 | 在标签名中 | 在搜索中 | 说明 |
|------|-----------|---------|------|
| `.` | 作为标签名开头是隐藏标签 | 搜索输入中 `public` 可见性下被移除 | 隐私标记 |
| `*` | 作为标签名会被存储 | 搜索中作为通配符 | [tag2matchterm](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L512-L540) 将 `*` 转为 `[^sep]*?` |
| `-` | `setTags` 时自动去除前导 `-` | 搜索中作为排除前缀 | 持久化时被清理 |
| `+` | 作为标签名的一部分保存 | 搜索中作为显式 AND 前缀 | 仅搜索时消费前缀 |
| `~` | 作为标签名的一部分保存 | 搜索中作为 OR 前缀 | 仅搜索时消费前缀 |
| `,` | `BookmarkFilter::tagsStrToArray` 中被替换为空格 | 在 `tags_str2array` 中由正则处理 | 兼容旧格式 |
| 空白 | `tags_filter` 中被 trim | `tags_str2array` 用 `\s+` 分割 | 多余空白被忽略 |

### 6.2 标签正则安全

`tag2matchterm` 中对非通配符字符使用 `preg_quote` 转义：

```php
$term .= preg_quote(substr($tag, $i, $offset - $i + 1), '/');
```

这确保标签名中的正则特殊字符（如 `.`、`?`、`(`、`)`）不会破坏正则表达式。

但 `.` 前缀标签作为隐藏标签有特殊处理——搜索时如果输入 `.secret`，在 `public` 可见性下会被直接移除，不会进入正则构建。

### 6.3 描述中的 hashtag 正则

```php
public static $HASHTAG_CHARS = '\p{Pc}\p{N}\p{L}\p{Mn}';
```

hashtag 匹配正则 `/(?<![' . self::$HASHTAG_CHARS . '])#([' . self::$HASHTAG_CHARS . ']+?)\b/sm`：

- `\p{Pc}`：下划线等连接符
- `\p{N}`：任何文字的数字
- `\p{L}`：任何语言的字母
- `\p{Mn}`：组合标记（重音符号等）
- `(?<!...)`：负向后顾确保 `#` 前不是有效 hashtag 字符（防止匹配 URL 中的 `#`）

---

## 七、大小写归一化

### 7.1 标签搜索的归一化

[BookmarkFilter::filterTags](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L317-L415) 中：

```php
if (!$casesensitive) {
    $re .= 'i';  // 追加 i 标志使正则不区分大小写
}
```

- 默认 `$casesensitive = false`，正则追加 `i` 标志
- 此时 `Linux`、`linux`、`LINUX` 均匹配

### 7.2 标签输入的归一化

[BookmarkFilter::tagsStrToArray](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L463-L470)（未被当前代码直接调用，但提供了独立的大小写归一化）：

```php
public static function tagsStrToArray(string $tags, bool $casesensitive): array
{
    $tagsOut = $casesensitive ? $tags : mb_convert_case($tags, MB_CASE_LOWER, 'UTF-8');
    $tagsOut = str_replace(',', ' ', $tagsOut);
    return preg_split('/\s+/', $tagsOut, -1, PREG_SPLIT_NO_EMPTY);
}
```

- 当 `$casesensitive = false` 时，将输入标签转为 UTF-8 小写
- 逗号被替换为空格

### 7.3 全文搜索的归一化

[BookmarkFilter::filterFulltext](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L211-L303) 中：

```php
$search = mb_convert_case(html_entity_decode($searchterms), MB_CASE_LOWER, 'UTF-8');
```

以及 `buildFullTextSearchableLink` 中：

```php
$content  = mb_convert_case($link->getTitle(), MB_CASE_LOWER, 'UTF-8') . '\\';
$content .= mb_convert_case($link->getDescription(), MB_CASE_LOWER, 'UTF-8') . '\\';
// ...
```

- **搜索词和被搜索内容都转为 UTF-8 小写**，实现不区分大小写的全文搜索
- 使用 `mb_convert_case` 而非 `strtolower`，正确处理 Unicode 字符

### 7.4 标签云的大小写合并

[BookmarkFileService::bookmarksCountPerTag](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L324-L363) 中：

```php
$caseMapping = [];
foreach ($searchResult->getBookmarks() as $bookmark) {
    foreach ($bookmark->getTags() as $tag) {
        // ...
        if (!isset($caseMapping[strtolower($tag)])) {
            $caseMapping[strtolower($tag)] = $tag;
            $tags[$caseMapping[strtolower($tag)]] = 0;
        }
        $tags[$caseMapping[strtolower($tag)]]++;
    }
}
```

- 标签云使用 `strtolower` 做大小写合并
- **首次遇到的写法保留为显示名称**，后续不同写法的同一标签被合并计数
- 例如：先遇到 `Linux`，后遇到 `linux`，标签云显示 `Linux`，计数为 2

注意：这里用的是 `strtolower` 而非 `mb_convert_case`，对于纯 ASCII 标签无影响，但对某些 Unicode 字符可能不能正确合并。

---

## 八、搜索结果与模板渲染

### 8.1 数据流

```
用户请求 (searchtags, searchterm)
    │
    ▼
BookmarkListController::index
    │ normalize_spaces → escape
    │ sessionManager → visibility, untaggedonly, linksPerPage
    ▼
BookmarkFileService::search
    │ 默认可见性：未登录→public, 已登录→all
    ▼
BookmarkFilter::filter (FILTER_TAG | FILTER_TEXT)
    │
    ├─ filterTags → 正则匹配 + 可见性过滤
    │    └─ search_highlight 不在此层设置
    │
    └─ filterFulltext → 文本匹配 + 可见性过滤
         └─ 设置 search_highlight 到 Bookmark.additionalContent
    ▼
SearchResult::getSearchResult → 分页切片
    ▼
BookmarkDefaultFormatter::format → HTML 格式化
    │ filterTagList → 过滤隐藏标签
    │ tokenizeSearchHighlightField → 插入高亮标记
    │ replaceTokens → 替换为 <span class="search-highlight">
    ▼
模板 linklist.html 渲染
```

### 8.2 标签云数据流

```
用户请求 (searchtags, sort)
    │
    ▼
TagCloudController::processRequest
    │ 登录→从session获取visibility
    │ searchtags → filteringTags (explode)
    ▼
BookmarkFileService::bookmarksCountPerTag
    │ 先 search() 获取候选书签
    │ 遍历书签标签 → 大小写合并 + 隐藏标签过滤
    │ array_multisort → 按计数降序+字母升序
    ▼
TagCloudController
    │ TYPE_CLOUD → formatTagsForCloud (对数缩放字号)
    │ TYPE_LIST  → 直接传递
    │ alphabetical_sort → 按字母排序（cloud和alpha模式）
    ▼
模板 tag.cloud.html / tag.list.html 渲染
```

### 8.3 标签云字号缩放

[TagCloudController::formatTagsForCloud](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/visitor/TagCloudController.php#L103-L122)：

```php
$maxCount = count($tags) > 0 ? max($tags) : 0;
$logMaxCount = $maxCount > 1 ? log($maxCount, 30) : 1;
foreach ($tags as $key => $value) {
    $size = log($value, 15) / $logMaxCount * 2.2 + 0.8;
    $tagList[$key] = ['count' => $value, 'size' => number_format($size, 2, '.', '')];
}
```

- 对数缩放：`log(count, 15) / log(maxCount, 30) * 2.2 + 0.8`
- 字号范围大约 0.8em ~ 3.0em
- 归一化到最大计数的对数值

### 8.4 模板中的标签交互

#### 标签云 (tag.cloud.html)

```html
<a href="{$base_path}/?searchtags={$tags_url.$key1}{$tags_separator|urlencode}{$search_tags_url}"
   style="font-size:{$value.size}em;">{$key}</a>
<a href="{$base_path}/add-tag/{$tags_url.$key1}" title="{'Filter by tag'|t}" class="count">{$value.count}</a>
```

- 点击标签名 → 跳转到书签列表并按该标签过滤（`/add-tag/` 路由会追加标签到当前搜索）
- 点击计数 → 同样通过 `/add-tag/` 追加标签
- 已有搜索标签时，新标签会追加到现有搜索中

#### 书签列表 (linklist.html)

搜索结果中的标签：

```html
<span class="label label-tag" title="{$strAddTag}">
  <a href="{$base_path}/add-tag/{$value1.taglist_urlencoded.$key2}">{$value1.taglist_html.$key2}</a>
</span>
```

搜索条件中的标签（可移除）：

```html
<span class="label label-tag" title="{'Remove tag'|t}">
  <a href="{$base_path}/remove-tag/{function="$search_tags_url.$key1"}">
    {$value}<span class="remove"><i class="fa fa-times"></i></span>
  </a>
</span>
```

#### 标签增删路由

[TagController::addTag](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/visitor/TagController.php#L22-L72)：
- 从 HTTP_REFERER 解析当前搜索参数
- 将新标签追加到 `searchtags` 参数
- 防止重复添加
- 移除 `page` 参数（结果已变化，页码无意义）

[TagController::removeTag](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/front/controller/visitor/TagController.php#L79-L119)：
- 使用 `array_diff` 从搜索标签中移除指定标签
- 如果移除后搜索标签为空，删除 `searchtags` 参数

### 8.5 插件过滤

所有过滤方法都通过 `pluginManager->filterSearchEntry` 调用插件钩子：

```php
if (!$this->pluginManager->filterSearchEntry($bookmark, [...])) {
    continue;
}
```

插件可以基于上下文信息（source、搜索词、可见性等）自定义过滤逻辑。

---

## 九、关键交互总结

### 9.1 搜索参数传递链

```
URL参数: ?searchtags=linux+ubuntu&searchterm=hello
    │
    ▼  BookmarkListController
    ├── normalize_spaces($searchtags)
    ├── normalize_spaces($searchterm) → escape
    │
    ▼  BookmarkFileService::search
    ├── ['searchtags' => 'linux ubuntu', 'searchterm' => 'hello']
    │
    ▼  BookmarkFilter::filter
    ├── filterTags('linux ubuntu', false, $visibility, false)
    │   ├── tags_str2array('linux ubuntu', ' ') → ['linux', 'ubuntu']
    │   ├── tag2regex('linux') → '(?=.*(?:^| )linux(?:$| ))'
    │   ├── tag2regex('ubuntu') → '(?=.*(?:^| )ubuntu(?:$| ))'
    │   └── 正则: /^(?=.*(?:^| )linux(?:$| ))(?=.*(?:^| )ubuntu(?:$| ))).*$/i
    │
    └── filterFulltext('hello', $visibility)
        ├── mb_convert_case → 'hello'
        └── 在 title/description/url/tags 中搜索
```

### 9.2 可见性决策树

```
用户请求
    │
    ├── 未登录？
    │   └── visibility = public
    │       ├── 私有书签 → 不返回
    │       ├── 隐藏标签(.) → 从搜索输入中移除，从标签云中不显示
    │       └── privacy.hide_public_links = true → 空数据
    │
    └── 已登录？
        └── visibility = session['visibility'] ?? all
            ├── all → 全部书签
            ├── public → 仅公开书签
            └── private → 仅私有书签
```

---

## 十、深度分析：BookmarkFilter 核心代码追踪

### 10.1 filterFulltext 三路搜索：mb_strpos 与 strpos 混用

`filterFulltext` 包含精确匹配、AND 包含、NOT 排除三条独立的搜索路径，但字符串位置函数使用不一致。

[BookmarkFilter.php#L273-L290](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L273-L290)：

```php
// 路径1 + 路径2：精确搜索和AND搜索 → mb_strpos
foreach ([$exactSearch, $andSearch] as $search) {
    for ($i = 0; $i < count($search) && $found !== false; $i++) {
        $found = mb_strpos($content, $search[$i]);  // 多字节安全，返回位置
        if ($found === false) {
            break;
        }
        $foundPositions[] = ['start' => $found, 'end' => $found + mb_strlen($search[$i])];
    }
}

// 路径3：排除搜索 → strpos
for ($i = 0; $i < count($excludeSearch) && $found !== false; $i++) {
    $found = strpos($content, $excludeSearch[$i]) === false;  // 只判断存在，不关心位置
}
```

#### 三路搜索对比

| 路径 | 搜索类型 | 使用函数 | 返回值用途 | 多字节安全 |
|------|---------|---------|-----------|-----------|
| 1 | 精确短语 `"..."` | `mb_strpos` | 返回匹配位置，用于 `search_highlight` 高亮 | ✅ 是 |
| 2 | AND 关键词 | `mb_strpos` | 返回匹配位置，用于 `search_highlight` 高亮 | ✅ 是 |
| 3 | NOT 排除 `-word` | `strpos` | 仅判断 `=== false`，不记录位置 | ❌ 否 |

#### 为什么混用？

1. **精确和 AND 搜索需要位置**：命中后要记录 `start` 和 `end` 位置，在 `BookmarkDefaultFormatter` 中渲染 `<span class="search-highlight">` 高亮。`mb_strpos` 返回的字符位置（而非字节位置）与 `mb_strlen` 计算的长度匹配，才能正确高亮多字节字符。

2. **排除搜索只关心存在性**：`strpos(...) === false` 只需要知道"有没有"，不需要"在哪里"。只要搜索词和被搜索内容都经 `mb_convert_case` 转成小写 UTF-8，**纯 ASCII 排除词不会有问题**。

#### 潜在边界问题

如果排除词是纯中文（如 `-测试`），`strpos` 按字节搜索可能出现：
- 被搜索内容中某汉字的某字节与排除词的某字节巧合匹配
- 导致错误地排除或不排除

实际风险较低，因为中文排除词前面有 `-` 前缀（ASCII），`strpos` 匹配 `-` 后继续匹配后续字节，在 UTF-8 编码下不会跨字符边界误匹配（UTF-8 首字节和续字节有明确范围区分）。但严格来说这是不一致的编码风格。

---

### 10.2 tag2regex：单字符早返回与链式前缀剥离顺序

[BookmarkFilter::tag2regex](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L481-L500)：

```php
protected function tag2regex(string $tag): string
{
    $tagsSeparator = $this->conf->get('general.tags_separator', ' ');
    // 早返回：单字符或~开头直接跳过
    if (!$tag || $tag === "-" || $tag === "*" || $tag[0] === "~") {
        return '';
    }
    $negate = false;
    // 前缀剥离顺序：先 '+' → 再 '-'
    if ($tag[0] === "+" && $tag[1]) {
        $tag = substr($tag, 1);
    }
    if ($tag[0] === "-") {
        $tag = substr($tag, 1);
        $negate = true;
    }
    $term = $this->tag2matchterm($tag);
    return $this->term2match($term, $negate);
}
```

#### 单字符早返回逻辑

| 输入标签 | 条件判断 | 返回值 | 说明 |
|---------|---------|--------|------|
| `''` | `!$tag` → true | `''` | 空标签跳过 |
| `'-'` | `$tag === "-"` → true | `''` | 单独 `-` 无意义 |
| `'*'` | `$tag === "*"` → true | `''` | 单独 `*` 无意义 |
| `'~'` | `$tag[0] === "~"` → true | `''` | 单独 `~` 无意义 |
| `'~foo'` | `$tag[0] === "~"` → true | `''` | **所有 ~ 开头的标签都跳过**，因为 OR 标签有单独处理路径 |

注意：`$tag[0] === "~"` 优先级最高，任何 `~` 开头的标签（无论后面是什么）都会直接返回空字符串，不会进入前缀剥离逻辑。

#### 链式前缀剥离顺序

代码顺序是**先处理 `+`，再处理 `-`**，这个顺序会影响多前缀组合的语义：

| 输入 | 步骤1: 剥 `+` | 步骤2: 剥 `-` | 最终标签 | `negate` | 语义 |
|------|-------------|-------------|---------|----------|------|
| `+-foo` | 匹配 `+` → `-foo` | 匹配 `-` → `foo` | `foo` | `true` | **排除 `foo`** |
| `-+bar` | 第1字符是 `-`，不匹配 `+` | 匹配 `-` → `+bar` | `+bar` | `true` | **排除 `+bar` 这个标签名本身** |
| `~-baz` | `tag[0] === '~'` → 早返回 `''` | - | - | - | OR 标签，在另一路径处理 |
| `+-~qux` | 匹配 `+` → `-~qux` | 匹配 `-` → `~qux` | `~qux` | `true` | 排除标签名为 `~qux` 的标签 |

**关键结论**：
- `+-foo` 等同于 `-foo`：先剥 `+` 再剥 `-`，最终排除 `foo`
- `-+bar` 不等同于 `+bar` 或 `-bar`：由于 `+` 只在第1字符时被剥，`-+bar` 剥掉 `-` 后剩下 `+bar` 作为标签名，最终排除的是字面量 `+bar` 标签
- 这是一个设计上的不对称性：`+` 前缀必须是**第一个字符**才会被剥离，而 `-` 前缀在剥完 `+` 后只要在第一位就会被剥离

---

### 10.3 隐私旁路：`~.draft` 在 `visibility=public` 下的完整路径

这是一个真实的隐私漏洞。让我们逐行追踪 `searchtags='~.draft'` 在未登录（`visibility=public`）时的执行路径。

#### 路径追踪图

```
输入: searchtags='~.draft', visibility='public'
    │
    ▼ filterTags [BookmarkFilter.php#L317]
    │
    ├─ tags_str2array('~.draft', ' ') → ['~.draft']
    │
    ├─ 🔴 第332-340行：public可见性隐藏标签过滤
    │   if ($visibility === self::$PUBLIC) {
    │       $inputTags = array_values(array_filter($inputTags, function ($tag) {
    │           return ! startsWith($tag, '.');
    │       }));
    │   }
    │
    │   startsWith('~.draft', '.') → false （第1字符是'~'不是'.'）
    │   array_filter 保留 '~.draft' ✓
    │   $inputTags = ['~.draft']
    │
    ├─ 第343行：构建 AND 正则
    │   $re_and = implode(array_map('tag2regex', ['~.draft']))
    │   tag2regex('~.draft') → 第484行 $tag[0] === '~' → return ''
    │   $re_and = ''
    │
    ├─ 🔴 第346-348行：提取 OR 标签
    │   $orTags = array_filter(array_map(function ($tag) {
    │       return startsWith($tag, '~') ? substr($tag, 1) : null;
    │   }, $inputTags));
    │
    │   startsWith('~.draft', '~') → true
    │   substr('~.draft', 1) → '.draft'  ⚠️  隐藏标签泄露！
    │   $orTags = [0 => '.draft']
    │
    ├─ 第350行：构建 OR 正则
    │   $re_or = implode('|', array_map('tag2matchterm', ['.draft']))
    │   tag2matchterm('.draft') → 第533行 preg_quote('.', '/') → '\.'
    │   结果: '\.draft'
    │
    ├─ 第352-353行：包装成正向前瞻
    │   $re_or = '(\.draft)'
    │   $re .= term2match('(\.draft)', false)
    │        → '(?=.*(?:^| )(\.draft)(?:$| ))'
    │
    ├─ 第356-359行：最终正则
    │   $re = '/^' + '' + '(?=.*(?:^| )(\.draft)(?:$| ))' + '.*$/i'
    │   结果: /^(?=.*(?:^| )(\.draft)(?:$| )).*$/i
    │
    ▼ 遍历书签进行正则匹配
    │
    └─ 匹配公开书签的标签字符串（含.description中的hashtag）
       若某公开书签标签为 ['linux', '.draft']
       其标签字符串为 'linux .draft'
       正则匹配成功 ✓ → 该书签被返回
```

#### 漏洞根源

问题出在 [BookmarkFilter.php#L333-L335](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L333-L335) 的 `array_filter`：

```php
$inputTags = array_values(array_filter($inputTags, function ($tag) {
    return ! startsWith($tag, '.');
}));
```

它只检查标签**直接以 `.` 开头**的情况，但 `~.draft` 以 `~` 开头，绕过了过滤。随后在第347行 `substr($tag, 1)` 剥掉 `~` 后，`.draft` 这个隐藏标签名被暴露给 OR 正则构建。

**攻击向量**：未登录用户构造 URL `/?searchtags=~.hidden_tag_name`，即可搜索到**包含该隐藏标签的公开书签**。虽然书签本身是公开的，但隐藏标签（通常用于内部分类如 `.draft`、`.review`、`.internal`）的存在性信息被泄露。

**修复思路**：在第347行 `substr` 之后再次检查是否为 `.` 前缀，或在 `tag2matchterm` 中加入可见性判断过滤隐藏标签。

---

### 10.4 真实组合正则示例

以下是通过静态代码推导的各种组合搜索生成的真实正则表达式（分隔符为空格，大小写不敏感）：

| 搜索标签 | 生成的正则 | 说明 |
|---------|-----------|------|
| `linux` | `/^(?=.*(?:^| )linux(?:$| )).*$/i` | 简单 AND |
| `linux ubuntu` | `/^(?=.*(?:^| )linux(?:$| ))(?=.*(?:^| )ubuntu(?:$| )).*$/i` | 双 AND |
| `linux -windows` | `/^(?=.*(?:^| )linux(?:$| ))(?!.*(?:^| )windows(?:$| )).*$/i` | AND + 排除 |
| `~ubuntu ~debian` | `/^(?=.*(?:^| )(ubuntu|debian)(?:$| )).*$/i` | 纯 OR（注意：无 AND 条件时匹配所有含任一标签的书签） |
| `linux ~ubuntu ~debian` | `/^(?=.*(?:^| )linux(?:$| ))(?=.*(?:^| )(ubuntu|debian)(?:$| )).*$/i` | AND + OR 组合 |
| `+linux -windows` | `/^(?=.*(?:^| )linux(?:$| ))(?!.*(?:^| )windows(?:$| )).*$/i` | 显式 AND + 排除（与 `linux -windows` 等价） |
| `+-secret` | `/^(?!.*(?:^| )secret(?:$| )).*$/i` | `+-` 链式：先剥 `+` 再剥 `-`，最终排除 secret |
| `-+secret` | `/^(?!.*(?:^| )\+secret(?:$| )).*$/i` | `-+` 链式：剥 `-` 后 `+` 保留为标签名一部分，排除字面量 `+secret` |
| `pro*` | `/^(?=.*(?:^| )pro[^ ]*?(?:$| )).*$/i` | 通配符：匹配 pro 开头的标签 |
| `~.draft` (public) | `/^(?=.*(?:^| )(\.draft)(?:$| )).*$/i` | **⚠️ 隐私漏洞**：OR 前缀绕过 `.` 过滤，匹配隐藏标签 |
| `.secret` (public) | `[]` (空) | 正常情况：`.secret` 被 array_filter 移除，返回空结果 |
| `linux ~.draft` (public) | `/^(?=.*(?:^| )linux(?:$| ))(?=.*(?:^| )(\.draft)(?:$| )).*$/i` | **⚠️ 组合漏洞**：AND 正常标签 + OR 隐藏标签 |

#### tag2matchterm 特殊字符转义验证

`tag2matchterm` 对非 `*` 字符使用 `preg_quote($str, '/')` 转义，确保正则安全：

| 标签名 | tag2matchterm 输出 | 说明 |
|--------|-------------------|------|
| `.draft` | `\.draft` | `.` 转义为 `\.` |
| `tag?` | `tag\?` | `?` 转义为 `\?` |
| `tag+name` | `tag\+name` | `+` 转义为 `\+` |
| `tag(name)` | `tag\(name\)` | 括号转义 |
| `tag[name]` | `tag\[name\]` | 方括号转义 |
| `tag$` | `tag\$` | `$` 转义为 `\$` |
| `tag^start` | `tag\^start` | `^` 转义为 `\^` |
| `pro*` | `pro[^ ]*?` | `*` 特殊处理为通配符，不转义 |

所有正则元字符都被正确转义 ✓。但通配符 `*` 被特殊处理为 `[^分隔符]*?`，这是有意设计的语法特性。

---

### 10.5 正则构建关键函数协作图

```
filterTags()
    │
    ├─ 输入: tags字符串 → tags_str2array() → $inputTags数组
    │
    ├─ [public可见性] array_filter 过滤 '.' 前缀
    │   └── ⚠️  漏洞点: '~.tag' 绕过检查
    │
    ├─ AND 部分: array_map(tag2regex, $inputTags) → implode
    │   │
    │   └─ tag2regex($tag)
    │       ├─ 早返回: 空/-/*/~开头 → ''
    │       ├─ 剥 '+' 前缀（仅第1字符）
    │       ├─ 剥 '-' 前缀，设置 negate=true
    │       ├─ tag2matchterm($tag) → 转义正则字符，*→[^sep]*?
    │       └─ term2match($term, $negate) → (?=...) 或 (?!...)
    │
    ├─ OR 部分: array_map 提取 '~' 前缀标签 → substr($tag,1)
    │   │
    │   └── ⚠️  漏洞点: '~.tag' → substr → '.tag' → 进入 tag2matchterm
    │
    ├─ 组合: '/^' + $re_and + $re_or_lookahead + '.*$' + 'i'
    │
    └─ 遍历书签: preg_match($re, $searchString)
        └─ $searchString = tagsString + ' ' + descriptionHashtags
```

### 10.6 关键不一致性总结

| 问题 | 位置 | 影响 |
|------|-----|------|
| `mb_strpos` vs `strpos` 混用 | filterFulltext 第278/289行 | 理论上多字节排除词可能不准确，实际风险低 |
| `+` 前缀仅在第1字符剥离 | tag2regex 第489行 | `-+bar` 语义反直觉，`+` 成为标签名一部分 |
| OR 标签路径无隐藏标签二次过滤 | filterTags 第346-354行 | **高风险隐私漏洞**：`~.hidden` 绕过 public 可见性检查 |
| `strtolower` vs `mb_convert_case` | bookmarksCountPerTag 第341行 | 纯 ASCII 没问题，Unicode 大小写合并可能不完整 |
| 通配符 `*` 不转义 | tag2matchterm 第520-522行 | 这是设计特性，不是 bug，但与其他字符处理不一致 |

---

## 十一、进阶分析：UTF-8 安全边界与 Unicode 大小写陷阱

### 11.1 strpos 在 UTF-8 字节扫描的两端安全性

[BookmarkFilter::filterFulltext](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L287-L290) 第289行使用 `strpos` 而非 `mb_strpos` 进行排除词匹配。虽然 `strpos` 是字节级扫描，但在排除搜索场景下具有**两端安全**特性。

#### UTF-8 编码结构基础

UTF-8 是变长编码，字节范围严格区分：

| 字节数 | 首字节格式 | 续字节格式 | 码点范围 |
|-------|-----------|-----------|---------|
| 1 | `0xxxxxxx` | 无 | U+0000 ~ U+007F (ASCII) |
| 2 | `110xxxxx` | `10xxxxxx` | U+0080 ~ U+07FF |
| 3 | `1110xxxx` | `10xxxxxx` | U+0800 ~ U+FFFF |
| 4 | `11110xxx` | `10xxxxxx` | U+10000 ~ U+10FFFF |

**关键属性**：
- 续字节固定为 `10xxxxxx`（范围 0x80 ~ 0xBF）
- 首字节范围与续字节范围**完全不重叠**
- ASCII 字符 `0xxxxxxx`（0x00 ~ 0x7F）永远不会出现在多字节字符内部

#### 排除词处理流程：

```
用户输入: "-测试" (排除词: 测试)
    │
    ▼ mb_convert_case(html_entity_decode("-测试"), MB_CASE_LOWER, 'UTF-8')
    │  排除词提取: substr("-测试", 1) → "测试"
    │
    ▼ 编码: UTF-8 字节序列
   "测" = E6 B5 8B
   "试" = E8 AF 95
    │
    ▼ strpos($content, "测试") === false
```

#### 为什么字节扫描不会跨字符边界误匹配：

```
被搜索内容 $content 也经 mb_convert_case 转成 UTF-8 小写。

搜索词 "测试" 首字节 0xE6 (11100110) 是 3 字节字符首字节格式。

如果 $content 中某多字节字符内部某字节碰巧是 0xE6？不可能！
因为多字节字符内部字节都是 0x80-0xBF (10xxxxxx)，
0xE6 (11100110) 高 4 位是 1110，属于首字节范围，
绝不会出现在字符内部。

结论：strpos 匹配到 0xE6 时，它必然是某个字符首字节，
后续字节也会继续匹配，不会跨字符边界。
```

#### 两端安全的三层保障

| 安全边界 | 说明 |
|-----------|------|
| 首字节不重叠性 | UTF-8 首字节范围与续字节范围完全不重叠，strpos 找到的首字节必然是真实字符起始 |
| 排除词前缀是 ASCII `-` | 排除词由 `substr($needle, 1) 去除 `-` 前缀，但 `-` 是 ASCII，不参与 strpos 搜索的排除词即使是纯多字节，其首字节也不会误匹配 |
| 两端都经 UTF-8 规范化 | 搜索词和被搜索内容都经 `mb_convert_case` 转成 UTF-8 小写，编码一致 |

#### 理论风险边界情况：

```
极端情况：被搜索内容含 ASCII 字符，而搜索词是某多字节字符的某字节序列？
例如：搜索词 "а" (U+0430)，UTF-8 D0 B0
如果被搜索内容有字符 "Ð°" (U+00D0 U+00B0)，UTF-8 C3 90 C2 B0
D0 是首字节格式 (11010000)，不会出现在字符内部。
即使后续字节 B0 是续字节格式 (10110000)，但 strpos 是连续字节匹配，
找到 D0 时已确认是字符首字节，后续匹配 B0 也必然是同一字符的续字节。

结论：在 UTF-8 编码下，strpos 字节扫描不会跨字符边界匹配。
```

**最终结论**：虽然 `strpos` 是字节级函数，但由于 UTF-8 编码的自同步特性（首字节与续字节范围不重叠），在排除搜索场景下是安全的。但代码风格上与 `mb_strpos` 不一致，应统一使用 `mb_strpos` 以避免未来维护者误解。

---

### 11.2 OR 标签：~+foo、~-baz、~.draft、~pro* 不再剥前缀的转换过程

OR 标签路径与 AND 标签路径是两条**完全独立**的处理流水线。AND 路径经过 `tag2regex` 会剥 `+`/`-` 前缀，但 OR 路径仅经过 `tag2matchterm`，**不剥任何前缀**。

#### 处理路径对比

```
AND 标签路径 (tag2regex):
  +foo → 剥 '+' → foo → tag2matchterm → term2match → (?=.*foo)
  -bar → 剥 '-' → bar → tag2matchterm → term2match → (?!.*bar)

OR 标签路径 (substr + tag2matchterm):
  ~+foo → substr(1) → +foo → tag2matchterm → \+foo → (?=.*\+foo)
  ~-baz → substr(1) → -baz → tag2matchterm → \-baz → (?=.*\-baz)
  ~.draft → substr(1) → .draft → tag2matchterm → \.draft → (?=.*\.draft)
  ~pro* → substr(1) → pro* → tag2matchterm → pro[^ ]*? → (?=.*pro[^ ]*?)
```

#### 逐个转换追踪

**1. ~+foo**

| 步骤 | 处理 | 结果 |
|-----|------|------|
| 输入 | `~+foo` | |
| tag2regex | `$tag[0] === '~'` → 返回 `''` | AND 部分跳过 |
| OR 提取 | `substr('~+foo', 1) → `+foo` | 剥掉 `~`，保留 `+` |
| tag2matchterm | `preg_quote('+')` → `\+` | `+` 被转义为 `\+` |
| 正则片段 | `(?=.*(?:^| )\+foo(?:$| )` | 匹配字面量 `+foo` 标签 |

**2. ~-baz**

| 步骤 | 处理 | 结果 |
|-----|------|------|
| 输入 | `~-baz` | |
| tag2regex | `$tag[0] === '~'` → 返回 `''` | AND 部分跳过 |
| OR 提取 | `substr('~-baz', 1) → `-baz` | 剥掉 `~`，保留 `-` |
| tag2matchterm | `preg_quote('-')` → `\-` | `-` 被转义为 `\-` |
| 正则片段 | `(?=.*(?:^| )\-baz(?:$| )` | 匹配字面量 `-baz` 标签 |

**3. ~.draft (public 可见性)**

| 步骤 | 处理 | 结果 |
|-----|------|------|
| 输入 | `~.draft` | |
| array_filter | `startsWith('~.draft', '.')` → `false` | 绕过 `.` 过滤 ✓ |
| tag2regex | `$tag[0] === '~'` → 返回 `''` | AND 部分跳过 |
| OR 提取 | `substr('~.draft', 1) → `.draft` | 剥掉 `~`，保留 `.` |
| tag2matchterm | `preg_quote('.')` → `\.` | `.` 被转义为 `\.` |
| 正则片段 | `(?=.*(?:^| )(\.draft)(?:$| ))` | ⚠️ 匹配 `.draft` 隐藏标签 |

**4. ~pro***

| 步骤 | 处理 | 结果 |
|-----|------|------|
| 输入 | `~pro*` | |
| tag2regex | `$tag[0] === '~'` → 返回 `''` | AND 部分跳过 |
| OR 提取 | `substr('~pro*', 1) → `pro*` | 剥掉 `~`，保留 `*` |
| tag2matchterm | `'*'` → `[^ ]*?` | `*` 转为通配符 |
| 正则片段 | `(?=.*(?:^| )pro[^ ]*?(?:$| ))` | 匹配 `pro` 开头的标签 |

#### 设计意图与不一致性

OR 路径不剥 `+`/`-` 前缀的原因可能是：
- OR 语法设计为"匹配这些标签中的任意一个"，如果标签名本身就包含 `+` 或 `-` 前缀，需要精确匹配
- 但这与 AND 路径的行为不一致，可能导致用户困惑

| 搜索 | 搜索意图 | 实际匹配 |
|-------|---------|---------|
| `+foo` | 必须包含 foo | ✓ 匹配 foo |
| `~+foo` | OR 匹配 foo 或... | ❌ 匹配字面量 `+foo` 标签 |
| `-bar` | 必须排除 bar | ✓ 排除 bar |
| `~-baz` | OR 排除... | ❌ 匹配字面量 `-baz` 标签 |

---

### 11.3 strtolower 对 Unicode 大小写合并的反例

[BookmarkFileService::bookmarksCountPerTag](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L340-L345) 第341行使用 `strtolower($tag)` 进行大小写合并，但 `strtolower` 是**单字节函数**，只能正确处理 ISO-8859-1 (Latin-1) 字符，对 Unicode 字符处理不正确。

#### strtolower vs mb_convert_case 对比

| 函数 | 编码感知 | 处理范围 |
|------|---------|---------|
| `strtolower` | ❌ 单字节 | 仅 ISO-8859-1 (U+0000 ~ U+00FF) |
| `mb_convert_case(MB_CASE_LOWER)` | ✅ Unicode | 全部 Unicode 字符 |

#### 反例 1：Étude (法语"练习曲")

```
书签1 标签: "Étude"  (U+00C9 tude)
书签2 标签: "étude"  (U+00E9 tude)

strtolower("Étude"):
  É (U+00C9) 在 ISO-8859-1 范围内 (0xC9)
  strtolower(0xC9) 映射到 0xE9 é，**单字节正确 ✓

等等，É 实际上在 ISO-8859-1 中存在！
ISO-8859-1: É = 0xC9, é = 0xE9
strtolower 能正确转换 É → é ✓

但更复杂的 Unicode 字符呢？
```

**真正的反例：带重音符号的大写字母不在 Latin-1 范围时：**

| 字符 | Unicode | strtolower 结果 | mb_convert_case 结果 | 是否一致？ |
|-----|---------|---------------|----------------------|-----------|
| É | U+00C9 | é (U+00E9) | é (U+00E9) | ✅ 一致（在 Latin-1 范围） |
| é | U+00E9 | é | é | ✅ |
| ẞ | U+1E9E)（德语大写 ß) | ẞ (不变，因为 U+1E9E 不在 Latin-1) | ss (U+0073 U+0073) | ❌ 不一致 |
| ß | U+00DF) | ß (不变，strtolower 对已经是小写) | ß | ✅ 一致（但大写是 SS) |
| Ğ | U+011E)（土耳其语 G 带-breve) | Ğ (不变，U+011E 不在 Latin-1) | ğ (U+011F) | ❌ 不一致 |
| ğ | U+011F) | ğ | ğ | ✅ |
| Σ | U+03A3)（希腊大写 Sigma) | Σ (不变，U+03A3 不在 Latin-1) | σ (U+03C3) 或 ς (U+03C2) | ❌ 不一致 |
| σ | U+03C3) | σ | σ | ✅ |

**反例 2：德语 ß/SS 大小写对

```
书签1 标签: "Straße"  (含 ß U+00DF)
书签2 标签: "STRASSE"  (全大写)

处理流程:
  strtolower("Straße")  →  "straße"  (ß 保持不变，strtolower 不认识 Unicode 小写字符)
  strtolower("STRASSE") →  "strasse"  (A-Z → a-z)

结果:
  $caseMapping["straße"]  →  "Straße"
  $caseMapping["strasse"] 不存在！
  两个标签被当作不同标签计数，计数分别累计

但 mb_convert_case:
  mb_convert_case("Straße", MB_CASE_LOWER, 'UTF-8')  →  "straße"
  mb_convert_case("STRASSE", MB_CASE_LOWER, 'UTF-8')  →  "strasse"
  注意：ß 的大写是 SS，所以 STRASSE 小写还是 strasse

实际上 ß 和 SS 小写不同，这是语言特性。
```

**反例 3：土耳其语 Ğ/ğ

```
书签1 标签: "Ğüne"  (Ğ U+011E)
书签2 标签: "ğüne"  (ğ U+011F)

strtolower("Ğüne")  →  "Ğüne"  (Ğ 保持不变！U+011E 不在 Latin-1)
strtolower("ğüne")  →  "ğüne"

结果:
  $caseMapping["Ğüne"]  →  "Ğüne"
  $caseMapping["ğüne"]  →  "ğüne"
  两个标签被当作不同标签 ❌

mb_convert_case 正确:
  mb_convert_case("Ğüne", MB_CASE_LOWER, 'UTF-8')  →  "ğüne"
  mb_convert_case("ğüne", MB_CASE_LOWER, 'UTF-8')  →  "ğüne"
  正确合并为同一个标签 ✓
```

**反例 4：希腊语 Σ/σ/ς

```
书签1 标签: "Σύνταξη"  (Σ U+03A3)
书签2 标签: "σύνταξη"  (σ U+03C3)

strtolower("Σύνταξη")  →  "Σύνταξη"  (Σ 保持不变！)
strtolower("σύνταξη")  →  "σύνταξη"

结果:
  两个标签被当作不同标签 ❌

mb_convert_case 正确:
  mb_convert_case("Σύνταξη", MB_CASE_LOWER, 'UTF-8')  →  "σύνταξη"
  正确合并 ✓
```

#### 实际影响范围

| 语言 | 受影响字符示例 | 影响 |
|-----|---------------|------|
| 德语 | ß/SS, ẞ | 大小写不合并 |
| 土耳其语 | Ğ/ğ, İ/i, I/ı | 大量字符不合并 |
| 希腊语 | Σ/σ/ς | 大小写不合并 |
| 俄语 | А/а, П/п | 所有西里尔字母 |
| 中文/日文 | 无大小写概念 | 无影响 |
| 法语/西班牙语 | É/é, Ñ/ñ | 在 Latin-1 范围内的字符正确，扩展字符不正确 |

对于 Shaarli 标签云来说，这个 bug 影响非 Latin-1 脚本的用户。

---

### 11.4 真实运行正则匹配测试（预期输出）

> ⚠️ 环境未安装 PHP，以下为预期输出推导结果。可执行命令：`php -r '...'

#### 测试 1：AND + 排除组合

```php
$re = '/^(?=.*(?:^| )linux(?:$| ))(?!.*(?:^| )windows(?:$| )).*$/i';
$tags1 = 'linux ubuntu';
$tags2 = 'linux windows';
$tags3 = 'ubuntu windows';
var_dump(preg_match($re, $tags1));  // int(1) ✓
var_dump(preg_match($re, $tags2));  // int(0) ✓
var_dump(preg_match($re, $tags3));  // int(0) ✓
```

**预期输出**：
```
int(1)
int(0)
int(0)
```

#### 测试 2：AND + OR 组合

```php
$re = '/^(?=.*(?:^| )linux(?:$| ))(?=.*(?:^| )(ubuntu|debian)(?:$| )).*$/i';
$tags1 = 'linux ubuntu';       // int(1) ✓
$tags2 = 'linux debian';       // int(1) ✓
$tags3 = 'linux gentoo';       // int(0) ✓
$tags4 = 'ubuntu debian';    // int(0) ✓
```

**预期输出**：
```
int(1)
int(1)
int(0)
int(0)
```

#### 测试 3：~.draft 隐私旁路验证

```php
// visibility=public，搜索 ~.draft
$re = '/^(?=.*(?:^| )(\.draft)(?:$| )).*$/i';
$tags1 = 'linux .draft';     // int(1) ⚠️ 匹配到隐藏标签
$tags2 = 'linux draft';        // int(0) ✓ 不匹配普通 draft
$tags3 = '.draft';            // int(1) ⚠️ 匹配到隐藏标签

// 正常情况：搜索 .draft（public）
// array_filter 移除 .draft，返回空数组，搜索结果为空
```

**预期输出**：
```
int(1)   ⚠️ 隐私漏洞：公开书签含 .draft 被搜到
int(0)
int(1)   ⚠️
```

#### 测试 4：通配符匹配

```php
$re = '/^(?=.*(?:^| )pro[^ ]*?(?:$| )).*$/i';
$tags1 = 'programming';       // int(1) ✓
$tags2 = 'project';           // int(1) ✓
$tags3 = 'pro';                 // int(1) ✓
$tags4 = 'apropos';           // int(0) ✓ 必须是完整标签
```

**预期输出**：
```
int(1)
int(1)
int(1)
int(0)
```

#### 测试 5：链式前缀 ~+foo 匹配

```php
// 搜索 ~+foo
$re = '/^(?=.*(?:^| )(\+foo)(?:$| )).*$/i';
$tags1 = '+foo bar';           // int(1) ✓ 匹配字面量 +foo
$tags2 = 'foo bar';           // int(0) ❌ 用户预期匹配 foo
```

**预期输出**：
```
int(1)
int(0)
```

#### 测试 6：Unicode 大小写测试

```php
$re = '/^(?=.*(?:^| )étude(?:$| )).*$/i';
$tags1 = 'Étude';              // int(1) ✓ 正则 i 标志不区分大小写
$tags2 = 'étude';              // int(1) ✓

// 但 strtolower 合并测试
$caseMapping = [];
$tag1 = 'Étude';
$tag2 = 'étude';
$caseMapping[strtolower($tag1)] = $tag1;  // "étude" => "Étude"
$caseMapping[strtolower($tag2)] = $tag2;  // "étude" => "étude" （覆盖）
// 结果：$caseMapping["étude"] = "étude"，正确合并 ✓
// 但如果是希腊语 Σ
$tag3 = 'Σύνταξη';
$caseMapping[strtolower($tag3)] = $tag3;  // "Σύνταξη" => "Σύνταξη" （Σ 不变）
$tag4 = 'σύνταξη';
$caseMapping[strtolower($tag4)] = $tag4;  // "σύνταξη" => "σύνταξη"
// 结果：两个条目，未合并 ❌
```

**预期输出**：
```
int(1)
int(1)
```

---

### 11.5 三路搜索与 bookmarksCountPerTag 完整协作

#### filterFulltext 三路搜索执行流程图：

```
用户输入: searchterm='hello "world code" -php
    │
    ▼ mb_convert_case(..., MB_CASE_LOWER, 'UTF-8')
    │  → 'hello "world code" -php'
    │
    ├─ 精确短语提取: /"([^"]+)"/
    │   preg_match_all → $exactSearch = ['world code']
    │
    ├─ 剩余部分: preg_replace 去掉精确短语 → 'hello  -php'
    │   explode(' ') → ['hello', '', '-php']
    │
    ├─ 分离 AND / 排除:
    │   $andSearch = ['hello']
    │   $excludeSearch = ['php']
    │
    ▼ 遍历书签:
    │
    ├─ buildFullTextSearchableLink:
    │   title\description\url\tags 全部小写化
    │
    ├─ 精确搜索: mb_strpos($content, 'world code')
    │   返回位置 → 记录 $foundPositions
    │
    ├─ AND 搜索: mb_strpos($content, 'hello')
    │   返回位置 → 记录 $foundPositions
    │
    └─ 排除搜索: strpos($content, 'php') === false
    │
    └─ 全部命中 → 设置 search_highlight → 加入结果
```

#### bookmarksCountPerTag 完整流程图：

```
调用: bookmarksCountPerTag(['linux'], 'all')
    │
    ▼ $this->search(['searchtags' => ['linux']], 'all')
    │   → 过滤出含 linux 标签的书签
    │
    ▼ 遍历书签的标签:
    │
    ├─ 过滤条件:
    │   ├─ 空标签 → 跳过
    │   ├─ 未登录 + . 开头 → 跳过
    │   ├─ nomarkdown → 跳过
    │   └─ 搜索过滤标签本身（即 linux) → 跳过
    │
    ├─ 大小写合并:
    │   $key = strtolower($tag)
    │   if (!isset($caseMapping[$key])) {
    │       $caseMapping[$key] = $tag;  // 首次遇到的写法
    │       $tags[$caseMapping[$key]] = 0;
    │   }
    │   $tags[$caseMapping[$key]]++;
    │
    ▼ 排序:
        array_multisort($tags, SORT_DESC, $tmpTags, SORT_ASC)
        → 按计数降序，同计数按字母升序
```

#### 关键交互：

| 场景 | 说明 |
|-----|------|
| 搜索过滤标签本身被排除 | 如果搜索 `linux`，标签云不会显示 `linux` 标签计数 |
| 首次写法优先 | 先遇到 `Étude`，后遇到 `étude`，显示 `Étude` |
| Unicode 大小写不合并 | 希腊语 `Σ` 和 `σ` 被当作不同标签 |
| 隐藏标签过滤 | 未登录时 `.` 开头标签不显示 |

---

### 11.6 代码缺陷汇总与修复建议

| 缺陷 | 位置 | 严重程度 | 修复建议 |
|-----|------|---------|---------|
| `~.hidden` 隐私旁路 | filterTags 第346-354行 | 🔴 高 | OR 标签 `substr` 后再次检查 `.` 前缀过滤 |
| `strtolower` Unicode 大小写 | bookmarksCountPerTag 第341行 | 🟡 中 | 改用 `mb_convert_case($tag, MB_CASE_LOWER, 'UTF-8') |
| `strpos` vs `mb_strpos` 不一致 | filterFulltext 第289行 | 🟢 低 | 统一使用 `mb_strpos` |
| OR 路径不剥 `+`/`-` 前缀 | filterTags 第347行 | 🟡 中 | OR 提取后调用与 AND 一致的前缀剥离 |
| 单 `-+bar` 语义反直觉 | tag2regex 第489-496行 | 🟡 中 | 明确文档或统一前缀剥离顺序 |

#### 隐私旁路修复代码示例：

```php
// 修复前:
$orTags = array_filter(array_map(function ($tag) {
    return startsWith($tag, '~') ? substr($tag, 1) : null;
}, $inputTags));

// 修复后:
$orTags = array_filter(array_map(function ($tag) use ($visibility) {
    if (!startsWith($tag, '~')) {
        return null;
    }
    $orTag = substr($tag, 1);
    // public 可见性下二次过滤隐藏标签
    if ($visibility === self::$PUBLIC && startsWith($orTag, '.')) {
        return null;
    }
    return $orTag;
}, $inputTags));
```

#### Unicode 大小写修复：

```php
// 修复前:
if (!isset($caseMapping[strtolower($tag)])) {
    $caseMapping[strtolower($tag)] = $tag;
}

// 修复后:
$lowerTag = mb_convert_case($tag, MB_CASE_LOWER, 'UTF-8');
if (!isset($caseMapping[$lowerTag])) {
    $caseMapping[$lowerTag] = $tag;
}
```
