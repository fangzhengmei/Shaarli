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
            $this->errors = array_unique(array_merge($this->errors, [$error]));
        }
    }
}
```

### 3.2 单插件加载

核心私有方法 [PluginManager::loadPlugin()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L163-L202) 执行四步操作：

1. **文件校验**：检查目录与 `<pluginName>.php` 是否存在，不存在抛出 `PluginFileNotFoundException`
2. **代码载入**：使用 `include_once $pluginFilePath` —— 注意这是**无条件 include**，无沙箱、无语法预检查
3. **初始化函数**：调用 `{pluginName}_init($conf)`（如果存在），其返回值被当作错误数组追加到 `$this->errors`
4. **路由注册**：调用 `{pluginName}_register_routes()`，返回路由数组经 `validateRouteRegistration()` 校验后存入 `$this->registeredRoutes[$pluginName][]`
5. **标记已加载**：`$this->loadedPlugins[] = $pluginName`

### 3.3 加载顺序的真实影响

加载顺序完全由配置项 `general.enabled_plugins` 的数组顺序决定。关键影响点：

| 影响面 | 说明 |
|--------|------|
| **Hook 执行顺序** | `executeHooks()` 按 `$loadedPlugins` 顺序遍历，后执行的插件覆盖前者写入的 `$data` 键 |
| **路由注册顺序** | Slim 路由按注册顺序匹配，先注册优先（`/plugin/<name>/...` 分组内部） |
| **init 副作用** | 前一个插件 `_init()` 写入的 `$conf` 会被后续插件读取到（如 demo_plugin 设置 `translation.extensions.demo`） |
| **错误掩盖** | 前一个插件抛出 `\Throwable` 被 catch 吞掉，后续插件仍继续加载，但用户只看到最后一次错误提示 |

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
    // 1. 注入元数据上下文
    foreach (['target'=>'_PAGE_', 'loggedin'=>'_LOGGEDIN_', ...] as $p => $k) {
        if (array_key_exists($p, $params)) {
            $data[$k] = $params[$p];
        }
    }

    // 2. 顺序执行所有已加载插件的对应 Hook
    foreach ($this->loadedPlugins as $plugin) {
        $hookFunction = $this->buildHookName($hook, $plugin);
        if (function_exists($hookFunction)) {
            $data = call_user_func($hookFunction, $data, $this->conf);
        }
    }

    // 3. 清理元数据上下文
    foreach (['_PAGE_', '_LOGGEDIN_', ...] as $k) {
        unset($data[$k]);
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

### 4.4 特殊 Hook：filter_search_entry

[PluginManager::filterSearchEntry()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L306-L323) 不走 `executeHooks()`，而是单独实现：

- 返回值为 `bool`，任何一个插件返回 `false` 即**短路**（后续插件不再调用），该书签被过滤掉
- 为性能考虑，使用 `loadFilterSearchEntryHooks()` 预加载可调用函数名列表，避免每次搜索都重复调用 `function_exists()`

### 4.5 返回值合并的真实风险

| 风险 | 说明 |
|------|------|
| **键名冲突即覆盖** | 两个插件向 `$data['buttons_toolbar'][]` 追加元素是安全的，但如果都写 `$data['custom_key'] = ...`，后者覆盖前者，无任何警告 |
| **未 return 导致数据丢失** | 插件若在 Hook 函数中忘记 `return $data`，`call_user_func` 返回 `null`，后续所有插件收到 `null`，整个管道数据被清空 |
| **类型污染** | 一个插件返回非数组类型（如 `true`/字符串），导致后续插件对 `$data` 做数组操作时产生 `TypeError`，该错误会被 catch 吞掉但后续插件不执行 |
| **上下文元数据冲突** | 注入的 `_PAGE_`、`_LOGGEDIN_` 等键若插件本身使用同名键，会被覆盖后再 unset，导致插件数据丢失 |

---

## 五、配置管理与配置表单

### 5.1 插件参数存储

参数扁平存储在 `ConfigManager` 的 `plugins` 命名空间下，见 [PluginsController::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L54-L84)：

```php
// POST 请求区分两种表单：
if (isset($parameters['parameters_form'])) {
    // 保存插件参数
    foreach ($parameters as $param => $value) {
        $this->container->conf->set('plugins.' . $param, escape($value));
    }
} else {
    // 保存启用/禁用与排序
    $this->container->conf->set('general.enabled_plugins', save_plugin_config($parameters));
}
```

参数读取通过 [ConfigPlugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/config/ConfigPlugin.php#L111-L127) 的 `load_plugin_parameter_values()` 合并进插件元数据。

### 5.2 配置表单渲染

- 展示页面 [PluginsController::index()](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L22-L49) 将插件分为 `enabledPlugins`（按 order 排序）和 `disabledPlugins`
- 每个已启用插件可通过 `save_plugin_parameters` Hook 在保存前介入修改（如 demo_plugin 自动追加 `_SUFFIX`）

### 5.3 配置表单的真实风险

| 风险 | 说明 |
|------|------|
| **参数名全局扁平** | `escape($value)` 做了转义，但参数名本身是插件开发者自定义的字符串。若恶意插件选择参数名 `general.title`，会因 `conf->set('plugins.general.title', ...)` 写入错误命名空间吗？不会——因为前缀是 `plugins.`，但插件间同名参数互相覆盖 |
| **save_plugin_parameters Hook 可篡改任意字段** | 该 Hook 在参数保存前执行，接收整个 `$_POST` 数组的引用，插件可修改其它插件参数甚至删除 `token` 字段 |
| **无类型校验** | 所有参数值都被 `escape()` 转成字符串保存，布尔值、数值都会变成字符串，插件需自行转换 |
| **_init 期间可写配置** | `demo_plugin_init()` 中直接 `$conf->set(...); $conf->write(true)`，插件可在加载时覆盖任意配置项（包括 `credentials.*`），而此时登录校验尚未完成（`index.php` 中 `checkLoginState()` 在 `$pluginManager->load()` 之前执行完毕，但插件目录里的代码本身就需要管理员安装） |

---

## 六、自定义路由注册机制

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
    // 2. callable 键必须存在
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
})->add('\Shaarli\Front\ShaarliMiddleware');
```

