# Shaarli 插件加载与路由扩展 — 代码理解

## 一、整体架构概览

Shaarli 的插件系统围绕 [PluginManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php) 构建，采用**全局函数命名约定**而非面向对象的接口约束。插件生命周期可分为四个阶段：

```
启动加载 (index.php)
    ↓
元数据解析 (.meta INI 文件)
    ↓
PHP 文件 include_once → init 函数 → 路由注册
    ↓
运行时 Hook 执行 (管道式顺序调用)
```

核心入口位于 [index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/index.php#L96-L97)：

```php
$pluginManager = new PluginManager($conf);
$pluginManager->load($conf->get('general.enabled_plugins', []));
```

---

## 二、插件元数据定义与解析

### 2.1 元数据文件格式

每个插件必须包含一个 `<pluginName>.meta` 的 INI 格式文件，位于 `plugins/<pluginName>/` 目录下。以 [demo_plugin.meta](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/demo_plugin.meta) 为例：

```ini
description="A demo plugin covering all use cases..."
parameters="DEMO_PLUGIN_PARAMETER;DEMO_PLUGIN_OTHER_PARAMETER"
parameter.DEMO_PLUGIN_PARAMETER="This is a parameter dedicated..."
parameter.DEMO_PLUGIN_OTHER_PARAMETER="Other demo parameter"
```

元数据字段说明：

| 字段 | 说明 |
|------|------|
| `description` | 插件描述，支持 `t()` 国际化翻译 |
| `parameters` | 分号分隔的参数名列表 |
| `parameter.<NAME>` | 单个参数的描述文本（可选） |

### 2.2 元数据解析流程

解析逻辑位于 [PluginManager::getPluginsMeta()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L230-L270)：

1. 用 `glob()` 扫描 `plugins/*` 所有子目录
2. 读取每个目录下的 `<name>.meta` 文件，使用 PHP 原生 `parse_ini_file()` 解析
3. 将 `parameters` 字符串按分号拆分为数组，结构化为 `$plugins[$name]['parameters'][$param]['value'|'desc']`
4. 注入 `order` 字段：`array_search($plugin, $this->authorizedPlugins)`，值为 `false` 表示未启用

### 2.3 元数据真实风险

- **INI 注入风险**：`parse_ini_file()` 对特殊字符敏感，但当前通过文件系统路径隔离，攻击者需要写入权限才能利用
- **参数值无序存储**：参数保存在 `$conf['plugins']` 的**扁平一维数组**中（见 [PluginsController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L66-L68)），不同插件若使用相同参数名会**互相覆盖**，无命名空间隔离

---

## 三、插件加载流程

### 3.1 加载入口

加载逻辑在 [PluginManager::load()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L84-L107)：

```php
public function load($authorizedPlugins)
{
    $this->authorizedPlugins = $authorizedPlugins;
    $dirs = glob(self::$PLUGINS_PATH . '/*', GLOB_ONLYDIR);
    $dirnames = array_map('basename', $dirs);

    foreach ($this->authorizedPlugins as $plugin) {
        $index = array_search($plugin, $dirnames);
        if ($index === false) {
            continue;   // 配置中存在但目录不存在 → 静默跳过
        }
        try {
            $this->loadPlugin($dirs[$index], $plugin);
        } catch (PluginFileNotFoundException $e) {
            error_log($e->getMessage());
        } catch (\Throwable $e) {
            // 不中断后续插件，仅记录错误
            $error = $plugin . t(' [plugin incompatibility]: ') . $e->getMessage();
            $this->errors = array_unique(array_merge($this->errors, [$error]));
        }
    }
}
```

### 3.2 单插件加载步骤与顺序不可逆性

核心私有方法 [PluginManager::loadPlugin()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L163-L202) 执行五步操作，顺序至关重要：

```
[1] 文件校验
    ↓  通过
[2] include_once $pluginFilePath   ← PHP 代码被执行，全局函数注册
    ↓
[3] 调用 {plugin}_init($conf)      ← 可能产生副作用（写配置、注册翻译域等）
    ↓  返回值合并进 $this->errors
[4] 调用 {plugin}_register_routes() → validateRouteRegistration()
    ↓  任一校验失败
    ↓  throw PluginInvalidRouteException   ← ★ 此时步骤 2、3 已不可逆
    ↓
[5] $this->loadedPlugins[] = $pluginName
```

1. **文件校验**：检查目录与 `<pluginName>.php` 是否存在，不存在抛出 `PluginFileNotFoundException`
2. **代码载入**：使用 `include_once $pluginFilePath` —— 注意这是**无条件 include**，无沙箱、无语法预检查
3. **初始化函数**：调用 `{pluginName}_init($conf)`（如果存在），其返回值被当作错误数组追加到 `$this->errors`
4. **路由注册**：调用 `{pluginName}_register_routes()`，返回路由数组经 `validateRouteRegistration()` 校验后存入 `$this->registeredRoutes[$pluginName][]`
5. **标记已加载**：`$this->loadedPlugins[] = $pluginName`

### 3.3 Hook 容错与错误累积逻辑

`$this->errors` 数组是整个插件系统的**全局错误累积器**，错误来源有三处，各自的容错策略不一致：

#### 来源一：load() 顶层 catch \Throwable

位于 [PluginManager.php L102-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L102-L104)：

```php
} catch (\Throwable $e) {
    $error = $plugin . t(' [plugin incompatibility]: ') . $e->getMessage();
    $this->errors = array_unique(array_merge($this->errors, [$error]));
}
```

- **容错行为**：当前插件跳过后续步骤、不加入 `loadedPlugins`、后续插件继续加载
- **累积方式**：`array_merge($this->errors, [$error])` —— 包裹成单元素数组后合并
- **去重方式**：`array_unique()` —— 纯字符串去重，同一插件同一异常消息多次出现只保留一条

#### 来源二：_init() 返回值数组

位于 [PluginManager.php L178-L182](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L178-L182)：

```php
$errors = call_user_func($initFunction, $this->conf);
if (!empty($errors)) {
    $this->errors = array_merge($this->errors, $errors);
}
```

- **容错行为**：即使返回错误数组，步骤 4、5 仍正常执行（路由注册+标记加载）—— **不会因为 init 报错而禁用插件**
- **累积方式**：直接 `array_merge($this->errors, $errors)` —— 假定返回值是数组
- **真实风险**：若插件 `_init()` 返回字符串/`true` 等非数组值，`array_merge(null|string, ...)` 会产生 PHP Warning：`array_merge(): Expected parameter 1 to be an array, null given`，该 Warning 在生产环境 `error_reporting` 下通常被静默，导致用户看不到任何错误提示
- **不一致性**：此处**未使用 `array_unique()`**，同一条错误消息若 init 返回多次重复元素会重复累积（与 executeHooks / load() 的行为不一致）

#### 来源三：executeHooks() 运行时 catch \Throwable

位于 [PluginManager.php L138-L143](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L138-L143)：

```php
try {
    $data = call_user_func($hookFunction, $data, $this->conf);
} catch (\Throwable $e) {
    $error = $plugin . t(' [plugin incompatibility]: ') . $e->getMessage();
    $this->errors = array_unique(array_merge($this->errors, [$error]));
}
```

- **容错行为**：当前插件 Hook 跳过（`$data` 不被该插件修改）、`$data` 保留上一个插件的结果、后续插件 Hook 继续执行
- **累积方式**：与 load() 顶层一致 —— `[$error]` 单元素数组合并 + `array_unique()`
- **数据连续性风险**：该插件 Hook 不 return 也不抛异常时，`call_user_func` 返回 `null`，`$data = null` 会覆盖管道前序所有修改（见 4.5 节）
- **管道完整性风险**：若插件抛异常，`$data` **不会被赋值**（PHP try 块内的赋值语句被中断），因此 `$data` 保持上一个插件的返回值 —— 这是"正确"的行为但与"不 return 导致 null"的行为**不一致**，给插件开发者造成困惑

### 3.4 错误累积的系统性问题汇总

| 问题 | 位置 | 后果 |
|------|------|------|
| **三处累积 API 不一致** | load() 用 `[$error]` + unique；_init() 直接 `$errors` 无 unique；executeHooks() 用 `[$error]` + unique | 同一种错误在不同阶段产生的累积条目数量不同，界面显示不稳定 |
| **错误消息无结构化** | 所有错误都是拼接的字符串：`<pluginName> [plugin incompatibility]: <message>` | 前端界面无法按插件聚合错误，无法区分"加载失败"与"运行时 Hook 失败" |
| **错误不持久化** | `$this->errors` 只存在于当前请求的 PluginManager 对象 | 页面渲染时 `plugins_errors` 变量只在当前页面显示，下次刷新就消失，管理员容易错过 |
| **_init 返回值无类型校验** | L178-182 无 `is_array($errors)` 检查 | 插件返回字符串导致 PHP Warning，Warning 被吞后无任何错误记录 |

### 3.5 路由校验失败抛出特定异常的风险

路由校验失败时的异常抛出逻辑位于 [PluginManager::loadPlugin()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L191-L199)：

```php
if ($routes !== null) {
    foreach ($routes as $route) {
        if (static::validateRouteRegistration($route)) {
            $this->registeredRoutes[$pluginName][] = $route;
        } else {
            throw new PluginInvalidRouteException($pluginName);
        }
    }
}
```

[PluginInvalidRouteException](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/exception/PluginInvalidRouteException.php) 是特定异常类，但**不携带插件名参数**（其构造函数忽略参数，硬编码消息为 "trying to register invalid route."）。

**真实风险链**：

| 时序步骤 | 问题 | 后果 |
|----------|------|------|
| 步骤 2 `include_once` 已执行 | 全局函数已注册到 PHP 运行时 | 即使后续异常，函数仍留在内存中，可被其他插件通过 `function_exists()` 发现 |
| 步骤 3 `_init()` 已执行完 | `_init()` 产生的副作用（`$conf->set(...)`、`$conf->write(true)`、注册语言域等）**不可回滚** | 出现"半加载"状态：配置被改了、但该插件不参与 Hook 和路由 |
| 异常在 `foreach ($routes)` 循环中抛出 | 只校验了前 N-1 个路由合法，第 N 个不合法 | 前 N-1 个已写入 `registeredRoutes[$pluginName][]`，但第 N 个导致异常；外层 catch 不清理已注册的部分路由 → **不一致状态** |
| `throw new PluginInvalidRouteException($pluginName)` | 异常类构造函数忽略 `$pluginName`，硬编码消息 | 错误日志仅记录 "trying to register invalid route."，不包含是哪个插件出问题，管理员难以定位 |
| 外层 catch `\Throwable` | 异常被吞掉 → 该插件不执行步骤 5（不加入 `loadedPlugins`） | 已 include 的 PHP 代码、已执行的 init 副作用、已部分注册的路由都残留，但 Hook 永远不会为该插件调用 |
| **已部分注册的路由仍生效** | index.php 中 `getRegisteredRoutes()` 会返回这些残留路由 | `/plugin/<name>/...` 下前 N-1 条路径被 Slim 注册，但用户在管理界面看不到该插件已启用（因为不 `loadedPlugins`），产生**幽灵路由** |

测试案例 [PluginManagerTest.php 中 invalid route 相关测试](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/tests/PluginManagerTest.php) 证明了这一点：使用非法 HTTP 方法 → 异常被吞 → `getLoadedPlugins()` 不包含该插件 → 但全局 Hook 函数仍可被 `function_exists()` 返回 true。

### 3.6 加载顺序的真实影响

加载顺序完全由配置项 `general.enabled_plugins` 的数组顺序决定。关键影响点：

| 影响面 | 说明 |
|--------|------|
| **Hook 执行顺序** | `executeHooks()` 按 `$loadedPlugins` 顺序遍历，后执行的插件覆盖前者写入的 `$data` 键 |
| **路由注册顺序** | Slim 路由按注册顺序匹配，先注册优先（`/plugin/<name>/...` 分组内部） |
| **init 副作用** | 前一个插件 `_init()` 写入的 `$conf` 会被后续插件读取到（如 demo_plugin 设置 `translation.extensions.demo`） |
| **错误掩盖** | 前一个插件抛出 `\Throwable` 被 catch 吞掉，后续插件仍继续加载，但用户只看到最后一次错误提示 |
| **错误累积顺序** | 三个错误来源按加载先后进入 `$this->errors`，相同插件的 init 错误与 Hook 错误不聚合显示 |

配置加载顺序的校验逻辑位于 [ConfigPlugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/config/ConfigPlugin.php#L21-L72) 的 `save_plugin_config()`，它会：
- 过滤掉非 `order_` 前缀且非插件目录名的字段
- 用 `validate_plugin_order()` 检查 order 值是否唯一（不允许重复排序号）
- 新启用的插件（无 order 字段）被追加到数组末尾

---

## 四、Hook 执行机制与返回值合并

### 4.1 Hook 命名约定

函数名格式由 [PluginManager::buildHookName()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L215-L218) 定义：

```
hook_<pluginName>_<hookName>
```

例如 demo_plugin 的 header Hook 为 `hook_demo_plugin_render_header`。

### 4.2 执行流程

核心方法 [PluginManager::executeHooks()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L118-L150)：

```php
public function executeHooks($hook, &$data, $params = [])
{
    // 1. 注入元数据上下文（5 个键）
    foreach ([
        'target' => '_PAGE_',
        'loggedin' => '_LOGGEDIN_',
        'basePath' => '_BASE_PATH_',
        'rootPath' => '_ROOT_PATH_',
        'bookmarkService' => '_BOOKMARK_SERVICE_',
    ] as $parameter => $metaKey) {
        if (array_key_exists($parameter, $params)) {
            $data[$metaKey] = $params[$parameter];
        }
    }

    // 2. 顺序执行所有已加载插件的对应 Hook（独立 try-catch）
    foreach ($this->loadedPlugins as $plugin) {
        $hookFunction = $this->buildHookName($hook, $plugin);
        if (function_exists($hookFunction)) {
            try {
                $data = call_user_func($hookFunction, $data, $this->conf);
            } catch (\Throwable $e) {
                $error = $plugin . t(' [plugin incompatibility]: ') . $e->getMessage();
                $this->errors = array_unique(array_merge($this->errors, [$error]));
            }
        }
    }

    // 3. 清理元数据上下文
    foreach (['_PAGE_', '_LOGGEDIN_', '_BASE_PATH_', '_ROOT_PATH_', '_BOOKMARK_SERVICE_'] as $metaKey) {
        unset($data[$metaKey]);
    }
}
```

### 4.3 返回值合并策略 — 管道式顺序覆盖

**关键设计**：`$data` 既是引用参数又被赋值返回值。这意味着：

1. 第 1 个插件的返回值 → 成为第 2 个插件的输入
2. 第 2 个插件的返回值 → 成为第 3 个插件的输入
3. N 个插件串行处理，最后一个插件的输出为最终结果

这是典型的**管道（Pipeline）模式**，合并规则是**完全覆盖**而非递归合并：

```php
// 伪代码示意
$data = hook_A($data, $conf);   // A 修改后
$data = hook_B($data, $conf);   // B 接收 A 的结果，再修改
$data = hook_C($data, $conf);   // C 接收 B 的结果，再修改
// 最终 $data 是 C 的返回值
```

### 4.4 全量 Hook 触发点清单

代码中 Hook 被触发的位置按功能分类如下（全部通过 `executePageHooks()` 或 `executeDefaultHooks()` 间接调用 `executeHooks()`）：

#### 渲染类 Hook（页面输出阶段）

| Hook 名 | 触发位置 | $data 初始内容 | 目标模板 |
|---------|----------|----------------|----------|
| `render_includes` | [ShaarliVisitorController::executeDefaultHooks()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php#L78-L97) | `[]` | `plugins_includes`（`<head>` 内） |
| `render_header` | 同上 | `[]` | `plugins_header`（头部导航后） |
| `render_footer` | 同上 | `[]` | `plugins_footer`（页脚前） |
| `render_linklist` | [BookmarkListController.php L120](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/BookmarkListController.php#L120) 和 [L156](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/BookmarkListController.php#L156) | `['links'=>..., 'pagetitle'=>...]` | linklist 模板 |
| `render_picwall` | [PictureWallController.php L46](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/PictureWallController.php#L46) | `['linksToDisplay'=>...]` | picwall 模板 |
| `render_tagcloud` | [TagCloudController.php L84](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/TagCloudController.php#L84) | `['tags'=>..., 'search_tags'=>...]` | tag.cloud 模板 |
| `render_taglist` | 同上 | 同上 | tag.list 模板 |
| `render_feed` | [FeedController.php L49](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/FeedController.php#L49) | Feed 构建器数组 | feed.rss / atom 模板 |
| `render_daily` | [DailyController.php L66](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/DailyController.php#L66) | `['linksToDisplay'=>...]` | daily 模板 |
| `render_tools` | [ToolsController.php L25](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ToolsController.php#L25) | `['pageabsaddr'=>..., 'sslenabled'=>...]` | tools 模板 |
| `render_editlink` | [ShaarePublishController.php L57](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaarePublishController.php#L57)（批量）和 [L167](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaarePublishController.php#L167)（单条） | `buildFormData()` 返回的表单数据数组 | editlink 模板（书签编辑表单） |

#### 持久化类 Hook（数据变更阶段）

| Hook 名 | 触发位置 | $data 初始内容 | 执行时数据状态 |
|---------|----------|----------------|--------------|
| `save_plugin_parameters` | [PluginsController.php L61](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L61) | **整个 POST 数组引用** `$parameters = $request->getParams()` | 在 `escape()` 之前执行，值为原始用户输入；插件可修改任意字段 |
| `save_link` | [ShaareManageController.php L131](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaareManageController.php#L131)、[L173](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaareManageController.php#L173)、[L274](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaareManageController.php#L274) | `$formatter->format($bookmark)` — 已**整体 escape 过的书签数组** | Hook 返回值 → `$bookmark->fromArray($data)` → 写入数据库；若插件对已转义字符串再处理后 return，会产生双转义或正则失效 |
| `delete_link` | [ShaareManageController.php L55](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaareManageController.php#L55) | `$formatter->format($bookmark)` — 已格式化的书签数组 | 在 `bookmarkService->remove()` 之前执行；demo_plugin 在此 Hook 中直接 `exit()` 证明此处可终止整个请求 |

#### 过滤类 Hook（不走 executeHooks，独立实现）

| Hook 名 | 触发位置 | 机制说明 |
|---------|----------|----------|
| `filter_search_entry` | [BookmarkFilter.php L149](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/bookmark/BookmarkFilter.php#L149)、[L242](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/bookmark/BookmarkFilter.php#L242)、[L368](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/bookmark/BookmarkFilter.php#L368)、[L429](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/bookmark/BookmarkFilter.php#L429) | 返回 bool，短路语义 |

### 4.5 返回值合并的真实风险

| 风险 | 说明 |
|------|------|
| **键名冲突即覆盖** | 两个插件向 `$data['buttons_toolbar'][]` 追加元素是安全的，但如果都写 `$data['custom_key'] = ...`，后者覆盖前者，无任何警告 |
| **未 return 导致数据丢失** | 插件若在 Hook 函数中忘记 `return $data`，`call_user_func` 返回 `null`，后续所有插件收到 `null`，整个管道数据被清空 |
| **类型污染** | 一个插件返回非数组类型（如 `true`/字符串），导致后续插件对 `$data` 做数组操作时产生 `TypeError`，该错误会被 catch 吞掉但后续插件不执行 |
| **上下文元数据冲突** | 注入的 `_PAGE_`、`_LOGGEDIN_` 等 5 个键若插件本身使用同名键，会被覆盖后再 unset，导致插件数据丢失 |
| **save_link 返回值直接写库** | `$bookmark->fromArray($data)` 不做额外转义，插件若在 Hook 中手动 `unescape()` 了某个字段后 return，会把未转义 HTML 直接写入数据库 |

### 4.6 filter_search_entry 与 executeHooks 容错策略差异

两种 Hook 调用机制的容错设计存在**系统性不一致**，这是最容易被忽略的架构风险：

| 维度 | executeHooks() | filterSearchEntry() |
|------|---------------|---------------------|
| **try-catch 包裹** | ✅ 每个插件独立 `catch (\Throwable)` | ❌ **完全没有 try-catch** |
| **异常后果** | 记录错误，继续下一个插件 | 异常直接冒泡到上层 → 整个搜索结果页面 500 错误 |
| **返回值类型** | `array`（管道覆盖） | `bool`（短路判断） |
| **短路语义** | ❌ 不支持，全量遍历（除非异常） | ✅ 任一返回 `false` 立即停止后续插件 |
| **元数据注入** | 注入 `_PAGE_`/`_LOGGEDIN_`/`_BASE_PATH_`/`_ROOT_PATH_`/`_BOOKMARK_SERVICE_` 五个键，执行后清理 | 无元数据注入，直接传 Bookmark 对象 + context 数组 |
| **Conf 对象传入** | 每个 Hook 函数接收 `$this->conf` 作为第二参数 | ❌ 不传入，Hook 函数签名无 Conf（只能用全局变量或自己读配置文件） |
| **性能优化** | 每次循环都调用 `function_exists()` 检查 | 首次调用时预加载 callable 列表到 `$filterSearchEntryHooks`（[PluginManager.php L329-L340](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L329-L340)），后续直接调用 |
| **错误累积** | 异常消息写入 `$this->errors` | ❌ 不累积，异常直接抛出（上层也未做全局 try-catch 捕获） |
| **预加载失效** | N/A | 若插件在 `loadedPlugins` 之后才定义 filter_search_entry 函数（如通过其他插件的 include），预加载列表不会包含它 → 永远不会被调用 |

**风险场景**（测试用例可验证）：在 `hook_xxx_filter_search_entry` 中执行 `new UnknownClass()` → PHP Fatal Error/Error 未被捕获 → 整个书签列表页面（RSS/PicWall/Daily/TagCloud 等所有走 BookmarkFilter 的页面）崩溃，而同样的代码在 `render_header` Hook 中只会被记为一条兼容性错误后继续运行。

**API 签名差异**：
- `executeHooks()` 调用的是：`call_user_func($hookFunction, $data, $this->conf)`
- `filterSearchEntry()` 调用的是：`$filterSearchEntryHook($bookmark, $context)`（第二参数是 context 数组，无 Conf）

插件开发者如果把 render_* Hook 的写法复制粘贴到 filter_search_entry，会发现 `$conf` 变量不存在导致报错——而这个错误不会被捕获。

---

## 五、配置管理与配置表单

### 5.1 插件参数存储

参数扁平存储在 `ConfigManager` 的 `plugins` 命名空间下，见 [PluginsController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L54-L84)：

```php
// POST 请求区分两种表单：
if (isset($parameters['parameters_form'])) {
    // 保存插件参数
    unset($parameters['parameters_form']);
    unset($parameters['token']);
    foreach ($parameters as $param => $value) {
        $this->container->conf->set('plugins.' . $param, escape($value));
    }
} else {
    // 保存启用/禁用与排序
    $this->container->conf->set('general.enabled_plugins', save_plugin_config($parameters));
}
```

参数读取通过 [ConfigPlugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/config/ConfigPlugin.php#L111-L127) 的 `load_plugin_parameter_values()` 合并进插件元数据：

```php
function load_plugin_parameter_values($plugins, $conf)
{
    $out = $plugins;
    foreach ($plugins as $name => $plugin) {
        if (empty($plugin['parameters'])) { continue; }
        foreach ($plugin['parameters'] as $key => $param) {
            if (!empty($conf[$key])) {
                $out[$name]['parameters'][$key]['value'] = $conf[$key];
                // ⚠ 未调用 unescape()，直接把已转义字符串赋值给 value
            }
        }
    }
    return $out;
}
```

### 5.2 配置表单渲染

- 展示页面 [PluginsController::index()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L22-L49) 将插件分为 `enabledPlugins`（按 order 排序）和 `disabledPlugins`
- 每个已启用插件可通过 `save_plugin_parameters` Hook 在保存前介入修改（如 demo_plugin 自动追加 `_SUFFIX`）

### 5.3 配置表单的真实风险

| 风险 | 说明 |
|------|------|
| **参数名全局扁平** | `escape($value)` 做了转义，但参数名本身是插件开发者自定义的字符串。若恶意插件选择参数名 `general.title`，会因 `conf->set('plugins.general.title', ...)` 写入错误命名空间吗？不会——因为前缀是 `plugins.`，但插件间同名参数互相覆盖 |
| **save_plugin_parameters Hook 可篡改任意字段** | 该 Hook 在参数保存前执行，接收整个 `$_POST` 数组的引用，插件可修改其它插件参数甚至删除 `token` 字段 |
| **无类型校验** | 所有参数值都被 `escape()` 转成字符串保存，布尔值、数值都会变成字符串，插件需自行转换 |
| **_init 期间可写配置** | `demo_plugin_init()` 中直接 `$conf->set(...); $conf->write(true)`，插件可在加载时覆盖任意配置项（包括 `credentials.*`） |

### 5.4 参数保存与读取的双向转义陷阱

#### 5.4.1 escape / unescape 函数定义

[Utils.php L96-L126](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/Utils.php#L96-L126) 定义了一对核心转义函数：

```php
function escape($input)
{
    if (null === $input) { return null; }
    // bool/int/float/DateTime 直接原样返回
    if (is_bool($input) || is_int($input) || is_float($input) || $input instanceof DateTimeInterface) {
        return $input;
    }
    // 数组递归转义，键名和值都 escape
    if (is_array($input)) {
        $out = [];
        foreach ($input as $key => $value) {
            $out[escape($key)] = escape($value);
        }
        return $out;
    }
    // ⚠ 第 4 参数 double_encode=false：不会对已有的 &amp; 再次编码
    return htmlspecialchars($input, ENT_COMPAT, 'UTF-8', false);
}

function unescape($str) {
    return htmlspecialchars_decode($str);
}
```

关键参数：
- `ENT_COMPAT`：只转双引号（`"`），不转单引号（`'`）
- `double_encode=false`：遇到已转义的实体（如 `&amp;`）不重复编码
- 数组递归：`escape($key)` 和 `escape($value)` 都执行（键名也被转义！）

**unescape 仅被使用一次**：全代码库搜索显示只有 [BookmarkMarkdownFormatter::reverseEscapedHtml()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/formatter/BookmarkMarkdownFormatter.php#L229-L232) 调用了它，这暗示"保存时转义、读取时反转义"的惯例并未在全项目一致推行。

#### 5.4.2 插件参数的完整转义链路（陷阱所在）

```
管理员输入值: <script>alert('x')</script>
    │
    ▼  POST /admin/plugins - PluginsController::save()
    │
    ├─► 步骤 1: executePageHooks('save_plugin_parameters', $parameters)
    │       └─ 插件收到原始用户输入（未转义）
    │       └─ demo_plugin 在此追加 _SUFFIX 到值末尾
    │       └─ 值仍是原始字符串: <script>alert('x')</script>_SUFFIX
    │
    ├─► 步骤 2: foreach ($parameters as $param => $value)
    │       └─ $conf->set('plugins.' . $param, escape($value));
    │       └─ escape() 执行:
    │           "<script>alert('x')</script>_SUFFIX"
    │           → &lt;script&gt;alert('x')&lt;/script&gt;_SUFFIX
    │           （⚠ ENT_COMPAT 不转单引号 → ' 仍然是 '）
    │
    ▼  写入配置文件 data/config.*.php
    │   值存储为: &lt;script&gt;alert('x')&lt;/script&gt;_SUFFIX
    │
    ├─► 读取路径 A: 管理界面回显 (PluginsController::index)
    │       └─ load_plugin_parameter_values() → 不 unescape
    │       └─ RainTPL 模板渲染变量 {$param.value}
    │       └─ → 浏览器显示为文本 &lt;script&gt;... ✅（安全，但如果用户以为
    │           看到的就是实际值，会产生"值被自动修改了"的困惑）
    │
    ├─► 读取路径 B: 插件 Hook 中直接读 conf
    │       └─ $value = $this->conf->get('plugins.DEMO_PLUGIN_PARAMETER')
    │       └─ → 拿到的是已转义的 &lt;script&gt;alert('x')&lt;/script&gt;_SUFFIX
    │       └─ 插件若用 if (endsWith($value, '_SUFFIX')) → ✅ 匹配成功
    │       └─ 插件若用 if (str_contains($value, "<script>")) → ❌ 匹配失败
    │       └─ 插件若再调用 escape() 输出 → double_encode=false 不变，但如果
    │           传给 eval/exec/正则等上下文，语义已错位
    │
    ├─► 读取路径 C: 插件输出到 HTML 属性（双引号）
    │       └─ → &lt;script&gt; 在 HTML 中显示为文本 ✅
    │
    ├─► 读取路径 D: 插件输出到 JavaScript 字符串（单引号）
    │       └─ <script>var x = '{$value}';</script>
    │       └─ → 值中的单引号 ' 未被 ENT_COMPAT 转义
    │       └─ → 实际上若原输入含 '，如 <a onclick='alert(1)'>x</a>
    │           → escape() 后为 &lt;a onclick='alert(1)'&gt;x&lt;/a&gt;
    │           → 放入 JS 单引号字符串中: '...onclick='alert(1)'...'
    │           → 单引号提前闭合 → JS 注入 → XSS ❌
    │
    └─► 二次保存路径 E: 管理界面再次保存（用户没改值）
            └─ 模板回显的是已转义值（&lt;script&gt;...）
            └─ 用户提交时浏览器把 &lt; 作为字面量发送（不是 <）
            └─ save_plugin_parameters Hook 再次处理
            └─ escape(&lt;script&gt;...) → double_encode=false → 保持不变
            └─ → 这一次是"正确的"（不会变成 &amp;lt;）
            └─ 但如果用户手动编辑为 &amp;test; → escape() 不会再编码 →
               最终渲染为 &test; 实体（可能触发 HTML 实体解析异常）
```

#### 5.4.3 转义链路上的已知问题汇总

| 问题 | 详情 |
|------|------|
| **不对称转义** | 保存 escape、读取不 unescape → 插件开发者若不读源码会误以为拿到的是原始字符串 |
| **ENT_COMPAT 不转单引号** | 保存值中单引号原样保留，若插件拼接到 JS 字符串（用单引号定界）会直接注入 |
| **键名也被转义** | `escape()` 递归处理数组时键名也 escape，若插件在 `save_plugin_parameters` Hook 中向 POST 添加了含特殊字符的字段名，被转义后 `conf->set('plugins.<转义后键名>', ...)`，后续读取时必须用同样转义后的键名才能找到 |
| **save_plugin_parameters Hook 在 escape 之前执行** | [PluginsController.php L59-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L59-L68)：Hook 执行早于 escape，插件若在 Hook 中把参数值改成嵌套数组（含对象），escape 的数组递归会遍历并转义所有子元素，可能破坏插件预期的数据结构 |
| **Markdown 格式化器反向 unescape** | [BookmarkMarkdownFormatter.php L231](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/formatter/BookmarkMarkdownFormatter.php#L231) 明确调用 `unescape()`，说明至少一条渲染链路假定了"存储时已转义"；但插件输出不走这条链路，导致相同存储值在不同渲染路径下的表现不一致 |
| **save_link Hook 接收已 escape 的数组** | [ShaarePublishController](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaarePublishController.php) 在保存书签时先对整个书签数组执行 `escape()`，再传给 `save_link` Hook；若插件在 Hook 中对字段做字符串操作（如 `preg_replace('/<script>/' ...)`），操作的是已转义的 `&lt;script&gt;` 文本，正则匹配会失败，插件逻辑静默失效 |
| **delete_link 触发顺序敏感** | `save_link` 执行在 `bookmarkService->set()` 之前（返回值会被 `fromArray()` 反向写回对象），而 `delete_link` 执行在 `bookmarkService->remove()` 之前——但 demo_plugin 中 `hook_demo_plugin_delete_link()` 直接 `exit()`，说明插件可以通过终止请求来**阻止删除操作**（等同于 veto 权限），而这个能力在官方文档中并未说明 |

---

## 六、自定义路由注册与管理员鉴权机制

### 6.1 路由注册流程

插件通过 `<pluginName>_register_routes()` 函数返回路由定义数组，例如 [demo_plugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/demo_plugin.php#L66-L75)：

```php
function demo_plugin_register_routes(): array
{
    return [
        [
            'method'   => 'GET',
            'route'    => '/custom',
            'callable' => 'Shaarli\DemoPlugin\DemoPluginController:index',
        ],
    ];
}
```

### 6.2 路由校验

[PluginManager::validateRouteRegistration()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L352-L366) 仅检查三项：

```php
protected static function validateRouteRegistration(array $input): bool
{
    // 1. method 必须存在且属于 GET/PUT/PATCH/POST/DELETE
    if (!array_key_exists('method', $input)
        || !in_array(strtoupper($input['method']), ['GET', 'PUT', 'PATCH', 'POST', 'DELETE'])) {
        return false;
    }
    // 2. callable 键必须存在（不检查是否可执行）
    if (!array_key_exists('callable', $input)) {
        return false;
    }
    return true;
    // ⚠ 注意：route 键、callable 可执行性均不校验！
}
```

### 6.3 路由挂载到 Slim

在 [index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/index.php#L174-L182)：

```php
$app->group('/plugin', function () use ($pluginManager) {
    foreach ($pluginManager->getRegisteredRoutes() as $pluginName => $routes) {
        $this->group('/' . $pluginName, function () use ($routes) {
            foreach ($routes as $route) {
                $this->{strtolower($route['method'])}(
                    '/' . ltrim($route['route'], '/'),
                    $route['callable']
                );
            }
        });
    }
})->add('\Shaarli\Front\ShaarliMiddleware');  // ⚠ 仅 ShaarliMiddleware
```

最终路由路径：`/plugin/<pluginName>/<route>`

### 6.4 Slim 全局路由组的中间件链对照

为了理解插件路由的鉴权风险，需要对照 index.php 中全部路由组的中间件挂载：

| 路由组 | 前缀 | 挂载的中间件 | 鉴权要求 |
|--------|------|-------------|----------|
| **管理员页面** | `/admin` | `ShaarliAdminMiddleware` | ✅ 强制登录，未登录重定向 `/login` |
| **插件自定义路由** | `/plugin` | `ShaarliMiddleware`（仅此一个） | ❌ **不强制登录**，完全依赖插件控制器自行检查 |
| **公开页面** | 无组或其他 | `ShaarliMiddleware` | 取决于 `privacy.hide_public_links` + `privacy.force_login` 配置 |

### 6.5 两层中间件的鉴权逻辑差异

#### ShaarliMiddleware（插件路由在用）

源码见 [ShaarliMiddleware.php L40-L62](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/ShaarliMiddleware.php#L40-L62)：

```php
public function __invoke(Request $request, Response $response, callable $next): Response
{
    $this->initBasePath($request);
    try {
        if (
            !$this->container->loginManager->isLoggedIn()
            && $this->container->conf->get('privacy.hide_public_links')
            && $this->container->conf->get('privacy.force_login')
            && !in_array($next->getName(), ['login', 'processLogin', 'atom', 'rss'], true)
        ) {
            throw new UnauthorizedException();  // 重定向到 login
        }
        return $next($request, $response);
    } catch (UnauthorizedException $e) {
        return $response->withRedirect($this->container->basePath . '/login?returnurl=' . ...);
    }
}
```

鉴权规则（对插件路由生效）：
- 默认：**完全公开**（`hide_public_links=false` 或 `force_login=false` 时，所有访问都放行）
- 严格模式（两个隐私配置都为 true）：仅未登录用户访问 `login/processLogin/atom/rss` 四个路由名时放行，**其他路由名（包括插件路由的 Slim 自动生成名）都会被重定向**
- 但注意：插件路由的 `$next->getName()` 返回的是 Slim 自动生成的内部名称（如 `route42`），不在白名单中 → 严格模式下插件路由也会被强制登录

**风险**：插件路由是否鉴权完全取决于 Shaarli 的全局隐私配置，而非插件自身的安全需求。当 Shaarli 处于"公开分享"模式时，所有插件自定义路由（包括执行删除/导出等写操作的 POST 路由）对匿名用户**完全开放**。

#### ShaarliAdminMiddleware（插件路由**未用**）

源码见 [ShaarliAdminMiddleware.php L15-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/ShaarliAdminMiddleware.php#L15-L26)：

```php
public function __invoke(Request $request, Response $response, callable $next): Response
{
    $this->initBasePath($request);
    if (true !== $this->container->loginManager->isLoggedIn()) {
        $returnUrl = urlencode($this->container->environment['REQUEST_URI']);
        return $response->withRedirect($this->container->basePath . '/login?returnurl=' . $returnUrl);
    }
    return parent::__invoke($request, $response, $next);  // 再执行 ShaarliMiddleware 的逻辑
}
```

鉴权规则（供对比）：
- 无条件检查 `isLoggedIn()`，未登录直接 302 重定向
- 通过后再执行 ShaarliMiddleware 的 open-shaarli 检查（此时已登录不会触发）
- 插件开发者若想保护自己的控制器，必须**手动**：要么继承 `ShaarliAdminController`（靠 `/admin` 路由组的中间件），要么在每个 action 方法开头 `$this->container->loginManager->isLoggedIn()` 判断

### 6.6 DemoPluginController 的鉴权真相

[DemoPluginController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/DemoPluginController.php) 声明：

```php
class DemoPluginController extends ShaarliAdminController
```

这意味着：
1. 它继承了 `ShaarliAdminController → ShaarliVisitorController` 的所有方法（含 `assignView`、`executePageHooks` 等）
2. 但！**它并不走 `/admin` 路由组**，而是挂载在 `/plugin/demo_plugin/` 下
3. 因此 `ShaarliAdminMiddleware` 的登录检查**不会自动执行**
4. DemoPluginController 的每个 action 方法**必须自行检查登录状态**（或在插件路由级别手动 `->add(ShaarliAdminMiddleware::class)`，但 Slim 路由注册代码中未提供此能力）

**结论**：即使插件控制器继承了 `ShaarliAdminController`，在插件路由组下也**不会自动获得管理员鉴权**。DemoPluginController 中的 `checkToken()` 等方法是可用的，但 `isLoggedIn()` 的检查需要插件开发者主动调用。

### 6.7 路由注册的真实风险汇总

| 风险 | 说明 |
|------|------|
| **中间件仅为 ShaarliMiddleware** | 插件路由组**不经过 `ShaarliAdminMiddleware`**！默认所有插件自定义路由在公开模式下**无需登录即可访问**，包括 POST/PUT/DELETE 写操作 |
| **callable 不校验存在性** | 注册时不检查 `callable` 是否可调用，运行时 Slim 才会报 500 错误，无降级策略 |
| **route 路径校验缺失** | `validateRouteRegistration()` 甚至不检查 `route` 键是否存在，若缺失则 Slim 注册时会用空字符串路径，导致 `/plugin/<name>` 本身被匹配 |
| **路径遍历风险** | `ltrim($route['route'], '/')` 只去除左斜杠，若插件传入 `../other-plugin/foo`，Slim 仍会解析为 `/plugin/<name>/../other-plugin/foo` 即 `/plugin/other-plugin/foo`，理论上可伪装为其他插件的路由 |
| **方法名动态调用** | `$this->{strtolower($route['method'])}()` 使用可变方法名，虽然 method 已被白名单限制，但在 PHP 层面 `get`/`post` 等仍是 Slim 的公开方法 |
| **路由无法指定中间件** | `_register_routes()` 返回值的结构不支持 `middleware` 字段，插件无法为单个路由挂载 `ShaarliAdminMiddleware`，只能在控制器代码中做运行时检查 |
| **路由名无法自定义** | 无法为路由指定 name，导致 `checkOpenShaarli()` 中的白名单匹配无法识别插件路由（`$next->getName()` 是 Slim 内部生成的名字），严格隐私模式下要么全放行要么全拦截 |

---

## 七、安全隔离的系统性风险

### 7.1 代码执行层面

| 风险点 | 位置 | 说明 |
|--------|------|------|
| **无沙箱的 include** | [PluginManager.php L175](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L175) | `include_once $pluginFilePath` 直接执行 PHP 代码。任何能写入 `plugins/` 目录的人（或被上传漏洞利用）均可执行任意代码 |
| **全局命名空间污染** | 所有插件 | 插件的 Hook 函数、`_init()` 均定义在全局命名空间。两个插件同名函数会导致 PHP Fatal error（Cannot redeclare），直接让站点瘫痪 |
| **\Throwable 被选择性吞掉** | [PluginManager.php L102-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L102-L104) 和 [L140-L142](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L140-L142) | `load()` 和 `executeHooks()` 会吞异常，但**无法捕获用户态 `exit()`/`die()`**——demo_plugin 的 `hook_demo_plugin_delete_link()` 就直接 `exit()`，会终止整个请求，无任何中间层可阻止 |
| **Bookmark 对象可变** | [PluginManager.php L317](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L317) | `filter_search_entry` Hook 接收 `Bookmark` 对象，文档注释说"should NOT be altered"但无强制，插件可调用 setter 修改书签内容（修改不会自动持久化，但会影响后续格式化输出） |
| **filter_search_entry 无 try-catch** | [PluginManager.php L306-L323](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L306-L323) | 一个插件的 filter 出错 → 整个搜索页面 500，与 executeHooks 的容错策略不一致 |

### 7.2 权限控制层面

| 风险点 | 位置 | 说明 |
|--------|------|------|
| **插件路由默认公开（非严格模式）** | [index.php L182](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/index.php#L182) | `/plugin/*` 组只加了 `ShaarliMiddleware`，不含管理员校验。`hide_public_links=false` 时未登录用户可直接访问所有插件自定义路由 |
| **继承 ShaarliAdminController ≠ 自动鉴权** | 插件路由组挂载方式 | 即使控制器类继承了 Admin 基类，只要不挂在 `/admin` 组下，`ShaarliAdminMiddleware` 就不会执行 |
| **_init 阶段写配置** | 各插件 `_init()` | 插件可在 `_init()` 中调用 `$conf->set('credentials.hash', ...)` 甚至 `$conf->write(true)` 重置管理员密码，此时登录会话已建立但 CSRF token 检查不适用 |
| **Hook 接收 Conf 引用** | [PluginManager.php L139](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L139) | **每个 executeHooks 调用**都把 `$this->conf` 作为第二参数传给插件，插件可在任意 Hook（包括公开的 `render_includes` 在匿名用户访问时）中读取和修改全部配置 |
| **save_plugin_parameters 无边界** | [PluginsController.php L61](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L61) | 该 Hook 接收整个 POST 数据引用，可修改任何字段包括 CSRF token、order 等；执行早于 `checkToken()` 之后但早于 `escape()` |
| **delete_link Hook 可 veto 删除** | demo_plugin 示例 | 在删除前执行的 Hook 中调用 `exit()` 可阻止书签被删除（无官方文档说明此能力） |

### 7.3 数据输出层面

| 风险点 | 位置 | 说明 |
|--------|------|------|
| **Hook 返回数据直接渲染** | [ShaarliVisitorController.php L88-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php#L88-L96) | Hook 返回的 `$data` 被直接 `assignView` 到模板变量 `plugins_header` / `plugins_footer` / `plugins_includes`，模板中使用 `{$plugins_footer}` 原样输出（无 autoescape）——插件注入的 `<script>` 直接执行 |
| **demo_plugin 中的示范** | [demo_plugin.php L205-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/demo_plugin.php#L205-L210) | 官方演示插件就直接拼接 HTML 字符串：`'<marquee>...</marquee>'`，说明插件输出默认不转义是"设计行为" |
| **参数转义不对称** | 保存 escape，读取不 unescape | 插件读取到的是 HTML 实体文本，若在 `<script>` 中使用（如 `JSON.parse('{$param}')`），会因实体未反转导致 JSON 解析失败，或者若插件手动 `unescape()` 后再输出，可能引入 XSS |
| **单引号未转义** | `ENT_COMPAT` 模式 | 参数值中的单引号 `'` 在 escape 后仍为 `'`，放入模板的 JS 单引号字符串中直接闭合，构成 JS 注入 |

### 7.4 加载过程的竞态与部分状态

| 风险点 | 说明 |
|--------|------|
| **加载中断的部分状态** | 如果第 N 个插件 `_init()` 中调用了 `exit()`，前面 N-1 个插件已加载、路由已注册但后续插件不加载，产生不一致状态；`exit()` 无法被 catch 捕获 |
| **路由校验失败的幽灵路由** | 前 N-1 条路由合法、第 N 条非法 → 前 N-1 条路由写入 `registeredRoutes` 后才抛异常 → 这部分路由最终仍被 Slim 挂载 → 管理界面看不到插件启用，但路由可访问 |
| **无重载机制** | `include_once` 保证单个请求内不会重复加载，但配置变更后必须刷新整个 PHP 生命周期（FastCGI 下需重启进程），无热重载；若插件文件在运行时被修改，只有下次 PHP 进程启动才会生效 |
| **错误数组 array_merge 累积** | `$this->errors = array_unique(array_merge($this->errors, $errors))`，若某插件 init 返回非数组（如字符串），`array_merge` 会产生 Warning，被错误报告设置吞掉；若 `$this->errors` 尚未初始化（PHP 7.4+ 下访问未声明属性会产生 Warning），也会导致错误累积失效 |
| **多插件相同 _init 副作用冲突** | 插件 A `_init()` 调用 `$conf->set('translation.extensions.foo', 'xxx')` 后 `$conf->write(true)`；插件 B 同样 `set('translation.extensions.foo', 'yyy')` 再 write → 插件 B 覆盖 A；如果 A 还读取了自己写入的值（如注册语言域），会读到 B 写入的内容 |

---

## 八、核心文件索引

| 文件 | 职责 |
|------|------|
| [PluginManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php) | 插件加载、Hook 执行（两处容错机制）、路由注册、元数据解析、错误累积 |
| [ConfigPlugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/config/ConfigPlugin.php) | 插件排序校验、参数值合并（直接读取不 unescape） |
| [ConfigManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/config/ConfigManager.php) | 配置读写（含插件参数扁平存储） |
| [PluginsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php) | 插件管理页与配置保存（save_plugin_parameters Hook 在 escape 之前执行） |
| [Utils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/Utils.php) | escape() / unescape() 定义，ENT_COMPAT + double_encode=false |
| [index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/index.php) | 应用启动、路由组挂载（插件组只用 ShaarliMiddleware） |
| [ShaarliVisitorController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php) | 基础控制器，封装 executeDefaultHooks / executePageHooks |
| [ShaarliMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/ShaarliMiddleware.php) | 前端路由中间件（插件路由仅用它，取决于隐私配置） |
| [ShaarliAdminMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/ShaarliAdminMiddleware.php) | 管理员中间件（强制登录；插件路由组未使用） |
| [ShaarliAdminController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaarliAdminController.php) | 管理员控制器基类（继承不等于自动鉴权） |
| [ShaareManageController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/ShaareManageController.php) | save_link / delete_link Hook 触发点 |
| [BookmarkFilter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/bookmark/BookmarkFilter.php) | filter_search_entry 4 处调用点（均无 try-catch） |
| [PluginInvalidRouteException.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/exception/PluginInvalidRouteException.php) | 路由校验异常类（构造函数忽略插件名，硬编码消息） |
| [demo_plugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/demo_plugin.php) | 官方演示插件（展示了 exit、save_plugin_parameters、路由注册等所有用法） |
| [DemoPluginController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/DemoPluginController.php) | 演示插件自定义控制器（继承 ShaarliAdminController 但不自动鉴权） |
| [PluginManagerTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/tests/PluginManagerTest.php) | 插件系统单元测试（覆盖错误累积、路由异常、元数据解析等） |
