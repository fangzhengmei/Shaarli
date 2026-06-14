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

## 9. 关键代码索引

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