最终路由路径：`/plugin/<pluginName>/<route>`

### 6.4 路由注册的真实风险

| 风险 | 说明 |
|------|------|
| **中间件仅为 ShaarliMiddleware** | 插件路由组**不经过 `ShaarliAdminMiddleware`**！这意味着默认所有插件自定义路由都是**公开可访问**的，无需登录。插件控制器必须自行检查登录状态（DemoPluginController 通过继承 `ShaarliAdminController` 实现，但这是插件开发者的自觉行为，框架不强制） |
| **callable 不校验存在性** | 注册时不检查 `callable` 是否可调用，运行时 Slim 才会报 500 错误，无降级策略 |
| **route 路径校验缺失** | `validateRouteRegistration()` 甚至不检查 `route` 键是否存在，若缺失则 Slim 注册时会用空字符串路径，导致 `/plugin/<name>` 本身被匹配 |
| **路径遍历风险** | `ltrim($route['route'], '/')` 只去除左斜杠，若插件传入 `../other-plugin/foo`，Slim 仍会解析为 `/plugin/<name>/../other-plugin/foo` 即 `/plugin/other-plugin/foo`，理论上可伪装为其他插件的路由 |
| **方法名动态调用** | `$this->{strtolower($route['method'])}()` 使用可变方法名，虽然 method 已被白名单限制，但在 PHP 层面 `get`/`post` 等仍是 Slim 的公开方法 |

---

## 七、安全隔离的系统性风险

### 7.1 代码执行层面

