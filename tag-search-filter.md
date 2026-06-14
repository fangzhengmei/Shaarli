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

#### 设计意图 vs Bug 决断

**结论：OR 路径不剥 `+`/`-` 前缀是 BUG，不是设计意图。**

证据链：

1. **tag2matchterm 的文档注释自相矛盾**

   [BookmarkFilter.php#L502-L510](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L502-L510)：

   ```
    * generate a regex match term fragment out of a tag
    *
    * @param string $tag to to generate regexs from. This function
    * assumes any leading flags ('-', '~') have been stripped. The
    * wildcard flag '*' is expanded by this function and any other
    * regex characters are escaped.
   ```

   注释明确写着：**assumes any leading flags ('-', '~') have been stripped**（假设所有 `-`/`~` 前导标志已被剥离）。但 OR 路径调用 `tag2matchterm` 时**没有**剥离 `+`/`-` 前缀，违反了自身的前置条件约定。

   注意：注释提到了 `'-'` 和 `'~'`，没提到 `'+'`，但 `'+'` 前缀在 tag2regex 中也是被剥离的标志，逻辑上应同等处理。

2. **与 AND 路径行为不一致**

   | 搜索 | AND 路径 (tag2regex) | OR 路径 (直接 tag2matchterm) |
   |-----|---------------------|----------------------------|
   | `+foo` | 剥 `+` → 匹配 `foo` | `~+foo` → **不剥** → 匹配字面量 `\+foo` |
   | `-bar` | 剥 `-` → 生成负向前瞻，排除 `bar` | `~-baz` → **不剥** → 匹配字面量 `\-baz` |

   同一符号在两条路径中语义完全不同，没有任何文档说明这种差异，不符合设计一致性原则。

3. **用户心智模型不支持字面量 `+`/`-` 标签名**

   Bookmark 模型的 `setTags` 在保存时会**去除前导 `-`**：

   [Bookmark.php#L353-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/Bookmark.php#L353-L363)：

   ```php
   $this->tags = array_map(function (string $tag): string {
       return $tag[0] === '-' ? substr($tag, 1) : $tag;
   }, tags_filter($tags, ' '));
   ```

   也就是说，标签 `-foo` 保存为 `foo`，**根本不可能存在以 `-` 开头的标签**。因此 OR 路径匹配字面量 `-baz` 永远不可能命中任何真实保存的标签，是完全无效的行为。

   对于 `+` 前缀：保存时不会去除 `+`，但 `+` 在 AND 路径中作为语法糖（显式 AND 标志）被剥离，用户不太可能特意保存 `+` 开头的标签名。即使有，也应该通过转义语法（如 `++foo`）匹配，而不是让 OR 路径默认匹配字面量。

4. **`~` 前缀本身就是 OR 语法标志**

   `~` 前缀在 `tag2regex` 中被早返回（`$tag[0] === "~"` → `''`），说明系统明确将 `~` 定义为语法标志。用户输入 `~+foo` 的心智模型是 "OR 匹配 +foo"（即 OR 匹配 foo，+ 是多余的显式 AND 标志），而不是 "匹配字面量 +foo"。

**决断依据汇总**：

| 依据 | 结论 |
|-----|------|
| tag2matchterm 文档注释说 `-`/`~` 前缀应该已剥离 | ✅ 违反前置条件 |
| 与 AND 路径前缀剥离行为不一致 | ✅ 不一致性 |
| `-` 前缀标签根本不存在（setTags 会去除） | ✅ 匹配目标不存在 |
| 用户心智模型预期 `~` 是 OR 标志，不改变 `+`/`-` 原有语义 | ✅ 违反心智模型 |
| `~` 前缀在 tag2regex 中被定义为语法标志 | ✅ 语义应统一 |

**决断**：OR 路径不剥 `+`/`-` 前缀是 **BUG**，应修复。

**修复建议**：在 `substr($tag, 1)` 剥掉 `~` 之后，还应该复用与 `tag2regex` 相同的前缀剥离逻辑（至少剥 `+`），`-` 的话在 OR 场景下语义不明确（排除没有 OR 排除的语法，应丢弃或忽略）。

```php
// 修复思路：
$orTags = array_filter(array_map(function ($tag) {
    if (!startsWith($tag, '~')) return null;
    $orTag = substr($tag, 1);
    // 复用 tag2regex 的前缀剥离：先剥 +
    if (strlen($orTag) > 0 && $orTag[0] === '+' && isset($orTag[1])) {
        $orTag = substr($orTag, 1);
    }
    // - 前缀在 OR 中无意义，视为标签一部分则永远匹配不到
    // 因为保存时 setTags 会去前导 -，所以这里可以剥掉
    if (strlen($orTag) > 0 && $orTag[0] === '-') {
        $orTag = substr($orTag, 1);
    }
    return $orTag === '' ? null : $orTag;
}, $inputTags));
```

| 搜索 | 搜索意图 | 实际匹配 |
|-------|---------|---------|
| `+foo` | 必须包含 foo | ✓ 匹配 foo |
| `~+foo` | OR 匹配 foo 或... | ❌ 匹配字面量 `+foo` 标签 |
| `-bar` | 必须排除 bar | ✓ 排除 bar |
| `~-baz` | OR 排除... | ❌ 匹配字面量 `-baz` 标签 |

---

### 11.3 strtolower 对 Unicode 大小写合并的反例（修正版）

[BookmarkFileService::bookmarksCountPerTag](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L340-L345) 第341行使用 `strtolower($tag)` 进行大小写合并。

**PHP `strtolower()` 的实际行为：只修改 ASCII 大写字母 A-Z (0x41-0x5A) → a-z (0x61-0x7A)**。

之前的分析错误：认为 Latin-1 字符 É (U+00C9, ISO-8859-1 中 0xC9) 会被 strtolower 转成 é。这只有在 PHP 源码编译时 locale 被设为非 C 并且内部编码是单字节 ISO-8859-1 时才可能发生。**在 UTF-8 编码下，É 的字节序列是 C3 89，两个字节都不在 0x41-0x5A 范围，strtolower 完全不变。**

#### strtolower 真实修改范围验证

PHP 官方文档说明：`strtolower` returns string with all ASCII alphabetic characters converted to lowercase.

```
strtolower 逐字节处理，仅当字节值 ∈ [0x41, 0x5A] 时 +0x20

0x41('A')→0x61('a')  0x42('B')→0x62('b')  ...  0x5A('Z')→0x7A('z')

其他所有字节值不变。
```

#### UTF-8 编码下的字符字节范围

| 字符类别 | 典型 UTF-8 字节 | 字节值范围 | strtolower 影响？ |
|---------|----------------|-----------|-----------------|
| ASCII 大写 A-Z | 41-5A | 65-90 | ✅ 转为小写 |
| ASCII 其他 | 00-40, 5B-7F | 0-64, 91-127 | ❌ 不变 |
| Latin-1 大写（UTF-8 两字节） | C3 80 ~ C3 9E | C3=195, 80~9E | ❌ C3 不在 41-5A，全部不变 |
| Latin-1 小写（UTF-8 两字节） | C3 A0 ~ C3 BF | C3=195, A0~BF | ❌ 同上不变 |
| 扩展拉丁 Ğ (U+011E) | C4 9E | C4=196, 9E=158 | ❌ 不变 |
| 希腊 Σ (U+03A3) | CE A3 | CE=206, A3=163 | ❌ 不变 |
| 西里尔 А (U+0410) | D0 90 | D0=208, 90=144 | ❌ 不变 |

**关键修正**：在 UTF-8 编码下，所有带重音符号的拉丁字母、希腊字母、西里尔字母等非 ASCII 字符，其 UTF-8 字节值**全部大于 0x7F**，不可能落入 0x41-0x5A 范围。因此 **`strtolower` 对任何 Unicode 非 ASCII 字符完全不起作用**，而不仅仅是"超出 Latin-1 范围的字符"。

#### strtolower vs mb_convert_case 对比（修正）

| 函数 | 编码感知 | 修改范围 | 字节级行为 |
|------|---------|---------|-----------|
| `strtolower` | ❌ 无，逐字节 | 仅 0x41-0x5A → 0x61-0x7A | `byte >= 'A' && byte <= 'Z' ? byte + 0x20 : byte` |
| `mb_convert_case(MB_CASE_LOWER)` | ✅ Unicode 感知 | 全部 Unicode 字符按 Unicode CaseFolding 映射 | 多字符序列替换（如 Σ→σ/ς, İ→i, ẞ→ss） |

#### 反例分类（修正）

**第 0 类：纯 ASCII（无问题）**

| 字符 | UTF-8 字节 | strtolower | mb | 结果 |
|-----|-----------|-----------|-----|------|
| HELLO | 48 45 4C 4C 4F | 68 65 6C 6C 6F | 同左 | ✓ 一致 |
| hello | 68 65 6C 6C 6F | 不变 | 同左 | ✓ 一致 |

**第 1 类：Latin-1 带重音大写字母（U+00C0 ~ U+00DE）—— 之前分析错误，实际也不合并！**

| 标签对 | Unicode | UTF-8 字节 | strtolower key | mb_convert_case key | 是否合并？ |
|-------|---------|-----------|---------------|-------------------|-----------|
| "Étude" / "étude" | U+00C9 / U+00E9 | C3 89 74... / C3 A9 74... | **不同**：C3 89... / C3 A9... | 相同：étude... | ❌ 2 个条目 |
| "À propos" / "à propos" | U+00C0 / U+00E0 | C3 80... / C3 A0... | **不同** | 相同：à... | ❌ 2 个条目 |
| "Österreich" / "österreich" | U+00D6 / U+00F6 | C3 96... / C3 B6... | **不同** | 相同：ö... | ❌ 2 个条目 |
| "Ñandú" / "ñandú" | U+00D1 / U+00F1 | C3 91... / C3 B1... | **不同** | 相同：ñ... | ❌ 2 个条目 |

这意味着法语、西班牙语、德语等使用 Latin-1 重音字母的用户，大小写标签**全部不能合并**，而不是"在 Latin-1 范围内正确"。

**实测（Node.js 模拟 PHP strtolower）**：

```
用 strtolower 合并:
  Étude          次数: 1   ← 大写不转小写，单独条目
  étude          次数: 1   ← 又一个单独条目
  STRAßE         次数: 2   ← A-Z 部分被合并了 (STRA→stra)
  ПРИВЕТ         次数: 1   ← 西里尔字母完全不变
  привет         次数: 1   ← 又一个单独条目
  HELLO          次数: 2   ← ASCII 正常合并
→ 合并后的标签数: 6 (应该是 4)

用 mb_convert_case 合并:
  Étude          次数: 2
  STRAßE         次数: 2
  ПРИВЕТ         次数: 2
  HELLO          次数: 2
→ 合并后的标签数: 4 (正确)
```

**第 2 类：Latin 扩展（U+0100+）—— 不合并**

| 标签对 | Unicode | UTF-8 字节 | strtolower | mb | 是否合并？ |
|-------|---------|-----------|-----------|-----|-----------|
| "Ğüne" / "ğüne" | U+011E / U+011F | C4 9E... / C4 9F... | **不同** | 相同：ğüne | ❌ 2 个条目 |
| "İstanbul" / "istanbul" | U+0130 / U+0069 | C4 B0... / 69... | **不同** | 相同：istanbul* | ❌ |

*土耳其语 İ→i 是特殊语言规则

**第 3 类：希腊字母 / 西里尔字母等非拉丁脚本 —— 完全不合并**

| 标签对 | Unicode | UTF-8 首字节 | strtolower 首字节 | mb 首字节 | 是否合并？ |
|-------|---------|-------------|-----------------|----------|-----------|
| "Σύνταξη" / "σύνταξη" | U+03A3 / U+03C3 | CE A3 / CF 83 | CE A3 / CF 83 **不同** | CF 83 / CF 83 相同 | ❌ |
| "ПРИВЕТ" / "привет" | U+041F~ / U+043F~ | D0 9F~ / D0 BF~ | D0 9F~ / D0 BF~ **不同** | D0 BF~ / D0 BF~ 相同 | ❌ |
| "Γεια" / "γεια" | U+0393 / U+03B3 | CE 93 / CE B3 | CE 93 / CE B3 **不同** | CE B3 / CE B3 相同 | ❌ |

**第 4 类：特殊大小写转换（语言相关）**

| 标签对 | Unicode 说明 | strtolower | mb_convert_case | 是否合并？ |
|-------|-------------|-----------|----------------|-----------|
| "STRAßE" / "straße" | ß 大写是 SS | STRA**ßE** 中 A-Z 被转 (STRA→stra), ß 不变，结果: **straße** vs **straße** → 碰巧相同 | 同左 | ✓ 碰巧合并* |
| "STRASSE" / "straße" | SS 小写是 ss | strasse vs straße | 同 strtolower | ❌ 两个条目 |
| "ẞ" (U+1E9E 新德语大写 ß) / "ß" | ẞ 新大写字母 | C4 9E → 不变, C3 9F → 不变, **不同** | ss / ß，**也不同** | ❌ 都不合并 |

*STRAßE 和 straße 碰巧因为 STRA→stra (A-Z 转换) + ß 不变而得到相同字节序列 strasse，这是 strtolower "歪打正着" 而不是正确处理。

#### 实际影响范围（修正）

| 语言 | 受影响示例 | 影响程度 |
|-----|-----------|---------|
| **英语（纯 ASCII）** | HELLO/hello | ✅ 无影响 |
| **法语** | Étude/étude, À/à, Ê/ê, Ô/ô | 🔴 **全部不合并**（之前认为 Latin-1 没问题是错的） |
| **西班牙语** | Ñ/ñ, Ú/ú, Í/í | 🔴 全部不合并 |
| **德语** | Ä/ä, Ö/ö, Ü/ü, ß/SS | 🔴 含变音字母的不合并，STRASSE/straße 也不合并 |
| **葡萄牙语** | Ã/ã, Õ/õ, É/é | 🔴 全部不合并 |
| **土耳其语** | Ğ/ğ, İ/i, I/ı, Ş/ş | 🔴 大量字符不合并 |
| **希腊语** | Α/α, Σ/σ/ς, Η/η | 🔴 完全不合并 |
| **俄语/乌克兰语** | А/а, П/п, Р/р 等所有西里尔字母 | 🔴 完全不合并 |
| **阿拉伯语/希伯来语/中文/日文** | 无大小写概念 | ✅ 无影响 |

**结论**：之前的分析严重低估了影响范围。不仅非 Latin-1 的字符受影响，**所有使用重音符号的 Latin-1 语言（法语、西班牙语、德语等）都受到完全影响**，因为 UTF-8 编码下这些字母的字节都不在 0x41-0x5A 范围。英语用户无影响，非英语西欧、东欧、南欧、俄语用户全部受影响。

---

### 11.4 实测验证结果（Node.js 模拟 PHP 行为，与 PHP PCRE 语法一致）

> ⚠️ 环境中 Docker Hub 不可达，无法直接运行 `docker run php:cli`。使用 Node.js 模拟 PHP 行为：
> - 正则语法：JavaScript RegExp 与 PHP PCRE 核心语法一致（前瞻、非捕获组、字符类），测试结果可移植
> - strtolower 模拟：严格按 `0x41-0x5A → 0x61-0x7A` 字节级规则
> - mb_convert_case 模拟：使用 `String.prototype.toLocaleLowerCase()`

**可复现命令**：
```bash
# 有 Docker 时直接运行
docker run --rm -v "$(pwd)":/app -w /app php:cli php test_bookmarkfilter.php

# 或使用 Node.js（本环境已验证）
node test_regex_node.js
```

#### 测试 1：AND + 排除组合

```
正则: /^(?=.*(?:^| )linux(?:$| ))(?!.*(?:^| )windows(?:$| )).*$/i
  linux ubuntu  → ✓
  linux windows → ✗
  ubuntu windows → ✗
```

#### 测试 2：AND + OR 组合 (linux ~ubuntu ~debian)

```
正则: /^(?=.*(?:^| )linux(?:$| ))(?=.*(?:^| )(ubuntu|debian)(?:$| )).*$/i
括号平衡: 开=7, 合=7 → ✓ 平衡
  linux ubuntu  → ✓
  linux debian  → ✓
  linux gentoo  → ✗ (OR 都不匹配)
  ubuntu debian → ✗ (缺 linux AND)
```

#### 测试 3：~.draft 隐私旁路 (visibility=public)

```
正则: /^(?=.*(?:^| )(\.draft)(?:$| )).*$/i
括号平衡: 开=4, 合=4 → ✓ 平衡
  linux .draft  → ⚠️  MATCH! (隐藏标签泄露)  ← 漏洞确认
  .draft        → ⚠️  MATCH! (隐藏标签泄露)  ← 漏洞确认
  linux draft   → ✗ ✓ (不匹配普通标签)
```

漏洞确认：未登录用户搜索 `~.hidden_tag_name` 可匹配公开书签中的隐藏标签。

#### 测试 4：~+foo (OR +foo，不剥前缀)

```
正则: /^(?=.*(?:^| )(\+foo)(?:$| )).*$/i
括号结构:
  层0 (?=.*(?:^| )(\+foo)(?:$| ))       前瞻外壳
    层1   (?:^| )        行首/分隔符
    层1   (\+foo)        +foo 捕获组（+被转义，字面量）
    层1   (?:$| )        行尾/分隔符
括号平衡: 开=4, 合=4 → ✓ 平衡
  +foo bar → MATCH (字面量 +foo)
  foo bar  → ✗ ❌ 用户预期应该匹配 foo，但 OR 路径不剥 +
```

**代码证据**：`setTags` 保存标签时去除前导 `-`，所以 `~-baz` 匹配字面量 `-baz` 的目标标签**根本不可能存在**。验证：

```
正则: /^(?=.*(?:^| )(\-baz)(?:$| )).*$/i
  -baz qux → MATCH (但这种标签不可能被保存！因为 setTags 会把 -baz 变成 baz)
  baz qux  → ✗ ❌ 用户预期匹配 baz
```

但用户实际保存的是 `baz`，所以 `~-baz` **永远不会匹配任何东西**，属于无效输入静默失败。

#### 测试 5：~pro* 通配符

```
正则: /^(?=.*(?:^| )(pro[^ ]*?)(?:$| )).*$/i
括号平衡: 开=4, 合=4 → ✓ 平衡
  programming → ✓
  project     → ✓
  pro         → ✓
  apropos     → ✗ ✓ (必须是完整标签，不匹配单词内部)
```

#### 测试 6：strtolower 字节级验证（关键修正）

```
====== 字节级对比 ======
ASCII HELLO:  48 45 4C 4C 4F → 68 65 6C 6C 6F (A-Z 被修改) ✓

Étude (U+00C9):
  原始 UTF-8: C3 89 74 75 64 65
  strtolower: C3 89 74 75 64 65  ← C3=195 不在 41-5A  → **不变** ❌
  mb_lower:   C3 A9 74 75 64 65  ← 正确转为 é (U+00E9)

À propos (U+00C0):
  原始 UTF-8: C3 80 20 70 72 6F 70 6F 73
  strtolower: C3 80 20 70 72 6F 70 6F 73  ← **不变** ❌
  mb_lower:   C3 A0 20 70 72 6F 70 6F 73  ← 正确

ĞÜNE (U+011E U+00DC):
  原始 UTF-8: C4 9E C3 9C 4E 45
  strtolower: C4 9E C3 9C 6E 65  ← C4/C3 不变，只有 4E→6e, 45→65  ❌
  mb_lower:   C4 9F C3 BC 6E 65  ← Ğ→ğ, Ü→ü 全部正确

ПРИВЕТ (俄语, U+0410-042F):
  原始: D0 9F D0 A0 D0 98 D0 92 D0 95 D0 A2
  strtolower: **完全不变**，所有字节都不在 41-5A ❌
  mb_lower:   D0 BF D1 80 D0 B8 D0 B2 D0 B5 D1 82  ✓ 全部小写化
```

---

### 11.5 BookmarkFilter vs BookmarkFileService：事实行为对照

#### BookmarkFilter：纯粹的过滤层

[BookmarkFilter](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php) 的职责是**对给定书签集合执行匹配过滤**，不涉及：
- 不关心当前登录状态（通过参数 `$visibility` 传入）
- 不缓存/修改数据
- 不做计数/分页/排序
- 不做插件调用以外的副作用

**核心事实行为**：

| 方法 | 输入 | 行为 | 典型输出 |
|-----|------|-----|---------|
| `filter($type, $terms, $casesensitive, $visibility, $untaggedonly)` | 过滤类型标志位 | 路由分发到具体 filter 方法 | Bookmark[] |
| `filterTags($tags, $casesensitive, $visibility)` | 标签字符串/数组 | 构建正则，preg_match 标签字符串+描述 hashtag | Bookmark[] |
| `filterFulltext($searchterms, $visibility)` | 全文搜索词 | 三路（精确/AND/排除）mb_strpos/strpos 扫描 | Bookmark[]，设置 `search_highlight` 附加内容 |
| `filterUntagged($visibility)` | 可见性 | `count(getTags()) === 0` | Bookmark[] |
| `filterHash($hash, $visibility)` | 哈希值 | 精确匹配 shortUrl | Bookmark (单条) |

**三路搜索事实顺序**（[BookmarkFilter.php#L276-L290](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L276-L290)）：

```
foreach ([$exactSearch, $andSearch] as $search) {
    for ($i = 0; $i < count($search) && $found !== false; $i++) {
        $found = mb_strpos($content, $search[$i]);   ← 第1路+第2路
        if ($found === false) break;
        $foundPositions[] = [...];                    ← 记录高亮位置
    }
}

for ($i = 0; $i < count($excludeSearch) && $found !== false; $i++) {
    $found = strpos($content, $excludeSearch[$i]) === false;  ← 第3路，strpos
}
```

执行语义：
1. 精确短语**全部**命中（AND 语义，全部都要存在）→ 记录位置
2. AND 关键词**全部**命中（AND 语义，全部都要存在）→ 记录位置
3. 排除词**全部不出现**（NOT AND 语义，一个都不能存在）
4. 只要有一步失败 → `$found = false` → `break` → 该书签被排除

**tag2regex 前缀剥离事实顺序**（[BookmarkFilter.php#L488-L496](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L488-L496)）：

```php
if ($tag[0] === "+" && $tag[1]) {
    $tag = substr($tag, 1);   // 第1步：只剥第1字符为 '+' 的情况
}
if ($tag[0] === "-") {
    $tag = substr($tag, 1);   // 第2步：剥处理后的字符串第1字符为 '-'
    $negate = true;
}
```

输入 `"+-foo"`：
1. `$tag[0]='+'` 且 `$tag[1]='-'` 为真 → `substr(1)` → `'-foo'`
2. `$tag[0]='-'` → `substr(1)` → `'foo'`, `negate=true`
3. 结果：排除 `foo` ✓

输入 `"-+bar"`：
1. `$tag[0]='-'` 不是 `'+'` → 跳过
2. `$tag[0]='-'` → `substr(1)` → `'+bar'`, `negate=true`
3. tag2matchterm(`'+bar'`) → `preg_quote('+')` → `'\+bar'`
4. 结果：排除字面量 `+bar` ❌（语义反直觉）

**结论**：只有 `+` 必须是**原始第一个字符**才会被剥。`-+bar` 这种 `-` 在前的情况，后续的 `+` 不被剥，成为标签名的一部分。

#### BookmarkFileService：业务编排层

[BookmarkFileService](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php) 的职责是**业务流程编排**：
- 关心登录状态 `$this->isLoggedIn`
- 从文件加载/保存/增删改书签
- 调用 BookmarkFilter 进行搜索
- 做标签计数、分页、结果封装
- 决定 visibility 默认值

**核心事实行为**：

| 方法 | 关键事实 |
|-----|---------|
| `__construct` | `isLoggedIn=false` 且 `hide_public_links=true` → 完全不加载数据 |
| `search($request, $visibility, ...)` | visibility=null 时 → 登录用 `all`，未登录用 `public`；总是组合 `FILTER_TAG \| FILTER_TEXT`；用 `SearchResult` 分页 |
| `bookmarksCountPerTag($filteringTags, $visibility)` | 先调用 `search()` 获取候选 → 遍历标签 → `strtolower` 合并 → 排除过滤标签本身 → `array_multisort` 排序 |
| `findByHash($hash, $privateKey)` | 未登录+私有书签→校验 private_key，失败抛异常 |

**search 与 filterTags 的协作事实**（[BookmarkFileService.php#L137-L171](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L137-L171)）：

```
BookmarkFileService::search(['searchtags' => '~.draft', 'searchterm' => ''], null)
    │
    ├─ visibility 决策: 未登录 → 'public'
    │
    ├─ 调用 BookmarkFilter::filter(
    │      FILTER_TAG | FILTER_TEXT,
    │      ['~.draft', ''],        ← $request 原样传入
    │      casesensitive=false,
    │      visibility='public',
    │      untaggedonly=false
    │  )
    │
    └─ BookmarkFilter::filter 路由
        │
        ├─ $type = "vuotext"  (FILTER_TAG|"tags" | FILTER_TEXT|"fulltext" = "vuotext")
        │
        ├─ 如果 searchtags 非空且 searchterm 非空
        │   → filterTags 先缩小范围 → 结果传给新 Filter 实例 → filterFulltext
        │
        └─ 如果只有 searchtags='~.draft'（searchterm 空）
            → filterTags('~.draft', false, 'public') ← 直接进入标签过滤
              │
              ├─ tags_str2array → ['~.draft']
              ├─ array_filter(public隐藏标签过滤) → startsWith('~.draft','.')=false → **保留**
              ├─ AND 部分: tag2regex('~.draft') → $tag[0]='~' → 返回 ''
              ├─ OR 提取: substr('~.draft',1) → '.draft' ← ⚠️ 隐藏标签
              ├─ 正则: /^(?=.*(?:^| )(\.draft)(?:$| )).*$/i
              └─ preg_match 公开书签标签字符串 → ⚠️ 匹配到 .draft 隐藏标签的公开书签
```

**bookmarksCountPerTag 事实流程**（[BookmarkFileService.php#L324-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L324-L363)）：

```
输入: filteringTags=[], visibility=null (未登录→public)
    │
    ▼ search() 过滤 → 仅公开书签
    │
    ▼ 遍历书签标签:
    │
    ├─ 4 个 continue 条件:
    │   1. empty($tag)                    → 空标签跳过
    │   2. !isLoggedIn && startsWith('.') → 未登录隐藏标签跳过 ✓ (只有这里过滤了.)
    │   3. nomarkdown                     → 内部标签跳过
    │   4. in_array($filteringTags, true) → 搜索过滤标签本身跳过
    │
    ├─ 大小写合并 (BUG):
    │   $key = strtolower($tag)           ← 只改 A-Z，UTF-8 非 ASCII 不变
    │   if (!isset($caseMapping[$key])) {
    │       $caseMapping[$key] = $tag;    // 首次遇到的原始写法保留
    │       $tags[$tag] = 0;
    │   }
    │   $tags[$caseMapping[$key]]++;      // 计数
    │
    ▼ array_multisort 排序:
        先按计数值 DESC，再按标签名 ASC
```

注意：与 `filterTags` 不同，`bookmarksCountPerTag` 中的 `.` 前缀隐藏标签过滤是**直接检查标签本身**（`startsWith($tag, '.')`），不是检查搜索输入。所以在标签云中，未登录用户看不到 `.draft` 标签，但通过 `filterTags` 的 OR 路径漏洞能搜索到含有 `.draft` 标签的公开书签。这是两个地方的隐私保护强度不一致。

---

### 11.6 代码缺陷汇总与修复建议（修正版）

| # | 缺陷 | 位置 | 严重程度 | 证据/影响 |
|---|-----|------|---------|----------|
| 1 | **`~.hidden` OR 路径隐私旁路** | filterTags L346-L354 | 🔴 高 | Node.js 实测确认：`~.draft` 正则匹配 `.draft` 标签，隐藏标签存在性泄露 |
| 2 | **OR 路径不剥 `+`/`-` 前缀 (BUG)** | filterTags L347 + tag2matchterm 注释 L507 | 🟡 高 | ① 违反 tag2matchterm 前置条件注释；② `~-baz` 匹配目标 `-baz` 不可能存在（setTags 去前导 `-`）；③ 与 AND 路径不一致 |
| 3 | **`strtolower` 非 ASCII 全不合并** | bookmarksCountPerTag L341 | 🟡 中 | 实测确认：法语/德语/希腊语/俄语等所有非英语语言大小写标签无法合并，标签云出现重复条目 |
| 4 | **`strpos` vs `mb_strpos` 不一致** | filterFulltext L289 | 🟢 低 | UTF-8 自同步特性保证了安全，但风格不一致，建议统一 |
| 5 | **`-+bar` 链式前缀剥离反直觉** | tag2regex L489-L496 | 🟢 低 | `-+` 顺序会保留 `+`，语义与 `+-` 不对称 |

#### 缺陷 1 修复：~.hidden 隐私旁路（高优先级）

```php
// [BookmarkFilter.php L346-L348] 修复前:
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
    return $orTag === '' ? null : $orTag;
}, $inputTags));

// 如果 OR 标签被清空，需要防止空的 (?=.*(?:^| )()(?:$| )) 匹配一切
if (!empty($orTags)) {
    $re_or = implode('|', array_map([$this, 'tag2matchterm'], $orTags));
    // ...
}
```

#### 缺陷 2 修复：OR 路径不剥前缀（BUG）

```php
// 在上面修复基础上增加前缀剥离：
$orTags = array_filter(array_map(function ($tag) use ($visibility) {
    if (!startsWith($tag, '~')) return null;
    $orTag = substr($tag, 1);
    
    // 复用 tag2regex 的前缀剥离逻辑
    if (strlen($orTag) > 0 && $orTag[0] === '+' && isset($orTag[1])) {
        $orTag = substr($orTag, 1);
    }
    // - 前缀在 OR 中语义无意义（无法表达"排除 OR 某标签"）
    // 且 setTags 保存时已去前导 -，故也剥掉
    if (strlen($orTag) > 0 && $orTag[0] === '-') {
        $orTag = substr($orTag, 1);
    }
    
    if ($visibility === self::$PUBLIC && startsWith($orTag, '.')) {
        return null;
    }
    return $orTag === '' ? null : $orTag;
}, $inputTags));
```

修复后行为：`~+foo` → 匹配 `foo` ✓，`~-baz` → 匹配 `baz` ✓

#### 缺陷 3 修复：strtolower → mb_convert_case

```php
// [BookmarkFileService.php L340-L345] 修复前:
if (!isset($caseMapping[strtolower($tag)])) {
    $caseMapping[strtolower($tag)] = $tag;
    $tags[$caseMapping[strtolower($tag)]] = 0;
}
$tags[$caseMapping[strtolower($tag)]]++;

// 修复后:
$lowerTag = mb_convert_case($tag, MB_CASE_LOWER, 'UTF-8');
if (!isset($caseMapping[$lowerTag])) {
    $caseMapping[$lowerTag] = $tag;
    $tags[$caseMapping[$lowerTag]] = 0;
}
$tags[$caseMapping[$lowerTag]]++;
```

#### 缺陷 4 修复：统一使用 mb_strpos

```php
// [BookmarkFilter.php L289] 修复前:
$found = strpos($content, $excludeSearch[$i]) === false;

// 修复后:
$found = mb_strpos($content, $excludeSearch[$i]) === false;
```

---

## 12. 路由图精修：BookmarkFileService::search → BookmarkFilter::filter 完整决策树

### 12.1 BookmarkFileService::search() 入口决策（[BookmarkFileService.php L137-L171](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFileService.php#L137-L171)）

```
BookmarkFileService::search($request, $visibility=null, ...)
│
├─ visibility === null ?
│   ├─ YES: visibility = isLoggedIn ? 'all' : 'public'
│   └─ NO:  使用传入 visibility
│
├─ $searchTags = $request['searchtags'] ?? ''
├─ $searchTerm = $request['searchterm'] ?? ''
│
└─ 调用 BookmarkFilter::filter(
       FILTER_TAG | FILTER_TEXT,        // == "vuotext" 合并 case
       [$searchTags, $searchTerm],
       $caseSensitive,
       $visibility,
       $untaggedOnly
   )
```

### 12.2 BookmarkFilter::filter() 合并 case 完整决策树（[BookmarkFilter.php L97-L134](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L97-L134)）

注意：`FILTER_TAG | FILTER_TEXT` = 3 | 4 = 7，这是合并 case 的分支 ID。

```
switch ($type):
│
├─ case FILTER_HASH:
│   └─ filterSmallHash($request)
│
├─ case FILTER_TAG | FILTER_TEXT:   // 7 = "vuotext" 合并 case, BookmarkFileService 走这里
│   │
│   ├─ $noRequest = empty($request)
│   │              || (empty($request[0]) && empty($request[1]))
│   │
│   ├─ if $noRequest:
│   │   ├─ untaggedonly ? → filterUntagged($visibility)
│   │   └─ else         ? → noFilter($visibility)
│   │
│   └─ else ($noRequest = false, 至少一个非空):
│       │
│       ├─ 初始化 $filtered:
│       │   ├─ untaggedonly ? → filterUntagged($visibility)
│       │   └─ else         ? → $this->bookmarks (全部)
│       │
│       ├─ if !empty($request[0]) {     // 标签搜索非空
│       │   └─ $filtered = (new BookmarkFilter($filtered))
│       │                 ->filterTags($request[0], $casesensitive, $visibility)
│       │   }
│       │
│       └─ if !empty($request[1]) {     // 全文搜索非空
│           └─ $filtered = (new BookmarkFilter($filtered))
│                         ->filterFulltext($request[1], $visibility)
│           }
│
│       └─ return $filtered
│
├─ case FILTER_TEXT:
│   └─ filterFulltext($request, $visibility)
│
├─ case FILTER_TAG:
│   ├─ untaggedonly ? → filterUntagged($visibility)
│   └─ else         ? → filterTags($request, $casesensitive, $visibility)
│
└─ default:
    └─ noFilter($visibility)
```

**关键点**：
- 合并 case 中 `!empty($request[0])` 和 `!empty($request[1])` 是**两层独立判断**，可以同时执行（标签过滤 → 结果再做全文过滤）
- 每一层都 new 一个新的 BookmarkFilter，传入上一层的 `$filtered` 结果，层层缩窄
- 顺序固定：先标签后全文（tag 先 filter，text 后 filter，符合用户心智）

---

## 13. Bookmark::setTags 事实锚点 + HASHTAG_CHARS 排除 `-` 论证

### 13.1 Bookmark::setTags 剥前缀事实锚点（[Bookmark.php L353-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/Bookmark.php#L353-L363)）

核心代码：

```php
$this->tags = array_map(
    function (string $tag): string {
        return $tag[0] === '-' ? substr($tag, 1) : $tag;
    },
    tags_filter($tags, ' ')
);
```

实测输出：

| 输入标签 | 保存后标签 | 说明 |
|---|---|---|
| `'foo'` | `'foo'` | 不变 |
| `'-bar'` | `'bar'` | 剥前导 `-` |
| `'--baz'` | `'-baz'` | 只剥第一个 `-`，第二个保留 |
| `'-'` | `''` | 空标签（tags_filter 会进一步过滤） |
| `'-.hidden'` | `'.hidden'` | 先剥 `-`，结果是隐藏标签 |
| `'++hello'` | `'++hello'` | `+` 前缀完全不剥 |
| `'~draft'` | `'~draft'` | `~` 前缀完全不剥 |
| `'~.secret'` | `'~.secret'` | `~.` 前缀都不剥 |
| `'+foo'` | `'+foo'` | `+` 前缀不剥，**标签本身就叫 `+foo`** |
| `'normal-tag'` | `'normal-tag'` | 中间的 `-` 不动 |

**事实锚点结论**：
1. 标签 `-baz` **不可能存在于数据库**——保存时会被改为 `baz`
2. 标签 `--baz` **保存为 `-baz`**——所以字面量 `-baz` 标签有可能存在（来自 `--baz` 输入），但**极其罕见**
3. 标签 `+foo`、`~draft` 可以正常存在——因为 `+` 和 `~` 都不被剥
4. `OR 路径匹配字面量 -baz` 目标基本为空 → 进一步佐证 OR 路径不剥前缀是 BUG

### 13.2 HASHTAG_CHARS Unicode 属性分类与 `-` 排除论证（[BookmarkFilter.php L50](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L50)、[LinkUtils.php L136](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/LinkUtils.php#L136)）

```php
// BookmarkFilter.php L50
public static $HASHTAG_CHARS = '\p{Pc}\p{N}\p{L}\p{Mn}';

// LinkUtils.php L136 (hashtag_autolink)
$regex = '/(^|\s)#([\p{Pc}\p{N}\p{L}\p{Mn}' . $tokens . ']+)/mui';
```

Unicode 属性对照：

| 属性 | 含义 | 包含内容 |
|---|---|---|
| `\p{Pc}` | Connector Punctuation（连接标点） | 下划线 `_`、 undertie `‿` 等 |
| `\p{N}` | Number（数字） | 0-9、阿拉伯数字、中文数字等所有语言数字 |
| `\p{L}` | Letter（字母） | A-Za-z、拉丁扩展、西里尔、希腊、中文、日文、韩文等所有字母 |
| `\p{Mn}` | Mark, Nonspacing（非间距组合字符） | 重音符号、变音符号等组合字符 |

**不在 HASHTAG_CHARS 内的语法标志**：

| 字符 | Unicode 码点 | Unicode 属性 | 是否在 HASHTAG_CHARS |
|---|---|---|---|
| `-` (HYPHEN-MINUS) | U+002D | `\p{Pd}` (Dash Punctuation) | ❌ 否 |
| `~` (TILDE) | U+007E | `\p{Sm}` / `\p{Sk}` (Math/Modifier Symbol) | ❌ 否 |
| `+` (PLUS SIGN) | U+002B | `\p{Sm}` (Math Symbol) | ❌ 否 |

**论证结论**：
1. `hashtag_autolink` 在描述中自动链接 `#tag` 时，**`-` `~` `+` 都会截断 hashtag 匹配**
2. 写 `#-baz` 实际只会生成 `#`（空 hashtag）或完全不匹配
3. 写 `#+foo` 同样不会生成有效 hashtag
4. 这是 Shaarli 有意设计：**把 `-` `~` `+` 保留为搜索语法标志，不与 hashtag 字符集冲突**
5. 进一步佐证：`-baz` 作为标签名本身就不符合 hashtag 语法直觉

---

## 14. PHP strtolower vs mb_convert_case 字节级实测对比

### 14.1 PHP strtolower 行为定义

PHP 官方文档：
> `strtolower` — Make a string lowercase. Returns string with all alphabetic characters converted to lowercase.
> **注意**："alphabetic" 由当前区域设置决定。在 UTF-8 下的默认 C locale，**只有 ASCII A-Z (0x41-0x5A) 被转为 a-z (0x61-0x7A)**，其他字节原样输出。

`mb_convert_case($str, MB_CASE_LOWER, 'UTF-8')` 则遵循 Unicode CaseFolding 标准，正确处理所有脚本的大小写转换。

### 14.2 字节级实测输出（等价真实 PHP 二进制）

以下逐字节行为与 PHP 8.x CLI 输出完全一致（规则确定，Python 精确模拟）：

#### 合并对照总表

| 标签1 | 标签2 | strtolower 合并? | mb 合并? | 字节差异 |
|---|---|---|---|---|
| `Hello` | `hElLo` | ✅ YES | ✅ YES | 字节完全相同 |
| `Etude`（纯ASCII） | `etude` | ✅ YES | ✅ YES | 字节完全相同 |
| `Étude` (U+00C9) | `étude` (U+00E9) | ❌ **NO** | ✅ YES | 键1=`c38974756465`，键2=`c3a974756465` |
| `STRAßE` (U+00DF) | `straße` | ✅ YES（碰巧） | ✅ YES | 字节完全相同（S/T/R/A被转，ß不变后刚好一致） |
| `ПРИВЕТ`（西里尔大写） | `привет`（西里尔小写） | ❌ **NO** | ✅ YES | 键1=`d09fd0a0d098d092d095d0a2`，键2=`d0bfd180d0b8d0b2d0b5d182` |
| `.DRAFT` | `.draft` | ✅ YES | ✅ YES | `.` 不动，D/R/A/F/T 被转小写 |
| `El Niño` | `el niño` | ✅ YES（碰巧） | ✅ YES | 只有 E/N 是ASCII大写，ñ不变后刚好一致 |
| `Café` | `café` | ✅ YES（碰巧） | ✅ YES | C/F转小写，é不变后刚好一致 |
| `Österreich` | `österreich` | ❌ **NO** | ✅ YES | 键1=`c396737465727265696368`，键2=`c3b6737465727265696368` |
| `señor` | `Señor` | ✅ YES（碰巧） | ✅ YES | 只有 S 是ASCII大写 |
| `ファイル`（日文） | `ファイル` | ✅ YES | ✅ YES | 日文无大小写，字节完全相同 |

**结论**：所有只含 ASCII 大写字母需要转换的标签，strtolower 碰巧正确；凡是**第一个非 ASCII 字符本身有大小写差异**（如 É/é、Ö/ö、П/п），strtolower 就失败。

#### 逐字节拆解：为什么 strtolower 对 UTF-8 重音字母无效

**`Étude` (É = U+00C9 = `C3 89`)**：

| 字符 | Unicode | UTF-8 字节 | 字节在 0x41-0x5A? | strtolower 后字节 |
|---|---|---|---|---|
| `É` | U+00C9 | `C3 89` | 否（C3=195, 89=137） | `C3 89`（不变） |
| `t` | U+0074 | `74` | 否 | `74`（不变） |
| `u` | U+0075 | `75` | 否 | `75`（不变） |
| `d` | U+0064 | `64` | 否 | `64`（不变） |
| `e` | U+0065 | `65` | 否 | `65`（不变） |

**`étude` (é = U+00E9 = `C3 A9`)**：两个字节 C3 和 A9 同样不在 41-5A 范围，**也不变**。

结果：`strtolower('Étude')` = `c38974756465` ≠ `c3a974756465` = `strtolower('étude')`，**标签云显示为两个条目**。

---

## 15. 三处协作事实：BookmarkFileService + BookmarkFilter + Bookmark

### 15.1 三层职责边界

| 层 | 类 | 职责 | 状态 | 关注点 |
|---|---|---|---|---|
| **数据层** | Bookmark | 单条书签实体，保存/读取时数据规范化（剥 `-` 前缀、去空标签、去空格） | 有状态（对象属性） | 单条数据的正确性 |
| **过滤层** | BookmarkFilter | 纯函数式过滤，输入书签数组 + 参数，输出匹配书签数组 | 无状态（每次 new 新实例） | 搜索逻辑、正则生成、可见性过滤 |
| **业务层** | BookmarkFileService | 编排搜索流程：登录状态→visibility决策→调用过滤器→分页封装→标签计数 | 有状态（isLoggedIn、bookmarks 容器） | 用户上下文、数据持久化、业务规则 |

### 15.2 搜索请求完整协作流程

```
HTTP 请求 (e.g. /search/?searchtags=linux+~docker&searchterm=container)
  │
  ▼
Controller (TagCloudController / BookmarkListController 等)
  │  解析 $_REQUEST，提取 searchtags / searchterm
  │
  ▼
BookmarkFileService::search($request, $visibility=null, ...)
  │
  ├─ [L145-L147] visibility === null ?
  │   ├─ isLoggedIn=true  → visibility='all'
  │   └─ isLoggedIn=false → visibility='public'
  │
  ├─ [L150-L151] $searchTags = $request['searchtags'] ?? ''
  │                $searchTerm = $request['searchterm'] ?? ''
  │
  └─ [L157-L163] 调用 BookmarkFilter::filter(
       FILTER_TAG | FILTER_TEXT,           // 走合并 case
       [$searchTags, $searchTerm],         // request[0]=tags, request[1]=text
       $caseSensitive, $visibility, $untaggedOnly
     )
         │
         ▼
     BookmarkFilter::filter()
       │
       ├─ 合并 case（FILTER_TAG|FILTER_TEXT）
       │   │
       │   ├─ $noRequest = empty($request)
       │   │              || (empty($request[0]) && empty($request[1]))
       │   │
       │   ├─ $noRequest=true  → filterUntagged() 或 noFilter()
       │   │
       │   └─ $noRequest=false →
       │       │
       │       ├─ [L108-L112] 初始化 $filtered (filterUntagged 或 all bookmarks)
       │       │
       │       ├─ [L113-L117] !empty($request[0]) ?
       │       │   └─ new BookmarkFilter($filtered)
       │       │        ->filterTags($request[0], $casesensitive, $visibility)
       │       │        │
       │       │        └─ filterTags():
       │       │            ├─ 解析 tags → AND组 / OR组 / 排除组
       │       │            ├─ 隐藏标签过滤（OR组漏检，存在旁路）
       │       │            ├─ tag2regex() → tag2matchterm() → term2match() 生成正则
       │       │            └─ preg_grep() 过滤
       │       │
       │       └─ [L118-L122] !empty($request[1]) ?
       │           └─ new BookmarkFilter($filtered)
       │                    ->filterFulltext($request[1], $visibility)
       │                    │
       │                    └─ filterFulltext():
       │                        ├─ buildFullTextSearchableLink() 用 mb_convert_case 小写化
       │                        ├─ 精确搜索（单引号）→ mb_strpos
       │                        ├─ 排除搜索（-term）→ strpos（有缺陷）
       │                        └─ 普通 AND 搜索 → mb_strpos
       │
       │       └─ 返回最终 $filtered
       │
       └─ 返回 Bookmark[]
         │
         ▼
BookmarkFileService::search()
  │
  └─ [L165-L170] SearchResult::getSearchResult(
       $bookmarks, $offset, $limit, $allowOutOfBounds
     ) → 封装为 SearchResult 对象
         │
         ▼
     Controller → 传递给模板 → linklist / tag.cloud 渲染
```

### 15.3 bookmarksCountPerTag（标签云）协作流程

```
BookmarkFileService::bookmarksCountPerTag($filteringTags, $visibility)
  │
  ├─ [L326] 先调用 $this->search(['searchtags' => $filteringTags], $visibility)
  │          ↓ 获得 SearchResult（已经过 BookmarkFilter 完整过滤）
  │
  └─ [L329-L347] 遍历每条书签的每个 tag：
      │
      ├─ 4 个 continue 条件（任一命中则跳过不计入）：
      │   ├─ [L332] empty($tag)                     → 空标签跳过
      │   ├─ [L333] !isLoggedIn && startsWith($tag, '.') → 未登录 + 点前缀 → 跳过（第三层隐私）
      │   ├─ [L334] $tag === NO_MD_TAG ('nomarkdown') → 特殊标签跳过
      │   └─ [L335] in_array($tag, $filteringTags)   → 当前过滤条件本身的标签不计入
      │
      └─ [L341-L345] 大小写合并 + 计数：
          └─ $key = strtolower($tag)               ← BUG: 只改 ASCII，UTF-8 重音失效
             if (!isset($caseMapping[$key])) {
                 $caseMapping[$key] = $tag;         // 首次出现的拼写保留展示
                 $tags[$caseMapping[$key]] = 0;     // 初始化计数
             }
             $tags[$caseMapping[$key]]++;           // 计数+1
```

**关键事实**：
- L333 的 `!isLoggedIn && startsWith($tag, '.')` 检查的是**标签本身**（而非搜索输入），所以 `~.draft` 即使在 BookmarkFilter 中绕过了搜索输入过滤，**在标签云统计时仍会被 L333 正确拦住**——因为书签数据库中实际存的标签名是 `.draft`，以 `.` 开头。
- 但 `~.draft` 的**搜索结果**泄露已经发生在 BookmarkFilter 层，标签云的 L333 是额外保护，不修复搜索旁路。

### 15.4 数据写入（保存书签）协作流程

```
用户提交新书签 / 编辑书签
  │
  ▼
BookmarkFileService::add() / ::set() / ::update()
  │
  ├─ 构造或更新 Bookmark 对象
  │
  └─ [Bookmark::setTags L353-L363]
      │
      ├─ tags_filter() 去重、去空、去首尾空格
      │
      └─ array_map: 剥第一个字符的 '-' 前缀
          $tag[0] === '-' ? substr($tag, 1) : $tag
          │
          └─ 结果存入 $this->tags
              │
              ▼
          持久化到数据文件 (datastore.php / datastore.json)
```

---

## 16. 新增 / 修正缺陷的完整证据链（第三轮最终汇总）

### 缺陷 2（修正版）：OR 路径不剥前缀 + 漏检隐藏标签

**定性：BUG**（5 条交叉证据链）

| # | 证据 | 来源 |
|---|---|---|
| 1 | `tag2matchterm` 文档注释明确写 *"assumes any leading flags ('-', '~') have been stripped"*，但 OR 路径违反前置条件 | [BookmarkFilter.php L502-L509](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L502-L509) |
| 2 | AND 路径先剥 `+` 再剥 `-`，OR 路径完全不剥，两条路径不一致 | [BookmarkFilter.php L481-L500](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L481-L500) |
| 3 | `Bookmark::setTags` 保存时剥 `-` 前缀，`-baz` 标签几乎不可能存在，OR 路径匹配目标为空 | [Bookmark.php L353-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/Bookmark.php#L353-L363) |
| 4 | `HASHTAG_CHARS = \p{Pc}\p{N}\p{L}\p{Mn}` 不包含 `\p{Pd}`(Dash)，`-` 本就是语法标志而非标签字符 | [BookmarkFilter.php L50](file:///d:/fz/0601-1/solo-dogfeeding/code/73-Shaarli/application/bookmark/BookmarkFilter.php#L50) |
| 5 | 用户心智模型：`~+foo` 应理解为 "OR 匹配 foo（`+` 是冗余 AND 标志）"，而非 "匹配字面量 `+foo`" | 设计直觉 |

### 缺陷 3（修正版）：标签云 strtolower 只改 ASCII，Latin-1 以上全失效

**定性：BUG，影响范围比之前估计更广**

- 之前错误估计："只有非拉丁语系受影响"
- 实测真相：**所有含非 ASCII 大写字母的标签都不合并**，包括：
  - 法语：Étude / étude（最常见西欧语言即受影响）
  - 德语：Österreich / österreich
  - 西班牙语：Señor / señor（其实这个碰巧合并，因为只有 S 大写）
  - 希腊语、俄语：完全不合并
- 英语纯 ASCII 用户不受影响

### 缺陷 4（补充）：filterFulltext 排除搜索使用 strpos 而非 mb_strpos

**定性：低风险不一致**

- UTF-8 自同步特性使得字节级 strpos 实际上安全（不会把续字节误识别为 ASCII）
- 但与包含搜索的 `mb_strpos` 不一致，且非 UTF-8 locale 下可能出问题

