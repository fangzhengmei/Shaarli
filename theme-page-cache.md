# Shaarli 渲染层、缓存与前端行为分析

## 目录

- [一、模板选择机制](#一模板选择机制)
- [二、页面构建流程](#二页面构建流程)
- [三、缓存刷新机制](#三缓存刷新机制)
- [四、前端行为绑定](#四前端行为绑定)

---

## 一、模板选择机制

### 1.1 主题发现与配置

Shaarli 使用 RainTPL 作为模板引擎，模板选择机制通过以下核心组件实现：

**主题工具类 [ThemeUtils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/ThemeUtils.php)**

```php
public static function getThemes($tplDir)
{
    $tplDir = rtrim($tplDir, '/');
    $allTheme = glob($tplDir . '/*', GLOB_ONLYDIR);
    $themes = [];
    foreach ($allTheme as $value) {
        $themes[] = str_replace($tplDir . '/', '', $value);
    }
    return $themes;
}
```

**工作原理**：
- 通过扫描 `tpl/` 目录下的所有子目录来发现可用主题
- 每个子目录代表一个独立主题（默认主题为 `default`，另有 `vintage` 主题）
- 在 [ConfigureController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ConfigureController.php) 中被调用，用于管理后台的主题选择

### 1.2 模板常量定义

**模板页面接口 [TemplatePage.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/TemplatePage.php)**

定义了所有可用模板的常量映射，共 25 种页面类型，包括：

| 常量名 | 模板值 | 用途 |
|--------|--------|------|
| `LINKLIST` | `linklist` | 书签列表首页 |
| `ADD_LINK` | `addlink` | 添加书签 |
| `EDIT_LINK` | `editlink` | 编辑书签 |
| `FEED_ATOM` | `feed.atom` | ATOM 订阅源 |
| `FEED_RSS` | `feed.rss` | RSS 订阅源 |
| `DAILY` | `daily` | 每日汇总 |
| `PICTURE_WALL` | `picwall` | 图片墙 |
| `TAG_CLOUD` | `tag.cloud` | 标签云 |
| ... | ... | ... |

### 1.3 RainTPL 配置初始化

在入口文件 [index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/index.php#L89-L90) 中配置模板引擎：

```php
RainTPL::$tpl_dir = $conf->get('resource.raintpl_tpl') . '/' . $conf->get('resource.theme') . '/';
RainTPL::$cache_dir = $conf->get('resource.raintpl_tmp');
```

**配置参数（[ConfigManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/config/ConfigManager.php#L354-L358)）**：

```php
$this->setEmpty('resource.raintpl_tpl', 'tpl/');
$this->setEmpty('resource.theme', 'default');
$this->setEmpty('resource.raintpl_tmp', 'tmp/');
$this->setEmpty('resource.page_cache', 'pagecache');
```

**模板目录结构**：
```
tpl/
├── default/          # 默认主题
│   ├── linklist.html
│   ├── includes.html
│   ├── addlink.html
│   └── ...
└── vintage/          # 经典主题
    └── ...
```

---

## 二、页面构建流程

### 2.1 请求生命周期

完整的页面请求处理流程：

```
HTTP 请求
    ↓
[index.php] 初始化环境、配置、容器
    ↓
[Slim 路由] 匹配控制器
    ↓
[Controller] 执行业务逻辑，获取数据
    ↓
[PageBuilder] 模板变量赋值
    ↓
[RainTPL] 模板编译与渲染
    ↓
HTTP 响应
```

### 2.2 页面构建器核心

**[PageBuilder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageBuilder.php)** 是渲染层的核心类：

#### 初始化阶段 [initialize()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageBuilder.php#L96-L167)

采用**懒加载**模式，首次调用 `assign()` 或 `render()` 时才初始化：

```php
private function initialize()
{
    $this->tpl = new RainTPL();
    
    // 1. 版本检查与更新通知
    $version = ApplicationUtils::checkUpdate(...);
    
    // 2. 全局模板变量赋值（共 30+ 个默认变量）
    $this->tpl->assign('is_logged_in', $this->isLoggedIn);
    $this->tpl->assign('feedurl', escape(index_url($_SERVER)));
    $this->tpl->assign('pagetitle', $this->conf->get('general.title', 'Shaarli'));
    $this->tpl->assign('token', $this->token);  // CSRF 令牌
    $this->tpl->assign('language', $this->conf->get('translation.language'));
    $this->tpl->assign('thumbnails_enabled', ...);
    $this->tpl->assign('links_per_page', $this->session['LINKS_PER_PAGE'] ?? 20);
    // ... 更多变量
}
```

#### 变量赋值

```php
// 单个变量赋值
public function assign($placeholder, $value)
{
    if ($this->tpl === false) {
        $this->initialize();  // 懒加载触发点
    }
    $this->tpl->assign($placeholder, $value);
}

// 批量赋值
public function assignAll($data)
```

#### 渲染阶段 [render()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageBuilder.php#L244-L253)

```php
public function render(string $page, string $basePath): string
{
    if ($this->tpl === false) {
        $this->initialize();
    }
    $this->finalize($basePath);  // 最终处理
    return $this->tpl->draw($page, true);  // 返回渲染后的字符串
}
```

#### 最终处理 [finalize()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageBuilder.php#L173-L197)

```php
protected function finalize(string $basePath): void
{
    // 1. 处理 Session 中的提示消息（成功/警告/错误）
    foreach ($messageKeys as $messageKey) {
        if (!empty($_SESSION[$messageKey])) {
            $this->tpl->assign('global_' . $messageKey, $_SESSION[$messageKey]);
            unset($_SESSION[$messageKey]);  // 消费后删除
        }
    }
    
    // 2. 计算资源路径
    $this->assign('asset_path',
        $rootPath . '/' .
        rtrim($this->conf->get('resource.raintpl_tpl', 'tpl'), '/') . '/' .
        $this->conf->get('resource.theme', 'default')
    );
}
```

### 2.3 控制器调用流程

以 [BookmarkListController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/BookmarkListController.php) 为例：

```php
public function index(Request $request, Response $response): Response
{
    // 1. 业务逻辑：查询书签数据
    $searchResult = $this->container->bookmarkService->search(...);
    
    // 2. 数据格式化
    $formatter = $this->container->formatterFactory->getFormatter();
    foreach ($searchResult->getBookmarks() as $key => $bookmark) {
        $links[$key] = $formatter->format($bookmark);
    }
    
    // 3. 组装模板数据
    $data = array_merge(
        $this->initializeTemplateVars(),
        [
            'links' => $links,
            'previous_page_url' => $previousPageUrl,
            'next_page_url' => $nextPageUrl,
            'result_count' => $searchResult->getTotalCount(),
            // ... 更多数据
        ]
    );
    
    // 4. 执行插件钩子
    $this->executePageHooks('render_linklist', $data, TemplatePage::LINKLIST);
    
    // 5. 赋值并渲染
    $this->assignAllView($data);
    return $response->write($this->render(TemplatePage::LINKLIST));
}
```

基类 [ShaarliVisitorController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/ShaarliVisitorController.php#L57-L72) 的 `render()` 方法：

```php
protected function render(string $template): string
{
    // 注入模板标识
    $this->assignView('_PAGE_', $template);
    $this->assignView('template', $template);
    
    // 注入统计数据
    $this->assignView('linkcount', $this->container->bookmarkService->count(BookmarkFilter::$ALL));
    $this->assignView('publicLinkcount', $this->container->bookmarkService->count(BookmarkFilter::$PUBLIC));
    $this->assignView('privateLinkcount', $this->container->bookmarkService->count(BookmarkFilter::$PRIVATE));
    
    // 执行公共插件钩子（includes/header/footer）
    $this->executeDefaultHooks($template);
    $this->assignView('plugin_errors', $this->container->pluginManager->getErrors());
    
    // 调用 PageBuilder 渲染
    return $this->container->pageBuilder->render($template, $this->container->basePath);
}
```

### 2.4 RainTPL 模板编译机制

**[rain.tpl.class.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php)** 核心流程：

#### 模板检查与编译 [check_template()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L254-L279)

```php
protected function check_template($tpl_name)
{
    // 计算编译后文件名（包含配置哈希，确保配置变更时重新编译）
    $temp_compiled_filename = self::$cache_dir . $tpl_basename . "." . 
        md5($tpl_dir . implode('', self::$config_name_sum));
    
    $this->tpl['compiled_filename'] = $temp_compiled_filename . '.rtpl.php';
    
    // 检查是否需要重新编译
    if (!file_exists($this->tpl['compiled_filename']) || 
        (self::$check_template_update && 
         filemtime($this->tpl['compiled_filename']) < filemtime($this->tpl['tpl_filename']))) {
        $this->compileFile(...);  // 编译模板
        return true;
    }
}
```

#### 模板编译 [compileFile()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L294-L325)

```php
protected function compileFile($tpl_basename, $tpl_basedir, $tpl_filename, $cache_dir, $compiled_filename)
{
    // 1. 读取原始模板
    $template_code = file_get_contents($tpl_filename);
    
    // 2. XML 标签处理（避免与 PHP 标签冲突）
    $template_code = preg_replace("/<\?xml(.*?)\?>/s", "##XML\\1XML##", $template_code);
    
    // 3. 禁用 PHP 标签
    if (!self::$php_enabled) {
        $template_code = str_replace(array("<?","?>"), array("&lt;?","?&gt;"), $template_code);
    }
    
    // 4. 编译模板语法为 PHP 代码
    $template_compiled = "<?php if(!class_exists('raintpl')){exit;}?>" . 
        $this->compileTemplate($template_code, $tpl_basedir);
    
    // 5. 写入编译后的 PHP 文件
    file_put_contents($compiled_filename, $template_compiled);
}
```

#### 模板语法编译 [compileCode()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L374-L584)

支持的模板标签：

| 标签 | 示例 | 编译后 |
|------|------|--------|
| 变量输出 | `{$title}` | `<?php echo $title;?>` |
| 循环 | `{loop="links"}` | `<?php foreach($links as $key => $value){ ?>` |
| 条件 | `{if="$is_logged_in"}` | `<?php if($is_logged_in){ ?>` |
| 包含 | `{include="includes"}` | 递归渲染子模板 |
| 函数 | `{function="t('Delete')"}` | `<?php echo t('Delete');?>` |
| 修饰器 | `{$title\|escape}` | `<?php echo escape($title);?>` |

#### 渲染执行 [draw()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L160-L207)

```php
function draw($tpl_name, $return_string = false)
{
    $this->check_template($tpl_name);  // 确保模板已编译
    
    // 提取变量到当前作用域
    extract($this->var);
    
    // 包含编译后的 PHP 文件（实际执行渲染）
    include $this->tpl['compiled_filename'];
}
```

### 2.5 模板包含与资源加载

以 [linklist.html](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/tpl/default/linklist.html) 为例：

```html
<!DOCTYPE html>
<html{if="$language !== 'auto'"} lang="{$language}"{/if}>
<head>
  {include="includes"}  <!-- 包含头部资源 -->
</head>
<body>
{include="page.header"}  <!-- 包含导航栏 -->

<!-- 页面内容 -->
<div id="linklist">
  {loop="links"}
    <div class="linklist-item" data-id="{$value.id}">
      <h2><a href="{$value.real_url}">{$value.title_html}</a></h2>
      <!-- ... -->
    </div>
  {/loop}
</div>

{include="page.footer"}  <!-- 包含页脚 -->

<!-- 页面特定 JS -->
<script src="{$asset_path}/js/thumbnails.min.js?v={$version_hash}#"></script>
{if="$is_logged_in && $async_metadata"}
  <script src="{$asset_path}/js/metadata.min.js?v={$version_hash}#"></script>
{/if}
</body>
</html>
```

**[includes.html](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/tpl/default/includes.html)** 资源加载：

```html
<title>{$pagetitle}</title>
<link type="text/css" rel="stylesheet" href="{$asset_path}/css/shaarli.min.css?v={$version_hash}#" />

{if="strpos($formatter, 'markdown') !== false"}
  <link type="text/css" rel="stylesheet" href="{$asset_path}/css/markdown.min.css?v={$version_hash}#" />
{/if}

{loop="$plugins_includes.css_files"}
  <link type="text/css" rel="stylesheet" href="{$root_path}/{$value}?v={$version_hash}#"/>
{/loop}

{if="is_file('data/user.css')"}
  <link type="text/css" rel="stylesheet" href="{$root_path}/data/user.css#" />
{/if}
```

**资源版本控制**：使用 `$version_hash` 进行缓存清除，由 [ApplicationUtils::getVersionHash()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/helper/ApplicationUtils.php) 计算，基于版本号和 salt。

---

## 三、缓存刷新机制

Shaarli 实现了**三层缓存架构**：
1. **RainTPL 编译缓存**（`tmp/` 目录）- 模板编译后的 PHP 文件
2. **页面缓存**（`pagecache/` 目录）- RSS/ATOM 等页面的完整缓存
3. **缩略图缓存**（`cache/` 目录）- 书签缩略图

### 3.1 页面缓存管理器

**[PageCacheManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageCacheManager.php)**

```php
class PageCacheManager
{
    protected $pageCacheDir;
    protected $isLoggedIn;

    public function __construct(string $pageCacheDir, bool $isLoggedIn)
    {
        $this->pageCacheDir = $pageCacheDir;
        $this->isLoggedIn = $isLoggedIn;  // 登录用户不使用缓存
    }

    // 清除所有页面缓存
    public function purgeCachedPages(): ?string
    {
        if (!is_dir($this->pageCacheDir)) {
            return sprintf(t('Cannot purge %s: no directory'), $this->pageCacheDir);
        }
        array_map('unlink', glob($this->pageCacheDir . '/*.cache'));
        return null;
    }

    // 使缓存失效（数据库变更或用户登出时调用）
    public function invalidateCaches(): void
    {
        $this->purgeCachedPages();
    }

    // 获取缓存页面实例
    public function getCachePage(string $pageUrl, DatePeriod $validityPeriod = null): CachedPage
    {
        return new CachedPage(
            $this->pageCacheDir,
            $pageUrl,
            false === $this->isLoggedIn,  // 仅匿名用户使用缓存
            $validityPeriod
        );
    }
}
```

### 3.2 缓存页面实现

**[CachedPage.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/feed/CachedPage.php)**

```php
class CachedPage
{
    protected $cacheDir;
    protected $shouldBeCached;
    protected $filename;
    protected $validityPeriod;

    public function __construct($cacheDir, $url, $shouldBeCached, ?DatePeriod $validityPeriod)
    {
        $this->cacheDir = $cacheDir;
        // URL 的 SHA1 哈希作为缓存文件名
        $this->filename = $this->cacheDir . '/' . sha1($url) . '.cache';
        $this->shouldBeCached = $shouldBeCached;
        $this->validityPeriod = $validityPeriod;
    }

    // 读取缓存
    public function cachedVersion()
    {
        if (!$this->shouldBeCached) return null;
        if (!is_file($this->filename)) return null;
        
        // 有效期检查（可选）
        if ($this->validityPeriod !== null) {
            $cacheDate = \DateTime::createFromFormat('U', (string) filemtime($this->filename));
            if ($cacheDate < $this->validityPeriod->getStartDate() ||
                $cacheDate > $this->validityPeriod->getEndDate()) {
                return null;
            }
        }
        return file_get_contents($this->filename);
    }

    // 写入缓存
    public function cache($pageContent)
    {
        if (!$this->shouldBeCached) return;
        file_put_contents($this->filename, $pageContent);
    }
}
```

### 3.3 缓存使用示例

**[FeedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/FeedController.php#L28-L57)** 中的缓存使用：

```php
protected function processRequest(string $feedType, Request $request, Response $response): Response
{
    $response = $response->withHeader('Content-Type', 'application/' . $feedType . '+xml; charset=utf-8');

    $pageUrl = page_url($this->container->environment);
    $cache = $this->container->pageCacheManager->getCachePage($pageUrl);

    // 1. 尝试读取缓存
    $cached = $cache->cachedVersion();
    if (!empty($cached)) {
        return $response->write($cached);  // 直接返回缓存
    }

    // 2. 无缓存，重新生成
    $data = $this->container->feedBuilder->buildData($feedType, $request->getParams());
    $this->executePageHooks('render_feed', $data, 'feed.' . $feedType);
    $this->assignAllView($data);
    $content = $this->render('feed.' . $feedType);

    // 3. 写入缓存供下次使用
    $cache->cache($content);

    return $response->write($content);
}
```

### 3.4 缓存失效触发点

缓存失效通过调用 `invalidateCaches()` 触发，发生在以下场景：

| 触发位置 | 触发时机 | 代码引用 |
|----------|----------|----------|
| 书签数据变更 | 新增/修改/删除书签 | [BookmarkFileService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/bookmark/BookmarkFileService.php#L318) |
| 用户登出 | 登录状态改变 | [LogoutController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/LogoutController.php#L22) |
| 配置变更 | 修改系统设置（含主题切换） | [ConfigureController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ConfigureController.php#L117) |
| 中间件检测 | 登录状态变更时 | [ShaarliMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/ShaarliMiddleware.php#L81) |
| 手动清除 | 管理员点击清除缓存 | [ServerController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ServerController.php#L71-L100) |

**书签保存时自动失效 [BookmarkFileService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/bookmark/BookmarkFileService.php#L318)**：

```php
public function save(): void
{
    // ... 保存数据 ...
    
    // 自动使页面缓存失效
    $this->pageCacheManager->invalidateCaches();
}
```

### 3.5 手动缓存清除

**[ServerController::clearCache()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ServerController.php#L71-L100)** 支持两种清除类型：

```php
public function clearCache(Request $request, Response $response): Response
{
    $exclude = ['.htaccess'];

    if ($request->getQueryParam('type') === static::CACHE_THUMB) {
        // 仅清除缩略图缓存
        $folders = [$this->container->conf->get('resource.thumbnails_cache')];
        $this->saveWarningMessage(t('Thumbnails cache has been cleared.'));
    } else {
        // 清除主缓存（页面缓存 + RainTPL 编译缓存）
        $folders = [
            $this->container->conf->get('resource.page_cache'),
            $this->container->conf->get('resource.raintpl_tmp'),
        ];
        $this->saveSuccessMessage(t('Shaarli\'s cache folder has been cleared!'));
    }

    foreach ($folders as $folder) {
        FileUtils::clearFolder($folder, false, $exclude);
    }

    return $this->redirect($response, '/admin/server');
}
```

### 3.6 客户端缓存控制

在 [init.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/init.php#L82-L86) 中禁用客户端缓存：

```php
// 防止客户端或代理缓存
header("Last-Modified: " . gmdate("D, d M Y H:i:s") . " GMT");
header("Cache-Control: no-store, no-cache, must-revalidate");
header("Cache-Control: post-check=0, pre-check=0", false);
header("Pragma: no-cache");
```

---

## 四、前端行为绑定

Shaarli 的前端代码采用 **原生 JavaScript + ES6 模块化** 架构，使用 `IIFE`（立即调用函数表达式）模式封装，避免全局污染。

### 4.1 前端资源结构

```
assets/
├── common/          # 跨主题共享资源
│   ├── css/
│   │   └── markdown.css
│   └── js/
│       ├── metadata.js        # 异步元数据获取
│       ├── shaare-batch.js    # 批量书签操作
│       ├── thumbnails-update.js
│       └── thumbnails.js      # 图片懒加载
├── default/         # 默认主题资源
│   ├── js/
│   │   ├── base.js          # 核心交互逻辑（主文件）
│   │   └── plugins-admin.js # 插件管理 JS
│   └── scss/
│       └── shaarli.scss
└── vintage/         # 经典主题资源
    └── js/
        └── base.js
```

### 4.2 核心行为绑定

**[base.js](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/assets/default/js/base.js)** 是前端核心，包含 15+ 种交互行为。

#### 初始化模式

```javascript
(() => {
    // 从隐藏输入字段读取配置
    const basePath = document.querySelector('input[name="js_base_path"]').value;
    const tagsSeparatorElement = document.querySelector('input[name="tags_separator"]');
    const tagsSeparator = tagsSeparatorElement ? tagsSeparatorElement.value || ' ' : ' ';

    // 所有事件绑定在此处执行
    initResponsiveMenu();
    initFoldButtons();
    initDeleteConfirm();
    initAlertClose();
    initVersionDismiss();
    initAutofocus();
    initSubHeaders();
    initTextareaResizer();
    initBulkActions();
    initTagManagement();
    // ... 更多初始化
})();
```

#### 1. 响应式菜单

```javascript
function toggleMenu(menu) {
    if (menu.classList.contains('open')) {
        setTimeout(toggleHorizontal, 500);
    } else {
        toggleHorizontal();
    }
    menu.classList.toggle('open');
    document.getElementById('menu-toggle').classList.toggle('x');
}

const menuToggle = document.getElementById('menu-toggle');
if (menuToggle != null) {
    menuToggle.addEventListener('click', () => toggleMenu(menu));
}

// 窗口大小变化时自动关闭菜单
window.addEventListener(WINDOW_CHANGE_EVENT, () => closeMenu(menu));
```

#### 2. 折叠/展开按钮

```javascript
function toggleFold(button, description, thumb) {
    if (button.classList.contains('fa-chevron-up')) {
        button.title = document.getElementById('translation-expand').innerHTML;
        if (description != null) description.style.display = 'none';
        if (thumb != null) thumb.style.display = 'none';
    } else {
        button.title = document.getElementById('translation-fold').innerHTML;
        if (description != null) description.style.display = 'block';
        if (thumb != null) thumb.style.display = 'block';
    }
    button.classList.toggle('fa-chevron-down');
    button.classList.toggle('fa-chevron-up');
}

// 单个折叠
[...foldButtons].forEach((foldButton) => {
    foldButton.addEventListener('click', (event) => {
        event.preventDefault();
        toggleFold(event.target, description, thumbnail);
    });
});

// 全部折叠/展开
[...foldAllButtons].forEach((foldAllButton) => {
    foldAllButton.addEventListener('click', (event) => {
        event.preventDefault();
        // 遍历所有折叠按钮，同步状态
        [].forEach.call(foldButtons, (foldButton) => {
            toggleFold(foldButton.firstElementChild, description, thumbnail);
        });
    });
});
```

#### 3. 删除确认

```javascript
const deleteLinks = document.querySelectorAll('.confirm-delete');
[...deleteLinks].forEach((deleteLink) => {
    deleteLink.addEventListener('click', (event) => {
        const type = event.currentTarget.getAttribute('data-type') || 'link';
        if (!confirm(document.getElementById(`translation-delete-${type}`).innerHTML)) {
            event.preventDefault();  // 取消操作
        }
    });
});
```

#### 4. 版本更新提示（localStorage）

```javascript
// 检查是否需要隐藏更新提示
if (newVersionMessage != null
    && localStorage.getItem('newVersionDismiss') != null
    && parseInt(localStorage.getItem('newVersionDismiss'), 10) + (7 * 24 * 60 * 60 * 1000) > (new Date()).getTime()
) {
    newVersionMessage.style.display = 'none';
}

// 用户点击"关闭"时记录时间戳
if (newVersionDismiss != null) {
    newVersionDismiss.addEventListener('click', () => {
        localStorage.setItem('newVersionDismiss', (new Date()).getTime().toString());
    });
}
```

#### 5. 文本域自动调整高度

```javascript
function init(description) {
    function resize() {
        const scrollTop = window.pageYOffset || ...;
        
        // 自动计算高度
        description.style.height = 'auto';
        description.style.height = `${description.scrollHeight + 10}px`;
        
        window.scrollTo(0, scrollTop);  // 防止跳动
    }

    // 绑定多个事件
    const observe = (element, event, handler) => {
        element.addEventListener(event, handler, false);
    };
    observe(description, 'change', resize);
    observe(description, 'cut', delayedResize);
    observe(description, 'paste', delayedResize);
    observe(description, 'drop', delayedResize);
    observe(description, 'keydown', delayedResize);

    resize();  // 初始化时执行一次
}
```

#### 6. 批量操作

```javascript
// 复选框变化时显示/隐藏操作栏
[...linkCheckboxes].forEach((checkbox) => {
    checkbox.addEventListener('change', () => {
        const count = [...document.querySelectorAll('.link-checkbox:checked')].length;
        if (count === 0 && bar.classList.contains('open')) {
            bar.classList.toggle('open');
        } else if (count > 0 && !bar.classList.contains('open')) {
            bar.classList.toggle('open');
        }
    });
});

// 批量删除
if (deleteButton != null && token != null) {
    deleteButton.addEventListener('click', (event) => {
        event.preventDefault();
        
        // 收集选中的书签
        const links = [];
        [...linkCheckedCheckboxes].forEach((checkbox) => {
            links.push({
                id: checkbox.value,
                title: document.querySelector(`.linklist-item[data-id="${checkbox.value}"] .linklist-link`).innerHTML,
            });
        });

        // 构建确认消息
        let message = `Are you sure you want to delete ${links.length} links?\n`;
        message += 'This action is IRREVERSIBLE!\n\nTitles:\n';
        links.forEach((item) => {
            message += `  - ${item.title}\n`;
        });

        if (window.confirm(message)) {
            window.location = `${basePath}/admin/shaare/delete?id=${ids.join('+')}&token=${token.value}`;
        }
    });
}

// 全选/取消全选
[...selectAllButtons].forEach((selectAllButton) => {
    selectAllButton.addEventListener('click', (e) => {
        e.preventDefault();
        const checked = selectAllButton.classList.contains('filter-off');
        [...linkCheckboxes].forEach((linkCheckbox) => {
            linkCheckbox.checked = checked;
            linkCheckbox.dispatchEvent(new Event('change'));  // 触发 change 事件
        });
    });
});
```

#### 7. 标签管理（AJAX）

```javascript
// 重命名标签
[...renameTagSubmits].forEach((rename) => {
    rename.addEventListener('click', (event) => {
        event.preventDefault();
        
        const xhr = new XMLHttpRequest();
        xhr.open('POST', `${basePath}/admin/tags`);
        xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
        xhr.onload = () => {
            if (xhr.status !== 200) {
                alert(`An error occurred. Return code: ${xhr.status}`);
                location.reload();
            } else {
                // 成功后更新 DOM（无需刷新页面）
                block.setAttribute('data-tag', totag);
                block.querySelector('a.tag-link').innerHTML = he.encode(totag);
                block.querySelector('a.tag-link').setAttribute('href', 
                    `${basePath}/?searchtags=${encodeURIComponent(totag)}`);
                
                // 刷新自动完成列表
                existingTags = existingTags.map((tag) => (tag === fromtag ? totag : tag));
                awesomepletes = updateAwesompleteList('.rename-tag-input', existingTags, awesomepletes, tagsSeparator);
                
                refreshToken(basePath);  // 刷新 CSRF token
            }
        };
        xhr.send(`renametag=1&fromtag=${fromtagUrl}&totag=${encodeURIComponent(totag)}&token=${refreshedToken}`);
    });
});

// 删除标签
[...deleteTagButtons].forEach((rename) => {
    rename.addEventListener('click', (event) => {
        event.preventDefault();
        if (confirm(`Are you sure you want to delete the tag "${tag}"?`)) {
            const xhr = new XMLHttpRequest();
            xhr.open('POST', `${basePath}/admin/tags`);
            xhr.onload = () => {
                block.remove();  // 直接从 DOM 移除
            };
            xhr.send(`deletetag=1&fromtag=${tagUrl}&token=${refreshedToken}`);
        }
    });
});
```

#### 8. CSRF Token 刷新

```javascript
function refreshToken(basePath, callback) {
    const xhr = new XMLHttpRequest();
    xhr.open('GET', `${basePath}/admin/token`);
    xhr.onload = () => {
        // 更新所有表单中的 token
        const elements = document.querySelectorAll('input[name="token"]');
        [...elements].forEach((element) => {
            element.setAttribute('value', xhr.responseText);
        });

        if (callback) {
            callback(xhr.response);
        }
    };
    xhr.send();
}

// AJAX 请求后刷新 token（防止 token 过期）
refreshToken(basePath);
```

#### 9. 标签自动完成（Awesomplete）

```javascript
function createAwesompleteInstance(element, separator, tags = []) {
    const awesome = new Awesomplete(Awesomplete.$(element));

    // 自定义过滤：支持搜索标志（-、~、+）
    awesome.filter = (text, input) => {
        let filterFunc = Awesomplete.FILTER_CONTAINS;
        let term = input.match(new RegExp(`[^${separator}]*$`))[0];
        const termFlagged = term.replace(/^[-~+]/, '');
        if (term !== termFlagged) {
            term = termFlagged;
            filterFunc = Awesomplete.FILTER_STARTSWITH;
        }
        return filterFunc(text, term);
    };

    // 选中后自动添加分隔符
    awesome.replace = (text) => {
        const before = awesome.input.value.match(new RegExp(`^(.+${separator}+)?[-~+]?|`))[0];
        awesome.input.value = `${before}${text}${separator}`;
    };

    // 不显示已选择的标签
    awesome.data = (item, input) => {
        while ((match = reg.exec(input))) {
            if (item === match[1]) return '';
        }
        return item;
    };

    return awesome;
}
```

### 4.3 异步元数据获取

**[metadata.js](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/assets/common/js/metadata.js)**

```javascript
// 编辑页面自动获取标题、描述
[...inputTitles].forEach((inputTitle) => {
    if (inputTitle.value.length > 0) {
        clearLoaders(loaders);
        return;
    }

    const url = form.querySelector('input[name="lf_url"]').value;
    const xhr = new XMLHttpRequest();
    xhr.open('GET', `${basePath}/admin/metadata?url=${encodeURI(url)}`, true);
    xhr.onload = () => {
        const result = JSON.parse(xhr.response);
        Object.keys(result).forEach((key) => {
            if (result[key] !== null && result[key].length) {
                const element = form.querySelector(`input[name="lf_${key}"], textarea[name="lf_${key}"]`);
                if (element != null && element.value.length === 0) {
                    element.value = he.decode(result[key]);  // HTML 实体解码
                }
            }
        });
        clearLoaders(loaders);
    };
    xhr.send();
});

// 列表页异步加载缩略图
[...thumbsToLoad].forEach((divElement) => {
    const { id } = divElement.closest('[data-id]').dataset;
    updateThumb(basePath, divElement, id);
});

function updateThumb(basePath, divElement, id) {
    const xhr = new XMLHttpRequest();
    xhr.open('PATCH', `${basePath}/admin/shaare/${id}/update-thumbnail`);
    xhr.responseType = 'json';
    xhr.onload = () => {
        if (xhr.status === 200 && response.thumbnail !== false) {
            const imgElement = divElement.querySelector('img');
            imgElement.src = response.thumbnail;
            imgElement.dataset.src = response.thumbnail;
            divElement.classList.remove('hidden');
        }
    };
    xhr.send();
}
```

### 4.4 图片懒加载

**[thumbnails.js](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/assets/common/js/thumbnails.js)**

```javascript
import Blazy from 'blazy';

(() => {
    new Blazy();  // 自动处理所有 data-src 属性的图片
})();
```

模板中的懒加载标记：
```html
<img data-src="{$root_path}/{$value.thumbnail}#" class="b-lazy" src="" alt="" />
```

### 4.5 批量书签操作

**[shaare-batch.js](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/assets/common/js/shaare-batch.js)**

```javascript
// AJAX 提交单个书签表单
const sendBookmarkForm = (basePath, formElement) => {
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        xhr.open('POST', `${basePath}/admin/shaare`);
        xhr.onload = () => {
            if (xhr.status !== 200) {
                alert(`An error occurred. Return code: ${xhr.status}`);
                reject();
            } else {
                formElement.closest('.edit-link-container').remove();
                resolve();
            }
        };
        xhr.send(formData);
    });
};

// 批量保存（显示进度条）
[...saveAllButtons].forEach((saveAllButton) => {
    saveAllButton.addEventListener('click', (e) => {
        e.preventDefault();
        
        const forms = [...getForms()];
        const nbForm = forms.length;
        let current = 0;
        
        document.querySelector('.dark-layer').style.display = 'block';
        
        // 并行发送所有请求
        const promises = [];
        forms.forEach((formElement) => {
            promises.push(sendBookmarkForm(basePath, formElement).then(() => {
                current += 1;
                progressBar.style.width = `${(current * 100) / nbForm}%`;
                progressBarCurrent.innerHTML = current;
            }));
        });

        Promise.all(promises).then(() => {
            window.location.href = `${basePath}/`;
        });
    });
});
```

### 4.6 前端-模板绑定模式

#### 模板标记 → JS 选择器的对应关系

| 模板标记（HTML） | JS 选择器 | 行为 |
|-----------------|-----------|------|
| `class="confirm-delete"` | `document.querySelectorAll('.confirm-delete')` | 删除确认 |
| `class="fold-button"` | `document.getElementsByClassName('fold-button')` | 折叠/展开 |
| `class="link-checkbox"` | `document.querySelectorAll('.link-checkbox')` | 批量选择 |
| `data-async-thumbnail="1"` | `document.querySelectorAll('div[data-async-thumbnail]')` | 异步缩略图 |
| `data-multiple` | `document.querySelectorAll('input[data-multiple]')` | 标签自动完成 |
| `name="lf_description"` | `document.getElementById('lf_description')` | 文本域自动调整 |
| `class="pure-alert-close"` | `document.querySelectorAll('.pure-alert-close')` | 关闭警告 |

#### 国际化（i18n）绑定方式

模板中定义翻译隐藏元素：
```html
<div id="translation-delete-link" style="display:none">{'Delete this link?'|t}</div>
<div id="translation-fold" style="display:none">{'Fold'|t}</div>
<div id="translation-expand" style="display:none">{'Expand'|t}</div>
```

JS 中读取：
```javascript
button.title = document.getElementById('translation-fold').innerHTML;
```

---

## 总结

Shaarli 的渲染、缓存与前端架构体现了以下设计特点：

1. **分层清晰**：模板选择 → 页面构建 → 缓存管理 → 前端交互，各层职责明确
2. **性能优先**：三级缓存策略（编译缓存、页面缓存、缩略图缓存），懒加载初始化
3. **安全可靠**：CSRF token 保护、XSS 转义、模板沙箱（RainTPL 黑名单）
4. **用户体验**：无刷新 AJAX 操作、localStorage 状态持久化、响应式设计
5. **可扩展性**：插件钩子系统、主题切换机制、模块化前端代码

核心文件速查：
- 渲染层：[PageBuilder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageBuilder.php)
- 缓存管理：[PageCacheManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/render/PageCacheManager.php)
- 模板引擎：[rain.tpl.class.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php)
- 前端核心：[base.js](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/assets/default/js/base.js)
- 入口配置：[index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/index.php)
