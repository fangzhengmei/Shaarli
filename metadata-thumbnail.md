# Shaarli 元数据抓取与缩略图：远程请求、解析和安全边界分析

## 1. 整体架构概览

Shaarli 的远程元数据与缩略图系统由三条主线组成：

| 功能 | 入口控制器 | 核心服务 | 底层 HTTP |
|------|-----------|---------|-----------|
| 元数据抓取（标题/描述/标签） | [MetadataController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/MetadataController.php#L13-L28) | [MetadataRetriever](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/MetadataRetriever.php#L12-L79) | [HttpAccess](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/HttpAccess.php#L15-L48) → [get_http_response()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/HttpUtils.php#L40-L161) |
| 缩略图获取 | [ThumbnailsController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ThumbnailsController.php#L17-L64) | [Thumbnailer](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/Thumbnailer.php#L14-L130) | WebThumbnailer（第三方库） |
| 表单回填 | [ShaarePublishController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L16-L273) | 同上 MetadataRetriever | 同上 |

依赖注入在 [ContainerBuilder](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/container/ContainerBuilder.php#L105-L107) 中完成，`metadataRetriever` 和 `thumbnailer` 均作为闭包延迟实例化。

---

## 2. 远程请求机制

### 2.1 请求发起流程

元数据抓取的完整调用链：

```
MetadataController::ajaxRetrieveTitle()
  → MetadataRetriever::retrieve($url)
    → HttpAccess::getHttpResponse($url, $timeout, $maxBytes, $headerCallback, $downloadCallback)
      → get_http_response($url, ...)
        → cURL / file_get_contents fallback
```

### 2.2 URL 校验与清洗

在 [get_http_response()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/HttpUtils.php#L47-L51) 中，URL 经过两步校验：

1. **[Url](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/Url.php#L62-L71) 对象构造**：`parse_url()` 解析，缺省 scheme 自动补 `http`；`idnToAscii()` 将 IDN 域名转 ASCII。
2. **[isHttp()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/Url.php#L214-L217) + FILTER_VALIDATE_URL**：双重检查——`filter_var` 验证 URL 合法性，`isHttp()` 确保 scheme 以 `http` 开头。非 HTTP(s) URL 直接返回 `['Invalid HTTP UrlUtils', false]`。

控制器层面，[MetadataController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/MetadataController.php#L22-L24) 再次验证 `get_url_scheme($url)` 必须包含 `http`，双重防线。

### 2.3 请求超时

| 配置项 | 默认值 | 用途 |
|--------|--------|------|
| `general.download_timeout` | 30 秒 | cURL `CURLOPT_TIMEOUT`（含 DNS + 连接 + 传输的总超时） |
| `general.download_max_size` | 4194304（4 MiB） | `CURLOPT_PROGRESSFUNCTION` 中断下载阈值 |
| WebThumbnailer `timeout` | 10 秒 | 独立于元数据抓取，在 [web-thumbnailer.json](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/inc/web-thumbnailer.json#L7) 配置 |

关键行为：
- cURL **没有**设置 `CURLOPT_CONNECTTIMEOUT`，只有总超时。若 DNS 解析缓慢或服务端慢速发送，30 秒内连接+传输都受此限制。
- `CURLOPT_MAXREDIRS = 3`，限制重定向深度。
- `CURLOPT_FOLLOWLOCATION = true` + `CURLOPT_AUTOREFERER = true`，自动跟随重定向。

### 2.4 下载大小控制

```php
// HttpUtils.php#L101-L110
curl_setopt($ch, CURLOPT_PROGRESSFUNCTION,
    function ($arg0, $arg1, $arg2, $arg3, $arg4) use ($maxBytes) {
        return ($arg2 > $maxBytes) ? 1 : 0;  // 非零返回值终止下载
    }
);
```

进度回调在每 16KB（`CURLOPT_BUFFERSIZE = 1024 * 16`）的 chunk 后被调用，当已下载字节超过 `maxBytes` 时终止。但实际上，元数据抓取的 `WRITEFUNCTION` 回调通常会更早中断下载（一旦提取到 title/description/tags 即返回 `false`）。

### 2.5 请求身份伪装

```php
// HttpUtils.php#L54-L55
$userAgent = 'Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:115.0) Gecko/20100101 Firefox/115.0';
$acceptLanguage = substr(get_locale(LC_COLLATE), 0, 2) . ',en-US;q=0.7,en;q=0.3';
```

伪装为 Firefox 115 浏览器，Accept-Language 按服务器 locale 自动设置。部分网站会根据 UA 返回不同内容，此设计旨在获取与普通浏览器一致的 HTML。

---

## 3. 内容类型过滤与提前终止

### 3.1 Header 回调：[get_curl_header_callback()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/HttpUtils.php#L511-L541)

这是 `CURLOPT_HEADERFUNCTION` 回调，逐行处理响应头：

```
接收头部行 → 检查状态码 → 检查 Content-Type → 提取 charset → 决定是否继续
```

**关键决策点**：

1. **状态码 301/302**：标记 `$isRedirected = true`，跳过后续头部处理（等待最终响应）。
2. **状态码非 200 且非重定向**：`return false` 立即终止下载。
3. **Content-Type 不含 `text/html`**：`return false` 终止。这意味着 PDF、图片、JSON API 等非 HTML 资源在头部阶段就被拒绝，不会浪费带宽。
4. **重定向后 Content-Type 继承问题**：代码通过 `$isRedirected` 标志和显式检测 `content-type` 头部行来避免使用旧请求的 Content-Type（见第 529 行的注释和逻辑）。

### 3.2 Download 回调：[get_curl_download_callback()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/HttpUtils.php#L555-L636)

这是 `CURLOPT_WRITEFUNCTION` 回调，逐 chunk 处理 HTML 内容：

```
接收 chunk → 提取 charset → 提取 title → 提取 description → 提取 keywords → 判断是否可终止
```

**提前终止策略**（第 620-632 行）：

```php
if (
    (!empty($responseCode) && !empty($contentType) && !empty($charset)) 
    && $foundChunk !== null
    && (! $retrieveDescription
        || $foundChunk < $currentChunk
        || (!empty($title) && !empty($description) && !empty($keywords)))
) {
    return false;  // 终止下载
}
```

核心思路：一旦在某个 chunk 找到 title/description/keywords，继续扫描下一个 chunk；如果下一个 chunk 没有找到新元数据，就停止。这意味着通常只需要下载 HTML 的前几 KB 就能获得全部元数据。

**注意**：`$responseCode` 和 `$contentType` 在闭包内从未被赋值（它们在 header callback 中设置但不在 download callback 的 `use` 声明中），这意味着 **该终止条件中的前半部分永远为 false**，实际效果是下载会继续直到达到 `maxBytes` 或 cURL 超时。这是一个潜在的 bug——早期终止逻辑实际未生效，元数据抓取总是下载到大小上限或超时。

---

## 4. 元数据解析逻辑

### 4.1 解析函数族

| 函数 | 位置 | 提取目标 |
|------|------|----------|
| [html_extract_title()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/LinkUtils.php#L13-L19) | LinkUtils | `<title>` 标签内容 |
| [html_extract_tag()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/LinkUtils.php#L66-L88) | LinkUtils | `<meta>` 标签：支持 `property`/`name`/`itemprop` 属性，优先 OpenGraph（`og:`） |
| [header_extract_charset()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/LinkUtils.php#L28-L36) | LinkUtils | HTTP `Content-Type` 头中的 `charset=` |
| [html_extract_charset()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/LinkUtils.php#L45-L54) | LinkUtils | HTML `<meta charset="...">` 标签 |

### 4.2 元数据提取优先级

download callback 中的提取顺序：

1. **charset**：先从 HTML `<meta charset>` 提取（header callback 已尝试从 Content-Type 头提取）
2. **title**：先尝试 `html_extract_title()`（匹配 `<title>` 标签），再尝试 `html_extract_tag('title')`（匹配 `<meta property="og:title">` 等）
3. **description**：仅当 `retrieve_description` 配置开启时提取，调用 `html_extract_tag('description')`
4. **keywords**：仅当 `retrieve_description` 配置开启时提取，调用 `html_extract_tag('keywords')`，然后按逗号拆分并转换为 Shaarli 标签格式

### 4.3 OpenGraph 支持

[html_extract_tag()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/LinkUtils.php#L66-L88) 同时匹配两种模式：

- **标准 meta 标签**：`<meta name="description" content="...">`
- **OpenGraph 标签**：`<meta property="og:description" content="...">`

正则还支持 `property="og:unrelated og:description"` 这种多值属性，以及属性顺序颠倒（`content` 在 `property` 之前，如 GitHub 的做法）。

### 4.4 字符编码转换

在 [MetadataRetriever::retrieve()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/MetadataRetriever.php#L59-L67) 中：

```php
if (!empty($title) && strtolower($charset) !== 'utf-8') {
    $title = mb_convert_encoding($title, 'utf-8', $charset);
}
```

charset 来源有两个（按优先级）：
1. HTTP `Content-Type` 头中的 `charset=`（header callback 提取）
2. HTML `<meta charset="...">` 标签（download callback 提取，覆盖前者）

如果 charset 为空或已经是 UTF-8，则不做转换。

### 4.5 解析失败的容错

[MetadataRetriever::cleanMetadata()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/MetadataRetriever.php#L76-L79)：

```php
protected function cleanMetadata($data): ?string
{
    return !is_string($data) || empty(trim($data)) ? null : trim($data);
}
```

- 解析失败时 `html_extract_*` 返回 `false`，`cleanMetadata` 将其转为 `null`
- 最终返回结构始终是 `['title' => ..., 'description' => ..., 'tags' => ...]`，只是值为 `null`
- **无异常抛出**：网络错误、DNS 解析失败、非 HTML 内容类型等场景都通过返回空值/null 来静默处理

---

## 5. 缩略图获取

### 5.1 Thumbnailer 架构

[Thumbnailer](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/Thumbnailer.php#L14-L130) 封装了第三方库 `web-thumbnailer`：

- 构造时加载 [web-thumbnailer.json](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/inc/web-thumbnailer.json) 配置（缓存永久、超时 10 秒）
- 设置 `maxWidth`/`maxHeight` 来自 Shaarli 配置 `thumbnails.width`/`thumbnails.height`
- 启用裁剪（`crop(true)`）

### 5.2 三种缩略图模式

| 模式 | 常量 | 行为 |
|------|------|------|
| `all` | `MODE_ALL` | 对所有 HTTP URL 尝试获取缩略图 |
| `common` | `MODE_COMMON` | 仅对 [COMMON_MEDIA_DOMAINS](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/Thumbnailer.php#L16-L32) 列表中的域名或 `.jpg/.png/.jpeg` 结尾的 URL 尝试 |
| `none` | `MODE_NONE` | 完全禁用缩略图 |

### 5.3 [isCommonMediaOrImage()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/Thumbnailer.php#L108-L121) 过滤逻辑

```php
foreach (self::COMMON_MEDIA_DOMAINS as $domain) {
    if (strpos($url, $domain) !== false) {
        return true;
    }
}
if (endsWith($url, '.jpg') || endsWith($url, '.png') || endsWith($url, '.jpeg')) {
    return true;
}
```

- 使用 `strpos` 而非精确域名匹配，存在误匹配风险（如 `evilimgur.com.example.com` 会匹配 `imgur.com`）
- 文件扩展名检查使用 `endsWith`，不考虑查询参数（如 `image.jpg?width=100` 不会匹配）

### 5.4 错误处理

```php
try {
    return $this->wt->thumbnail($url);
} catch (\Throwable $e) {
    error_log(get_class($e) . ': ' . $e->getMessage());
}
return false;
```

所有异常被捕获并记录到 error_log，不向调用者传播。`thumbnail` 字段设为 `false` 表示无缩略图。

### 5.5 GD 依赖检查

构造函数中检查 `extension_loaded('gd')`，若不可用则强制设 `thumbnails.mode = none` 并 `die()` 输出错误信息。这是一个硬性终止，用户体验较差（TODO 注释也承认需要改进）。

---

## 6. 书签表单回填流程

### 6.1 同步模式（`enable_async_metadata = false`）

[ShaarePublishController::buildLinkDataFromUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L182-L226)：

```
用户提交 URL → 检查数据库是否已存在 → 不存在且无 title 参数 → 同步调用 MetadataRetriever::retrieve()
→ 将 title/description/tags 回填到表单数据 → 渲染编辑表单
```

同步模式下，**用户必须等待远程请求完成**后才能看到表单页面。若目标站点响应慢，管理员体验会很差。

### 6.2 异步模式（`enable_async_metadata = true`，默认）

1. 服务端直接渲染空表单，`async_metadata` 标志传给模板
2. 前端 [metadata.js](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/assets/common/js/metadata.js#L57-L94) 在 DOM 加载后自动发起 AJAX：

```javascript
xhr.open('GET', `${basePath}/admin/metadata?url=${encodeURI(url)}`, true);
xhr.onload = () => {
    const result = JSON.parse(xhr.response);
    Object.keys(result).forEach((key) => {
        if (result[key] !== null && result[key].length) {
            const element = form.querySelector(`input[name="lf_${key}"], textarea[name="lf_${key}"]`);
            if (element != null && element.value.length === 0) {
                element.value = he.decode(result[key]);
            }
        }
    });
    clearLoaders(loaders);
};
```

**回填规则**：
- 仅当表单字段当前为空时才填入远程值（`element.value.length === 0`）
- 使用 `he.decode()` 对 HTML 实体解码
- 支持 `lf_title`（input）、`lf_description`（textarea）、`lf_tags`（input）三个字段

### 6.3 缩略图异步更新

[metadata.js](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/assets/common/js/metadata.js#L98-L106) 同时处理缩略图懒加载：

```javascript
const thumbsToLoad = document.querySelectorAll('div[data-async-thumbnail]');
thumbsToLoad.forEach((divElement) => {
    const { id } = divElement.closest('[data-id]').dataset;
    updateThumb(basePath, divElement, id);
});
```

`updateThumb()` 发送 PATCH 请求到 `/admin/shaare/{id}/update-thumbnail`，服务端调用 `Thumbnailer::get()` 获取缩略图后更新书签数据，返回格式化后的书签 JSON。

### 6.4 保存时的缩略图处理

[ShaarePublishController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L121-L127)：

```php
if (
    $this->container->conf->get('thumbnails.mode', Thumbnailer::MODE_NONE) !== Thumbnailer::MODE_NONE
    && true !== $this->container->conf->get('general.enable_async_metadata', true)
    && $bookmark->shouldUpdateThumbnail()
) {
    $bookmark->setThumbnail($this->container->thumbnailer->get($bookmark->getUrl()));
}
```

仅在 **同步模式** 下保存时获取缩略图。异步模式下缩略图由前端 AJAX 单独请求。

### 6.5 Bookmark.shouldUpdateThumbnail() 判断

[Bookmark::shouldUpdateThumbnail()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/Bookmark.php#L398-L405)：

```php
return $this->thumbnail !== false
    && !$this->isNote()
    && startsWith(strtolower($this->url), 'http')
    && (null === $this->thumbnail || !is_file($this->thumbnail));
```

条件：thumbnail 未被标记为不可用（`!== false`）、非笔记、URL 以 http 开头、且 thumbnail 为 null 或缓存文件不存在。

---

## 7. 安全边界与 SSRF 风险分析

### 7.1 现有防护措施

| 防护层 | 实现 | 有效性 |
|--------|------|--------|
| 协议限制 | `isHttp()` + `get_url_scheme()` 检查必须为 http(s) | ✅ 阻止 `file://`、`gopher://`、`ftp://` 等危险协议 |
| URL 合法性 | `FILTER_VALIDATE_URL` + `parse_url()` | ✅ 阻止格式错误的 URL |
| 认证保护 | `MetadataController` 和 `ThumbnailsController` 继承自 `ShaarliAdminController`，需要登录 | ✅ 防止匿名用户滥用 |
| 内容类型过滤 | Header callback 拒绝非 `text/html` | ⚠️ 仅限元数据抓取，缩略图不受此限制 |
| 下载大小限制 | `CURLOPT_PROGRESSFUNCTION` + `maxBytes` | ✅ 防止内存耗尽 |
| 超时限制 | cURL `CURLOPT_TIMEOUT` | ⚠️ 无连接超时，DNS 慢解析可阻塞 |
| 重定向深度 | `CURLOPT_MAXREDIRS = 3` | ✅ 限制重定向链 |

### 7.2 SSRF（服务端请求伪造）风险

**风险等级：高**

核心问题：Shaarli 以服务器身份发起 HTTP 请求，攻击者（已登录的管理员）可构造 URL 让服务器访问内网资源。

#### 场景 1：内网端口扫描

```
GET /admin/metadata?url=http://192.168.1.1:8080/
```

通过观察响应时间或返回内容，可推断内网服务是否存活。虽然 `CURLOPT_TIMEOUT = 30s` 限制了单次请求时间，但攻击者可逐个探测。

#### 场景 2：访问云元数据服务

```
GET /admin/metadata?url=http://169.254.169.254/latest/meta-data/
```

在 AWS/GCP/Azure 环境中，此 URL 返回实例元数据（可能含 IAM 凭证）。由于 header callback 会拒绝非 `text/html` 响应，元数据 API 若返回纯文本会被过滤，但 **如果 API 返回 `Content-Type: text/html` 则不会被阻止**。

#### 场景 3：WebThumbnailer SSRF

`Thumbnailer` 使用独立的 `web-thumbnailer` 库，有独立的超时（10s）和下载逻辑。该库的请求不受 `get_curl_header_callback` 的 `text/html` 过滤，**任何 URL 都会被请求以尝试提取缩略图**，扩大了攻击面。

#### 场景 4：DNS Rebinding

攻击者注册域名，首次解析返回公网 IP（通过 `FILTER_VALIDATE_URL`），随后在 TTL 过期后解析到内网 IP。由于 cURL `FOLLOWLOCATION = true`，重定向后的请求可能访问内网。

### 7.3 缺失的防护

1. **无内网 IP 黑名单**：未检查目标 IP 是否为私有地址段（`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`127.0.0.0/8`、`169.254.0.0/16`）
2. **无 DNS 解析后校验**：未在 DNS 解析后验证目标 IP，存在 DNS rebinding 风险
3. **无请求频率限制**：除登录认证外，无 API 调用速率限制，可被用于大量内网扫描
4. **无 URL 目标白名单**：任何 HTTP(s) URL 都会被请求
5. **WebThumbnailer 独立请求通道**：缩略图获取走第三方库，绕过了元数据抓取的部分安全过滤
6. **Fallback 方法使用 `stream_context_set_default`**：[get_http_response_fallback()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/HttpUtils.php#L183-L222) 修改了全局默认流上下文，可能影响其他 HTTP 请求

### 7.4 其他安全注意事项

1. **mb_convert_encoding 注入**：若 `charset` 被恶意控制（如通过 `Content-Type: text/html; charset=<malicious>`），`mb_convert_encoding($data, 'utf-8', $charset)` 可能触发意外行为。虽然 PHP 8.x 对无效编码处理更严格，但恶意 charset 值仍可能导致异常或意外输出。

2. **正则 DoS**：[html_extract_tag()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/LinkUtils.php#L66-L88) 中的正则较复杂，对于精心构造的超长 HTML 可能存在回溯爆炸风险。不过由于 `maxBytes` 限制（4 MiB），实际影响有限。

3. **XSS via metadata**：前端 [metadata.js](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/assets/common/js/metadata.js#L84-L85) 使用 `element.value = he.decode(result[key])` 赋值，对于 `<input>` 和 `<textarea>` 的 `.value` 赋值是安全的（不会被解析为 HTML）。但后端模板使用 `escape()` 函数对表单数据进行转义，提供了纵深防御。

4. **CURLOPT_FOLLOWLOCATION + 开放重定向**：自动跟随重定向意味着如果目标网站返回 302 到内网地址，Shaarli 会跟随。虽然有 `MAXREDIRS = 3` 限制，但仍可被利用。

---

## 9. 深度追查：WebThumbnailer 第三方库的实际 HTTP 请求路径

### 9.1 版本与来源

Shaarli 使用的 WebThumbnailer 版本为 `v2.2.0`，来自 [arthurhoaro/web-thumbnailer](https://github.com/ArthurHoaro/web-thumbnailer)（[composer.lock](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/composer.lock#L10-L56)）。

### 9.2 请求架构：双重 HTTP 调用

WebThumbnailer 的缩略图获取流程涉及 **两次独立的 HTTP 请求**，均绕过了 Shaarli 自身 `HttpUtils` 中的 `get_curl_header_callback` 内容类型过滤：

```
① Finder 查找阶段（DefaultFinder::find()）
   ├─ 判断 URL 是否为图片扩展名 → 是则直接返回原 URL
   └─ 否则请求目标页面 HTML → 解析 <meta property="og:image"> 提取缩略图 URL

② 下载阶段（Thumbnailer::thumbnailDownload()）
   └─ 请求 Finder 返回的缩略图 URL（可能是完全不同的域名/路径）
     → 下载图片二进制数据 → GD 库裁剪缩放 → 保存到 cache/ 目录
```

#### 阶段 ①：Finder 查找

[DefaultFinder::find()](https://github.com/ArthurHoaro/web-thumbnailer/blob/v2.2.0/src/Finder/DefaultFinder.php#L35-L83) 中的 WebAccess 调用：

```php
list($headers, $content) = $this->webAccess->getContent(
    $this->url,
    (int) ConfigManager::get('settings.default.timeout', 30),
    (int) ConfigManager::get('settings.default.max_img_dl', 16777216),  // 16 MiB
    $callback,
    $content
);
```

- **超时**：默认 30s，可被 Shaarli 的 [web-thumbnailer.json](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/inc/web-thumbnailer.json#L7) 覆盖为 10s
- **最大下载量**：默认 16 MiB（`max_img_dl`），而非 Shaarli 元数据抓取的 4 MiB——**WebThumbnailer 允许下载 4 倍于元数据抓取的数据量**

#### 阶段 ②：缩略图下载

[Thumbnailer::thumbnailDownload()](https://github.com/ArthurHoaro/web-thumbnailer/blob/v2.2.0/src/Application/Thumbnailer.php#L239-L303) 中的第二次 WebAccess 调用：

```php
$webaccess = WebAccessFactory::getWebAccess($thumbUrl);
list($headers, $data) = $webaccess->getContent(
    $thumbUrl,
    $this->options[WebThumbnailer::DOWNLOAD_TIMEOUT],
    $this->options[WebThumbnailer::DOWNLOAD_MAX_SIZE]
);
```

- `thumbUrl` 来自 Finder 提取的 `og:image` 或直接图片 URL，**完全由远程页面内容决定**，没有任何域名白名单校验
- 使用独立的 WebAccess 实例，cURL cookie jar 在两次请求间共享（见下文）

### 9.3 WebAccess 实现与超时/大小控制

#### [WebAccessCUrl](https://github.com/ArthurHoaro/web-thumbnailer/blob/v2.2.0/src/Application/WebAccess/WebAccessCUrl.php)

核心 cURL 配置（与 Shaarli 自身的 [get_http_response()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/HttpUtils.php#L40-L161) 对比）：

| 配置项 | Shaarli HttpUtils | WebThumbnailer WebAccessCUrl |
|--------|-------------------|------------------------------|
| `CURLOPT_TIMEOUT` | 30s（可配置） | 30s 默认，JSON 配置可覆盖为 10s |
| `CURLOPT_MAXREDIRS` | 3 | **6**（重定向链更长） |
| `CURLOPT_BUFFERSIZE` | 16KB | 16KB |
| `CURLOPT_COOKIESESSION` | 未设置 | **设置**，配合 `CURLOPT_COOKIEFILE`/`COOKIEJAR` 持久化 cookie |
| 下载大小限制方式 | PROGRESSFUNCTION + maxBytes | PROGRESSFUNCTION + maxBytes |
| 默认 maxBytes | 4 MiB | **16 MiB**（`max_img_dl`） |
| User-Agent | Firefox 115 (Fedora) | Firefox **45** (Linux) + `WebThumbnailer` 标识 |
| Header callback 内容类型过滤 | ✅ 仅 text/html | ❌ **无全局过滤**，仅在 DefaultFinder 的 WRITEFUNCTION 回调中实现 |

#### [WebAccessPHP](https://github.com/ArthurHoaro/web-thumbnailer/blob/v2.2.0/src/Application/WebAccess/WebAccessPHP.php)（cURL 不可用时的 fallback）

使用 `file_get_contents()` + PHP stream context：
- 无内容类型检查
- 直接读取 `$maxBytes` 字节，使用 `get_headers()` 手动追踪重定向
- 同样的 `CURLOPT_MAXREDIRS` 等效限制为 3 次

### 9.4 WebThumbnailer 的内容类型过滤策略

WebThumbnailer **没有独立的 Header callback**，它将内容类型判断嵌入到 `CURLOPT_WRITEFUNCTION` 回调中（[DefaultFinder::getCurlCallback()](https://github.com/ArthurHoaro/web-thumbnailer/blob/v2.2.0/src/Finder/DefaultFinder.php#L89-L177)），这带来了关键差异：

```php
// WRITEFUNCTION 回调中的内容类型分流
if (
    !empty($contentType)
    && strpos($contentType, 'image/') !== false
    && strpos($contentType, 'application/octet-stream') === false
) {
    $thumbnail = $url;  // 直接把原 URL 当作缩略图
    return false;       // 终止下载
} elseif (
    !empty($contentType)
    && strpos($contentType, 'text/html') === false
    && strpos($contentType, 'application/octet-stream') === false
) {
    return false;       // 非 HTML 非二进制流 → 终止
}
```

**关键观察**：

1. **`application/octet-stream` 被当作 HTML 处理继续下载**——二进制流不会被早期终止，可能浪费带宽
2. **`image/*` 类型被当作缩略图直接返回**——这是 WebThumbnailer 处理直接图片链接的方式
3. **内容类型过滤仅在 Finder 阶段生效**，在下载阶段（`thumbnailDownload()`）完全没有内容类型检查——恶意 `og:image` 指向非图片资源时，完整内容会被下载并送入 `ImageUtils::generateThumbnail()` 由 GD 库验证
4. **仅当使用 `WebAccessCUrl` 时才启用回调**——如果 PHP 没装 cURL，`WebAccessPHP` 分支完全不执行 WRITEFUNCTION 回调，**直接下载完整内容后才用 `extractMetaTag()` 解析**，整个响应内容将被载入内存

### 9.5 两次请求间的 Cookie 持久化

```php
// WebAccessCUrl.php
$cookie = ConfigManager::get('settings.path.cache') . '/cookie.txt';
curl_setopt($ch, CURLOPT_COOKIESESSION, true);
curl_setopt($ch, CURLOPT_COOKIEFILE, $cookie);
curl_setopt($ch, CURLOPT_COOKIEJAR, $cookie);
```

- Finder 阶段和缩略图下载阶段共用同一个 `cache/cookie.txt` 文件
- 这意味着：① 阶段中目标网站设置的 cookie（如登录态、CSRF token）会被 ② 阶段自动携带
- **跨请求的 cookie 共享可能被利用**：若 Finder 阶段触发目标网站设置某个危险 cookie，下载阶段会带上它请求另一域名的缩略图（虽然存在同源 cookie 限制，但仍需注意）

---

## 10. cURL 协议限制核查：CURLOPT_PROTOCOLS 与 CURLOPT_REDIR_PROTOCOLS

### 10.1 核查结论：两处代码均 **未设置**

| 设置 | Shaarli `get_http_response()` | WebThumbnailer `WebAccessCUrl` |
|------|-------------------------------|--------------------------------|
| `CURLOPT_PROTOCOLS` | ❌ 未设置 | ❌ 未设置 |
| `CURLOPT_REDIR_PROTOCOLS` | ❌ 未设置 | ❌ 未设置 |

### 10.2 未设置的默认行为（PHP 7.1+ / cURL 7.19.4+）

- **`CURLOPT_PROTOCOLS` 默认值**：`CURLPROTO_HTTP | CURLPROTO_HTTPS | CURLPROTO_FTP | CURLPROTO_FTPS`
- **`CURLOPT_REDIR_PROTOCOLS` 默认值**（自 PHP 7.1.9 / cURL 7.19.4）：`CURLPROTO_HTTP | CURLPROTO_HTTPS | CURLPROTO_FTP | CURLPROTO_FTPS`

这意味着：

1. **FTP 协议被默认允许**——虽然 Shaarli 自身的 `Url::isHttp()` 会在发起请求前拦截非 HTTP(s) scheme，但 `FOLLOWLOCATION` 重定向到 `ftp://` URL 不会被 `isHttp()` 拦截（校验只在初始 URL 做）
2. **没有被白名单限制为仅 HTTP/HTTPS**——与最佳安全实践不符
3. **WebThumbnailer 更危险**：其 Finder 阶段和下载阶段均未做 scheme 校验，初始 URL 理论上可以是任何被 cURL 接受的协议

### 10.3 重定向协议风险链

```
攻击者构造初始 URL: http://evil.com/redirect
  → evil.com 返回 302 Location: ftp://internal-server/secret.txt
    → cURL 按默认配置允许 FTP 协议重定向，Shaarli WebThumbnailer 将尝试下载
```

虽然 FTP 下载最终因 `Content-Type` 不符合或 GD 库无法解析图片而失败，但：
- **连接已建立**，内网端口/服务可达性可被盲探测
- 若目标 FTP 服务有匿名登录且提供公开文件，下载字节数可从响应时间推断

### 10.4 与 Shaarli 前端 URL 白名单的差异

书签保存时的 [whitelist_protocols()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/UrlUtils.php#L75-L89)：

```php
function whitelist_protocols($url, $protocols)
{
    $protocols = array_merge(['http', 'https'], $protocols);
    $protocol = preg_match('#^(\w+):/?/?#', $url, $match);
    if ($protocol === 1 && !in_array($match[1], $protocols)) {
        $url = str_replace($match[0], 'http://', $url);
    }
    return $url;
}
```

此函数仅在 `Bookmark::setUrl()` 保存书签时被调用，它替换非白名单协议。但 **缩略图更新时使用的 URL 直接来自 `Bookmark::getUrl()`**，即已通过 `whitelist_protocols` 清洗的 URL——因此 `update-thumbnail` 端点的初始 URL 是相对安全的。然而：

1. `MetadataController::ajaxRetrieveTitle()` 的 URL 来自 `$_GET['url']`，只经过 `get_url_scheme()` 检查，**没有经过 `whitelist_protocols()`**
2. 重定向过程中 cURL 可能跳转到 FTP 等协议，不受上述白名单保护

---

## 11. update-thumbnail 端点：完整鉴权与 URL 来源链路

### 11.1 路由注册

在 [index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/index.php#L156-L159) 中：

```php
$this->patch(
    '/shaare/{id:[0-9]+}/update-thumbnail',
    '\Shaarli\Front\Controller\Admin\ThumbnailsController:ajaxUpdate'
);
// ... 整个 /admin group 都挂载了 ShaarliAdminMiddleware
})->add('\Shaarli\Front\ShaarliAdminMiddleware');
```

### 11.2 完整鉴权链路

```
PATCH /admin/shaare/{id}/update-thumbnail
  │
  ├─ ① Slim 路由匹配 → 提取 {id} 为数字正则约束 ([0-9]+)
  │
  ├─ ② ShaarliAdminMiddleware
  │    └─ loginManager->isLoggedIn() === true？
  │       ├─ false → 302 重定向到 /login
  │       └─ true → 继续
  │            └─ 调用 parent: ShaarliMiddleware
  │                 ├─ 检查是否已安装（配置文件存在）
  │                 ├─ 执行数据库 updater
  │                 └─ 检查 open_shaarli 强制登录配置
  │
  ├─ ③ ThumbnailsController::ajaxUpdate()
  │    ├─ {id} 正则约束 + ctype_digit() 双重校验
  │    ├─ bookmarkService->get((int) $id) 从数据存储读取书签
  │    │    └─ BookmarkNotFoundException → 404
  │    ├─ 从已读取的 Bookmark 对象调用 getUrl() 获取 URL
  │    ├─ 调用 thumbnailer->get($bookmark->getUrl()) 获取缩略图
  │    ├─ bookmark->setThumbnail($result) 更新内存对象
  │    └─ bookmarkService->set($bookmark) 写回数据存储
  │
  └─ ④ 返回 raw formatter 格式化的书签 JSON
```

### 11.3 isLoggedIn() 的判定逻辑

[LoginManager::isLoggedIn()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/security/LoginManager.php#L128-L134)：

```php
public function isLoggedIn(): bool
{
    if ($this->openShaarli) {
        return true;   // ⚠️ 开放 Shaarli 模式下，任何人都算"已登录"
    }
    return $this->isLoggedIn;
}
```

**重要安全提示**：在 `security.open_shaarli = true` 模式下，**任何匿名用户都被视为已登录**，因此 `update-thumbnail`、`metadata` 等需要管理员权限的端点会暴露给所有人。这是设计特性（公开协作的 Shaarli 实例），但对于部署在可访问内网的服务器，这意味着 SSRF 攻击面也对外开放。

### 11.4 会话校验：checkLoginState()

在 `checkLoginState()` 中，即使 `openShaarli = true` 跳过，正常模式下的校验链为：

1. `staySignedIn` cookie 校验（SHA1 哈希：密码 + IP + salt）
2. 或 session：`!hasSessionExpired() && !hasClientIpChanged()`
3. IP 变更检测可通过 `security.session_protection_disabled = true` 关闭

### 11.5 ⚠️ CSRF Token 缺失

与 [ShaarePublishController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L98-L100) 对比：

```php
// save() 方法有 token 校验
$this->checkToken($request);

// 但 ThumbnailsController::ajaxUpdate() 中……
// ❌ 没有 checkToken() 调用！
```

`ThumbnailsController::ajaxUpdate()` **未调用 `checkToken()`**，即 **不校验 CSRF Token**。同样 `MetadataController::ajaxRetrieveTitle()` 也没有。

这意味着：若用户登录态有效（session cookie 未过期），攻击者可通过第三方页面构造 AJAX PATCH 请求触发缩略图更新——虽然无法读取响应（浏览器 CORS 阻止），但可作为 SSRF 的触发通道，让服务器请求任意已存为书签的 URL。

### 11.6 URL 来源可信度分析

| 端点 | URL 来源 | 是否可信 |
|------|---------|---------|
| `update-thumbnail` | `bookmarkService->get($id)->getUrl()` | ✅ **可信**——URL 已存入数据存储，在 `Bookmark::setUrl()` 时已通过 `whitelist_protocols()` 清洗；但仍受重定向 SSRF 影响 |
| `metadata` | `$request->getParam('url')`（`$_GET`） | ❌ **不可信**——直接来自用户输入，只经过 `get_url_scheme()` 做 `http` 前缀检查 |

**update-thumbnail 的特殊风险**：虽然 URL 本身来自可信存储，但如果攻击者先通过正常流程将恶意 URL（如内网地址 `http://10.0.0.1:6379/`）保存为书签，后续触发 `update-thumbnail` 时仍会发起内网请求。也就是说，**只要 URL 曾被保存成功，就等于通过了所有校验**。

### 11.7 书签 URL 保存路径（如何污染 URL）

攻击者可通过 `POST /admin/shaare` 保存 URL：

1. `ShaarePublishController::save()` → `Bookmark::setUrl($url, $allowedProtocols)`
2. `Bookmark::setUrl()` → `whitelist_protocols($url, ['http', 'https'] + $allowedProtocols)`
3. `whitelist_protocols()` 仅**替换协议前缀**为 `http://`，不校验域名/IP
4. 内网 IP 地址如 `http://10.0.0.1/admin` 完全通过校验——因为协议是合法的 HTTP

**结论**：update-thumbnail 的"可信 URL"仅意味着协议合法，不代表目标 IP/域名安全。SSRF 风险依然存在，只是需要先有一个保存步骤。

---

## 12. 总结：补充关键发现

| 领域 | 新增关键发现 |
|------|-------------|
| **WebThumbnailer 架构** | 涉及两次独立 HTTP 调用（Finder 查 og:image + 下载缩略图），均绕过 Shaarli 的内容类型 header callback |
| **下载大小** | WebThumbnailer Finder 默认允许下载 16 MiB（是元数据抓取的 4 倍），PHP fallback 分支无 WRITEFUNCTION 回调会下载完整内容 |
| **内容类型过滤** | 仅在 cURL WRITEFUNCTION 回调中实现；PHP fallback 无过滤；`application/octet-stream` 被宽容处理；缩略图下载阶段无内容类型校验 |
| **cURL 协议限制** | Shaarli 和 WebThumbnailer 均 **未设置** `CURLOPT_PROTOCOLS`/`CURLOPT_REDIR_PROTOCOLS`，默认允许 FTP/FTPS 协议及重定向到 FTP |
| **Cookie 持久化** | WebThumbnailer 两次请求共享 cookie jar，Finder 阶段设置的 cookie 会被带到下载阶段 |
| **update-thumbnail 鉴权** | 依赖 `ShaarliAdminMiddleware` + `LoginManager::isLoggedIn()`；open_shaarli 模式下匿名可访问；**CSRF token 校验缺失** |
| **URL 来源** | update-thumbnail 使用存储中已清洗的 URL（可信），但保存阶段不校验内网 IP，攻击者可预埋恶意 URL；`/metadata` 端点直接接受用户输入的 URL |
| **SSRF 深层风险** | ① open_shaarli + 未设置 PROTOCOLS → 匿名 FTP 内网探测；② CSRF 缺失 → 登录态下被第三方页面触发；③ 预埋 URL + update-thumbnail → 任意书签 URL 的 SSRF 触发 |

---

## 13. thumbnails.mode 默认值与开箱即用 SSRF 暴露面评估

### 13.1 默认值溯源

默认配置在 [ConfigManager::setDefaultValues()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/config/ConfigManager.php#L387-L389) 中：

```php
$this->setEmpty('thumbnails.mode', Thumbnailer::MODE_ALL);
$this->setEmpty('thumbnails.width', '125');
$this->setEmpty('thumbnails.height', '90');
```

同时 [ConfigManager::setDefaultValues()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/config/ConfigManager.php#L370-L371) 中：

```php
$this->setEmpty('general.enable_async_metadata', true);
$this->setEmpty('security.allowed_protocols', ['ftp', 'ftps', 'magnet']);
$this->setEmpty('security.open_shaarli', false);
```

**开箱即用的默认值组合**：

| 配置项 | 默认值 | SSRF 影响 |
|--------|--------|----------|
| `thumbnails.mode` | `MODE_ALL` | ✅ 对所有 HTTP URL 启用缩略图抓取，无域名限制 |
| `general.enable_async_metadata` | `true` | ✅ 前端自动触发 `/admin/metadata` AJAX 请求元数据 |
| `security.open_shaarli` | `false` | ❌ 关闭，SSR F 通道仅限登录用户 |
| `security.allowed_protocols` | `['ftp', 'ftps', 'magnet']` | ✅ FTP/FTPS 被加入书签 URL 协议白名单 |
| `general.download_timeout` | 30s（LegacyUpdater 设置） | 单次请求最长阻塞 30s |
| `general.download_max_size` | 4 MiB（LegacyUpdater 设置） | 元数据抓取单请求最多下载 4 MiB |

### 13.2 旧版本迁移行为（LegacyUpdater）

[LegacyUpdater::updateMethodWebThumbnailer()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/legacy/LegacyUpdater.php#L524-L539)：

```php
$thumbnailsEnabled = extension_loaded('gd') && $this->conf->get('thumbnail.enable_thumbnails', true);
$this->conf->set('thumbnails.mode', $thumbnailsEnabled ? Thumbnailer::MODE_ALL : Thumbnailer::MODE_NONE);
```

- 从旧版本（< v0.12.0）升级时，如果旧配置 `thumbnail.enable_thumbnails` 未显式关闭（默认 `true`）且 GD 可用，则迁移为 `MODE_ALL`
- **升级场景下也是默认启用缩略图**

### 13.3 GD 不可用的降级

[Thumbnailer 构造函数](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/Thumbnailer.php#L55-L61)：

```php
if (!extension_loaded('gd')) {
    $this->conf->set('thumbnails.mode', Thumbnailer::MODE_NONE);
    // TODO: improve user experience, redirect to server config with a warning message
    die('Please install the PHP GD extension to use thumbnails feature.');
}
```

GD 不可用时强制设为 `MODE_NONE` 并 `die()` 终止。这实际上 **保护了没有 GD 的服务器** 不暴露缩略图 SSRF 通道——但元数据抓取 `MetadataController` 不依赖 GD，依然可用。

### 13.4 开箱即用 SSRF 暴露面矩阵

| 场景 | 元数据抓取 `/admin/metadata` | 缩略图抓取（同步保存时） | 缩略图异步更新 `update-thumbnail` |
|------|-------------------------------|------------------------|----------------------------------|
| **默认配置（需要登录）** | ✅ 暴露于登录用户 | ✅ 暴露于登录用户（同步保存模式 `enable_async_metadata=false`） | ✅ 暴露于登录用户（书签列表页的 `data-async-thumbnail` 触发） |
| **open_shaarli=true** | 🟥 暴露于任意匿名用户 | 🟥 暴露于任意匿名用户 | 🟥 暴露于任意匿名用户 |
| **GD 不可用** | ✅ 暴露于登录用户（不依赖 GD） | ❌ 无（`MODE_NONE`） | ❌ 无 |
| **thumbnails.mode=common** | ✅ 暴露于登录用户 | ⚠️ 仅 COMMON_MEDIA_DOMAINS 命中或 `.jpg/.png/.jpeg` 结尾的 URL 才触发 | ⚠️ 同上 |

**结论：默认配置下 SSRF 暴露面为中等偏上**——登录用户即可对任意 HTTP(S) URL 发起元数据请求和缩略图请求。若管理员开启了 `open_shaarli`，暴露面扩大到全体互联网匿名用户。

---

## 14. cURL 重定向允许 FTP 协议的 libcurl 版本前提

### 14.1 CURLOPT_REDIR_PROTOCOLS 的引入历史

| 项目 | 版本 | 说明 |
|------|------|------|
| `CURLOPT_REDIR_PROTOCOLS` 选项加入 libcurl | **7.19.4**（2009-03） | 最初发布时默认值为 `CURLPROTO_ALL`，即重定向可跳转到所有协议（包括 file://, gopher://, scp:// 等危险协议） |
| PHP 支持该选项 | PHP **5.2.10+** | PHP curl 扩展开始暴露此常量 |
| **默认值收紧** | libcurl **7.65.2**（2019-06） | 此版本将默认值改为 `CURLPROTO_HTTP \| CURLPROTO_HTTPS \| CURLPROTO_FTP \| CURLPROTO_FTPS`——仅保留 Web 常用协议 |

### 14.2 当前实际风险评估

```
libcurl < 7.65.2 → 默认 CURLPROTO_ALL → 重定向可到 file://、gopher://、dict:// 等所有协议
                → 风险：极高（可读取本地文件 / 内网 Gopher 协议攻击 Redis/Memcached）

libcurl >= 7.65.2 → 默认 HTTP/HTTPS/FTP/FTPS → 重定向仅限这四类
                  → 风险：中等（仍可 FTP 内网探测，但无法直接读本地文件）
```

**关键前提验证**：Shaarli 代码中未显式设置 `CURLOPT_REDIR_PROTOCOLS`，因此 **完全依赖底层 libcurl 的默认行为**。

- 现代服务器（2020 年后部署）大多数使用 libcurl ≥ 7.65.2，风险被限制在 HTTP/HTTPS/FTP/FTPS
- 但企业内网 LTS 发行版（如 CentOS 7、RHEL 7）仍带 libcurl 7.29.0（2013），**默认允许所有协议**——此类部署存在高危 SSRF 到 `file:///etc/passwd` 的风险

### 14.3 CURLOPT_PROTOCOLS 与 CURLOPT_REDIR_PROTOCOLS 的差异

| 选项 | 作用 | 默认值（libcurl ≥ 7.65.2） |
|------|------|---------------------------|
| `CURLOPT_PROTOCOLS` | 限制初始请求允许的协议 | HTTP / HTTPS / FTP / FTPS |
| `CURLOPT_REDIR_PROTOCOLS` | 限制重定向目标允许的协议 | HTTP / HTTPS / FTP / FTPS |

两者默认值相同但语义不同：
- **初始 URL**：Shaarli 在应用层已做 `Url::isHttp()` 检查（仅允许 HTTP/HTTPS），所以 `CURLOPT_PROTOCOLS` 对初始请求的默认值被应用层校验覆盖
- **重定向 URL**：Shaarli **没有**对每次重定向目标重复做 `Url::isHttp()` 检查，完全依赖 `CURLOPT_REDIR_PROTOCOLS` 的默认值——这是协议风险的真正来源

### 14.4 Shaarli 代码路径中的差异

| 代码路径 | 初始 URL 协议校验 | 重定向协议保护 |
|---------|-----------------|-------------|
| `MetadataRetriever` → `HttpAccess::getHttpResponse()` → `get_http_response()` | ✅ `Url::isHttp()` + `FILTER_VALIDATE_URL` | ❌ 仅依赖 libcurl 默认 REDIR_PROTOCOLS |
| `ThumbnailsController` → `Thumbnailer::get()` → WebThumbnailer `WebAccessCUrl` | ❌ **完全无初始 URL 协议校验**（URL 来自 `Bookmark::getUrl()`，存储时仅经过 `whitelist_protocols`） | ❌ 仅依赖 libcurl 默认 REDIR_PROTOCOLS |
| WebThumbnailer 缩略图下载阶段（`og:image` URL） | ❌ **完全无校验**，URL 从远程页面 meta 标签解析 | ❌ 仅依赖 libcurl 默认 REDIR_PROTOCOLS |

**最危险路径**：WebThumbnailer 下载从远程 HTML 解析出的 `og:image` URL——攻击者在目标网页设置 `<meta property="og:image" content="file:///etc/passwd">`，在 libcurl < 7.65.2 环境中，cURL 会尝试读取本地文件并送入 GD 库处理。GD 无法解析图片会报错，但文件读取动作已发生（可通过响应时间差异做盲数据提取）。

---

## 15. 管理员目录所有 Controller 的 CSRF Token 校验清单

### 15.1 checkToken() 实现

[ShaarliAdminController::checkToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarliAdminController.php#L26-L37)：

```php
protected function checkToken(Request $request): bool
{
    if (!$this->container->sessionManager->checkToken($request->getParam('token'))) {
        $this->saveErrorMessage(t('Invalid token!'));
        $this->redirectFromReferer($request, $this->container->response, []);
    }
    return true;
}
```

token 从请求参数 `$_REQUEST['token']`（即 GET + POST + Cookie）读取，校验失败后重定向并显示错误消息。

### 15.2 完整清单（18 个 Admin Controller）

| # | Controller | 方法 | HTTP 方法 | 是否校验 CSRF Token | SSRF 相关 |
|---|-----------|------|-----------|-------------------|----------|
| 1 | [ConfigureController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ConfigureController.php#L67) | `save()` | POST | ✅ `checkToken($request)`（第 67 行） | 可配置 thumbnails.mode |
| 2 | [ExportController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ExportController.php#L37) | `process()` | POST | ✅ `checkToken($request)` | 否 |
| 3 | [ImportController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ImportController.php#L51) | `import()` | POST | ✅ `checkToken($request)` | 可批量导入 URL（SSR F 预埋入口） |
| 4 | [ManageTagController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ManageTagController.php#L46-L103) | `rename()` / `delete()` | POST | ✅ 两处均有 `checkToken` | 否 |
| 5 | [PasswordController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/PasswordController.php#L45) | `save()` | POST | ✅ `checkToken` | 否 |
| 6 | [PluginsController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/PluginsController.php#L56) | `save()` | POST | ✅ `checkToken` | 否 |
| 7 | [ShaareManageController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php) | `delete()` | POST | ✅（第 23 行） | 否 |
| 8 | [ShaareManageController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php) | `editSave()` | POST | ✅（第 84 行） | 可修改书签 URL |
| 9 | [ShaareManageController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php) | `bEdit()` | POST | ✅（第 150 行） | 可批量修改书签 URL |
| 10 | [ShaareManageController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php) | `pin()` | POST | ✅（第 186 行） | 否 |
| 11 | [ShaareManageController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php) | `bDelete()` | POST | ✅（第 214 行） | 否 |
| 12 | [ShaarePublishController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L100) | `save()` | POST | ✅ `checkToken($request)` | 可保存 URL 入库 |
| 13 | 🔴 [MetadataController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/MetadataController.php) | `ajaxRetrieveTitle()` | GET | ❌ **无 CSRF 校验** | **直接 SSRF 入口** |
| 14 | 🔴 [ThumbnailsController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ThumbnailsController.php) | `ajaxUpdate()` | PATCH | ❌ **无 CSRF 校验** | **SSRF 触发入口**（URL 来自存储） |
| 15 | 🔴 [ServerController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ServerController.php#L71) | `clearCache()` | GET | ❌ **无 CSRF 校验** | 可清除缩略图缓存（迫使后续请求重新下载） |
| 16 | [SessionFilterController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/SessionFilterController.php) | `visibility()` | GET | ❌ 无校验 | 否（仅切换可见性） |
| 17 | [TokenController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/TokenController.php) | `getToken()` | GET | ❌ 无校验 | 自身就是 token 提供方 |
| 18 | [LogoutController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/LogoutController.php) | `index()` | GET | ❌ 无校验 | 否（登出动作） |
| 19 | [ToolsController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ToolsController.php) | `index()` | GET | ❌ 无校验 | 否（只读页面） |
| 20 | [ShaareAddController](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareAddController.php) | `addShaare()` | GET | ❌ 无校验 | 否（仅展示表单） |

### 15.3 CSRF 缺失的实际影响

**MetadataController（GET `/admin/metadata?url=...`）**：

- 浏览器 CORS 阻止第三方网页读取跨域 AJAX 响应内容，但 **请求本身会被发送**（Simple Request）
- 攻击者可在钓鱼网站放置 `<img src="http://target-shaarli/admin/metadata?url=http://10.0.0.1:6379/">`，若用户当前已登录 Shaarli，浏览器会携带 session cookie 发起请求——服务器会向 `10.0.0.1:6379` 发起 HTTP 请求
- 虽无法读取响应，但可用于内网端口扫描、服务存活探测

**ThumbnailsController（PATCH `/admin/shaare/{id}/update-thumbnail`）**：

- PATCH 不是 Simple Request，浏览器会先发 CORS preflight OPTIONS 请求
- 若 Shaarli 未配置宽松的 CORS 策略，浏览器会阻止实际请求发出
- 但若浏览器为旧版本或 CORS 配置存在疏漏，仍可触发
- 更现实的攻击链：先通过 CSRF 保存恶意 URL 入库（ShaarePublishController `save()` 虽有 token，但如攻击者通过钓鱼诱导用户在已登录浏览器中提交），再通过图片墙页面正常触发缩略图下载

**ServerController::clearCache()（GET `/admin/clear-cache?type=thumbnails`）**：

- 可被 `<img>` 标签 CSRF 触发，清除缩略图缓存后，后续页面访问会强制重新从远程下载缩略图——配合预埋的恶意 URL 可放大 SSRF 攻击面

---

## 16. 书签 URL 入库的全部污染入口追查

### 16.1 URL 清洗管道：Bookmark::setUrl()

[Bookmark::setUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/Bookmark.php#L244-L253)：

```php
public function setUrl(?string $url, array $allowedProtocols = []): Bookmark
{
    $url = $url !== null ? trim($url) : '';
    if (! empty($url)) {
        $url = whitelist_protocols($url, $allowedProtocols);
    }
    $this->url = $url;
    return $this;
}
```

[whitelist_protocols()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/http/UrlUtils.php#L75-L89)：

```php
function whitelist_protocols($url, $protocols)
{
    $protocols = array_merge(['http', 'https'], $protocols);
    $protocol = preg_match('#^(\w+):/?/?#', $url, $match);
    if ($protocol === 1 && !in_array($match[1], $protocols)) {
        $url = str_replace($match[0], 'http://', $url);
    }
    return $url;
}
```

清洗逻辑**仅替换协议前缀**，不做任何域名/IP 校验。默认 `$allowedProtocols` 来自配置 `security.allowed_protocols = ['ftp', 'ftps', 'magnet']`，因此最终允许的协议为 `http / https / ftp / ftps / magnet`。

### 16.2 入口 ①：管理员手动保存书签

| 路径 | 文件位置 | CSRF 保护 | URL 来源 |
|------|---------|----------|---------|
| `POST /admin/shaare`（新建） | [ShaarePublishController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L100-L136) | ✅ token 校验 | `$request->getParam('lf_url')` → `$bookmark->setUrl($url, $this->container->conf->get('security.allowed_protocols'))` |
| `POST /admin/shaare/{id}`（编辑保存） | [ShaareManageController::editSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php#L84-L136) | ✅ token 校验 | 同上 |
| `POST /admin/batch/shaare`（批量编辑） | [ShaareManageController::bEdit()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php#L150-L177) | ✅ token 校验 | 逐本调用 `setUrl()` |

所有手动保存路径都经过 `setUrl()` 协议清洗。

### 16.3 入口 ②：书签文件批量导入

[NetscapeBookmarkUtils::import()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/netscape/NetscapeBookmarkUtils.php#L88-L191)：

```php
foreach ($bookmarks as $bkm) {
    // ...
    $link->setTitle($bkm['name']);
    $link->setUrl($bkm['url'], $this->conf->get('security.allowed_protocols'));  // 第 169 行
    // ...
    $this->bookmarkService->addOrSet($link, false);
}
```

- URL 来自 `NetscapeBookmarkParser` 解析用户上传的 Netscape 书签文件
- 经过 `setUrl()` 协议清洗
- 有 CSRF 保护（[ImportController::import()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ImportController.php#L51) 第 51 行 `checkToken`）
- **可被用作 SSRF 预埋批量入口**：管理员上传构造的书签文件，其中包含大量内网地址，导入后触发缩略图/元数据抓取

### 16.4 入口 ③：插件钩子 save_link 污染

[ShaarePublishController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L130-L135)：

```php
// To preserve backward compatibility with 3rd parties, plugins still use arrays
$formatter = $this->getFormatter('raw');
$data = $formatter->format($bookmark);
$this->executePageHooks('save_link', $data);
$bookmark->fromArray($data, $this->container->conf->get('general.tags_separator', ' '));
```

[Bookmark::fromArray()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/Bookmark.php#L69-L91)：

```php
public function fromArray(array $data, string $tagsSeparator = ' '): Bookmark
{
    // ...
    $this->url = $data['url'] ?? null;  // ⚠️ 第 73 行
    // ...
}
```

**🔴 关键安全漏洞**：`fromArray()` 直接将 `$data['url']` 赋值给 `$this->url`，**不经过 `setUrl()` 的 `whitelist_protocols()` 协议清洗**。

这意味着：
1. 第三方插件在 `save_link` 钩子中修改 `$data['url']` 时，可以注入任意协议（如 `file://`、`gopher://`）
2. 被污染的 URL 直接入库，后续 `update-thumbnail` 和元数据抓取将使用被污染的 URL 发起请求
3. 相同的 `save_link` → `fromArray()` 模式在以下 4 处存在：
   - [ShaarePublishController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaarePublishController.php#L135) 第 135 行
   - [ShaareManageController::editSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php#L132) 第 132 行
   - [ShaareManageController::bEdit()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php#L174) 第 174 行
   - [ShaareManageController::bDelete()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/front/controller/admin/ShaareManageController.php#L275) 第 275 行

### 16.5 入口 ④：LegacyUpdater 数据迁移

[LegacyUpdater::updateMethodBookmarksToEntities()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/legacy/LegacyUpdater.php#L589-L591)：

```php
$linksArray = new BookmarkArray();
foreach ($this->linkDB as $key => $link) {
    $linksArray[$key] = (new Bookmark())->fromArray($link, $this->conf->get('general.tags_separator', ' '));
}
```

- 旧数据格式的 URL 直接通过 `fromArray()` 载入，不经过 `setUrl()` 协议清洗
- 但旧链接在原存储中本就已入库，属于既有数据，不构成新的污染路径

### 16.6 入口 ⑤：BookmarkInitializer 默认书签

[BookmarkInitializer::initialize()](file:///d:/fz/0601-1/solo-dogfeeding/code/78-Shaarli/application/bookmark/BookmarkInitializer.php#L39-L114)：

- 硬编码的 3 条默认书签 URL（YouTube、Shaarli 项目等）
- **调用 `setUrl()`**：`$bookmark->setUrl('https://www.youtube.com/watch?v=DVEUcbPkb-c')`
- 无用户输入，无安全风险

### 16.7 污染入口风险汇总

| 入口 | URL 清洗方式 | CSRF 保护 | 风险级别 | 说明 |
|------|------------|----------|---------|------|
| 管理员新建/编辑书签 | ✅ `setUrl()` + `whitelist_protocols` | ✅ | 低 | 仅 HTTP(S)/FTP(S)/magnet |
| Netscape 书签导入 | ✅ `setUrl()` + `whitelist_protocols` | ✅ | 中 | 批量导入，可预埋大量内网 URL |
| `save_link` 插件钩子 | ❌ **`fromArray()` 直接赋值，绕过 `setUrl()`** | ✅ | 🟥 **高** | 恶意插件可注入任意协议 URL |
| LegacyUpdater 迁移 | ❌ `fromArray()` 直接赋值 | N/A | 低 | 既有数据迁移 |
| BookmarkInitializer | ✅ `setUrl()` | N/A | 无 | 硬编码安全 URL |

**最高风险发现**：插件通过 `save_link` 钩子可以将任意 URL（含 `file://`、`gopher://`、内网地址）写入书签，绕过协议白名单。虽然插件安装本身需要管理员权限，但这打破了"所有入库 URL 都经过 `whitelist_protocols` 清洗"的安全假设。

---

## 17. 最终汇总：全部安全发现全景

| 类别 | 发现 | 严重度 |
|------|------|--------|
| **SSRF 默认暴露** | `thumbnails.mode` 默认 `MODE_ALL`，登录用户即可对任意 HTTP(S) URL 发起缩略图抓取；`enable_async_metadata` 默认 `true`，前端自动触发元数据抓取 | 中 |
| **SSRF 匿名暴露** | `security.open_shaarli` 开启时，所有 SSRF 通道（metadata、update-thumbnail、保存书签）对外开放 | 高 |
| **cURL 协议白名单缺失** | 未设置 `CURLOPT_PROTOCOLS` / `CURLOPT_REDIR_PROTOCOLS`，libcurl < 7.65.2 默认允许所有协议（含 file://、gopher://） | 高（老系统）/ 中（新系统） |
| **WebThumbnailer og:image 无协议校验** | 从远程页面解析出的 `og:image` URL 无任何协议/域名校验，直接传入 WebAccess 请求 | 高 |
| **WebThumbnailer PHP fallback 无内容过滤** | cURL 不可用时，`WebAccessPHP` 不启用 WRITEFUNCTION 回调，整页内容载入内存 | 中 |
| **MetadataController CSRF 缺失** | GET `/admin/metadata?url=...` 无 token 校验，可被 `<img>` CSRF 触发内网请求 | 中 |
| **ThumbnailsController CSRF 缺失** | PATCH `/admin/shaare/{id}/update-thumbnail` 无 token 校验 | 低（需 CORS 绕过） |
| **ServerController clearCache CSRF 缺失** | GET `/admin/clear-cache?type=thumbnails` 无 token 校验，可被 CSRF 触发缓存清除以放大 SSRF | 低 |
| **fromArray() 绕过 setUrl()** | `Bookmark::fromArray()` 直接赋值 `$this->url`，不经过 `whitelist_protocols` 协议清洗；`save_link` 插件钩子可利用此路径污染 URL | 高 |
| **内网 IP 无黑名单** | 所有 HTTP 请求路径均未校验目标 IP 是否为私有地址段（10.x/8、172.16/12、192.168/16、127/8、169.254/16） | 高 |
| **DNS Rebinding 无防护** | 未对 DNS 解析结果进行校验或缓存，存在 DNS 重新绑定攻击面 | 中 |
| **无请求速率限制** | 除登录失败封禁外，元数据与缩略图抓取端点无调用速率限制，可被用于大规模内网扫描 | 中 |
| **下载提前终止 bug** | `get_http_response()` 的 download callback 终止条件依赖未传入的变量，永不触发，实际下载到 maxBytes 或超时 | 低（性能影响） |
| **COMMON_MEDIA_DOMAINS 误匹配** | `strpos` 模糊匹配，`evilimgur.com.example.com` 会命中 `imgur.com` | 低 |
| **无 Content-Type 继承 bug 修复不完善** | 重定向场景下 Content-Type 继承逻辑存在 edge case | 低 |


