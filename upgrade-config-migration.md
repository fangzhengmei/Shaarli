# Shaarli 升级器与配置迁移代码理解

## 一、整体架构概览

Shaarli 的升级系统采用**双升级器**设计，配合**配置管理器**和**书签数据服务**实现全链路的版本升级、配置兼容与数据迁移。核心组件分布在以下模块：

| 模块 | 职责 | 核心文件 |
|------|------|----------|
| 升级执行框架 | 扫描并执行 update 方法、记录已完成更新 | [Updater.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/Updater.php) |
| 遗留升级器 | 兼容历史版本数据迁移（PHP 数组→对象） | [LegacyUpdater.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php) |
| 升级工具 | 读写 `updates.txt` 记录文件 | [UpdaterUtils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/UpdaterUtils.php) |
| 配置管理器 | 统一配置读写，支持 PHP→JSON 过渡 | [ConfigManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigManager.php) |
| 配置 I/O 层 | JSON / PHP 两种格式的读写实现 | [ConfigJson.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigJson.php)、[ConfigPhp.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigPhp.php) |
| 书签数据服务 | 检测数据格式并触发遗留迁移 | [BookmarkFileService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkFileService.php) |
| 中间件触发点 | 每次登录请求自动执行升级 | [ShaarliMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/ShaarliMiddleware.php) |

---

## 二、升级触发流程与执行步骤

### 2.1 升级触发入口

升级流程有**两个独立触发点**，分别覆盖不同的升级场景：

#### 触发点 1：HTTP 中间件（当前版本升级）

