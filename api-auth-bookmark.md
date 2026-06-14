# API 鉴权与书签接口分析

## 一、REST API Token 鉴权机制

### 1.1 整体架构

API 鉴权通过 [ApiMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php) 中间件统一处理，在所有 API 控制器执行前进行请求校验。

中间件执行流程：
1. 检查 API 是否启用（`api.enabled` 配置）
2. 校验 JWT Token
3. 初始化包含私有书签的 BookmarkFileService
4. 捕获 ApiException 并转换为 JSON 错误响应
5. 添加 CORS 响应头

### 1.2 JWT Token 验证

Token 验证逻辑位于 [ApiUtils::validateJwtToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L24-L55)。

**Token 结构**：标准 JWT 三段式结构 `header.payload.signature`，使用 Base64Url 编码。

**验证步骤**：
1. **格式校验**：必须包含 3 个部分，且 header、payload 非空
2. **签名验证**：使用 HMAC-SHA512 算法，以 `api.secret` 为密钥，重新计算签名并比对
3. **Header 解析**：JSON 解码 header 部分
4. **Payload 解析**：JSON 解码 payload 部分
5. **签发时间校验**：
   - `iat`（issued at）必须存在
   - `iat` 不能晚于当前时间
   - Token 有效期为 9 分钟（`ApiMiddleware::$TOKEN_DURATION = 540` 秒）

**Token 获取方式**：
- 从 `Authorization` 请求头提取，格式为 `Bearer <token>`
- 兼容 `REDIRECT_HTTP_AUTHORIZATION` 环境变量（Apache 重写场景）

### 1.3 令牌 vs 页面会话

