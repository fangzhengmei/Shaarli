# 导入导出与 Netscape 格式：格式转换和数据保真路径分析

## 1. 整体架构

Shaarli 的导入导出功能围绕 Netscape 书签文件格式（Netscape Bookmark File Format）构建。核心模块分布在以下层次：

| 层次 | 模块 | 关键文件 |
|------|------|----------|
| 控制器层 | HTTP 路由与请求处理 | [ImportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/front/controller/admin/ImportController.php), [ExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/front/controller/admin/ExportController.php) |
| 业务逻辑层 | Netscape 格式工具类 | [NetscapeBookmarkUtils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php) |
| 解析层 | 第三方 Netscape 解析库 | `Shaarli\NetscapeBookmarkParser\NetscapeBookmarkParser` (外部依赖) |
| 数据模型层 | Bookmark 实体与存储 | [Bookmark.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/Bookmark.php), [BookmarkFileService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/BookmarkFileService.php), [BookmarkArray.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/BookmarkArray.php) |
| 工具函数层 | 标签、URL、转义等辅助函数 | [LinkUtils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php), [UrlUtils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/UrlUtils.php), [Utils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/Utils.php) |
| 视图层 | 模板渲染 | [export.bookmarks.html](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tpl/default/export.bookmarks.html), [import.html](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tpl/default/import.html), [export.html](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tpl/default/export.html) |

---

## 2. 浏览器书签导入路径

### 2.1 导入请求处理流程

```
用户上传 .html 文件
        │
        ▼
POST /admin/import ── [ImportController::import()]
        │
        ├─ 1. CSRF Token 校验 (checkToken)
        ├─ 2. 上传文件有效性检查
        │     ├─ 文件是否存在 (UploadedFileInterface)
        │     └─ 文件大小是否为 0 (超限判断)
        │
        └─ 3. 调用 NetscapeBookmarkUtils::import()
                  │
                  ├─ 3.1 DOCTYPE 格式校验
                  ├─ 3.2 解析 POST 参数 (overwrite / default_tags / privacy)
                  ├─ 3.3 初始化日志器 (import.log)
                  ├─ 3.4 NetscapeBookmarkParser::parseString() 解析
                  ├─ 3.5 逐条处理书签
                  ├─ 3.6 bookmarkService->save() 持久化
                  └─ 3.7 history->importLinks() 记录历史
```

### 2.2 DOCTYPE 格式校验