位于 [ShaarliMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/ShaarliMiddleware.php#L40-L83) 的 `__invoke()` 方法：

```php
public function __invoke(Request $request, Response $response, callable $next): Response
{
    $this->initBasePath($request);
    // ... 安装检查 ...
    $this->runUpdates();  // ← 每次请求都会尝试执行升级
    // ...
}
```

`runUpdates()` 方法 ([ShaarliMiddleware.php#L67-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/ShaarliMiddleware.php#L67-L83)) 的逻辑：

1. **权限校验**：仅登录用户可执行升级（`isLoggedIn() !== true` 则直接返回）
2. **设置 basePath**：传递 URL 子路径给升级器（用于 URL 修正类升级）
3. **执行升级**：调用 `$this->container->updater->update()`
4. **持久化进度**：若有新升级完成，写入 `data/updates.txt`
5. **清除缓存**：调用 `pageCacheManager->invalidateCaches()` 使页面缓存失效

#### 触发点 2：书签服务构造函数（遗留数据迁移）

位于 [BookmarkFileService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkFileService.php#L62-L105) 构造函数内：

```php
try {
    $this->bookmarks = $this->bookmarksIO->read();
} catch (...) {
    // 初始化空数据
}

// 关键：若读取的数据不是 BookmarkArray 类型，说明是遗留格式
if (! $this->bookmarks instanceof BookmarkArray) {
    $this->migrate();  // ← 触发 LegacyUpdater
    exit('Your data store has been migrated, please reload the page.');
}
```

`migrate()` 方法 ([BookmarkFileService.php#L422-L442](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkFileService.php#L422-L442))：

1. 实例化 `LegacyLinkDB` 读取旧格式数组数据
2. 实例化 `LegacyUpdater` 并传入已完成更新列表
3. 执行 `$updater->update()` 处理所有遗留升级方法
4. 将完成的升级方法名写入 `updates.txt`

### 2.2 Updater 核心执行流程

升级器基类的 `update()` 方法是统一执行引擎，[Updater.php#L76-L113](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/Updater.php#L76-L113)：

```
┌─────────────────────────────────────────────────┐
│              Updater::update()                  │
├─────────────────────────────────────────────────┤
│ 1. 登录检查：非登录用户直接返回 []               │
│ 2. 通过 Reflection 获取所有类方法                │
│ 3. 遍历每个方法：                                │
│    ├─ 方法名不以 'updateMethod' 开头 → 跳过      │
│    ├─ 方法名在 $doneUpdates 中 → 跳过            │
│    ├─ setAccessible(true) 允许调用 protected     │
│    ├─ invoke() 反射执行方法                      │
│    ├─ 返回值 === true → 加入 $updatesRan         │
│    └─ 捕获异常 → 包装为 UpdaterException 抛出    │
│ 4. 将成功的更新合并到 $doneUpdates                │
│ 5. 返回 $updatesRan 数组                         │
└─────────────────────────────────────────────────┘
```

**设计要点**：
- **反射扫描**：自动发现所有 `updateMethodXxx` 方法，无需手动注册
- **命名约定**：方法必须以 `updateMethod` 前缀开头
- **返回值约定**：必须返回 `true` 才被视为成功执行并记录
- **异常隔离**：单个更新方法失败不影响已成功的方法，但会终止后续执行

### 2.3 升级步骤时序图

```
HTTP Request
    │
    ▼
ShaarliMiddleware::__invoke()
    │
    ├─ initBasePath()
    │
    ├─ 检查是否已安装（未安装则跳转 /install）
    │
    └─ runUpdates()
          │
          ├─ isLoggedIn() === true? ──否──► 结束
          │
          ├─ updater->setBasePath()
          │
          ├─ updater->update()
          │     │
          │     ├─ 遍历 ReflectionMethod
          │     │    ├─ 前缀匹配 updateMethod?
          │     │    ├─ 已在 doneUpdates?
          │     │    ├─ 执行方法
          │     │    └─ 成功→记录 updatesRan
          │     │
          │     └─ 返回 updatesRan[]
          │
          ├─ updatesRan 非空?
          │     │
          │     ├─ 是→写入 updates.txt
          │     └─ 是→invalidateCaches()
          │
          └─ checkOpenShaarli()
```

---

## 三、配置文件读写机制

### 3.1 ConfigIO 接口设计

[ConfigIO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigIO.php) 定义了配置读写的抽象接口：

```php
interface ConfigIO {
    public function read($filepath);      // 读取→数组
    public function write($filepath, $conf);  // 数组→写入文件
    public function getExtension();       // 返回文件扩展名
}
```

有两个实现类，形成**渐进式迁移**：

| 实现类 | 扩展名 | 存储格式 | 定位 |
|--------|--------|----------|------|
| `ConfigPhp` | `.php` | PHP `$GLOBALS` 变量 | 遗留格式，只读模式用于过渡 |
| `ConfigJson` | `.json.php` | JSON 包裹在 PHP 注释中 | 当前推荐格式 |

### 3.2 ConfigJson 读写实现

[ConfigJson.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigJson.php)：

**读取流程** (`read()` [L14-L39](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigJson.php#L14-L39))：
1. 文件不可读 → 返回空数组
2. `file_get_contents()` 读取原始内容
3. 去除 PHP 包裹头 `<?php /*` 和尾 `*/ ?>`
4. `json_decode(..., true)` 解析为关联数组
5. 解析失败 → 抛出详细异常（含错误码、JSON lint 提示）

**写入流程** (`write()` [L44-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigJson.php#L44-L56))：
1. `json_encode($conf, JSON_PRETTY_PRINT)` 格式化输出
2. 拼接 PHP 安全包裹：`<?php /* JSON */ ?>`
3. `file_put_contents()` 写入
4. 写入失败 → 抛出 `IOException`

> **安全设计**：JSON 内容包裹在 PHP 注释标签中，即使配置文件被 HTTP 直接访问，PHP 引擎会执行 `<?php /* ... */ ?>`，返回空白页，避免泄露敏感信息。

### 3.3 ConfigPhp 遗留格式与键名映射

[ConfigPhp.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigPhp.php) 承担双重角色：

#### 角色 1：读取遗留 PHP 配置

`read()` 方法 ([L77-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigPhp.php#L77-L92))：
```php
include $filepath;  // 执行 PHP 文件，填充 $GLOBALS
// 提取 $ROOT_KEYS 中定义的全局变量
// 提取 $GLOBALS['config'] 和 $GLOBALS['plugins'] 子数组
```

#### 角色 2：提供键名映射表

`$LEGACY_KEYS_MAPPING` ([L35-L72](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigPhp.php#L35-L72)) 是**新旧键名双向映射**的核心：

```php
// 格式：'新分层键名' => '遗留扁平键名'
public static $LEGACY_KEYS_MAPPING = [
    'credentials.login'      => 'login',
    'credentials.hash'       => 'hash',
    'general.title'          => 'title',
    'general.header_link'    => 'titleLink',
    'privacy.default_private_links' => 'privateLinkByDefault',
    'resource.data_dir'      => 'config.DATADIR',
    // ... 共 40+ 条映射
];
```

ConfigManager 在 `get/set/remove/exists` 四个操作中都会自动应用此映射（当 ConfigIO 是 ConfigPhp 实例时）。

### 3.4 ConfigManager 核心逻辑

[ConfigManager.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigManager.php) 是上层统一入口。

#### 初始化与自动格式检测

`initialize()` 方法 ([L71-L79](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigManager.php#L71-L79))：
```php
if (file_exists($this->configFile . '.php')) {
    $this->configIO = new ConfigPhp();   // 检测到遗留 PHP 配置
} else {
    $this->configIO = new ConfigJson();  // 默认使用 JSON
}
$this->load();
```

#### 分层键访问

支持**点分隔的嵌套键**，如 `general.timezone` → `$conf['general']['timezone']`。

三个递归静态方法实现底层操作：
- `getConfig($settings, $conf)` - 递归读取
- `setConfig($settings, $value, &$conf)` - 递归写入
- `removeConfig($settings, &$conf)` - 递归删除

#### 默认值填充

`setDefaultValues()` ([L344-L411](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigManager.php#L344-L411)) 为缺失的配置项提供合理默认值，涵盖：
- 资源路径（data_dir、datastore、cache 等）
- 安全配置（ban_after、session_protection 等）
- 通用设置（title、links_per_page、timezone 等）
- 缩略图、翻译、插件、格式化器等

#### 写入权限与必填校验

`write()` 方法 ([L214-L241](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/config/ConfigManager.php#L214-L241)) 执行两级检查：

1. **权限校验**：配置文件已存在时，必须登录才能修改
2. **必填校验**：8 个强制字段必须存在：
   - `credentials.login`、`credentials.hash`、`credentials.salt`
   - `security.session_protection_disabled`
   - `general.timezone`、`general.title`、`general.header_link`
   - `privacy.default_private_links`

---

## 四、历史数据迁移机制

### 4.1 配置格式迁移：PHP → JSON

由 `LegacyUpdater::updateMethodConfigToJson()` 实现 ([LegacyUpdater.php#L173-L212](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L173-L212))。

**迁移步骤**：

```
┌───────────────────────────────────────────────────────┐
│ 1. 守卫条件：ConfigIO 已是 ConfigJson → return true   │
├───────────────────────────────────────────────────────┤
│ 2. 实例化 ConfigPhp / ConfigJson                       │
├───────────────────────────────────────────────────────┤
│ 3. 读取旧 config.php 内容到 $oldConfig                  │
├───────────────────────────────────────────────────────┤
│ 4. 备份：rename config.php → config.save.php           │
├───────────────────────────────────────────────────────┤
│ 5. 切换 ConfigIO 为 ConfigJson 并 reload()             │
├───────────────────────────────────────────────────────┤
│ 6. 遍历 $ROOT_KEYS，通过 LEGACY_KEYS_MAPPING 反向映射   │
│    array_flip() 将 'login' → 'credentials.login'       │
├───────────────────────────────────────────────────────┤
│ 7. 遍历子配置 ['config', 'plugins']，逐个键转换写入     │
├───────────────────────────────────────────────────────┤
│ 8. $conf->write() 持久化 JSON 文件                      │
│    失败 → error_log + return false                     │
└───────────────────────────────────────────────────────┘
```

### 4.2 废弃配置合并：options.php → config

`updateMethodMergeDeprecatedConfigFile()` ([LegacyUpdater.php#L147-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L147-L165))：

1. 检查 `data/options.php` 是否存在
2. `include` 该文件以注入 `$GLOBALS`
3. 将允许的键（`ConfigPhp::$ROOT_KEYS` + 'config'）从 `$GLOBALS` 迁移到 ConfigManager
4. `$conf->write()` 持久化
5. `unlink()` 删除旧 options.php

### 4.3 数据存储 ID 系统迁移

`updateMethodDatastoreIds()` ([LegacyUpdater.php#L247-L280](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L247-L280)) 将主键从**日期字符串**改为**自增整数 ID**。

**迁移前后对比**：

| 项目 | 迁移前 | 迁移后 |
|------|--------|--------|
| 主键格式 | `'20121206_182539'` (linkdate) | `0`, `1`, `2` ... (整数) |
| 日期存储 | `linkdate` 字段 | `created` (DateTime 对象) |
| 更新日期 | 无 | `updated` (DateTime 对象) |
| 短链接 | 无 | `shorturl` 字段 |

**迁移步骤**：
1. **守卫检查**：取第一条记录的 key，若已是整数则直接返回（幂等）
2. **自动备份**：`copy(datastore, datastore.YYYYMMDDHHmmss.php)`
3. **数据重建**：
   - 提取所有值到顺序数组
   - `array_reverse()` 翻转（旧数据在前）
   - 移除 `linkdate` 字段，添加 `id = cpt++`
   - LinkDB 构造器自动将日期字符串转为 DateTime
4. **持久化**：`$linkDB->save()` + `$linkDB->reorder()`

### 4.4 遗留数组 → Bookmark 对象迁移

`updateMethodMigrateDatabase()` ([LegacyUpdater.php#L581-L596](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L581-L596))：

```php
// 1. 备份 datastore → datastore.YYYYMMDDHHmmss_1.php
copy($this->conf->get('resource.datastore'), $save);

// 2. 将每条 LegacyLinkDB 数组转为 Bookmark 对象
$linksArray = new BookmarkArray();
foreach ($this->linkDB as $key => $link) {
    $linksArray[$key] = (new Bookmark())
        ->fromArray($link, $this->conf->get('general.tags_separator', ' '));
}

// 3. 通过 BookmarkIO 写入新格式（gzip + base64 + PHP 包裹）
$linksIo = new BookmarkIO($this->conf);
$linksIo->write($linksArray);
```

### 4.5 BookmarkIO 数据存储格式

[BookmarkIO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php) 定义了当前版本的数据序列化格式：

```
文件内容结构：
  <?php /* <base64(gzdeflate(serialize(Bookmark[])))> */ ?>
```

写入流程 (`write()` [L114-L142](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L114-L142))：
1. `serialize($links)` → PHP 序列化字符串
2. `gzdeflate()` → 压缩
3. `base64_encode()` → ASCII 安全编码
4. 拼接 PHP 注释包裹 `<?php /* ... */ ?>`
5. **互斥锁保护**：`$this->mutex->synchronized()` 确保并发安全
6. **磁盘空间检查**：预留 500KB 安全余量

读取流程 (`read()` [L75-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L75-L104))：反向执行上述解码步骤。

### 4.6 其他历史迁移方法一览

| 方法名 | 功能 |
|--------|------|
| `updateMethodEscapeUnescapedConfig` | 对 title、header_link 等配置做 HTML escape |
| `updateMethodRenameDashTags` | 移除标签前缀 `-` (支持排除搜索语法) |
| `updateMethodApiSettings` | 初始化 API enabled + secret |
| `updateMethodDefaultTheme` | 重构模板目录结构：tpl/ → tpl/{theme}/ |
| `updateMethodMoveUserCss` | `inc/user.css` → `data/user.css` |
| `updateMethodEscapeMarkdown` | 根据 markdown 插件启用情况设置 markdown_escape |
| `updateMethodPiwikUrl` | 自动补全 Piwik URL 的 http:// 前缀 |
| `updateMethodAtomDefault` | 默认启用 Atom Feed |
| `updateMethodResetHistoryFile` | 重置 history.php（日期格式变更） |
| `updateMethodReorderDatastore` | 重新排序 datastore 持久化 |
| `updateMethodVisibilitySession` | session key `privateonly` → `visibility` |
| `updateMethodDownloadSizeAndTimeoutConf` | 添加下载大小和超时默认配置 |
| `updateMethodWebThumbnailer` | 缩略图设置迁移到 WebThumbnailer |
| `updateMethodSetSticky` | 为所有书签添加 sticky=false 默认值 |
| `updateMethodRemoveRedirector` | 删除已废弃的 redirector 配置 |
| `updateMethodFormatterSetting` | markdown 插件→核心 formatter 配置迁移 |
| `updateMethodRelativeHomeLink` | `header_link='?'` → `'/subfolder/'` (Slim 路由兼容) |
| `updateMethodMigrateExistingNotesUrl` | 笔记 URL `?abcdef` → `/shaare/abcdef` |
| `updateMethodRemoveSettingRemoteBranch` | 删除 `updates.check_updates_branch` |

---

## 五、重复执行幂等性设计

Shaarli 升级系统通过**多层守卫**确保每个 update 方法可安全重复执行。

### 5.1 第一层：已完成记录过滤

升级器框架级别的幂等保护，位于 `Updater::update()` ([Updater.php#L89-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/Updater.php#L89-L96))：

```php
foreach ($this->methods as $method) {
    if (
        !startsWith($method->getName(), 'updateMethod')
        || in_array($method->getName(), $this->doneUpdates)  // ← 关键：已执行则跳过
    ) {
        continue;
    }
    // ...
}
```

**持久化存储**：`data/updates.txt` 以分号分隔的方法名列表。

读写实现见 [UpdaterUtils.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/UpdaterUtils.php)：
- `readUpdatesFile()`: `explode(';', $content)` 
- `writeUpdatesFile()`: `implode(';', $updates)`

### 5.2 第二层：方法内前置条件检查

每个 `updateMethodXxx` 在执行实际变更前都会做**状态检测**，已完成则直接 `return true`。典型模式：

#### 模式 A：配置项存在性检查

```php
// updateMethodApiSettings
public function updateMethodApiSettings()
{
    if ($this->conf->exists('api.secret')) {  // ← 守卫
        return true;
    }
    // 实际操作...
}
```

#### 模式 B：数据格式检查

```php
// updateMethodDatastoreIds
$first = 'update';
foreach ($this->linkDB as $key => $link) {
    $first = $key;
    break;
}
if (is_int($first)) {  // ← 主键已是整数，迁移完成
    return true;
}
```

#### 模式 C：文件/目录存在性检查

```php
// updateMethodMoveUserCss
if (!is_file('inc/user.css')) {  // ← 源文件不存在，无需迁移
    return true;
}
return rename('inc/user.css', 'data/user.css');
```

#### 模式 D：值格式检查

```php
// updateMethodPiwikUrl
if (!$this->conf->exists('plugins.PIWIK_URL')
    || startsWith($this->conf->get('plugins.PIWIK_URL'), 'http')) {  // ← 已有前缀
    return true;
}
```

#### 模式 E：ConfigIO 类型检查

```php
// updateMethodConfigToJson
if ($this->conf->getConfigIO() instanceof ConfigJson) {  // ← 已是 JSON
    return true;
}
```

### 5.3 幂等性测试验证

测试用例覆盖了每个 update 方法的 "nothing to do" 场景，例如 [LegacyUpdaterTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/tests/legacy/LegacyUpdaterTest.php) 中：

- `testConfigToJsonNothingToDo()` - JSON 已存在时不重复转换
- `testDatastoreIdsNothingToDo()` - ID 已迁移时不重建
- `testUpdateApiSettingsNothingToDo()` - API 配置已存在时不覆盖
- `testEscapeMarkdownSettingNothingToDoEnabled()` - markdown 配置已设置时保持
- `testUpdatePiwikUrlNothingToDo()` - URL 已是 http 开头时不变
- `testUpdateStickyNothingToDo()` - 任一书签已有 sticky 字段时跳过

测试模式统一为：**执行前记录校验和/值 → 执行 update → 执行后校验和/值不变**。

---

## 六、中断恢复机制

### 6.1 原子性与备份策略

升级系统采用**先备份后操作**的策略，确保任何中断都可回滚。

#### 数据存储备份

涉及 datastore 重写的升级方法都会先创建时间戳备份：

```php
// updateMethodDatastoreIds
$save = $this->conf->get('resource.data_dir') . '/datastore.' . date('YmdHis') . '.php';
copy($this->conf->get('resource.datastore'), $save);

// updateMethodMigrateDatabase
$save = $this->conf->get('resource.data_dir') . '/datastore.' . date('YmdHis') . '_1.php';
copy($this->conf->get('resource.datastore'), $save);
```

备份文件命名约定：
- 普通 ID 迁移：`datastore.YYYYMMDDHHmmss.php`
- 对象格式迁移：`datastore.YYYYMMDDHHmmss_1.php`（后缀 `_1` 区分）

#### 配置文件备份

```php
// updateMethodConfigToJson
rename($this->conf->getConfigFileExt(), $this->conf->getConfigFile() . '.save.php');
// config.php → config.save.php（不是删除，而是重命名保留）
```

### 6.2 进度持久化与断点续传

升级进度分**内存态**和**持久态**两层：

```
执行中 (内存态)           成功后 (持久态)
┌─────────────────┐      ┌──────────────────┐
│ $updatesRan[]   │ ───► │ data/updates.txt │
│ (单次请求内)    │      │ (跨请求持久)     │
└─────────────────┘      └──────────────────┘
```

关键代码在 [ShaarliMiddleware.php#L74-L82](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/ShaarliMiddleware.php#L74-L82)：

```php
$newUpdates = $this->container->updater->update();
if (!empty($newUpdates)) {
    $this->container->updater->writeUpdates(
        $this->container->conf->get('resource.updates'),
        $this->container->updater->getDoneUpdates()
    );
    $this->container->pageCacheManager->invalidateCaches();
}
```

**中断场景分析**：

| 中断时机 | 后果 | 恢复方式 |
|----------|------|----------|
| update() 执行中途 | 部分方法已执行但未写入 updates.txt | 下次请求重新执行已执行的方法（靠方法内幂等守卫保护） |
| update() 成功但 writeUpdates 前 | 所有方法执行完毕但未记录 | 下次请求重新扫描，但每个方法内部守卫会立即 return true |
| writeUpdates 写入失败 | 抛出异常显示错误页 | 用户刷新重试，同上 |
| migrate() 遗留迁移中途 | exit() 提示刷新 + 删除 updates.txt | 按提示操作或手动删除 data/updates.txt |

### 6.3 并发安全

书签读写通过 `malkusch/lock` 库的 `FlockMutex` 实现文件锁互斥：

```php
// ContainerBuilder.php
$container['bookmarkService'] = function (ShaarliContainer $container) {
    return new BookmarkFileService(
        // ...
        new FlockMutex(fopen(SHAARLI_MUTEX_FILE, 'r'), 2),  // ← 共享锁文件
        // ...
    );
};
```

在 `BookmarkIO::synchronized()` ([BookmarkIO.php#L152-L159](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L152-L159)) 中：
```php
protected function synchronized(callable $function): void
{
    try {
        $this->mutex->synchronized($function);
    } catch (LockAcquireException $exception) {
        $function();  // 获取锁失败时降级为无锁执行（兼容共享主机）
    }
}
```

### 6.4 错误处理与回滚指引

#### UpdaterException 包装

[UpdaterException.php](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/exception/UpdaterException.php) 会记录失败的方法名和原始异常：

```php
private function buildMessage($message)
{
    $out = '';
    if (!empty($message))  $out .= $message . PHP_EOL;
    if (!empty($this->method))  $out .= t('An error occurred while running the update ') . $this->method . PHP_EOL;
    if (!empty($this->previous)) $out .= '  ' . $this->previous->getMessage();
    return $out;
}
```

#### 迁移失败的手动恢复

BookmarkFileService 中的 migrate() 在迁移后给出明确操作指引 ([BookmarkFileService.php#L96-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkFileService.php#L96-L99))：

```php
exit(
    'Your data store has been migrated, please reload the page.' . PHP_EOL .
    'If this message keeps showing up, please delete data/updates.txt file.'
);
```

**手动恢复步骤**：
1. 刷新页面重试（断点续传）
2. 若持续报错，删除 `data/updates.txt` 重置升级进度
3. 从时间戳备份文件恢复 datastore
4. 从 `config.save.php` 恢复配置

---

## 七、版本升级完整操作手册

### 7.1 标准升级流程

1. **替换代码文件**：覆盖新版本 Shaarli 源码
2. **登录触发**：管理员浏览器访问 Shaarli 并登录
3. **自动执行**：ShaarliMiddleware 自动检测并执行所有待执行的 `updateMethod*`
4. **验证结果**：检查页面是否正常加载，书签数据完整
5. **清理备份**：确认无误后，可删除 `data/*.php` 时间戳备份和 `config.save.php`

### 7.2 升级进度检查

检查 `data/updates.txt` 内容，其中包含所有已成功执行的更新方法名（分号分隔）。

### 7.3 故障排查

| 现象 | 可能原因 | 解决方案 |
|------|----------|----------|
| 升级时显示 "An error occurred while running the update..." | 单个 update 方法异常 | 检查 PHP error_log，从备份恢复后重试 |
| 持续显示 "Your data store has been migrated" | 遗留迁移循环 | 删除 `data/updates.txt` |
| 配置丢失 | PHP→JSON 迁移失败 | 恢复 `config.save.php` 为 `config.php`，重新升级 |
| 数据乱码 | datastore 迁移中断 | 从 `datastore.YYYYMMDDHHmmss.php` 备份恢复 |

---

## 八、设计模式总结

| 设计模式 | 应用场景 |
|----------|----------|
| **策略模式** | ConfigIO 接口 + ConfigPhp/ConfigJson 实现，可互换 |
| **模板方法** | Updater::update() 定义执行骨架，子类提供具体 updateMethod |
| **反射扫描** | 自动发现 updateMethod 前缀方法，免注册 |
| **多层守卫** | 框架级 doneUpdates 过滤 + 方法级前置检查 = 强幂等 |
| **备忘录** | 升级前自动备份 datastore/config，支持回滚 |
| **断点续传** | updates.txt 持久化进度，支持从中断处继续 |
| **互斥锁** | FlockMutex 保证并发安全，降级兼容共享主机 |