| 维度 | API Token (JWT) | 页面 Session |
|------|-----------------|-------------|
| **实现类** | [ApiMiddleware](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php) | [SessionManager](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/security/SessionManager.php) + [LoginManager](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/security/LoginManager.php) |
| **载体** | HTTP Authorization 头 | Cookie + 服务端 Session |
| **认证方式** | 无状态 JWT 签名验证 | 有状态 Session ID 匹配 |
| **有效期** | 固定 9 分钟 | 1 小时（默认）/ 1 年（记住登录） |
| **登录凭证** | `api.secret` 配置项 | 用户名 + 密码（本地/LDAP） |
| **IP 绑定** | 否 | 是（可通过配置禁用） |
| **XSRF Token** | 不需要 | 需要（[SessionManager::generateToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/security/SessionManager.php#L81-L86)） |
| **书签访问权限** | 可访问全部（含私有） | 登录后可访问全部，未登录仅公开 |

**关键区别**：
- API 通过 `setLinkDb()` 方法强制创建 `isLoggedIn=true` 的 BookmarkFileService，始终可访问私有书签
- 前端页面通过 `LoginManager::isLoggedIn()` 判断权限，未登录时只能访问公开书签

## 二、参数校验机制

### 2.1 查询参数校验

在 [Links::getLinks()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L36-L84) 中实现：

**offset 参数**：
- 使用 `ctype_digit()` 校验必须为数字字符串
- 无效时抛出 `ApiBadParametersException('Invalid offset')`
- 默认值为 0

**limit 参数**：
- 空值时使用默认值 20（`Links::$DEFAULT_LIMIT`）
- 数字字符串转换为整数
- 特殊值 `'all'` 表示无限制（转换为 `null`）
- 其他值抛出 `ApiBadParametersException('Invalid limit')`

### 2.2 路径参数校验

**ID 参数校验**：
- 使用 `is_integer_mixed()` 函数判断是否为整数（支持字符串数字）
- 无效 ID 或不存在的书签均抛出 `ApiLinkNotFoundException`
- 见于 [Links::getLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L97-L107)、[Links::putLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L155-L189)、[Links::deleteLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L202-L212)

### 2.3 请求体参数处理

[ApiUtils::buildBookmarkFromRequest()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L100-L138) 负责将请求数据转换为 Bookmark 对象：

**URL 处理**：
- 调用 `cleanup_url()` 清理 URL
- 为空时生成便签（note）类型书签

**private 字段**：
- 使用 `filter_var($input['private'], FILTER_VALIDATE_BOOLEAN)` 布尔转换
- 未提供时使用 `default_private_links` 配置

**tags 字段**：
- 兼容字符串和数组两种格式
- 字符串格式通过 `tags_str2array()` 按分隔符拆分
- 单元素数组也会再次拆分（容错处理）

**时间字段**：
- `created` 和 `updated` 使用 `DateTime::ATOM` 格式解析
- 解析失败则忽略，使用默认值

**title/description**：
- 空字符串作为默认值

### 2.4 标签接口参数校验

[Tags::getTags()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L36-L76) 与 Links 接口校验逻辑一致。

[Tags::putTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L113-L140) 额外校验：
- 请求体必须包含 `name` 字段
- 为空时抛出 `ApiBadParametersException('New tag name is required in the request body')`

## 三、书签 CRUD 接口

### 3.1 查询书签列表

**接口**：`GET /api/v1/links`

**实现**：[Links::getLinks()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L36-L84)

**查询参数**：
- `visibility`：可见性过滤（all/public/private）
- `offset`：分页偏移量，默认 0
- `limit`：每页数量，默认 20，'all' 为全部
- `searchtags`：标签搜索
- `searchterm`：全文搜索

**返回**：200 OK + 书签数组

### 3.2 查询单条书签

**接口**：`GET /api/v1/links/{id}`

**实现**：[Links::getLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L97-L107)

**路径参数**：
- `id`：书签 ID

**返回**：200 OK + 单条书签数据

### 3.3 创建书签

**接口**：`POST /api/v1/links`

**实现**：[Links::postLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L117-L142)

**请求体**：
```json
{
  "url": "https://example.com",
  "title": "Example",
  "description": "...",
  "tags": ["tag1", "tag2"],
  "private": false,
  "created": "2024-01-01T00:00:00+00:00"
}
```

**重复检测**：
- 按 URL 查重，存在重复时返回 409 Conflict + 已有书签数据

**返回**：201 Created + Location 头 + 新书签数据

### 3.4 更新书签

**接口**：`PUT /api/v1/links/{id}`

**实现**：[Links::putLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L155-L189)

**更新逻辑**：
- 通过 [ApiUtils::updateLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L148-L157) 全量覆盖字段
- 覆盖字段：title、url、description、tags、private
- 注意：ID 保持不变

**重复检测**：
- 检测 URL 是否与其他书签冲突（排除自身）
- 冲突返回 409 Conflict

**返回**：200 OK + 更新后的书签数据

### 3.5 删除书签

**接口**：`DELETE /api/v1/links/{id}`

**实现**：[Links::deleteLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L202-L212)

**返回**：204 No Content

### 3.6 书签数据格式

[ApiUtils::formatLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L65-L86) 统一格式化输出：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 书签 ID |
| `url` | string | URL，便签类型为内部路径 |
| `shorturl` | string | 短链接哈希 |
| `title` | string | 标题 |
| `description` | string | 描述 |
| `tags` | array | 标签数组 |
| `private` | bool | 是否私有 |
| `created` | string | 创建时间（ATOM 格式） |
| `updated` | string | 更新时间（ATOM 格式，空字符串表示未更新） |

## 四、私有内容访问控制

### 4.1 API 层面的私有访问

API 鉴权通过后即视为管理员权限，可访问所有书签：

- [ApiMiddleware::setLinkDb()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php#L144-L154) 创建 `isLoggedIn=true` 的 BookmarkFileService
- `visibility` 参数允许显式指定 `all`、`public`、`private` 过滤

### 4.2 BookmarkFilter 可见性过滤

[BookmarkFilter](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFilter.php) 是书签过滤核心类。

**三种可见性**：
- `all`：全部书签
- `public`：仅公开书签
- `private`：仅私有书签

**过滤逻辑**（[noFilter()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFilter.php#L144-L167)）：
```php
if ($visibility === 'all') {
    $out[$key] = $value;
} elseif ($value->isPrivate() && $visibility === 'private') {
    $out[$key] = $value;
} elseif (!$value->isPrivate() && $visibility === 'public') {
    $out[$key] = $value;
}
```

### 4.3 隐藏标签（点开头标签）

标签搜索时的特殊处理（[BookmarkFilter::filterTags()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFilter.php#L317-L415)）：

- 以 `.` 开头的标签为隐藏标签
- `public` 可见性下自动过滤掉以 `.` 开头的标签搜索
- 仅登录用户/API 可搜索隐藏标签

### 4.4 私有分享链接（仅前端页面，API 路由不暴露）

**回核结论：API 路由**不**暴露 privateKey 访问私有书签的能力。privateKey 是前端页面（Visitor/Public 路由）独享的机制。**

#### 4.4.1 前端页面路由（privateKey 生效的地方）

**生成页面路由**：[index.php L149](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/index.php#L149)
```php
$this->get('/admin/shaare/private/{hash}', '\Shaarli\Front\Controller\Admin\ShaareManageController:sharePrivate');
```

**生成逻辑**：[ShaareManageController::sharePrivate()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/front/controller/admin/ShaareManageController.php#L184-L205)
1. 校验 XSRF token（`$this->checkToken($request)`）
2. 若书签是 public，直接重定向到 `/shaare/{hash}`
3. 若书签无 `private_key`，生成 `bin2hex(random_bytes(16))`（32 字符十六进制）并保存
4. 重定向到 `/shaare/{hash}?key=<private_key>`

**访问页面路由**：[BookmarkListController::permalink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/front/controller/visitor/BookmarkListController.php#L129-L160)
```php
$privateKey = $request->getParam('key');
$bookmark = $this->container->bookmarkService->findByHash($args['hash'], $privateKey);
```

#### 4.4.2 服务层实现

[BookmarkFileService::findByHash()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFileService.php#L110-L124)
```php
public function findByHash(string $hash, string $privateKey = null): Bookmark
{
    $first = current(
        $this->bookmarks->filter(function (Bookmark $bookmark) use ($hash, $privateKey) {
            return $bookmark->getShortUrl() === $hash
                && (!$bookmark->isPrivate()
                    || true === $this->isLoggedIn
                    || (!empty($privateKey)
                        && $privateKey === $bookmark->getAdditionalContentEntry('private_key'))
                );
        })
    );
    if (false === $first) {
        throw new BookmarkNotFoundException();
    }
    return $first;
}
```

**访问条件（任一满足即可）**：
1. 书签本身是公开的（`!$bookmark->isPrivate()`）
2. 已登录（`true === $this->isLoggedIn`）—— API 场景始终为 true
3. 提供正确的 `privateKey` 且与书签存储的 `private_key` 匹配

#### 4.4.3 API 路由不暴露 privateKey

**代码事实核查**：对 `application/api/` 目录搜索 `findByHash|privateKey|private_key`，**零匹配**。

API 书签查询全部走 `id` 路径：
- [Links::getLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L97-L107)：通过 `$args['id']` 用 `is_integer_mixed()` 校验后调用 `$this->bookmarkService->get($id)`
- [Links::getLinks()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L36-L84)：通过 `search()` 按可见性过滤

API 格式化输出 [ApiUtils::formatLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L65-L86) 的字段列表为：
`id, url, shorturl, title, description, tags, private, created, updated`
——**不含 private_key 字段**，也不接受 `key` 查询参数。

#### 4.4.4 结论

| 维度 | 前端 Visitor 路由 `/shaare/{hash}?key=...` | API 路由 `/api/v1/links` |
|------|------------------------------------------|------------------------|
| 身份标识 | Session Cookie (未登录) | JWT Authorization 头 (必须登录) |
| 私有书签访问 | privateKey 查询参数 | JWT 鉴权后直接访问 |
| 定位方式 | 短链接 hash (shorturl) | 整数书签 ID (id) |
| privateKey 是否可用 | ✅ 是 | ❌ 否 |
| BookmarkFileService::isLoggedIn | 取决于 LoginManager | 始终 `true` |

### 4.5 公开链接隐藏配置

[BookmarkFileService 构造函数](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFileService.php#L62-L105)：

- `privacy.hide_public_links = true` 时，未登录用户看不到任何书签（包括公开的）
- 配合 `privacy.force_login` 实现强制登录

## 五、错误响应与错误码语义

### 5.1 异常类体系

所有 API 异常继承自抽象类 [ApiException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiException.php)。

**异常类与 HTTP 状态码映射**：

| 异常类 | HTTP 状态码 | 语义 |
|--------|------------|------|
| [ApiAuthorizationException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiAuthorizationException.php) | 401 Unauthorized | 鉴权失败 |
| [ApiBadParametersException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiBadParametersException.php) | 400 Bad Request | 请求参数无效 |
| [ApiLinkNotFoundException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiLinkNotFoundException.php) | 404 Not Found | 书签不存在 |
| [ApiTagNotFoundException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiTagNotFoundException.php) | 404 Not Found | 标签不存在 |
| [ApiInternalException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiInternalException.php) | 500 Internal Server Error | 服务器内部错误 |

### 5.2 401 鉴权错误场景

[ApiAuthorizationException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiAuthorizationException.php) 包含多种鉴权失败原因：

| 错误消息 | 触发条件 |
|----------|---------|
| `API is disabled` | `api.enabled` 配置为 false |
| `JWT token not provided` | 缺少 Authorization 头 |
| `Token secret must be set in Shaarli's administration` | `api.secret` 未配置 |
| `Invalid JWT header` | Bearer 格式不匹配 |
| `Malformed JWT token` | JWT 结构不正确 |
| `Invalid JWT signature` | 签名验证失败 |
| `Invalid JWT header` | Header JSON 解析失败 |
| `Invalid JWT payload` | Payload JSON 解析失败 |
| `Invalid JWT issued time` | 签发时间无效或过期 |

**安全设计**：
- 生产环境（非 debug 模式）统一返回 `Not authorized`，不暴露具体原因
- Debug 模式下会附加原始错误信息，便于开发调试
- 见 [ApiAuthorizationException::setMessage()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiAuthorizationException.php#L29-L33)

### 5.3 400 参数错误场景

[ApiBadParametersException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiBadParametersException.php)：

| 错误消息 | 触发位置 |
|----------|---------|
| `Invalid offset` | offset 参数非数字 |
| `Invalid limit` | limit 参数格式无效 |
| `New tag name is required in the request body` | PUT tag 缺少 name 字段 |

### 5.4 404 资源未找到

**书签未找到**：[ApiLinkNotFoundException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiLinkNotFoundException.php)
- 固定消息：`Link not found`
- 触发于 GET/PUT/DELETE 单条书签接口，ID 无效或不存在

**标签未找到**：[ApiTagNotFoundException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiTagNotFoundException.php)
- 固定消息：`Tag not found`
- 触发于 GET/PUT/DELETE 单标签接口，标签名不存在

### 5.5 409 冲突

**URL 重复冲突**：
- 创建书签时 URL 已存在 → 409
- 更新书签时 URL 与其他书签冲突 → 409
- 响应体包含已存在的书签数据
- 不是通过异常类实现，而是直接返回响应

### 5.6 500 内部错误

[ApiInternalException](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiInternalException.php)：
- 通用服务器错误
- 用于未捕获的异常或系统级错误

### 5.7 响应格式

**生产环境**：
```json
"错误消息字符串"
```

**Debug 模式**（`dev.debug = true`）：
```json
{
  "message": "错误消息",
  "stacktrace": "异常类名: 堆栈跟踪"
}
```

响应格式由 [ApiException::getApiResponseBody()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiException.php#L39-L48) 控制。

### 5.8 CORS 响应头

所有 API 响应统一添加 CORS 头（[ApiMiddleware::__invoke()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php#L65-L84)）：

- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Headers: X-Requested-With, Content-Type, Accept, Origin, Authorization`
- `Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS`

## 六、其他 API 接口

### 6.1 实例信息

**接口**：`GET /api/v1/info`

**实现**：[Info::getInfo()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Info.php#L27-L43)

**返回内容**：
- `global_counter`：书签总数
- `private_counter`：私有书签数
- `settings.title`：站点标题
- `settings.header_link`：页头链接
- `settings.timezone`：时区
- `settings.enabled_plugins`：启用的插件
- `settings.default_private_links`：默认私有设置
- `settings.tags_separator`：标签分隔符

### 6.2 标签接口

| 接口 | 方法 | 实现 |
|------|------|------|
| 获取标签列表 | GET /api/v1/tags | [Tags::getTags()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L36-L76) |
| 获取单标签 | GET /api/v1/tags/{tagName} | [Tags::getTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L89-L98) |
| 重命名标签 | PUT /api/v1/tags/{tagName} | [Tags::putTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L113-L140) |
| 删除标签 | DELETE /api/v1/tags/{tagName} | [Tags::deleteTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L153-L173) |

## 八、JWT 安全深度分析

### 8.1 签名比对：未使用 hash_equals 存在时序攻击风险

**代码位置**：[ApiUtils::validateJwtToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L31-L34)

```php
$genSign = Base64Url::encode(hash_hmac('sha512', $parts[0] . '.' . $parts[1], $secret, true));
if ($parts[2] != $genSign) {
    throw new ApiAuthorizationException('Invalid JWT signature');
}
```

**问题分析**：
- 使用 `!=` 操作符进行字符串比较，而不是 `hash_equals()`
- **时序攻击（Timing Attack）风险**：普通字符串比较会在第一个不匹配字符处返回，攻击者可通过测量响应时间差异逐字节推断正确签名
- 代码库全局搜索 `hash_equals` 无任何匹配，确认未使用时序安全的比较函数
- 虽然 JWT 签名长度固定（HMAC-SHA512 为 64 字节 Base64Url 编码约 86 字符），但时序攻击在理论上仍可行

### 8.2 alg:none 可达性结论：不可达

**代码位置**：[ApiUtils::validateJwtToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L24-L55)

**完整验证流程**（严格按代码执行顺序）：
```
步骤 1: 拆分 JWT 为 3 段 → 非 3 段则 "Malformed JWT token"
步骤 2: 固定算法重算签名
        $genSign = Base64Url::encode(hash_hmac('sha512', $parts[0] . '.' . $parts[1], $secret, true))
步骤 3: 签名比对 if ($parts[2] != $genSign) → 不等则 "Invalid JWT signature"
步骤 4: Base64Url::decode($parts[0]) 然后 json_decode → 失败则 "Invalid JWT header"
步骤 5: Base64Url::decode($parts[1]) 然后 json_decode → 失败则 "Invalid JWT payload"
步骤 6: 检查 iat 存在且有效 → "Invalid JWT issued time"
```

**可达性分析**：

构造攻击 token：`{"alg":"none"}.{"iat":<valid>}.<empty_or_any>`

追踪执行：
1. 步骤 2 硬编码调用 `hash_hmac('sha512', header_b64.payload_b64, $secret, true)` — 无论 header 中写什么算法，这里始终以 HS512 重新计算
2. 攻击者无法用 `alg:none` 的空签名匹配 HS512 输出（除非猜到 `$secret`）
3. 因此在**步骤 3 必然抛出 `Invalid JWT signature`**，永远到不了步骤 4 解析 alg 字段

**结论：alg:none 不可达。** 这不是巧合安全，而是"先签名再解析"的顺序 + 硬编码算法的双重保障。代码在 header 被解析之前就已经用固定算法 HS512 重算了签名并比对。

**仍建议显式校验 alg 的理由（防御性编程）**：
- 即使当前不可达，未来若有人重构将步骤 4、5 提前到步骤 2 之前，或改为从 alg 动态分派算法，就会引入漏洞
- 遵循 JWT 安全最佳实践，显式校验是更严谨的做法

**修复建议**（定位为防御性编程，而非修复实际漏洞）：
```php
// 在签名比对成功后追加 alg 校验（防未来回归）
$header = json_decode(Base64Url::decode($parts[0]));
if ($header === null || !isset($header->alg) || $header->alg !== 'HS512') {
    throw new ApiAuthorizationException('Invalid JWT algorithm');
}
```

### 8.3 JWT Replay 链路分析：9 分钟窗口缺 jti 一次性消耗

**代码事实**：
- 全局搜索 `jti|JTI` 无任何匹配，代码库从未使用 `jti`（JWT ID）claim
- Token 有效期固定 9 分钟（[ApiMiddleware::$TOKEN_DURATION = 540](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php#L25)）
- `iat` 校验仅做时间窗口判断，不做任何一次性消耗记录

**iat 校验代码**：[ApiUtils::validateJwtToken() L43-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L43-L54)
```php
if (empty($payload->iat) || $payload->iat > time()) {
    throw new ApiAuthorizationException('Invalid JWT issued time');
}
if (time() - $payload->iat > ApiMiddleware::$TOKEN_DURATION) {
    throw new ApiAuthorizationException('Invalid JWT issued time');
}
```

**完整 Replay 链路追踪**：

```
攻击前提：攻击者已截获一个合法 JWT 请求（中间人/HTTPS 降级/日志泄露/Referer 泄漏）

链路 1: 写操作重放
  T=0s    合法用户: POST /api/v1/links  Authorization: Bearer <token> (iat=T0)
          → ApiMiddleware::checkRequest() → checkToken() → validateJwtToken() 通过
          → Links::postLink() → ApiUtils::buildBookmarkFromRequest() → BookmarkFileService::add()
          → BookmarkFileService::save() → 写入磁盘 → 返回 201 + Location

  T=300s  攻击者: 重放同一请求（相同 Authorization 头）
          → validateJwtToken(): time()-iat = 300 < 540 → 通过 ✅
          → Links::postLink() → URL 查重: 若原 URL 已存在 → 返回 409 Conflict
          → 但若原书签已被删除 → 重放成功创建新书签 ⚠️

链路 2: 删除操作重放
  T=0s    合法用户: DELETE /api/v1/links/42  Authorization: Bearer <token> (iat=T0)
          → 成功删除书签 42 → 返回 204 No Content

  T=300s  攻击者: 重放同一请求
          → validateJwtToken(): 通过 ✅
          → Links::deleteLink(): 若书签 42 已被恢复 → 再次删除 ⚠️
          → 若书签 42 不存在 → 返回 404（无害但暴露了 ID 存在性）

链路 3: 标签操作重放（影响面最大）
  T=0s    合法用户: DELETE /api/v1/tags/oldtag  Authorization: Bearer <token>
          → Tags::deleteTag() → 遍历所有含 oldtag 的书签 → Bookmark::deleteTag()
          → BookmarkFileService::set() × N → BookmarkFileService::save()

  T=300s  攻击者: 重放同一请求
          → validateJwtToken(): 通过 ✅
          → 若 oldtag 被重新添加到书签 → 再次删除所有含该标签的书签标签 ⚠️
          → 副作用: N 个书签 updated 时间戳被重置 + N 条历史记录 + 缓存全量失效

链路 4: PUT rename 重放
  T=0s    合法用户: PUT /api/v1/tags/foo  {name: "bar"}  Authorization: Bearer <token>
          → Tags::putTag() → 所有含 foo 的书签 renameTag("foo", "bar")

  T=300s  攻击者: 重放同一请求
          → validateJwtToken(): 通过 ✅
          → 若 foo 已被重新添加 → 再次 rename → 但此时 bar 可能已存在 → 触发合并 ⚠️
```

**与页面 Session 的 XSRF 对比**：

| 维度 | API JWT Token | 页面 Session + XSRF Token |
|------|--------------|--------------------------|
| 一次性消耗 | 无（同一 token 可用 9 分钟） | 有（每次表单 XSRF token 刷新） |
| 重放窗口 | 9 分钟固定 | 仅限当前会话 + 单次提交 |
| 写操作保护 | 依赖 HTTPS 保密性 | 依赖 XSRF token 一次性 + SameSite Cookie |

**官方文档佐证**：[REST-API.md L34-44](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/doc/md/REST-API.md#L34-L44) 的 PHP 示例每次请求都**重新生成 token**，但这是客户端约定而非服务器强制。

**缓解方案对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| 缩短有效期至 60 秒 | 无需改代码，改常量即可 | 对客户端时钟同步要求更高 |
| 添加 jti + 服务端黑名单 | 一次性消耗，严格防重放 | 引入有状态，破坏无状态设计；需要存储 + 清理过期 jti |
| 绑定请求方法+路径+Body 到签名 | Token 仅对特定请求有效 | 破坏"一个 token 多个请求"的使用模式 |
| 要求客户端每次请求重新生成 token | 已有官方示例支持 | 仍非服务器强制，恶意客户端可复用 |

## 九、CORS 安全分析

### 9.1 Allow-Origin: * 配合 Authorization 的安全风险

**代码位置**：[ApiMiddleware::__invoke()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php#L76-L83)

```php
return $response
    ->withHeader('Access-Control-Allow-Origin', '*')
    ->withHeader(
        'Access-Control-Allow-Headers',
        'X-Requested-With, Content-Type, Accept, Origin, Authorization'
    )
    ->withHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
```

**问题分析**：
- `Access-Control-Allow-Origin: *` 表示允许任意域名访问
- `Access-Control-Allow-Headers` 明确包含 `Authorization`，允许携带认证头
- **安全风险**：
  - 根据 W3C CORS 规范，当 `Allow-Origin` 为 `*` 时，浏览器**不应**允许 `withCredentials` 为 `true` 的请求
  - 但配置本身存在语义冲突：允许跨域 + 允许携带认证头
  - 如果浏览器实现存在漏洞，可能导致 CSRF 攻击：恶意网站可诱导用户携带 JWT Token 发起跨域请求
- 更安全的配置应使用白名单域名，并动态设置 `Allow-Origin`

### 9.2 OPTIONS 请求是否经过 ApiMiddleware

**路由配置**：[index.php 第 185-199 行](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/index.php#L185-L199)

```php
$app->group('/api/v1', function () {
    $this->get('/info', '\Shaarli\Api\Controllers\Info:getInfo')->setName('getInfo');
    // ... 其他路由定义
})->add('\Shaarli\Api\ApiMiddleware');
```

**Slim 3 框架特性**：
- Slim 3 内置 `Slim\Middleware\MethodOverrideMiddleware`，但未显式配置
- Slim 3 对 OPTIONS 请求的处理逻辑：
  - 若显式定义了 `$app->options()` 路由，则匹配该路由
  - 若未定义，Slim 的 `Router` 会自动处理匹配路径的 OPTIONS 请求，返回允许的方法列表
  - **关键点**：group 级别的 middleware 对自动生成的 OPTIONS 响应**同样生效**

**代码追踪结论**：
- API 组路由中未显式定义 OPTIONS 路由
- Slim 框架自动生成的 OPTIONS 响应**仍会经过 ApiMiddleware**
- 流程：
  1. 浏览器发送 OPTIONS 预检请求到 `/api/v1/links`
  2. Slim 路由器匹配到 `/api/v1` group
  3. 执行 ApiMiddleware 中间件
  4. ApiMiddleware 的 `checkRequest()` 会校验 JWT Token！
  5. 预检请求通常不携带 Authorization 头 → 抛出 401 错误
  6. 实际 CORS 预检失败

**这是一个 Bug**：CORS 预检请求（OPTIONS）不应要求认证，浏览器不会在预检请求中携带 Authorization 头。

## 十、is_integer_mixed 函数追踪

**定义位置**：[Utils.php 第 362-369 行](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/Utils.php#L362-L369)

```php
function is_integer_mixed($input)
{
    if (is_array($input) || is_bool($input) || is_object($input)) {
        return false;
    }
    $input = strval($input);
    return ctype_digit($input) || (startsWith($input, '-') && ctype_digit(substr($input, 1)));
}
```

**功能说明**：
- 检查输入是否为"混合类型整数"，支持字符串数字
- 排除数组、布尔值、对象类型
- 支持正整数和负整数字符串
- 内部使用 `ctype_digit()` 校验

**调用位置**：
- [Links::getLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L99)
- [Links::putLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L157)
- [Links::deleteLink()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Links.php#L204)

## 十一、ApiException 掩码逻辑分析

### 11.1 生产环境错误掩码

**核心逻辑**：[ApiAuthorizationException::setMessage()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiAuthorizationException.php#L29-L33)

```php
public function setMessage($message)
{
    $original = $this->debug === true ? ': ' . $this->getMessage() : '';
    $this->message = $message . $original;
}
```

**调用流程**：
1. 代码中抛出异常时设置具体错误消息，如 `throw new ApiAuthorizationException('JWT token not provided')`
2. 在 `getApiResponse()` 中调用 `$this->setMessage('Not authorized')`
3. 根据 `debug` 标志决定是否附加原始消息

**掩码效果**：

| debug 模式 | 原始异常消息 | 最终输出 |
|-----------|-------------|---------|
| `false`（生产） | `JWT token not provided` | `Not authorized` |
| `false`（生产） | `Invalid JWT signature` | `Not authorized` |
| `false`（生产） | `Invalid JWT issued time` | `Not authorized` |
| `true`（开发） | `JWT token not provided` | `Not authorized: JWT token not provided` |

**安全设计意图**：
- 防止攻击者通过错误消息枚举系统状态（如区分"token 不存在"和"token 过期"）
- 避免泄露 JWT 校验的具体实现细节

### 11.2 响应体格式掩码

**代码位置**：[ApiException::getApiResponseBody()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/exceptions/ApiException.php#L39-L48)

```php
protected function getApiResponseBody()
{
    if ($this->debug !== true) {
        return $this->getMessage();  // 仅返回字符串
    }
    return [
        'message' => $this->getMessage(),
        'stacktrace' => get_class($this) . ': ' . $this->getTraceAsString()
    ];
}
```

**掩码效果**：
- 生产环境：简单字符串响应，无结构信息
- 开发环境：JSON 对象，包含完整消息和堆栈跟踪

## 十二、putTag rename 操作副作用分析

**代码位置**：[Tags::putTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L113-L140)

### 12.1 执行流程

```
PUT /api/v1/tags/{tagName}
    ↓
1. 检查标签是否存在（bookmarksCountPerTag）
2. 解析请求体获取新标签名
3. 搜索所有包含旧标签的书签
4. 遍历每个书签：
   ├─ Bookmark::renameTag($from, $to)
   ├─ BookmarkFileService::set($bookmark, false)  // save=false，暂不写入
   └─ History::updateLink($bookmark)              // 记录历史
5. 调用 BookmarkFileService::save()                // 批量写入磁盘
```

### 12.2 renameTag 内部逻辑

**代码位置**：[Bookmark::renameTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/Bookmark.php#L513-L522)

```php
public function renameTag(string $fromTag, string $toTag): void
{
    if (($pos = array_search($fromTag, $this->tags ?? [])) !== false) {
        if (in_array($toTag, $this->tags ?? []) !== false) {
            $this->deleteTag($fromTag);  // 目标标签已存在 → 合并删除
        } else {
            $this->tags[$pos] = trim($toTag);  // 直接替换
        }
    }
}
```

### 12.3 副作用分析

| 操作 | 副作用 |
|------|-------|
| **标签合并** | 若书签同时包含 `fromTag` 和 `toTag`，则仅删除 `fromTag`，`toTag` 保留 |
| **标签去重** | `trim($toTag)` 会去除新标签名的首尾空白 |
| **批量更新** | 循环调用 `set($bookmark, false)` 多次更新内存，最后一次 `save()` 写入磁盘，性能较好 |
| **历史记录** | 每个被修改的书签都会产生一条 updateLink 历史记录 |
| **缓存失效** | `save()` 会调用 `PageCacheManager::invalidateCaches()` 使所有页面缓存失效 |
| **时间戳更新** | `BookmarkFileService::set()` 会自动设置 `$bookmark->setUpdated(new DateTime())`，所有受影响书签的 updated 时间被更新为当前时间 |
| **ID 不变** | 书签 ID 保持不变 |

## 十三、deleteTag 操作完整路径追踪

### 13.1 完整调用链

```
DELETE /api/v1/tags/{tagName}
    ↓ 路由解析 [index.php:196](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/index.php#L196)
    ↓ ApiMiddleware 校验 JWT [ApiMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php)
    ↓ Tags::deleteTag() [Tags.php:153-173](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/controllers/Tags.php#L153-L173)
        ↓ 1. bookmarksCountPerTag() 检查标签存在
        ↓ 2. search() 查找所有包含该标签的书签
        ↓ 3. 循环处理每个书签：
        │     ↓ Bookmark::deleteTag($tag) [Bookmark.php:539-545](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/Bookmark.php#L539-L545)
        │     ↓ BookmarkFileService::set($bookmark, false) [BookmarkFileService.php:200-217](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFileService.php#L200-L217)
        │     │     ↓ 检查 isLoggedIn 权限
        │     │     ↓ 检查书签存在
        │     │     ↓ $bookmark->validate() 校验
        │     │     ↓ 设置 updated 时间戳
        │     │     └ 更新内存中的书签数据
        │     └ History::updateLink($bookmark) 记录历史
        └ 4. BookmarkFileService::save() [BookmarkFileService.php:309-319](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFileService.php#L309-L319)
              ↓ reorder() 重新排序
              ↓ BookmarkIO::write() 写入数据文件
              └ invalidateCaches() 页面缓存失效
```

### 13.2 deleteTag 内部逻辑

**代码位置**：[Bookmark::deleteTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/Bookmark.php#L539-L545)

```php
public function deleteTag(string $tag): void
{
    while (($pos = array_search($tag, $this->tags ?? [])) !== false) {
        unset($this->tags[$pos]);
        $this->tags = array_values($this->tags);  // 重排索引
    }
}
```

**特点**：
- 使用 `while` 循环可删除标签数组中所有匹配项（防止重复标签）
- 删除后调用 `array_values()` 重新索引数组，避免产生稀疏数组

### 13.3 BookmarkFileService::set() 权限检查

**代码位置**：[BookmarkFileService::set()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFileService.php#L200-L217)

```php
public function set(Bookmark $bookmark, bool $save = true): Bookmark
{
    if (true !== $this->isLoggedIn) {
        throw new Exception(t('You\'re not authorized to alter the datastore'));
    }
    // ... 后续操作
}
```

**关键安全点**：
- API 场景下 `isLoggedIn=true`（由 ApiMiddleware::setLinkDb() 强制设置）
- 前端页面场景下由 LoginManager 控制
- 这是第二层权限校验，防止中间件绕过

## 十四、安全问题总结与修复建议

### 14.1 已识别的安全问题

| 问题 | 严重程度 | 位置 |
|------|---------|------|
| 未使用 hash_equals 比对签名 | 中 | [ApiUtils.php:32](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L32) |
| 未校验 JWT alg 字段 | 低 | [ApiUtils.php:36-39](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiUtils.php#L36-L39) |
| CORS Allow-Origin: * + Allow Authorization | 中 | [ApiMiddleware.php:77](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php#L77) |
| OPTIONS 预检请求需要 JWT 认证 | 高（可用性） | [ApiMiddleware.php:99](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/api/ApiMiddleware.php#L99) |

### 14.2 修复建议

**1. 时序攻击防护**：
```php
// 使用 hash_equals 替换 !=
if (!hash_equals($genSign, $parts[2])) {
    throw new ApiAuthorizationException('Invalid JWT signature');
}
```

**2. alg 字段校验**：
```php
$header = json_decode(Base64Url::decode($parts[0]));
if ($header === null || !isset($header->alg) || $header->alg !== 'HS512') {
    throw new ApiAuthorizationException('Invalid JWT algorithm');
}
```

**3. CORS 安全配置**：
```php
// 替代方案 1：使用白名单
$allowedOrigins = ['https://trusted.com'];
$origin = $request->getHeaderLine('Origin');
if (in_array($origin, $allowedOrigins)) {
    $response = $response->withHeader('Access-Control-Allow-Origin', $origin);
}

// 替代方案 2：若必须允许任意域，禁止携带凭证
// 移除 Authorization 从 Allow-Headers
```

**4. OPTIONS 请求跳过鉴权**：
```php
protected function checkRequest($request)
{
    if (! $this->conf->get('api.enabled', true)) {
        throw new ApiAuthorizationException('API is disabled');
    }
    // OPTIONS 预检请求不需要鉴权
    if ($request->getMethod() !== 'OPTIONS') {
        $this->checkToken($request);
    }
}
```

## 七、总结

Shaarli 的 REST API 设计体现了以下特点：

1. **鉴权清晰**：JWT Token 机制独立于页面 Session，无状态、短有效期，适合 API 场景
2. **参数校验分层**：基础格式校验在控制器层，业务逻辑校验在服务层
3. **权限模型简洁**：API 认证通过即拥有全部权限，私有内容通过 visibility 参数灵活过滤
4. **错误体系完整**：从 400/401/404/409/500 覆盖主要错误场景，生产环境隐藏敏感错误细节
5. **CORS 友好**：统一添加跨域头，但 Slim 3 OPTIONS 自动合成 Route 存在缺头的可用性 Bug
6. **alg:none 实际不可达**：硬编码 HS512 + 先签名后解析的顺序形成双重保护，但仍建议显式校验以防未来回归
7. **Replay 风险存在**：9 分钟窗口内无 jti 一次性消耗，写操作可被重放
8. **privateKey 与 API 隔离**：私有书签分享密钥机制仅存在于前端 Visitor 路由，API 路由完全不暴露该入口
9. **安全改进空间**：JWT 签名比对、CORS/OPTIONS 联合修复、Replay 窗口缩短等方面存在可优化点