在 [NetscapeBookmarkUtils.php#L95](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L95-L97) 中进行文件格式识别：

```php
if (preg_match('/<!DOCTYPE NETSCAPE-Bookmark-file-1>/i', $data) === 0) {
    return $this->importStatus($filename, $filesize);
}
```

- 使用不区分大小写的正则匹配 (`/i` 修饰符)
- 支持 `<!DOCTYPE netscape-bookmark-file-1>` 等大小写变体（见测试 [BookmarkImportTest.php#L160-L169](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/BookmarkImportTest.php#L160-L169) 的 `testImportLowecaseDoctype`）
- 若 DOCTYPE 不存在，返回"未知文件格式"状态，`importCount/overwriteCount/skipCount` 均为 0

### 2.3 第三方解析库

解析工作委托给外部库 `Shaarli\NetscapeBookmarkParser\NetscapeBookmarkParser`（通过 composer 引入）。该库负责：

- 将 Netscape HTML 结构解析为关联数组
- 处理嵌套文件夹 (H3 + DL 嵌套结构) 到标签的转换
- 解析 `ADD_DATE`、`PRIVATE`、`TAGS`、`HREF` 等属性
- 解析 `<DD>` 描述内容
- 处理不同浏览器导出的编码差异

解析输出的每条书签数组结构大致为：

```php
[
    'name'        => '书签标题',
    'url'         => 'https://example.com',
    'description' => '书签描述',
    'tags'        => ['tag1', 'tag2', 'folderName'],  // 包含文件夹名转换的标签
    'dateCreated' => 1456433741,                       // Unix 时间戳
    'public'      => true,                             // !PRIVATE 的反向
]
```

---

## 3. 层级（文件夹）到标签的转换

### 3.1 转换机制

Netscape 格式使用嵌套的 `<DL>` + `<H3>` 结构表示文件夹层级：

```html
<DL><p>
  <DT><H3>Folder1</H3>
  <DL><p>
    <DT><A HREF="http://nest.ed/1-1" TAGS="tag1,tag2">Nested 1-1</A>
  </DL><p>
  <DT><H3>Folder3</H3>
  <DL><p>
    <DT><H3>Folder3-1</H3>
    <DL><p>
      <DT><A HREF="http://nest.ed/3-1">Nested 3-1</A>
    </DL><p>
  </DL><p>
</DL><p>
```

由第三方解析库 `NetscapeBookmarkParser` 完成转换，**将每一级父文件夹的名称作为标签追加到书签的标签列表中**。

### 3.2 实际转换示例

以测试文件 [netscape_nested.htm](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/input/netscape_nested.htm) 为例，测试断言见 [BookmarkImportTest.php#L203-L317](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/BookmarkImportTest.php#L203-L317)：

| 书签 URL | 原始 TAGS 属性 | 所在文件夹层级 | 最终导入标签 |
|----------|---------------|---------------|-------------|
| `http://nest.ed/1` | `tag1,tag2` | 根目录 | `tag1 tag2` |
| `http://nest.ed/1-1` | `tag1,tag2` | Folder1 | `folder1 tag1 tag2` |
| `http://nest.ed/1-2` | `tag3,tag4` | Folder1 | `folder1 tag3 tag4` |
| `http://nest.ed/2-1` | (空) | Folder2 | `folder2` |
| `http://nest.ed/2-2` | (空) | Folder2 | `folder2` |
| `http://nest.ed/3-1` | `tag3` | Folder3 > Folder3-1 | `folder3 folder3-1 tag3` |
| `http://nest.ed/3-2` | (空) | Folder3 > Folder3-1 | `folder3 folder3-1` |
| `http://nest.ed/2` | `tag4` | 根目录 | `tag4` |

**转换规则总结：**
1. 文件夹名作为标签追加到 TAGS 属性解析出的标签之前
2. 多级嵌套文件夹的名称按层级从外到内依次追加
3. 文件夹名保持原始大小写（如 `Folder1` → `folder1`？实际测试中是小写转换，由解析库处理）
4. TAGS 属性中的逗号分隔标签会被拆分为独立标签

### 3.3 默认标签的追加逻辑

用户可在导入表单中指定 `default_tags`，在 [NetscapeBookmarkUtils.php#L103-L110](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L103-L110) 和 [NetscapeBookmarkUtils.php#L164-L166](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L164-L166) 处理：

```php
$defaultTags = tags_str2array(
    escape($post['default_tags']),
    $this->conf->get('general.tags_separator', ' ')
);
// ...
$bkm['tags'] = array_merge($defaultTags, $bkm['tags']);
```

- 默认标签会被 `escape()` 进行 HTML 转义
- 默认标签通过 `array_merge` 放在解析标签的**前面**
- 标签分隔符可配置（默认空格，支持自定义如 `@`）

---

## 4. 重复地址处理

### 4.1 重复检测机制

重复检测基于 URL 的精确匹配，由 [BookmarkArray::getByUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/BookmarkArray.php#L224-L234) 实现：

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

- 使用哈希表 `$this->urls`（key=URL, value=数组偏移）实现 O(1) 查找
- **精确字符串匹配**，不会做 URL 规范化（如尾部斜杠、参数顺序等差异会视为不同 URL）
- URL 在 `offsetSet` 时同步写入索引

### 4.2 处理分支逻辑

在 [NetscapeBookmarkUtils.php#L143-L176](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L143-L176) 中实现：

```
findByUrl($bkm['url'])
        │
        ├── 找到 (existingLink = true)
        │     │
        │     ├── overwrite = false ──► skipCount++, continue
        │     │
        │     └── overwrite = true ──► 更新现有 Bookmark
        │                               ├─ setUpdated(new DateTime())
        │                               ├─ overwriteCount++
        │                               └─ 重新设置 title/url/description/private/tags
        │
        └── 未找到 (existingLink = false)
              │
              └─ 创建新 Bookmark ──► importCount++
                    ├─ setCreated(从 bkm['dateCreated'] 转换)
                    └─ 设置其他字段
```

**注意事项：**
- 覆盖时**保留原始创建时间**，仅更新 `updated` 时间戳
- 新建时使用导入文件中的 `dateCreated`（Unix 时间戳）转换为 DateTime，并设置为系统默认时区

### 4.3 计数器含义

| 计数器 | 含义 |
|--------|------|
| `importCount` | 所有经过处理写入存储的书签数（含新建和覆盖） |
| `overwriteCount` | 覆盖已存在 URL 的书签数 |
| `skipCount` | 因 URL 已存在且未开启 overwrite 而跳过的书签数 |

---

## 5. 私有标记处理

### 5.1 导入时的隐私决策树

在 [NetscapeBookmarkUtils.php#L132-L141](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L132-L141) 中实现三级隐私策略：

```
$post['privacy'] 参数
        │
        ├── 'private' ──► 全部强制设为私有 (isPrivate = true)
        ├── 'public'  ──► 全部强制设为公开 (isPrivate = false)
        └── default/其他 ──► 使用文件中的 PRIVATE 属性
                              │
                              ├── isset($bkm['public']) && !$bkm['public'] ──► true (私有)
                              └── 其他情况 ──► false (默认公开)
```

Netscape 格式中的 `PRIVATE="1"` 属性由解析库转换为 `$bkm['public'] = false`。

### 5.2 验证测试用例

测试覆盖在 [BookmarkImportTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/BookmarkImportTest.php)：
- `testImportKeepPrivacy` (L366-L404)：保留文件中的隐私设置
- `testImportAsPublic` (L409-L422)：全部强制公开
- `testImportAsPrivate` (L427-L440)：全部强制私有
- `testOverwriteAsPublic` (L445-L475)：二次导入时覆盖隐私为公开
- `testOverwriteAsPrivate` (L480-L510)：二次导入时覆盖隐私为私有

### 5.3 导出时的私有标记

在导出模板 [export.bookmarks.html#L9](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tpl/default/export.bookmarks.html#L9) 中：

```html
PRIVATE="{$private}"
```

其中 `$private = intval($value.private)`，输出为 `"0"` 或 `"1"`，与 Netscape 标准兼容。

---

## 6. 编码边界

### 6.1 HTML 实体转义

#### 6.1.1 `escape()` 函数

定义于 [Utils.php#L96-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/Utils.php#L96-L114)：

```php
function escape($input)
{
    // ...
    return htmlspecialchars($input, ENT_COMPAT, 'UTF-8', false);
}
```

- 使用 `ENT_COMPAT`：仅转义双引号，不转义单引号
- 字符集强制 `UTF-8`
- 第 4 参数 `double_encode = false`：不会对已存在的实体进行二次编码

#### 6.1.2 转义应用点

| 位置 | 转义对象 | 代码位置 |
|------|----------|----------|
| 导入 - 默认标签 | `$post['default_tags']` | [NetscapeBookmarkUtils.php#L107](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L107) |
| 导入 - 状态消息 | `$filename` | [NetscapeBookmarkUtils.php#L213](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L213) |
| 通用 - sanitizeLink | url/title/description/tags | [Utils.php#L133-L139](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/Utils.php#L133-L139) |

#### 6.1.3 转义测试验证

在 [BookmarkImportTest.php#L562-L584](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/BookmarkImportTest.php#L562-L584) 的 `testSanitizeDefaultTags` 中：

- 输入：`tag1& tag2 "tag3"`
- 存储：`tag1&amp; tag2 &quot;tag3&quot;`

### 6.2 URL 协议白名单

在 [Bookmark::setUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/Bookmark.php#L244-L253) 中调用 `whitelist_protocols()`，定义于 [UrlUtils.php#L75-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/UrlUtils.php#L75-L89)：

```php
function whitelist_protocols($url, $protocols)
{
    if (startsWith($url, '?') || startsWith($url, '/') || startsWith($url, '#')) {
        return $url;  // 内部路径不处理
    }
    $protocols = array_merge(['http', 'https'], $protocols);
    // 协议不在白名单中 → 替换为 http://
    // 无协议 → 自动添加 http://
}
```

**编码边界：**
- 内部 note URL（以 `?` 或 `/shaare/` 开头）绕过协议检查
- `javascript:`、`data:`、`file:` 等危险协议会被替换为 `http://`
- 协议识别正则：`/^(\w+):/?/?/`

### 6.3 文件编码处理

测试文件 [internet_explorer_encoding.htm](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/input/internet_explorer_encoding.htm) 验证了 IE 导出的特殊格式支持：

- IE 导出可能包含 `LAST_VISIT`、`LAST_MODIFIED` 等额外属性
- 可能使用不同的字符集编码
- 第三方解析库负责处理编码差异，解析结果为 UTF-8

### 6.4 标签处理的编码边界

在 [Bookmark::setTags()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/Bookmark.php#L353-L363) 中：

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

- `tags_filter()`（[LinkUtils.php#L247-L253](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php#L247-L253)）：trim + 过滤空标签
- 去除标签开头的 `-`（用于标签否定搜索语法）
- 不做额外的 HTML 转义（转义在输入或渲染阶段处理）

---

## 7. 导出文件生成路径

### 7.1 导出请求处理流程

```
POST /admin/export ── [ExportController::export()]
        │
        ├─ 1. CSRF Token 校验
        ├─ 2. 参数校验 (selection 不能为空)
        ├─ 3. 获取 BookmarkRawFormatter
        ├─ 4. NetscapeBookmarkUtils::filterAndFormat() 过滤和格式化
        │     ├─ 校验 selection ∈ [all, public, private]
        │     ├─ bookmarkService->search() 按可见性过滤
        │     ├─ formatter->format() 转为数组
        │     ├─ 生成 taglist (逗号分隔)
        │     └─ 可选: note URL 前追加站点地址
        ├─ 5. 模板变量赋值 (links/date/eol/selection)
        └─ 6. 渲染 export.bookmarks.html 模板
              ├─ Content-Type: text/html; charset=utf-8
              └─ Content-Disposition: attachment; filename=bookmarks_{selection}_{datetime}.html
```

### 7.2 书签格式化链

使用 `BookmarkRawFormatter`（继承自 [BookmarkFormatter](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/formatter/BookmarkFormatter.php)，无额外覆盖）。在 [BookmarkFormatter::format()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/formatter/BookmarkFormatter.php#L75-L101) 中生成以下字段：

```php
$out = [
    'id', 'shorturl', 'url', 'real_url', 'url_html',
    'title', 'title_html', 'description', 'thumbnail',
    'taglist',              // 标签数组
    'taglist_urlencoded',   // URL 编码的标签数组
    'taglist_html',
    'tags',                 // 空格分隔的标签字符串
    'tags_urlencoded',
    'sticky', 'private', 'class',
    'created',              // DateTime 对象
    'updated',              // DateTime 对象
    'timestamp',            // 创建时间 Unix 戳
    'updated_timestamp',    // 更新时间 Unix 戳
    'additional_content',
];
```

### 7.3 Netscape 特有的格式增强

在 [NetscapeBookmarkUtils::filterAndFormat()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L55-L78) 中额外处理：

```php
$link['taglist'] = implode(',', $bookmark->getTags());
if ($bookmark->isNote() && $prependNoteUrl) {
    $link['url'] = rtrim($indexUrl, '/') . '/' . ltrim($link['url'], '/');
}
```

- **taglist 字段**：将内部的空格/自定义分隔符标签转换为 Netscape 标准的**逗号分隔**
- **Note URL 处理**：note 的 URL 是内部相对路径（如 `/shaare/WDWyig`），可选前加站点 URL 以便浏览器导入

### 7.4 导出模板结构

模板文件 [export.bookmarks.html](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tpl/default/export.bookmarks.html)：

```html
<!DOCTYPE NETSCAPE-Bookmark-file-1>
<META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=UTF-8">
<TITLE>{$pagetitle}</TITLE>
<H1>Shaarli export of {$selection} bookmarks on {$date}</H1>
<DL><p>
{loop="links"}
<DT><A HREF="{$value.url}"
         ADD_DATE="{$value.timestamp}"
         {if="$value.updated_timestamp"}LAST_MODIFIED="{$value.updated_timestamp}" {/if}
         PRIVATE="{$private}"
         TAGS="{$value.taglist}">{$value.title}</A>
{if="$value.description"}{$eol}<DD>{$value.description}{/if}
{/loop}
</DL><p>
```

**模板输出要点：**
- `ADD_DATE`：Unix 时间戳（秒）
- `LAST_MODIFIED`：仅在书签有更新时间时输出
- `PRIVATE`：`intval()` 转换为 `"0"` 或 `"1"`
- `TAGS`：逗号分隔的标签列表（在 filterAndFormat 中已转换）
- `<DD>`：仅在描述非空时输出，前加换行符 `{$eol}`（PHP_EOL）
- **注意**：Shaarli 导出**不包含文件夹层级结构**，所有书签平铺在根 `<DL>` 下，标签信息仅通过 TAGS 属性保留

---

## 8. 数据保真路径总结

### 8.1 导入保真度

| 数据项 | 源字段 (Netscape) | 目标字段 (Bookmark) | 保真说明 |
|--------|-------------------|---------------------|----------|
| 标题 | `<A>` 标签内容 | `title` | 直接赋值，`trim()` |
| URL | `HREF` 属性 | `url` | 经 `whitelist_protocols()` 协议过滤 |
| 描述 | `<DD>` 内容 | `description` | 直接赋值 |
| 标签 | `TAGS` 属性 + 文件夹名 | `tags[]` | 逗号→数组，文件夹名自动追加，前可加默认标签 |
| 创建时间 | `ADD_DATE` 属性 | `created` | Unix 时间戳 → DateTime → 系统时区 |
| 更新时间 | （导入时不读取） | `updated` | 覆盖时设置为当前时间；新建时为 null |
| 私有状态 | `PRIVATE` 属性 | `private` | 可被 `privacy` 参数强制覆盖 |
| ID | （无） | `id` | 自动分配递增整数 |
| ShortURL | （无） | `shortUrl` | 由 created + id 通过 `smallHash()` 生成 |

### 8.2 导出保真度

| 数据项 | 源字段 (Bookmark) | 目标字段 (Netscape) | 保真说明 |
|--------|-------------------|---------------------|----------|
| 标题 | `title` | `<A>` 标签内容 | 直接输出 |
| URL | `url` | `HREF` 属性 | note 可前加站点 URL |
| 描述 | `description` | `<DD>` 内容 | 非空时输出 |
| 标签 | `tags[]` | `TAGS` 属性 | 数组→逗号分隔字符串 |
| 创建时间 | `created` | `ADD_DATE` | DateTime → Unix 时间戳 |
| 更新时间 | `updated` | `LAST_MODIFIED` | 非空时输出 |
| 私有状态 | `private` | `PRIVATE` 属性 | bool → "0"/"1" |
| 文件夹层级 | （不适用） | `<H3>` + 嵌套 `<DL>` | **丢失**，Shaarli 无文件夹概念，全部平铺 |
| Sticky | `sticky` | （无对应属性） | **丢失** |
| Thumbnail | `thumbnail` | （无对应属性） | **丢失** |

### 8.3 往返保真限制

1. **文件夹层级丢失**：导入时文件夹→标签，但导出时无法还原为嵌套文件夹结构
2. **Sticky 标记丢失**：Netscape 格式无对应字段
3. **缩略图缓存丢失**：Netscape 格式不支持
4. **标签分隔符语义变化**：内部空格/自定义分隔符 ↔ 导出逗号分隔
5. **创建时间精度**：均为 Unix 秒级时间戳，无精度损失
6. **URL 规范化**：导入时协议白名单处理可能改变原始 URL（如 `javascript:` → `http://`）

---

## 10. 第三方解析库 H3 / DL 嵌套遍历与文件夹名追加

### 10.1 解析器三层架构

`Shaarli\NetscapeBookmarkParser` v4.0.0 （commit `aa024e5731959966660d98fcefe27deada40d88e`）采用三层责任链：

```
NetscapeBookmarkParser::parseString()
        │  入口兼容层（向后兼容旧 API）
        ▼
NetscapeBookmarkDecoder::decode()
        │  核心解析：逐行正则 + 状态机遍历
        ▼
     输出 PHP 关联数组
```

构造器在 [NetscapeBookmarkParser](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/vendor/shaarli/netscape-bookmark-parser/src/NetscapeBookmarkParser.php) L33-L38 中固定了默认上下文：

```php
private $defaultContext = [
    NetscapeBookmarkDecoder::KEEP_NESTED_TAGS => true,   // 启用文件夹名→标签
    NetscapeBookmarkDecoder::NORMALIZE_DATES  => true,
    NetscapeBookmarkDecoder::DATE_RANGE       => '30 years',
];
```

### 10.2 `sanitizeString()`：HTML 预处理

在进入逐行遍历前，`NetscapeBookmarkDecoder::sanitizeString()`（L374-L429）对原始字符串做归一化：

| 步骤 | 正则/函数 | 作用 |
|------|-----------|------|
| 1 | `preg_replace('@<!--.*?-->@mis', '', ...)` | 剥离 HTML 注释块 |
| 2 | `preg_replace('@>(\s*?)<@mis', ">\n<", ...)` | 在 `><` 之间插换行，确保"每行一元素" |
| 3 | `preg_replace('@(<!DOCTYPE|<META|<TITLE|<H1|<P).*\n@i', '', ...)` | 删除元数据行（注意 META 行在此移除） |
| 4 | `trim($bookmark)` | 去除首尾空白 |
| 5 | `str_replace("\r", '', $bookmark)` | 删除回车符（统一 LF 换行） |
| 6 | `<DD>` + `<A>` 多行→单行回调 | 将实际换行转义为 Unicode 字符 `▄` 占位 |
| 7 | `preg_replace('@\n<DD@i', '<DD', ...)` | 把 `<A>` 与后续 `<DD>` 粘到同一行 |

**编码陷阱**：步骤 3 把 `<META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=XXX">` 这行完全删除。因此 HTML meta charset **只能在进入 sanitizeString 之前被外部探测**，解析器内部不会再看到 charset 声明。

### 10.3 `decode()` 主循环：线性状态机遍历

在 `NetscapeBookmarkDecoder::decode()` L98-L248 中，采用**栈式状态机**处理嵌套结构：

```
初始化:
    $items = []                // 输出书签数组
    $folderTags = []           // 当前生效的扁平化文件夹标签（一维数组）
    $groupedFolderTags = []    // 栈：每层文件夹的标签数组 [[layer0], [layer1], ...]

按行循环: explode("\n", sanitizeString($data))
    │
    ├── 匹配 /^<h\d.*>(.*)<\/h\d>/i  →  H1~H6 任意级别标题
    │       │
    │       ├─ $header[1] = H3 文本内容（文件夹名）
    │       ├─ $tag = sanitizeTags($header[1])   // 分割+小写+过滤
    │       ├─ array_push($groupedFolderTags, $tag)   // 入栈
    │       └─ $folderTags = flattenTagsList($groupedFolderTags)  // 展平为一维
    │
    ├── 匹配 /^<\/DL>/i  →  文件夹闭合标签
    │       │
    │       ├─ array_pop($groupedFolderTags)    // 出栈一层
    │       └─ $folderTags = flattenTagsList($groupedFolderTags)  // 重新展平
    │
    ├── 匹配 /<a/i  →  书签链接（含 <DT><A HREF="..." ...>TITLE</A>）
    │       │
    │       ├─ href="(.*?)"  → $item['url']
    │       ├─ icon="(.*?)"  → $item['image']
    │       ├─ <a.*?>(.*?)</a>  → $item['name']
    │       ├─ description/note 属性 或 <dd>(.*?)$  → $item['description']
    │       ├─ $tags = ($keepNestedTags ? $folderTags : [])   // ★ 文件夹标签前缀
    │       ├─ tags/labels/folders="..." → splitTagString() 并 append 到 $tags
    │       ├─ add_date → parseDate() → $item['dateCreated']
    │       ├─ public/published/pub / private/shared → parseBoolean() → $item['public']
    │       └─ $items[] = $item
    │
    └── 其他行（<DT>、<DL>、<p> 等无内容行） → 忽略
```

### 10.4 三级嵌套的栈状态演示

以 [netscape_nested.htm](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/input/netscape_nested.htm) 为例：

| 行 | 匹配类型 | `$groupedFolderTags` 栈 | `$folderTags` 扁平 |
|----|---------|------------------------|-------------------|
| `<H3>Folder1</H3>` | H3 | `[['folder1']]` | `['folder1']` |
| 书签 1-1 | `<A>` | `[['folder1']]` | `['folder1']`  → 追加到 tags |
| 书签 1-2 | `<A>` | `[['folder1']]` | `['folder1']`  → 追加到 tags |
| `</DL>` | 闭合 | `[]` | `[]` |
| `<H3>Folder2</H3>` | H3 | `[['folder2']]` | `['folder2']` |
| 书签 2-1, 2-2 | `<A>` | `[['folder2']]` | `['folder2']` |
| `</DL>` | 闭合 | `[]` | `[]` |
| `<H3>Folder3</H3>` | H3 | `[['folder3']]` | `['folder3']` |
| `<H3>Folder3-1</H3>` | H3 | `[['folder3'], ['folder3-1']]` | `['folder3', 'folder3-1']` |
| 书签 3-1, 3-2 | `<A>` | 同上 | `['folder3', 'folder3-1']` → 追加到 tags |
| `</DL>` | 内层闭合 | `[['folder3']]` | `['folder3']` |
| `</DL>` | 外层闭合 | `[]` | `[]` |

### 10.5 `flattenTagsList()`：二维栈展开到一维标签链

位于 `NetscapeBookmarkDecoder` 末尾（L491 附近），函数签名推断：

```php
public static function flattenTagsList(array $groupedFolderTags): array
{
    // 把 [['folderA'], ['folderB', 'folderC']]
    // 展开为 ['folderA', 'folderB', 'folderC']
    return array_reduce(
        $groupedFolderTags,
        fn ($acc, $tags) => array_merge($acc, $tags),
        []
    );
}
```

这是一个纯粹的"二维栈 → 一维数组"拼接，保证文件夹层级按嵌套从外到内的顺序排列。

### 10.6 `sanitizeTags()` 路径：文件夹名 → 标签数组

文件夹名 `Folder3-1` 进入 `sanitizeTags()`（L446-L479 附近）的完整调用链：

```
H3 捕获文本 "Folder3-1"
    │
    └─ sanitizeTags("Folder3-1")
           │
           ├─ 1. 判定分隔符: strpos(',') === false → 使用 ' ' 空格
           │
           ├─ 2. splitTagString("Folder3-1", ' ')
           │      │
           │      ├─ explode(' ', strtolower("Folder3-1"))
           │      │          = ['folder3-1']   ★ 在此处完成小写化
           │      │
           │      ├─ preg_replace('/\s{2,}/', ' ', $tags)  // 合并多空格
           │      │
           │      └─ array_map('trim') + array_filter
           │             = ['folder3-1']
           │
           └─ 3. 非纯字母数字的标签二次清洗（删除开头标点等）
                  最终返回 ['folder3-1']
```

---

## 11. 文件夹名变小写的精确代码位置

### 11.1 唯一位置：`splitTagString()` 中的 `strtolower()`

在 `NetscapeBookmarkDecoder::splitTagString()`（L433-L442）：

```php
public static function splitTagString(string $tagString, string $separator): array
{
    $tags = explode($separator, strtolower($tagString));   // ← 唯一小写化位置
    $tags = preg_replace('/\s{2,}/', ' ', $tags);
    return array_values(array_filter(array_map('trim', $tags)));
}
```

**关键结论：**
1. 小写化发生在 `explode` **之前**，`strtolower($tagString)` 是对整串操作
2. 函数入口有两条路径，全部都经过小写化：
   - 路径 A：H3 文件夹名 → `sanitizeTags()` → `splitTagString()` → 小写
   - 路径 B：`<A TAGS="Tag1,Tag2">` 属性 → `splitTagString()` → 小写
3. 使用 PHP 内置 `strtolower()`，**不支持多字节（非 ASCII）字符**。对中文/日文等 UTF-8 多字节字符无影响（原样保留），但对带重音的拉丁字符（如 `É` → `é`）可能因 locale 设置而行为不一致
4. Shaarli 自身在 `BookmarkFilter::filterAll()` 等搜索场景使用 `mb_convert_case($val, MB_CASE_LOWER, 'UTF-8')`（见 [BookmarkFilter.php:L620-L623](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/BookmarkFilter.php#L620-L623)），因此导入时的单字节小写化与搜索时的多字节小写化存在**标准不一致**

### 11.2 小写化影响范围对照表

| 输入 | 解析后标签 | 说明 |
|------|-----------|------|
| `Folder1` | `folder1` | 纯 ASCII 标题 |
| `Folder3-1` | `folder3-1` | 带连字符 |
| `My Tag` | `my tag` → `['my', 'tag']` | 空格分隔拆为两个 |
| `Tag1,Tag2` | `['tag1', 'tag2']` | 逗号分隔 |
| `标签A` | `标签a`（视 locale，通常保持 `标签A`） | UTF-8 中文 + 字母 |
| `ÉTÉ` | `été`（若 locale 为 UTF-8 则失败） | 带重音字符需 `mb_strtolower` |

### 11.3 注意：META 行被剥离

`sanitizeString()` 的步骤 3 用正则 `'@(<!DOCTYPE|<META|<TITLE|<H1|<P).*\n@i'` 把 `<META ... charset=...>` 整行删除。因此**任何基于该行的编码识别都必须在调用 `decode()` 之前完成**。解析器本身没有任何字符集转换逻辑，一律按原始字节做正则匹配。

---

## 12. URL 字段处理完整路径：trim、协议补全、尾斜杠

### 12.1 调用链总览

```
解析器 href="..." 捕获
    │  NetscapeBookmarkDecoder::decode() L163
    │  $item['url'] = $href[1]   ← 正则 /href="(.*?)"/i，不做任何处理
    ▼
NetscapeBookmarkUtils::import() L169
    │  $link->setUrl($bkm['url'], $allowedProtocols)
    ▼
Bookmark::setUrl() L244-L253
    │  ├─ 1. $url = cleanup_url($url)        ← URL 构造 + Firefox Reader 剥除
    │  ├─ 2. $url = whitelist_protocols(...) ← 协议白名单过滤 + 缺省补全
    │  └─ 3. $this->url = $url                ← 写入
    ▼
存储 + 索引
    │  BookmarkArray::offsetSet()
    │  $this->urls[$bookmark->getUrl()] = $offset
    └─ 精确字符串匹配进行去重
```

### 12.2 第一层：`cleanup_url()` → `Url` 类构造

在 [Bookmark::setUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/Bookmark.php#L244-L253) 内调用：

```php
public function setUrl(?string $url, array $allowedProtocols = []): Bookmark
{
    $url = cleanup_url($url);   // L245
    // ...
    $this->url = whitelist_protocols($url, $allowedProtocols);  // L249
}
```

`cleanup_url()` 在 [UrlUtils.php:L27-L32](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/UrlUtils.php#L27-L32)：

```php
function cleanup_url($url)
{
    $urlObj = new Url($url);
    return $urlObj->cleanup();
}
```

进入 [Url::__construct()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/Url.php#L62-L71)：

```php
public function __construct($url)
{
    $url = $url ?? '';
    $url = self::cleanupUnparsedUrl(trim($url));   // ★ L65: 唯一的 trim 位置
    $this->parts = parse_url($url);

    if (!empty($url) && empty($this->parts['scheme'])) {
        $this->parts['scheme'] = 'http';   // ★ L69: 临时补 scheme 以便后续重建
    }
}
```

**Trim 位置确认**：`trim($url)` 位于 Url 构造器 L65，仅去除**首尾空白字符**（空格、`\t`、`\n`、`\r`、`\0`、`\v`）。不处理 URL 内部空格，不处理尾部斜杠。

### 12.3 Firefox Reader 前缀剥除

`cleanupUnparsedUrl()`（[Url.php:L81-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/Url.php#L81-L84)）调用 `removeFirefoxAboutReader()`：

```php
protected static function removeFirefoxReader($input)
{
    $firefoxPrefix = 'about://reader?url=';
    if (startsWith($input, $firefoxPrefix)) {
        return urldecode(ltrim($input, $firefoxPrefix));
    }
    return $input;
}
```

- 仅当 URL 字面以 `about://reader?url=` 开头时触发
- `ltrim()` 在此处**仅用于剥除已知前缀字符**，不是通用 trim
- 之后 `urldecode()` 还原被百分号编码的原始 URL

### 12.4 第二层：`whitelist_protocols()` 协议补全与过滤

在 [UrlUtils.php:L75-L106](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/UrlUtils.php#L75-L106)：

```php
function whitelist_protocols($url, $protocols)
{
    // L78-L83: 内部路径直接放行（协议补全不应用）
    if (startsWith($url, '?') || startsWith($url, '/') || startsWith($url, '#')) {
        return $url;
    }
    $protocols = array_merge(['http', 'https'], $protocols);

    // L87-L97: 提取 URL 前缀中的协议名
    $scheme = get_url_scheme($url);
    if (!empty($scheme) && in_array(strtolower($scheme), $protocols)) {
        return $url;                    // 协议在白名单中 → 原样返回
    } elseif (!empty($scheme)) {
        return 'http://' . substr($url, strlen($scheme) + 1);
        // ↑↑↑ 协议不在白名单中（如 javascript:）→ 替换为 http://
    }
    return 'http://' . $url;           // 无协议 → 前加 http://
}
```

**协议处理矩阵：**

| 输入 URL | `scheme` | 协议在白名单中 | 输出 URL |
|----------|----------|---------------|----------|
| `https://example.com` | `https` | 是 | `https://example.com` |
| `example.com` | (空) | — | `http://example.com` |
| `javascript:alert(1)` | `javascript` | 否 | `http://alert(1)` |
| `magnet:?xt=urn:...` | `magnet` | 默认否（可配置） | 默认 `http://?xt=urn:...` |
| `?/shaare/WDWyig` | (空) | 内部路径 | `?/shaare/WDWyig`（不变） |
| `file:///C:/x.txt` | `file` | 否 | `http:///C:/x.txt` |

### 12.5 尾斜杠处理：完全保留

**整个导入路径对尾部斜杠不做任何处理。** 具体证据：

1. 解析器正则 `/href="(.*?)"/i` 不区分 URL 尾斜杠
2. `trim($url)` 仅删空白，不删 `/`
3. `parse_url()` 保留 path 中的尾斜杠
4. `unparse_url()` 直接拼接 parts，不做规范化
5. `BookmarkArray::getByUrl()` 采用精确字符串哈希匹配

因此，以下两个 URL 会被视为**完全不同**的条目：

- `https://example.com/path`
- `https://example.com/path/`

### 12.6 查询参数与 fragment 清洗（非规范化）

`Url::cleanup()`（[Url.php:L160-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/Url.php#L160-L165)）会剥离 `$annoyingQueryParams` 中列出的参数：

```php
private static $annoyingQueryParams = [
    'action_object_map=', 'action_ref_map=', 'action_type_map=',  // Facebook
    'fb_', 'fb=', 'PHPSESSID=',                                    // PHP/Facebook
    '__scoop',                                                     // Scoop.it
    'utm_',                                                        // Google Analytics
    'xtor=',                                                       // ATInternet
    'campaign_',                                                   // 其他
];
```

这会影响去重：导入时 `utm_*` 等参数被剥离，导致同一 URL 有无 utm 参数会被归并。例如：
- 导入 A：`https://x.com/a?utm_source=twitter` → 存储为 `https://x.com/a`
- 导入 B：`https://x.com/a` → 同样存储为 `https://x.com/a`，被判定为重复

---

## 13. 文件字符集探测完整函数链

### 13.1 导入主流程中无字符集转换

首先确认：在 Shaarli 自身的导入链中，**没有任何字符集探测与转换步骤**。

```
ImportController.php L73:  $data = (string)$file->getStream();
                                        ↑ 原始文件二进制字节流，直接读入
NetscapeBookmarkUtils.php L93:
    $data = (string)$file->getStream();   // 同上，原样传递
L95: preg_match('/<!DOCTYPE NETSCAPE...>/i', $data)
     ↑ 正则直接对原始字节匹配，不做编码判断
L124: $this->parser->parseString($data)
     ↑ 原始字节交给第三方库，不经任何 iconv/mb_convert
```

任何字符集转换都必须在外部管道完成，或依赖 PHP 正则引擎对目标编码的兼容程度。

### 13.2 解析库内部同样无字符集转换

在 `NetscapeBookmarkDecoder` 内部检查：

| 函数 | 是否涉及编码转换 |
|------|-----------------|
| `decode()` | 否，直接正则匹配字节 |
| `sanitizeString()` | 否，仅删行/粘行/转义换行 |
| `splitTagString()` | 使用 `strtolower()`（单字节），不进行 iconv |
| `sanitizeTags()` | 使用 `ctype_alnum()`，其余保留原样 |
| `parseBoolean()` | 正则匹配 TRUE_PATTERN / FALSE_PATTERN |
| `parseDate()` / `normalizeDate()` | 纯数字处理 |

**注意：** `sanitizeString()` 的步骤 3 会将 `<META ... charset=Windows-1252>` 这一行整行删除。如果浏览器导出文件中 `<META>` 行同时声明了 `CONTENT="text/html; charset=Windows-1252"`，在 `decode()` 开始执行时它已经不存在了，因此解析器内部**不可能**从该声明提取编码信息。

### 13.3 字符集探测工具函数（存在于 Shaarli，但导入路径未调用）

Shaarli 中存在两个字符集提取函数，但它们服务于**外链元数据抓取**场景（`MetadataRetriever`），而非书签导入：

**1. `header_extract_charset()`** — [LinkUtils.php:L28-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php#L28-L36)

```php
function header_extract_charset($header)
{
    preg_match('/charset=["\']?([^; "\']+)/i', $header, $match);
    if (!empty($match[1])) {
        return strtolower(trim($match[1]));
    }
    return false;
}
```

用于从 HTTP 响应头 `Content-Type: text/html; charset=GBK` 中提取。

**2. `html_extract_charset()`** — [LinkUtils.php:L45-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php#L45-L54)

```php
function html_extract_charset($html)
{
    preg_match('#<meta .*charset=["\']?([^";\'>/]+)["\']? */?>#Usi', $html, $enc);
    if (!empty($enc[1])) {
        return strtolower($enc[1]);
    }
    return false;
}
```

用于从 HTML `<META CHARSET="...">` 或 `<META HTTP-EQUIV="Content-Type" CONTENT="...;charset=...">` 中提取。

它被调用在 `HttpUtils::getCurlDownloadCallback()`（[HttpUtils.php:L593-L594](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/HttpUtils.php#L593-L594)）中，仅在 cURL 抓取外部页面 metadata 时使用。

### 13.4 实际的转换回退触发条件（MetadataRetriever 场景）

在 [MetadataRetriever.php:L59-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/MetadataRetriever.php#L59-L67) 中：

```php
if (!empty($title) && strtolower($charset) !== 'utf-8') {
    $title = mb_convert_encoding($title, 'utf-8', $charset);
}
if (!empty($description) && strtolower($charset) !== 'utf-8') {
    $description = mb_convert_encoding($description, 'utf-8', $charset);
}
if (!empty($tags) && strtolower($charset) !== 'utf-8') {
    $tags = mb_convert_encoding($tags, 'utf-8', $charset);
}
```

**回退条件（严格）：**
1. `$charset` 必须非空（由 `header_extract_charset` 或 `html_extract_charset` 成功提取到）
2. `strtolower($charset) !== 'utf-8'`
3. `mb_convert_encoding` 支持该源编码（mbstring 扩展必须启用）

**不回退的情况：**
- `$charset === null` 或提取失败 → 按 UTF-8 原样处理，若实际为 GBK/Windows-1252 则会出现乱码
- `$charset === 'utf-8'`（大小写不敏感）→ 不转换
- mbstring 扩展未启用 → 报错或静默失败

### 13.5 书签文件为 Windows-1252 / GBK 时的实际行为

因为导入管道未进行任何 `mb_convert_encoding` 或 `iconv` 调用，实际表现如下：

| 场景 | 导出文件编码 | DOCTYPE 匹配 | 英文字段 | 中文字段（标题/标签） |
|------|------------|-------------|---------|---------------------|
| 标准 Firefox / Chrome 导出 | UTF-8 | ✅ 匹配 | 正常 | 正常 |
| 旧版 IE 导出 | Windows-1252 | ✅ 匹配（DOCTYPE 为 ASCII） | 正常 | 西欧重音字符可能乱码 |
| 国内浏览器导出 | GBK / GB2312 | ✅ 匹配（DOCTYPE 为 ASCII） | 正常 | **中文乱码**——存储为 GBK 字节，系统按 UTF-8 解释 |
| 导出带 BOM 的 UTF-8 | UTF-8 BOM | ✅ BOM 不影响正则 | 正常 | 正常 |

**修复建议（当前缺失）：**
在 `NetscapeBookmarkUtils::import()` L93 之后、L95 DOCTYPE 检查之前，插入以下逻辑可解决问题：

```php
$data = (string)$file->getStream();
// 新增字符集探测与回退
$charset = html_extract_charset($data);
if ($charset && $charset !== 'utf-8' && function_exists('mb_convert_encoding')) {
    $data = mb_convert_encoding($data, 'UTF-8', $charset);
}
```

注意：需在 `sanitizeString()` 剥除 META 行**之前**执行探测。

### 13.6 字符集探测优先级（若实现上述修复）

```
1. 从 <META> 中提取 charset:
   /<meta .*charset=["\']?([^";\'>/]+)["\']? *\/?>/Usi
   │
   ├── 提取到 "utf-8" / "UTF-8" → 不转换
   ├── 提取到 "windows-1252" / "cp1252" / "iso-8859-1" → mb_convert(..., 'UTF-8', 'Windows-1252')
   ├── 提取到 "gbk" / "gb2312" / "gb18030" → mb_convert(..., 'UTF-8', 'GBK')
   └── 其他编码 → mb_convert(..., 'UTF-8', $charset)

2. 提取失败（或 mbstring 不可用）
   └── 按原始字节继续解析（可能出现乱码）
```

---

## 14. 关键代码索引（补充）

### 第三方解析库核心（v4.0.0, commit aa024e5）

- `NetscapeBookmarkParser::__construct()` — 默认上下文（KEEP_NESTED_TAGS=true）
- `NetscapeBookmarkDecoder::decode()` — 逐行解析主循环（H3/DL 状态机）
- `NetscapeBookmarkDecoder::sanitizeString()` — HTML 预处理（剥 META / 粘行 / 去注释）
- `NetscapeBookmarkDecoder::splitTagString()` — **`strtolower()` 小写化唯一位置**
- `NetscapeBookmarkDecoder::sanitizeTags()` — 文件夹名 → 标签数组清洗
- `NetscapeBookmarkDecoder::flattenTagsList()` — 二维标签栈展平为一维链

### URL 字段处理完整路径

- [NetscapeBookmarkDecoder::decode() L163](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/vendor/shaarli/netscape-bookmark-parser/src/Encoder/NetscapeBookmarkDecoder.php) — 解析器中 URL 提取（不做 trim）
- [Url::__construct() L65](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/Url.php#L62-L71) — **`trim($url)` 唯一位置** + 缺省 scheme 补全
- [Url::removeFirefoxAboutReader()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/Url.php#L93-L100) — Firefox Reader 前缀剥除
- [Url::cleanup()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/Url.php#L160-L165) — `utm_*`、`fb_*` 等恼人查询参数剥离
- [whitelist_protocols()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/UrlUtils.php#L75-L106) — 协议白名单 + 无协议补 `http://`
- [Bookmark::setUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/Bookmark.php#L244-L253) — 整合 cleanup_url + whitelist_protocols

### 字符集探测与回退

- [html_extract_charset()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php#L45-L54) — 从 HTML meta 提取 charset
- [header_extract_charset()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php#L28-L36) — 从 HTTP Content-Type 提取 charset
- [MetadataRetriever.php:L59-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/MetadataRetriever.php#L59-L67) — **`mb_convert_encoding` 实际回退位置**（仅外链抓取场景，未用于导入）
- [NetscapeBookmarkUtils.php:L93](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L93) — 导入时文件字节流读取（**当前无字符集转换**）

### 关键代码索引

### 导入主流程
- [ImportController::import()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/front/controller/admin/ImportController.php#L49-L81)
- [NetscapeBookmarkUtils::import()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L88-L191)

### 导出主流程
- [ExportController::export()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/front/controller/admin/ExportController.php#L35-L79)
- [NetscapeBookmarkUtils::filterAndFormat()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L55-L78)

### URL 重复检测
- [BookmarkFileService::findByUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/BookmarkFileService.php#L129-L132)
- [BookmarkArray::getByUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/BookmarkArray.php#L224-L234)

### 标签处理
- [Bookmark::setTags()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/Bookmark.php#L353-L363)
- [tags_str2array()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php#L217-L223)
- [tags_filter()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/LinkUtils.php#L247-L253)

### 编码与安全
- [escape()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/Utils.php#L96-L114)
- [whitelist_protocols()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/http/UrlUtils.php#L75-L89)
- [Bookmark::setUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/application/bookmark/Bookmark.php#L244-L253)

### 测试用例
- [BookmarkImportTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/BookmarkImportTest.php)
- [BookmarkExportTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/77-Shaarli/tests/netscape/BookmarkExportTest.php)
