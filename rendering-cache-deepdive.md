# Shaarli 渲染与缓存深度分析

> 本文档深入剖析 RainTPL 编译缓存键、模板包含解析、插件渲染钩子全集、缓存清除范围差异，并修正中间件触发缓存失效的归因。

---

## 一、RainTPL 编译缓存键生成逻辑

### 1.1 三级缓存体系

RainTPL 内部实际上存在**三级缓存**，而非简单的"编译缓存"一种：

| 层级 | 缓存类型 | 文件后缀 | 作用 |
|------|---------|---------|------|
| 第一级 | 模板编译缓存 | `.rtpl.php` | 模板语法编译为 PHP 代码的结果 |
| 第二级 | 静态输出缓存 | `.s_{cache_id}.rtpl.php` | 渲染后的完整 HTML 输出（需显式启用） |
| 第三级 | Shaarli 页面缓存 | `.cache` | Shaarli 业务层对 RSS/ATOM 等页面的完整缓存 |

### 1.2 编译缓存键生成公式

核心代码见 [rain.tpl.class.php#L254-L279](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L254-L279)。

**编译缓存文件名公式**：

```
compiled_filename = cache_dir + tpl_basename + "." + md5(tpl_dir + implode('', config_name_sum)) + ".rtpl.php"
```

**静态输出缓存文件名公式**：

```
cache_filename = cache_dir + tpl_basename + "." + md5(tpl_dir + implode('', config_name_sum)) + ".s_" + cache_id + ".rtpl.php"
```

### 1.3 缓存键组成要素详解

```php
$temp_compiled_filename = self::$cache_dir 
    . $tpl_basename 
    . "." 
    . md5($tpl_dir . implode('', self::$config_name_sum));
```

| 组成部分 | 来源 | 说明 |
|---------|------|------|
| `cache_dir` | `self::$cache_dir` | 编译缓存根目录，配置为 `tmp/` |
| `tpl_basename` | `basename($tpl_name)` | 模板文件名（不含路径），如 `linklist` |
| `tpl_dir` | `self::$tpl_dir + $tpl_basedir` | 模板目录完整路径，含主题子目录，如 `tpl/default/` |
| `config_name_sum` | `self::$config_name_sum` | 所有 RainTPL 配置值的拼接数组，参与 MD5 计算 |
| `cache_id` | `$this->cache_id` | 静态缓存的标识（可选，Shaarli 未使用） |

### 1.4 `$config_name_sum` 的构建与 Bug

**构建方式**：在 `RainTPL::configure()` 中累计，见 [rain.tpl.class.php#L240-L248](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L240-L248)

```php
static function configure( $setting, $value = null ){
    if( is_array( $setting ) )
        foreach( $setting as $key => $value )
            self::configure( $key, $value );
    else if( property_exists( __CLASS__, $setting ) ){
        self::$$setting = $value;
        self::$config_name_sum[$key] = $value;  // ⚠️ Bug: $key 未定义
    }
}
```

**Bug 分析**：
- 当传入单个配置项时（非数组形式），`$key` 变量未定义
- 此时 `$config_name_sum` 数组的 key 为 `null`，导致所有后续单个配置都覆盖同一个 null 键
- 但由于最终使用 `implode('', self::$config_name_sum)`，只要值正确拼接，MD5 仍然能反映配置变化
- 实际影响：Shaarli 主要通过数组形式批量配置，在数组分支中 `$key` 是正确的，因此实际影响有限

**Shaarli 中的配置调用**（[index.php#L89-L90](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/index.php#L89-L90)）：

```php
RainTPL::$tpl_dir = $conf->get('resource.raintpl_tpl') . '/' . $conf->get('resource.theme') . '/';
RainTPL::$cache_dir = $conf->get('resource.raintpl_tmp');
```

> 注意：Shaarli 使用直接属性赋值而非 `configure()` 方法，因此 `$config_name_sum` 在 Shaarli 场景下实际上始终为空数组。这意味着编译缓存键的 MD5 部分实际上只由 `$tpl_dir` 决定。

### 1.5 编译触发条件

```php
if( !file_exists($this->tpl['compiled_filename']) 
    || ( self::$check_template_update 
         && filemtime($this->tpl['compiled_filename']) < filemtime($this->tpl['tpl_filename']) ) ){
    $this->compileFile(...);
    return true;
}
```

| 条件 | 说明 |
|------|------|
| 编译文件不存在 | 首次访问必然触发编译 |
| `check_template_update=true` 且源文件更新 | 开发模式下自动检测模板变更 |

**编译文件内容结构**：
```php
<?php if(!class_exists('raintpl')){exit;}?>
<!-- 编译后的 PHP 代码 -->
```

顶部的安全检查防止直接通过 Web 访问编译缓存文件。

### 1.6 缓存文件命名示例

假设：
- `cache_dir = tmp/`
- `tpl_dir = tpl/default/`
- 模板名 = `linklist`

则编译缓存文件为：
```
tmp/linklist.78a3b2c1d4e5f6... .rtpl.php
        └── md5("tpl/default/")
```

---

## 二、模板包含（include）解析机制

### 2.1 include 标签语法

RainTPL 支持两种 include 形式：

| 语法 | 示例 | 说明 |
|------|------|------|
| 普通包含 | `{include="includes"}` | 每次都重新渲染 |
| 缓存包含 | `{include="includes" cache="3600"}` | 启用子模板静态缓存 |

### 2.2 编译后的 PHP 代码

核心解析逻辑见 [rain.tpl.class.php#L409-L441](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L409-L441)。

**普通 include 编译结果**：

```php
<?php 
    $tpl = new RainTpl;
    $tpl_dir_temp = self::$tpl_dir;
    $tpl->assign( $this->var );               // 继承所有父模板变量
    // 如果在循环内，还会传递：
    // $tpl->assign( "key", $key1 ); 
    // $tpl->assign( "value", $value1 );
    $tpl->draw( dirname("includes") . ( substr("includes",-1,1) != "/" ? "/" : "" ) . basename("includes") );
?>
```

**带缓存的 include 编译结果**：

```php
<?php 
    $tpl = new RainTpl;
    if( $cache = $tpl->cache( $template = basename("includes") ) )
        echo $cache;
    else{
        $tpl_dir_temp = self::$tpl_dir;
        $tpl->assign( $this->var );
        $tpl->draw( ... );
    } 
?>
```

### 2.3 变量继承机制

include 的子模板**完全继承**父模板的所有变量，通过 `$tpl->assign($this->var)` 实现。

**特殊情况：循环内 include**

当 `{include="..."}` 出现在 `{loop="..."}` 内部时，会额外传递循环变量：

```php
if( !$loop_level ? null : '$tpl->assign( "key", $key'.$loop_level.' ); $tpl->assign( "value", $value'.$loop_level.' );' )
```

这使得子模板中可以使用 `{$key}` 和 `{$value}` 访问当前循环项。

### 2.4 include 的递归特性

每个 include 都会创建一个**全新的 RainTpl 实例**，因此：
- 支持无限嵌套 include（A 包含 B，B 包含 C）
- 每个子模板独立编译，有独立的编译缓存文件
- 子模板的配置（如 `tpl_dir`）继承自当前静态属性

### 2.5 Shaarli 中的 include 链

以 [linklist.html](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/tpl/default/linklist.html) 为例：

```
linklist.html
  ├── {include="includes"}     → includes.html
  ├── {include="page.header"}  → page.header.html
  └── {include="page.footer"}  → page.footer.html
```

**includes.html 的作用**：
- 定义页面 `<head>` 中的 meta、CSS 资源
- 注入插件 CSS 文件（`plugins_includes.css_files`）
- 注入用户自定义 CSS（`data/user.css`）

**page.header.html 的作用**：
- 导航栏
- 搜索框
- 插件工具按钮（`plugins_header.buttons_toolbar`）

**page.footer.html 的作用**：
- 页脚文本（`plugins_footer.text`）
- 页面底部插件内容（`plugins_footer.endofpage`）
- 插件 JS 文件（`plugins_footer.js_files`）

### 2.6 包含模板的缓存独立性

每个被 include 的子模板都有**独立**的编译缓存文件：
- `includes.xxx.rtpl.php`
- `page.header.xxx.rtpl.php`
- `page.footer.xxx.rtpl.php`

但由于 include 是**运行时动态执行**的（每次都 `new RainTpl` 并 `draw`），因此：
- 编译缓存是独立的（各自的编译文件）
- 但渲染过程每次都会重新执行
- 只有显式使用 `cache="..."` 参数的 include 才会有输出缓存

---

## 三、插件渲染钩子全集

### 3.1 钩子命名规范

插件钩子函数命名遵循 [PluginManager::buildHookName()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/plugin/PluginManager.php#L215-L218) 定义的规范：

```
hook_<plugin_name>_<hook_name>
```

例如：
- `hook_demo_plugin_render_linklist`
- `hook_isso_render_footer`
- `hook_archiveorg_save_link`

### 3.2 渲染类钩子（Render Hooks）

渲染钩子在模板渲染前执行，插件可以通过修改 `$data` 数组注入内容。

#### 通用页面钩子（每个页面都会执行）

在 [ShaarliVisitorController::executeDefaultHooks()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php#L78-L97) 中调用：

| 钩子名 | 触发时机 | 模板变量 | 典型用途 |
|--------|---------|---------|---------|
| `render_includes` | 页面 head 区域 | `plugins_includes` | 注入自定义 CSS、JS 引用 |
| `render_header` | 页面导航区域 | `plugins_header` | 添加工具栏按钮、搜索字段 |
| `render_footer` | 页脚区域 | `plugins_footer` | 添加页脚文本、统计代码 |

#### 页面特定钩子

由各个控制器显式调用 `executePageHooks()`：

| 钩子名 | 触发页面 | 调用位置 | 模板占位符 |
|--------|---------|---------|-----------|
| `render_linklist` | 书签列表页 | [BookmarkListController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/BookmarkListController.php#L120) | `action_plugin` `link_plugin` `plugin_start_zone` `plugin_end_zone` |
| `render_editlink` | 编辑/新增书签页 | [ShaarePublishController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ShaarePublishController.php#L57) | `edit_link_plugin` |
| `render_feed` | RSS/ATOM 订阅页 | [FeedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/FeedController.php#L49) | `feed_plugins_header` |
| `render_daily` | 每日汇总页 | [DailyController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/DailyController.php#L66) | `plugin_start_zone` `plugin_end_zone` |
| `render_picwall` | 图片墙页 | [PictureWallController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/PictureWallController.php#L46) | `plugin_start_zone` `plugin_end_zone` |
| `render_tagcloud` | 标签云页 | [TagCloudController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/TagCloudController.php#L84) | `plugin_start_zone` `plugin_end_zone` |
| `render_taglist` | 标签列表页 | [TagCloudController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/TagCloudController.php) (tag list action) | `plugin_start_zone` `plugin_end_zone` |
| `render_tools` | 工具页 | [ToolsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ToolsController.php#L25) | `tools_plugin` |

### 3.3 数据生命周期钩子

| 钩子名 | 触发时机 | 调用位置 | 用途 |
|--------|---------|---------|------|
| `save_link` | 保存书签前（新增/修改/批量操作） | [ShaarePublishController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ShaarePublishController.php#L133)、[ShaareManageController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ShaareManageController.php) | 保存前修改书签数据、验证 |
| `delete_link` | 删除书签前 | [ShaareManageController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ShaareManageController.php#L55) | 清理关联资源 |
| `save_plugin_parameters` | 保存插件配置时 | [PluginsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/PluginsController.php#L61) | 参数验证、持久化 |
| `filter_search_entry` | 搜索结果过滤（逐条） | [PluginManager::filterSearchEntry()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/plugin/PluginManager.php#L306-L323) | 自定义搜索过滤 |

### 3.4 钩子注入的模板占位符详解

插件通过向 `$data` 数组中添加特定 key 来在模板中占位。这些占位符在模板中通过 `{loop="..."}` 遍历渲染。

#### 通用占位符

| 占位符名 | 所在钩子 | 模板位置 | 内容类型 |
|---------|---------|---------|---------|
| `plugins_includes.css_files` | render_includes | `<head>` 内 | CSS 文件路径数组 |
| `plugins_header.buttons_toolbar` | render_header | 导航栏右侧 | 按钮属性数组（含 `attr` 和 `html`） |
| `plugins_header.fields_toolbar` | render_header | 搜索栏旁 | 表单字段 HTML |
| `plugins_footer.text` | render_footer | 页脚文本区 | 文本/HTML 片段数组 |
| `plugins_footer.endofpage` | render_footer | `</body>` 前 | HTML 片段数组 |
| `plugins_footer.js_files` | render_footer | 页面底部 | JS 文件路径数组 |

#### 页面特定占位符

| 占位符名 | 所在页面 | 说明 |
|---------|---------|------|
| `action_plugin` | linklist | 工具栏操作按钮（每个是 `[attr => [...], html => '...', on/off => true]` 结构） |
| `link_plugin` | linklist | 每条书签下方的图标/链接（追加到数组） |
| `plugin_start_zone` | linklist/daily/picwall/tagcloud | 页面内容开始处的 HTML 块 |
| `plugin_end_zone` | linklist/daily/picwall/tagcloud | 页面内容结束处的 HTML 块 |
| `edit_link_plugin` | editlink | 编辑表单中 tags 字段后的额外字段 |
| `tools_plugin` | tools | 工具页的额外工具项 |
| `feed_plugins_header` | feed | RSS/ATOM feed 的 header 区域 |

### 3.5 钩子执行时的元数据注入

在 [PluginManager::executeHooks()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/plugin/PluginManager.php#L118-L150) 中，会向 `$data` 临时注入以下元数据键（钩子执行完后移除）：

| 元数据键 | 说明 |
|---------|------|
| `_PAGE_` | 当前页面模板名（target） |
| `_LOGGEDIN_` | 用户是否登录 |
| `_BASE_PATH_` | URL 基础路径 |
| `_ROOT_PATH_` | URL 根路径（去掉 index.php） |
| `_BOOKMARK_SERVICE_` | 书签服务实例 |

插件可以通过这些元数据判断当前上下文，决定是否执行操作。

### 3.6 插件渲染钩子调用时序

```
控制器处理请求
    ↓
assignAllView($data)        // 模板变量赋值
    ↓
executePageHooks(...)       // 页面特定钩子（render_linklist 等）
    ↓
render($template)
    ├── assign linkcount    // 统计数据
    ├── executeDefaultHooks // 通用钩子（render_includes/header/footer）
    └── PageBuilder->render // RainTPL 实际渲染
         ├── initialize()   // 初始化全局变量
         └── draw()         // 模板编译与渲染
```

---

## 四、缓存清除范围差异

Shaarli 存在多个缓存层次，不同清除操作的范围不同，需仔细区分。

### 4.1 四层缓存总览

| 缓存层 | 目录配置项 | 默认路径 | 存储内容 | 清除方法 |
|--------|----------|---------|---------|---------|
| 页面缓存 | `resource.page_cache` | `pagecache/` | RSS/ATOM 等整页输出 | `purgeCachedPages()` |
| RainTPL 编译缓存 | `resource.raintpl_tmp` | `tmp/` | 模板编译后的 PHP 文件 | 删除目录内容 |
| 缩略图缓存 | `resource.thumbnails_cache` | `cache/` | 书签缩略图图片 | 删除目录内容 |
| 数据缓存 | - | `data/` 下各种 .php | 书签数据、配置等 | 不在缓存清除范围 |

### 4.2 PageCacheManager 的清除范围

**[PageCacheManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageCacheManager.php)**

#### `purgeCachedPages()` - 仅清除页面缓存

```php
public function purgeCachedPages(): ?string
{
    if (!is_dir($this->pageCacheDir)) {
        return sprintf(t('Cannot purge %s: no directory'), $this->pageCacheDir);
    }
    array_map('unlink', glob($this->pageCacheDir . '/*.cache'));
    return null;
}
```

**清除范围**：仅 `pagecache/*.cache` 文件（使用 glob 匹配 `.cache` 后缀）

**不清除**：
- RainTPL 编译缓存（`tmp/*.rtpl.php`）
- 缩略图缓存（`cache/` 目录）
- 其他非 `.cache` 后缀的文件

#### `invalidateCaches()` - 失效缓存（目前等同 purgeCachedPages）

```php
public function invalidateCaches(): void
{
    $this->purgeCachedPages();
}
```

目前实现与 `purgeCachedPages()` 完全相同，但语义上是"使缓存失效"，未来可能扩展为清除更多缓存类型。

### 4.3 ServerController 手动清除的范围差异

**[ServerController::clearCache()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ServerController.php#L71-L100)**

支持两种清除模式，通过 `?type=` 参数区分：

#### 模式一：默认模式（清除主缓存）

```php
$folders = [
    $this->container->conf->get('resource.page_cache'),     // pagecache/
    $this->container->conf->get('resource.raintpl_tmp'),    // tmp/
];
```

**清除范围**：
- ✅ 页面缓存（pagecache/）
- ✅ RainTPL 编译缓存（tmp/）
- ❌ 缩略图缓存

#### 模式二：thumb 模式（仅清除缩略图）

```php
if ($request->getQueryParam('type') === static::CACHE_THUMB) {
    $folders = [$this->container->conf->get('resource.thumbnails_cache')];  // cache/
}
```

**清除范围**：
- ✅ 缩略图缓存（cache/）
- ❌ 页面缓存
- ❌ RainTPL 编译缓存

### 4.4 FileUtils::clearFolder() - 底层清除工具

**[FileUtils::clearFolder()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/helper/FileUtils.php#L95-L129)**

```php
public static function clearFolder(string $path, bool $selfDelete, array $exclude = []): bool
```

**参数说明**：
- `$path`：要清除的目录路径
- `$selfDelete`：是否同时删除目录本身（false=只删内容）
- `$exclude`：排除的文件名数组（默认排除 `.htaccess`）

**安全机制**：
- 验证路径在 Shaarli 根目录内（`isPathInShaarliFolder()`）
- 防止越权删除系统文件

### 4.5 各类缓存失效触发场景汇总

| 场景 | 触发方式 | 清除页面缓存 | 清除编译缓存 | 清除缩略图缓存 |
|------|---------|------------|------------|--------------|
| 书签新增/修改/删除 | BookmarkFileService->save() | ✅ | ❌ | ❌ |
| 用户登出 | LogoutController | ✅ | ❌ | ❌ |
| 配置修改 | ConfigureController | ✅ | ❌ | ❌ |
| 数据库版本更新 | ShaarliMiddleware->runUpdates() | ✅ | ❌ | ❌ |
| 手动清除缓存（默认） | /admin/server?clearcache | ✅ | ✅ | ❌ |
| 手动清除缩略图 | /admin/server?clearcache&type=thumb | ❌ | ❌ | ✅ |
| 主题切换 | ConfigureController（间接） | ✅ | ❌（实际自动失效，因为 tpl_dir 变了） | ❌ |

### 4.6 编译缓存的"隐式失效"机制

RainTPL 编译缓存虽然不在 `invalidateCaches()` 的清除范围内，但在以下情况下会**自动失效**：

1. **模板文件修改**：如果 `check_template_update = true`，源文件 mtime 新于编译文件时自动重新编译
2. **主题切换**：切换主题会改变 `tpl_dir`，导致 MD5 缓存键变化，相当于自动切换到另一套编译缓存
3. **手动清除**：通过 ServerController 的默认清除模式删除 `tmp/` 目录

---

## 五、中间件触发缓存失效的归因修正

### 5.1 之前的归因（不准确）

之前的分析认为："登录状态变更时，中间件会触发缓存失效"。

### 5.2 真实原因：数据库版本升级

实际代码见 [ShaarliMiddleware::runUpdates()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/ShaarliMiddleware.php#L67-L83)：

```php
protected function runUpdates(): void
{
    if ($this->container->loginManager->isLoggedIn() !== true) {
        return;  // 未登录直接返回，不做任何事
    }

    $this->container->updater->setBasePath($this->container->basePath);
    $newUpdates = $this->container->updater->update();  // 执行数据库迁移
    if (!empty($newUpdates)) {
        $this->container->updater->writeUpdates(
            $this->container->conf->get('resource.updates'),
            $this->container->updater->getDoneUpdates()
        );

        $this->container->pageCacheManager->invalidateCaches();  // ⚠️ 仅在有新更新时才调用
    }
}
```

### 5.3 完整触发条件

中间件触发缓存失效**必须同时满足**以下所有条件：

| 条件 | 说明 |
|------|------|
| 1. 用户已登录 | `isLoggedIn() === true`，未登录直接跳过 |
| 2. 有新的数据库更新 | `$newUpdates` 非空（即版本号提升，需要执行迁移脚本） |
| 3. 更新执行成功 | 更新脚本执行完成，写入更新记录 |

### 5.4 触发时机详解

```
每次请求（登录用户）
    ↓
ShaarliMiddleware->__invoke()
    ↓
runUpdates()
    ├── 检查 isLoggedIn()
    │    └── 未登录 → 直接返回（不清除缓存）
    ├── 执行 upater->update()
    │    └── 获取待执行的更新列表
    └── 有新更新？
         ├── 否 → 正常继续（不清除缓存）
         └── 是 → 执行更新 → writeUpdates() → invalidateCaches() ✅
```

### 5.5 为什么升级后需要清缓存

数据库结构/数据迁移后，页面内容可能发生变化（如字段增减、数据格式变化），旧的页面缓存可能与新数据不兼容，因此需要使页面缓存失效。

### 5.6 其他缓存失效触发点的准确归因

为了完整，重新梳理所有 5 个 `invalidateCaches()` 调用点：

| # | 调用位置 | 触发条件 | 触发原因 |
|---|---------|---------|---------|
| 1 | [BookmarkFileService.php#L318](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/bookmark/BookmarkFileService.php#L318) | 每次保存书签数据（新增/修改/删除/批量操作后） | 书签数据变更，页面内容必然变化 |
| 2 | [LogoutController.php#L22](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/LogoutController.php#L22) | 用户登出时 | 登录状态变化，页面可见内容可能变化（私有书签可见性） |
| 3 | [ConfigureController.php#L117](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ConfigureController.php#L117) | 保存配置后且配置文件真的被修改了 | 配置变化可能影响页面内容和样式 |
| 4 | [ShaarliMiddleware.php#L81](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/ShaarliMiddleware.php#L81) | 数据库版本更新完成后 | 数据结构变化，旧缓存可能不兼容 |
| 5 | [LegacyLinkDB.php#L360](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/legacy/LegacyLinkDB.php#L360) | 旧版 LinkDB 保存后 | 兼容旧代码的数据变更 |

### 5.7 易错点澄清

1. **"登录时清除缓存"** ❌ 错误
   - 实际：只有登出时明确清除（LogoutController）
   - 登录时不清除缓存（登录状态下能看到更多内容，但缓存本来就对登录用户不生效）

2. **"中间件每次请求都清缓存"** ❌ 错误
   - 实际：只有数据库版本升级时才清，且只对登录用户检测

3. **"主题切换会清编译缓存"** ❌ 不直接
   - 实际：主题切换会让配置变更，触发 `invalidateCaches()`（清页面缓存）
   - 编译缓存因为 tpl_dir 变了，会自动使用不同的缓存键，不需要手动清除

---

## 附录：关键文件速查

| 文件 | 作用 |
|------|------|
| [rain.tpl.class.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php) | RainTPL 模板引擎核心（编译、缓存、include） |
| [PageBuilder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageBuilder.php) | 页面构建器（模板变量、渲染入口） |
| [PageCacheManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageCacheManager.php) | 页面缓存管理 |
| [CachedPage.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/feed/CachedPage.php) | 单页缓存读写实现 |
| [PluginManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/plugin/PluginManager.php) | 插件钩子执行器 |
| [ShaarliVisitorController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php) | 前端控制器基类（钩子调用、render 流程） |
| [ShaarliMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/ShaarliMiddleware.php) | 请求中间件（更新检测、登录检查） |
| [FileUtils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/helper/FileUtils.php) | 文件工具（目录清除） |
