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

### 4.4 私有分享链接

[BookmarkFileService::findByHash()](file:///d:/fz/0601-1/solo-dogfeeding/code/76-Shaarli/application/bookmark/BookmarkFileService.php#L110-L124) 支持私有链接分享：

- 私有书签可通过 `privateKey` 参数访问
- 未登录用户提供正确的 `privateKey` 也可查看单条私有书签
- Key 不匹配时抛出 `BookmarkNotFoundException`

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

## 七、总结

Shaarli 的 REST API 设计体现了以下特点：

1. **鉴权清晰**：JWT Token 机制独立于页面 Session，无状态、短有效期，适合 API 场景
2. **参数校验分层**：基础格式校验在控制器层，业务逻辑校验在服务层
3. **权限模型简洁**：API 认证通过即拥有全部权限，私有内容通过 visibility 参数灵活过滤
4. **错误体系完整**：从 400/401/404/409/500 覆盖主要错误场景，生产环境隐藏敏感错误细节
5. **CORS 友好**：统一添加跨域头，便于前端集成
