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

## 8. 总结：关键发现

| 领域 | 关键发现 |
|------|----------|
| **请求超时** | 仅有总超时（默认 30s），无连接超时；DNS 慢解析可长时间阻塞 |
| **内容类型** | Header callback 严格过滤 `text/html`，非 HTML 在头部阶段被拒绝；WebThumbnailer 无此限制 |
| **解析失败** | 静默返回 null，无异常传播，容错性好但难以排查问题 |
| **提前终止 bug** | download callback 的终止条件依赖未传入的 `$responseCode`/`$contentType` 变量，实际永不触发 |
| **表单回填** | 异步模式用户体验好；回填仅覆盖空字段，已有内容不被覆盖 |
| **SSRF 风险** | 无内网 IP 过滤、无 DNS 解析后校验、无频率限制；WebThumbnailer 扩大攻击面 |