| 风险点 | 位置 | 说明 |
|--------|------|------|
| **无沙箱的 include** | [PluginManager.php L175](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L175) | `include_once $pluginFilePath` 直接执行 PHP 代码。任何能写入 `plugins/` 目录的人（或被上传漏洞利用）均可执行任意代码 |
| **全局命名空间污染** | 所有插件 | 插件的 Hook 函数、`_init()` 均定义在全局命名空间。两个插件同名函数会导致 PHP Fatal error（Cannot redeclare），直接让站点瘫痪 |
| **\Throwable 被吞** | [PluginManager.php L102-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L102-L104) | 加载期的异常被 catch 后仅记录不抛出，恶意插件可能用致命错误（如 `exit()` / `die()`）绕过——demo_plugin 的 `hook_demo_plugin_delete_link()` 就直接 `exit()`，会终止整个请求 |
| **Bookmark 对象可变** | [PluginManager.php L317](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L317) | `filter_search_entry` Hook 接收 `Bookmark` 对象，文档注释说"should NOT be altered"但无强制，插件可调用 setter 修改书签内容 |

### 7.2 权限控制层面

| 风险点 | 位置 | 说明 |
|--------|------|------|
| **插件路由默认公开** | [index.php L182](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/index.php#L182) | `/plugin/*` 组只加了 `ShaarliMiddleware`，不含管理员校验。未登录用户可直接访问所有插件自定义路由 |
| **_init 阶段写配置** | 各插件 `_init()` | 插件可在 `_init()` 中调用 `$conf->set('credentials.hash', ...)` 甚至 `$conf->write(true)` 重置管理员密码 |
| **Hook 接收 Conf 引用** | [PluginManager.php L139](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php#L139) | 每个 Hook 都接收 `$this->conf` 作为第二个参数，插件可在任意 Hook 中读取和修改全部配置 |
| **save_plugin_parameters 无边界** | [PluginsController.php L61](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L61) | 该 Hook 接收整个 POST 数据，可修改任何字段包括 token、order 等 |

### 7.3 数据输出层面

| 风险点 | 位置 | 说明 |
|--------|------|------|
| **Hook 返回数据直接渲染** | [ShaarliVisitorController.php L88-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php#L88-L96) | Hook 返回的 `$data` 被直接 `assignView` 到模板，若插件在 `$data['endofpage'][]` 中注入 `<script>`，模板原样输出，构成存储型 XSS |
| **demo_plugin 中的示范** | [demo_plugin.php L205-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/demo_plugin.php#L205-L210) | 官方演示插件就直接拼接 HTML 字符串：`'<marquee>...</marquee>'`，说明插件输出默认不转义 |
| **参数仅保存时转义** | [PluginsController.php L67](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php#L67) | `escape($value)` 在保存时做 HTML 转义，但插件 Hook 读取参数后再输出时若再次拼接 HTML，可能存在双重转义或绕过 |

### 7.4 加载过程的竞态

| 风险点 | 说明 |
|--------|------|
| **加载中断的部分状态** | 如果第 N 个插件 `_init()` 中调用了 `exit()`，前面 N-1 个插件已加载、路由已注册但后续插件不加载，产生不一致状态 |
| **无重载机制** | `include_once` 保证单个请求内不会重复加载，但配置变更后必须刷新整个 PHP 生命周期（FastCGI 下需重启进程），无热重载 |
| **错误数组 array_merge 累积** | `$this->errors = array_unique(array_merge($this->errors, $errors))`，若某插件 init 返回非数组（如字符串），`array_merge` 会产生 Warning，被错误报告设置吞掉 |

---

## 八、核心文件索引

| 文件 | 职责 |
|------|------|
| [PluginManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/plugin/PluginManager.php) | 插件加载、Hook 执行、路由注册、元数据解析 |
| [ConfigPlugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/config/ConfigPlugin.php) | 插件排序校验、参数值合并 |
| [ConfigManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/config/ConfigManager.php) | 配置读写（含插件参数存储） |
| [PluginsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/admin/PluginsController.php) | 插件管理页与配置保存 |
| [index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/index.php) | 应用启动、路由挂载 |
| [ShaarliVisitorController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php) | 基础控制器，封装 Hook 调用 |
| [ShaarliMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/ShaarliMiddleware.php) | 前端路由中间件（插件路由仅用它） |
| [ShaarliAdminMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/application/front/ShaarliAdminMiddleware.php) | 管理员中间件（插件路由**不用**它） |
| [demo_plugin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/75-Shaarli/plugins/demo_plugin/demo_plugin.php) | 官方演示插件（所有 Hook 用法示例） |
