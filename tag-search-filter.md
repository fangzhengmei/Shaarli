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
