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

---

## 九、崩溃重试：方法体中途中断后的幂等行为逐路径推演

升级过程中，PHP 进程可能因 `max_execution_time` 超时、内存耗尽、用户手动刷新、服务器断电等原因在 `updateMethod` 内部中途崩溃。由于 `updates.txt` 只在所有方法成功返回后才写入，崩溃必然导致**进度未记录**，下次请求会重新从第一个未记录的方法开始执行。此时方法的安全性完全依赖自身内部的前置守卫。

以下对所有高风险迁移方法做**逐断点推演**。

### 9.1 updateMethodConfigToJson：PHP → JSON 配置迁移

代码位于 [LegacyUpdater.php#L173-L212](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L173-L212)，内部操作时序：

```
时序号  操作                                       崩溃后结果
──────  ─────────────────────────────────────────  ────────────────────────────
T1  实例化 ConfigPhp / ConfigJson                 无副作用，安全
T2  configPhp->read() 读取 $oldConfig             无副作用，安全
T3  rename(config.php → config.save.php)          ← 原子文件系统操作
T4  conf->setConfigIO(ConfigJson) + reload()      切换为 JSON IO
T5  遍历 $ROOT_KEYS 写入内存 ConfigManager        仅内存态，崩溃则丢失
T6  遍历子配置 config/plugins 写入内存             仅内存态，崩溃则丢失
T7  conf->write() 持久化 JSON 到磁盘              ← 真正落盘
T8  return true
```

**逐断点分析**：

| 崩溃时机 | 磁盘状态 | 下次重试行为 | 是否幂等安全 |
|----------|----------|--------------|--------------|
| T3 之前 | config.php 存在，无 save.php | 守卫 `instanceof ConfigJson` 不成立，重新从 T1 开始 | ✅ 安全 |
| T3 完成，T4 之前 | config.php 已消失，仅 config.save.php | 守卫 `instanceof ConfigJson` 仍不成立（ConfigManager 初始化时找不到 `.php` → 自动走 JSON，但 `config.json.php` 不存在 → read 返回空数组 → **丢失所有配置**） | ❌ **危险** |
| T5 期间 | config.save.php 存在，config.json.php 尚未写入 | 同上，空配置加载，守卫检查必填字段时在后续 write 抛 MissingFieldConfigException | ⚠️ 可恢复，需手动把 save.php 改回 .php |
| T7 完成之后 | config.json.php 已写入 | 守卫 `instanceof ConfigJson` 成立，直接 return true | ✅ 安全 |

> **关键缺陷**：T3 的 rename 是不可逆的单点，在 T7 完成 JSON 写入前若崩溃，会陷入「无配置可加载」状态。必须靠手动将 `config.save.php` 重命名回 `config.php` 才能回滚重试。

### 9.2 updateMethodDatastoreIds：日期主键 → 整数 ID

代码位于 [LegacyUpdater.php#L247-L280](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L247-L280)：

```
T1  取第一条记录 key 判断类型（is_int）
T2  copy(datastore → datastore.YYYYMMDDHHmmss.php)   ← 备份
T3  提取所有 value 到顺序数组 $links[]
T4  unset($this->linkDB[$offset]) 清空内存           ← 仅内存
T5  array_reverse($links) 翻转                       ← 仅内存
T6  遍历每条：unset(linkdate) + id = cpt++           ← 仅内存
T7  linkDB->save() 持久化到磁盘                      ← 真正落盘
T8  linkDB->reorder()
T9  return true
```

**逐断点分析**：

| 崩溃时机 | 磁盘状态 | 下次重试守卫 | 结果 |
|----------|----------|--------------|------|
| T2 之前 | 原 datastore（日期主键） | T1 取 key 是字符串，继续执行 | ✅ 安全 |
| T2 之后 T7 之前 | 原 datastore（日期）+ 备份文件存在 | T1 取 key 仍是字符串，**重复创建备份**（同一秒则覆盖，不同秒则新增多个备份） | ✅ 安全（但产生冗余备份） |
| T7 完成之后 | datastore 已改为整数主键 | T1 取 key 是 int → return true | ✅ 安全 |

### 9.3 updateMethodMigrateDatabase：数组 → Bookmark 对象

代码位于 [LegacyUpdater.php#L581-L596](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L581-L596)：

```
T1  copy(datastore → datastore.YYYYMMDDHHmmss_1.php)  ← 备份
T2  遍历每条 array → Bookmark::fromArray()           ← 仅内存
T3  BookmarkIO->write(BookmarkArray)                 ← 真正落盘
T4  return true
```

| 崩溃时机 | 磁盘状态 | BookFileService 识别 | 重试结果 |
|----------|----------|----------------------|----------|
| T3 之前 | datastore 仍是 Legacy 格式 | `instanceof BookmarkArray` 为 false → 走 migrate() | ✅ 安全，重复备份+重写 |
| T3 完成 | datastore 已是 BookmarkArray | `instanceof BookmarkArray` 为 true → 正常加载 | ✅ 安全 |

### 9.4 updateMethodMergeDeprecatedConfigFile：合并 options.php

代码位于 [LegacyUpdater.php#L147-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyUpdater.php#L147-L165)：

```
T1  is_file(options.php) 守卫
T2  include options.php 注入 $GLOBALS
T3  遍历写入 ConfigManager 内存
T4  conf->write() 持久化 config 文件                ← 落盘
T5  unlink(options.php)                             ← 删除源
T6  return true
```

| 崩溃时机 | 磁盘状态 | 重试行为 |
|----------|----------|----------|
| T4 之前 T5 之后 | options.php 还在，T1 守卫通过，重新执行 T2-T5，**配置被重复合并写入**（值相同则无影响），最后 unlink | ✅ 安全 |
| T4 之前 T4 之前 | T4 write 崩溃，配置未落盘，options.php 还在，下次全量重做 | ✅ 安全 |
| T5 之后 | options.php 已删，T1 守卫不成立，直接 return true | ✅ 安全 |

### 9.5 updateMethodApiSettings：初始化 API 密钥

```
T1  conf->exists('api.secret') 守卫                 ← 关键幂等点
T2  conf->set('api.enabled', true)                  ← 内存
T3  conf->set('api.secret', generate_api_secret())  ← 内存（每次生成不同值）
T4  conf->write()                                    ← 落盘
T5  return true
```

| 崩溃时机 | 重试行为 | 是否安全 |
|----------|----------|----------|
| T4 之前 | T1 守卫 `api.secret` 不存在 → 重新从 T2 执行，**每次生成的 secret 都不同**（取决于 login+salt 的 hash，但如果后续还有依赖这个 secret 的外部应用会出问题） | ⚠️ 功能安全，secret 改变导致旧 API token 失效 |
| T4 之后 | T1 守卫存在 → return true | ✅ 安全 |

### 9.6 updateMethodEscapeUnescapedConfig：二次转义风险

```
T1  conf->set('general.title', escape(原始值))       ← 如果原始值已经是转义后的
T2  conf->set('general.header_link', escape(...))
T3  conf->write()
T4  return true
```

**严重缺陷**：该方法**没有任何前置守卫**。如果 T3 之后崩溃但 updates.txt 未写入，下次重试会对**已经转义过的值再次调用 escape()**，导致双重转义。例如：
- 原始值：`<script>` → 第一次 escape：`&lt;script&gt;`
- 第二次 escape：`&amp;lt;script&amp;gt;` → 页面显示 `&lt;script&gt;` 而非正确的转义结果

这是整个升级系统中**幂等性最弱**的方法。

### 9.7 updateMethodMigrateExistingNotesUrl：笔记 URL 迁移

代码位于 [Updater.php#L151-L173](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/Updater.php#L151-L173)：

```
T1  遍历所有 bookmark
T2  检查 isNote() + startsWith(url, '?') + 正则匹配   ← 格式守卫
T3  setUrl('/shaare/' + hash) + bookmarkService->set(false)
T4  如果有 $updated=true，则 bookmarkService->save()   ← 批量落盘
T5  return true
```

| 崩溃时机 | 重试行为 | 安全性 |
|----------|----------|--------|
| T4 之前，部分已 set 但未 save | 下次遍历：已修改的 URL 已不符合 `startsWith('?')` 守卫 → 跳过；未修改的继续处理 | ✅ 安全（部分进度丢失但重复安全） |
| T4 之前，全部已 set 但 save 崩溃 | 同上，T2 守卫全部跳过，$updated=false，不执行 save，但 return true | ✅ 安全（但此时数据库其实未持久化，靠下次请求的 save？不——set(false) 未 save，数据仅在内存，下一次请求从磁盘重新读取，**所有 URL 恢复原状，重新迁移**） | ✅ 安全 |
| T4 完成 | T2 守卫全部不匹配，立即 return true | ✅ 安全 |

### 9.8 崩溃重试安全等级总表

| 方法 | 前置守卫 | 崩溃重试安全性 | 特殊风险 |
|------|----------|----------------|----------|
| updateMethodConfigToJson | ConfigIO 类型检查 | ⚠️ 中高 | T3 rename 与 T7 write 之间崩溃需手动回滚 save.php |
| updateMethodDatastoreIds | 主键 is_int 检查 | ✅ 高 | 仅产生冗余备份文件 |
| updateMethodMigrateDatabase | 无（靠 BookFileService 识别） | ✅ 高 | 同上 |
| updateMethodMergeDeprecatedConfigFile | is_file(options.php) | ✅ 高 | 仅重复合并（相同值无影响） |
| updateMethodApiSettings | exists(api.secret) | ⚠️ 中 | 崩溃前未 write → 重试生成不同 secret |
| updateMethodEscapeUnescapedConfig | **无** | ❌ **低** | 双重转义导致数据损坏 |
| updateMethodEscapeMarkdown | exists(markdown_escape) | ✅ 高 | |
| updateMethodDefaultTheme | file_exists(linklist.html) | ✅ 高 | |
| updateMethodMoveUserCss | is_file(inc/user.css) | ✅ 高 | |
| updateMethodPiwikUrl | startsWith(http) | ✅ 高 | |
| updateMethodSetSticky | 任一条目 isset(sticky) | ✅ 高 | 只要第一条有 sticky 就认为已完成（不完全但可接受） |
| updateMethodMigrateExistingNotesUrl | 每条 URL 格式检查 | ✅ 高 | |
| updateMethodRelativeHomeLink | header_link === '?' | ✅ 高 | |

---

## 十、并发竞态：多请求同时触发升级的竞争分析

升级流程虽然只会在 `isLoggedIn === true` 时触发，但管理员打开多个 Tab 同时访问页面、或 API/Web 并发请求等场景下，**可能有多个 PHP 进程同时进入 runUpdates**。由于升级流程整体没有分布式锁，所有竞态依赖文件系统操作的原子性和业务级守卫。

### 10.1 updates.txt 的读写竞态

关键代码路径位于 [ShaarliMiddleware.php#L74-L82](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/ShaarliMiddleware.php#L74-L82)：

```php
// 进程 A 和 进程 B 几乎同时执行以下代码：
$newUpdates = $this->container->updater->update();    // ← 读 updates.txt
if (!empty($newUpdates)) {
    $this->container->updater->writeUpdates(          // ← 写 updates.txt
        $this->container->conf->get('resource.updates'),
        $this->container->updater->getDoneUpdates()
    );
}
```

时间线推演（两进程同时升级同一方法 Mx）：

```
时刻   进程 A                                   进程 B
────   ──────────────────────────────────────   ──────────────────────────────────────
T0     readUpdatesFile() → [M1, M2]
T1                                              readUpdatesFile() → [M1, M2]
T2     update(): 发现 M3 待执行
T3     执行 M3 → 成功，updatesRan=[M3]
T4                                              update(): 发现 M3 待执行（A 还没写回）
T5                                              执行 M3 → 成功，updatesRan=[M3]
T6     writeUpdates([M1, M2, M3])  ✓
T7                                              writeUpdates([M1, M2, M3])  ✓
```

**结果分析**：
- 两进程都执行了 M3 方法体（重复执行），**安全性靠方法自身内部守卫**（见第九章）
- updates.txt 最终内容一致（写入相同的合并后数组），无文件内容损坏
- **方法体被重复执行**是最大风险，但方法的幂等守卫已覆盖大部分场景

### 10.2 时间戳备份文件的覆盖竞态

备份文件命名使用 `date('YmdHis')` **秒级精度**：

```php
// updateMethodDatastoreIds [LegacyUpdater.php#L260]
$save = $this->conf->get('resource.data_dir') . '/datastore.' . date('YmdHis') . '.php';

// updateMethodMigrateDatabase [LegacyUpdater.php#L583]
$save = $this->conf->get('resource.data_dir') . '/datastore.' . date('YmdHis') . '_1.php';
```

**竞态场景**：两进程在**同一秒**内进入同一会产生备份的方法：

| 命名模式 | 并发结果 |
|----------|----------|
| `datastore.YYYYMMDDHHmmss.php` | `copy()` 目标文件同名 → 后执行的进程**覆盖**先执行进程的备份 → 只保留一份（内容相同，无数据损失，但缺少「独立备份」） |
| `datastore.YYYYMMDDHHmmss_1.php` | 同上，后执行覆盖先执行 |
| 两方法在同一秒触发 | 一个带 `_1` 后缀一个不带，互相不冲突 |

> 注：备份文件内容相同（都是 copy 自同一个源 datastore），所以覆盖不造成数据损失，只是少了一份冗余。

### 10.3 datastore 并发迁移的写竞态

`updateMethodMigrateDatabase()` 中执行 `BookmarkIO->write()`：

BookmarkIO 的 write 方法 ([BookmarkIO.php#L114-L142](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L114-L142)) 有 **FlockMutex 互斥锁**保护：

```php
$this->synchronized(function () use ($data) {
    file_put_contents($this->datastore, $data);
});
```

但关键问题：**两个进程都在自己的内存中完成了 array→Bookmark 转换，然后竞争写同一个 datastore 文件**。

时间线推演：
```
T0  A: 读取旧 datastore（Legacy 数组格式）
T1  B: 读取旧 datastore（Legacy 数组格式）
T2  A: 在内存中转换为 BookmarkArray → 获取锁 → 写入磁盘 ✓
T3  B: 在内存中转换为 BookmarkArray → 等待锁 → 写入磁盘 ✓
```

结果：B 覆盖了 A 的写入，但由于两者输入相同，转换逻辑是纯函数，**写入内容完全一致**，覆盖不造成数据差异。

但若在 A 写入后、B 读取前的窗口，有用户**新增了一个书签**（新 Bookmark 格式），则 B 会：
1. 读取的是混合格式（部分 Bookmark 对象 + 部分数组？不可能，因为文件是整体 serialize）
2. 实际上 BookmarkIO::read() 整体 unserialize，如果 A 已成功写入 BookmarkArray，则 B 读到的是完整的 BookmarkArray，BookFileService 判定 `instanceof BookmarkArray` 为 true，不会走 migrate 路径，B 根本不会执行迁移。

因此在**迁移期间**（migrate 触发后 exit 强制用户刷新），正常业务路径无法写入数据，避免了迁移与业务写的竞态。

### 10.4 config.json.php 并发写竞态

`updateMethodConfigToJson()` 中：
```php
T3  rename(config.php → config.save.php)   // rename 是原子操作
T7  conf->write() → file_put_contents(config.json.php, ...)
```

- T3 rename 原子性：第一个进程 rename 成功，第二个进程 T3  rename 会失败（源文件不存在）产生 PHP Warning，但不会 fatal → 进程 B 继续执行 T4 reload() → 此时加载空 JSON 配置 → ❗ **写入一份只有默认值的 config.json.php，覆盖掉进程 A 可能正在写入或已写入的真实配置**

这是并发场景下最严重的竞态。修复思路：在 T3 rename 前加一段「目标 config.save.php 是否已存在」检查作为二次守卫，或使用 mkdir 原子性作为分布式锁。

### 10.5 并发竞态总表

| 资源 | 并发操作 | 保护机制 | 风险等级 | 后果 |
|------|----------|----------|----------|------|
| updates.txt | 多进程 read-modify-write | 无保护（最终写入内容一致） | ⚠️ 中 | 方法体被重复执行（依赖各自幂等守卫） |
| datastore.YYYYMMDDHHmmss.php | copy() 创建备份 | 秒级文件名可能冲突 | 低 | 备份被同内容覆盖（无数据损坏） |
| datastore 迁移写入 | BookmarkIO::write() 并发写 | FlockMutex 互斥锁 | ✅ 低 | 内容一致，覆盖安全 |
| config.php → save.php rename | 并发 rename | 无保护 | ❌ **高** | 第二个进程落到空配置路径，可能丢失配置写回 |
| config.json.php 写入 | file_put_contents 并发 | 无保护 | ❌ **高** | 后写的默认值配置覆盖先进程写入的真实配置 |
| 业务书签写入 vs 迁移 | migrate 后 exit 中断请求 | exit 强制中断 | ✅ 低 | 迁移期间业务操作被中断，刷新后重试 |

---

## 十一、格式识别链路：遗留数组 vs BookmarkArray 对象检测全程

### 11.1 三层存储格式演进

Shaarli 历史上有 **三种 datastore 文件格式**，全部使用相同的物理包装（`<?php /* base64(gzdeflate(serialize(...))) */ ?>`），区别仅在于 `serialize()` 的内部对象类型：

| 格式代号 | serialize 内部结构 | 主键类型 | 对应升级阶段 | 对应读取类 |
|----------|-------------------|----------|--------------|------------|
| **FORMAT_A** | `array[key=linkdate_string] = array[]` | 日期字符串 | 原始版本 | LegacyLinkDB |
| **FORMAT_B** | `array[key=int_id] = array[]` | 整数 ID | updateMethodDatastoreIds 之后 | LegacyLinkDB |
| **FORMAT_C** | `BookmarkArray 对象`（内部封装 `Bookmark[]`） | 整数 ID，对象封装 | updateMethodMigrateDatabase 之后 | BookmarkIO + BookmarkArray |

其中 **FORMAT_A / FORMAT_B 合称为「遗留数组格式」**，FORMAT_C 为「新对象格式」。

### 11.2 完整识别链路图

请求进入后，在 `index.php` 中构建容器，`BookmarkFileService` 作为容器服务被懒加载时触发识别：

```
HTTP Request
    │
    ▼
index.php → ContainerBuilder::build()
    │
    ▼
BookmarkFileService 被首次访问（如首页链接列表）
    │
    ▼
__construct()  [BookmarkFileService.php#L62-L105]
    │
    ├─ 实例化 BookmarkIO
    │
    ├─ bookmarksIO->read()  [BookmarkIO.php#L75-L104]
    │     │
    │     ├─ file_get_contents(datastore.php)
    │     ├─ 去除 PHP 包裹头/尾
    │     ├─ base64_decode → gzinflate → unserialize
    │     │
    │     └─ 返回值：
    │        · FORMAT_A → array (key=string)
    │        · FORMAT_B → array (key=int)
    │        · FORMAT_C → BookmarkArray 对象 ✓
    │
    ├─ catch (空/不存在异常) → 初始化空 BookmarkArray
    │
    ├─ 关键判定点：$this->bookmarks instanceof BookmarkArray
    │     │
    │     ├─ true  → FORMAT_C，正常继续，实例化 BookmarkFilter
    │     │
    │     └─ false → FORMAT_A 或 FORMAT_B，进入 migrate()
    │              │
    │              ├─ migrate()  [BookmarkFileService.php#L422-L442]
    │              │     │
    │              │     ├─ new LegacyLinkDB() 读取遗留数组
    │              │     │    (LegacyLinkDB::read 使用 FileUtils::readFlatDB，
    │              │     │     底层也是同样的 unserialize)
    │              │     │
    │              │     ├─ new LegacyUpdater(已完成更新列表)
    │              │     │    └─ update() 执行剩余迁移方法
    │              │     │
    │              │     └─ writeUpdatesFile() 写入记录
    │              │
    │              └─ exit() 提示用户刷新页面
    │
    └─ 实例化 pluginManager + bookmarkFilter
```

### 11.3 instanceof 判定的精确语义

位于 [BookmarkFileService.php#L94-L100](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkFileService.php#L94-L100)：

```php
if (! $this->bookmarks instanceof BookmarkArray) {
    $this->migrate();
    exit('Your data store has been migrated, please reload the page.');
}
```

这个判定条件的**真值表**：

| unserialize 返回值类型 | `instanceof BookmarkArray` | 是否走 migrate |
|------------------------|---------------------------|----------------|
| `array`（任意键类型） | **false** | ✅ 是 |
| `BookmarkArray` 对象 | **true** | ❌ 否 |
| `Bookmark` 对象 | false（但正常情况不会整体序列化单个 Bookmark） | ✅ 是（异常路径） |
| `false`（反序列化失败） | false | ✅ 是（但会在后续 LegacyLinkDB 中再次失败） |
| `null`（空文件） | false | 会被 EmptyDataStoreException 提前捕获，不会走到此处 |

### 11.4 LegacyLinkDB 内部 FORMAT_A vs FORMAT_B 的二次识别

LegacyLinkDB::read() ([LegacyLinkDB.php#L287-L332](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/legacy/LegacyLinkDB.php#L287-L332)) 内部有一段**向前兼容逻辑**，处理在 updateMethodDatastoreIds 之前（FORMAT_A）的数据：

```php
// LegacyLinkDB::read() 内部关键代码
foreach ($this->links as $key => &$link) {
    // ...
    if (!isset($link['created'])) {   // ← FORMAT_A 标记：无 created 字段
        $link['id'] = $link['linkdate'];           // 暂用 linkdate 当 id 过渡
        $link['created'] = DateTime::createFromFormat(
            self::LINK_DATE_FORMAT,
            $link['linkdate']
        );
        // ... updated 和 shorturl 同样补全
    }
    // ...
}
```

这段代码确保 **updateMethodDatastoreIds 不需要先执行**，后续的升级方法（如 updateMethodRenameDashTags）可以在未迁移 ID 的情况下工作，升级顺序不做强制要求。

### 11.5 识别链路的边界条件

| 场景 | 识别结果 | 后续行为 |
|------|----------|----------|
| 全新安装（datastore 不存在） | DatastoreNotInitializedException 捕获 → 初始化 BookmarkArray（含示例书签） | 正常流程 |
| 仅删除 bookmarks 条目（空 datastore，文件存在） | EmptyDataStoreException → 空 BookmarkArray | 正常流程 |
| datastore 部分损坏（unserialize 返回 false） | `!instanceof BookmarkArray` = true → 进入 migrate → LegacyLinkDB 再次反序列化同样失败 → PHP Warning | 需要从备份恢复 |
| FORMAT_B 中间态（已改 ID 为 int，但还是数组） | `!instanceof BookmarkArray` = true → 进入 migrate → updateMethodDatastoreIds 的守卫 `is_int(first)` 已通过 → 直接 return → 然后 updateMethodMigrateDatabase 执行对象转换 | ✅ 正确，只执行剩余迁移 |
| FORMAT_C 完成态 | `instanceof BookmarkArray` = true → 正常加载 | 走新 Updater（非 Legacy）的 updateMethod* |

---

## 十二、备份生命周期：累积清理策略与恢复操作手册

### 12.1 备份文件的产生源

每次执行涉及重写的升级方法，都会在 `data/` 目录（或 `resource.data_dir` 指定路径）产生备份文件：

| 备份文件名模式 | 产生方法 | 内容 | 保留策略 |
|----------------|----------|------|----------|
| `datastore.YYYYMMDDHHmmss.php` | updateMethodDatastoreIds | FORMAT_A（日期主键数组） | **永久**，不自动删除 |
| `datastore.YYYYMMDDHHmmss_1.php` | updateMethodMigrateDatabase | FORMAT_B（整数 ID 数组） | **永久**，不自动删除 |
| `config.save.php` | updateMethodConfigToJson | PHP 格式原始配置 | **永久**，不自动删除 |
| `data/updates.txt.bak` | 无 | 无（代码不自动创建备份） | 需用户手动备份 |

此外还有用户可手动导出的备份（Tools 页面导出 datastore / config），不在自动备份范畴。

### 12.2 备份累积规模估算

假设用户经历一次完整的「原始版本 → 最新版本」升级，会产生的备份文件集合：

```
data/
├── datastore.20240115143022.php       # ID 迁移备份（约等于 datastore 大小）
├── datastore.20240115143022_1.php     # 对象迁移备份（约等于 datastore 大小）
├── config.save.php                     # 配置备份（几 KB）
└── ... 如果升级被中断重试，可能多份时间戳不同的重复备份
```

大小估算：每个 datastore 备份 ≈ 实际 datastore.php 大小（gzip 压缩，一般几千条书签在 1-5MB 级别）。如果反复中断重试，每秒产生一对备份，极端情况下可能累积几十 MB。由于是纯文本+gzip，磁盘占用对现代服务器可忽略，但杂乱的文件列表会干扰排查。

### 12.3 识别可清理备份的准则

代码中**未提供自动清理机制**，需用户手动按以下准则判断：

| 准则 | 操作 |
|------|------|
| **确认升级成功后 7 天以上** | 可安全删除所有 `datastore.*.php`（保留最新的一对即可） |
| `data/updates.txt` 中已记录对应方法名 | 对应备份可删（方法已执行且不会再回退） |
| `config.json.php` 已存在且功能正常 | `config.save.php` 可归档后删除 |
| 时间戳相同、内容相同的多份重复备份 | 只保留一份 |

建议的 Linux 清理命令（需先人工确认）：
```bash
# 列出所有 datastore 备份按时间排序
ls -lt data/datastore.*.php

# 删除 30 天前的 datastore 备份（谨慎执行）
find data/ -name "datastore.*.php" -mtime +30 -delete
```

### 12.4 从备份完整恢复操作步骤（分场景）

#### 场景 A：updateMethodConfigToJson 中断，配置丢失

**症状**：升级后登录发现配置被重置为默认值、或页面报 MissingFieldConfigException。

**恢复步骤**：
```
步骤 1：进入 data/ 目录，确认文件状态
        ls -la data/config*
        预期看到：config.save.php（原始 PHP 配置）+ config.json.php（可能是默认值）

步骤 2：停止 Web 服务（或确保无人访问），防止并发写入
        sudo systemctl stop php-fpm nginx   # 或对应命令

步骤 3：回滚配置
        cd data/
        cp config.save.php config.php         # 恢复原始 PHP 格式
        rm -f config.json.php                 # 删除不完整的 JSON
        rm -f updates.txt                      # 重置升级进度，触发完整重跑

步骤 4：重新启动服务
        sudo systemctl start php-fpm nginx

步骤 5：浏览器登录 Shaarli，页面会自动重新执行 PHP→JSON 迁移
        完成后验证所有配置项是否正确：设置、插件、API 密钥等
```

#### 场景 B：updateMethodDatastoreIds 中断，数据格式混乱

**症状**：部分书签丢失、链接 ID 错乱、或页面报「Link not found」。

**恢复步骤**：
```
步骤 1：定位最近一份日期主键备份
        cd data/
        ls -lt datastore.*.php | head -5
        # 选择升级开始时刻附近的一份（不带 _1 后缀的）

步骤 2：确认备份内容格式
        php -r '
          $content = file_get_contents("datastore.YYYYMMDDHHmmss.php");
          $data = substr($content, 9, -6); // 去 PHP 包裹
          $arr = unserialize(gzinflate(base64_decode($data)));
          echo "第一条 key 类型: " . gettype(array_key_first($arr)) . "\n";
          echo "条目数: " . count($arr) . "\n";
        '
        # 期望输出: "第一条 key 类型: string"（证明是 FORMAT_A）

步骤 3：用备份覆盖
        cp datastore.YYYYMMDDHHmmss.php datastore.php
        rm -f updates.txt

步骤 4：刷新浏览器，重新登录触发升级
```

#### 场景 C：updateMethodMigrateDatabase 中断，混合格式

**症状**：反复显示 "Your data store has been migrated, please reload the page."

**恢复步骤**：
```
步骤 1：用 _1 后缀的备份恢复
        cd data/
        cp datastore.YYYYMMDDHHmmss_1.php datastore.php
        rm -f updates.txt

步骤 2：若 _1 备份也损坏，用更早的不带后缀的备份：
        cp datastore.YYYYMMDDHHmmss.php datastore.php
        rm -f updates.txt
        # 会先跑完 ID 迁移，再跑对象迁移

步骤 3：刷新登录，重新走完整迁移流程
```

#### 场景 D：updateMethodEscapeUnescapedConfig 重复执行，双重转义

**症状**：标题/页头显示 `&lt;` 而非 `<`，`&quot;` 而非 `"`。

**恢复步骤**：
```
步骤 1：如果 config.save.php 还在，从场景 A 方式整体回滚（最稳妥）

步骤 2：如果只有 JSON 配置，手动反向恢复被双重转义的字段：
        cd data/
        cp config.json.php config.json.php.bak
        php -r '
          // 读取 JSON 配置
          $content = file_get_contents("config.json.php");
          $data = substr($content, 9, -6);
          $conf = json_decode(trim($data), true);

          // 反向还原一次：htmlspecialchars_decode 一次
          $conf["general"]["title"] = htmlspecialchars_decode(
              $conf["general"]["title"], ENT_QUOTES
          );
          $conf["general"]["header_link"] = htmlspecialchars_decode(
              $conf["general"]["header_link"], ENT_QUOTES
          );
          // 若还有 redirector.url 也处理

          // 写回
          $out = "<?php /*" . json_encode($conf, JSON_PRETTY_PRINT) . "*/ ?>";
          file_put_contents("config.json.php", $out);
          echo "已反向恢复一次转义，请刷新验证\n";
        '
        # 如果还是异常，再跑一次 decode（可能被执行了 3+ 次）

步骤 3：删除 updates.txt 中 updateMethodEscapeUnescapedConfig 条目
        # 防止下次升级被判定为「已完成」而跳过守卫 → 不对，其实删了会重跑
        # 实际上正确做法是**保留**该条目，避免重跑
```

#### 场景 E：任意场景的通用兜底恢复

如果以上针对性恢复都失败，按「最后可用状态」思路：

```
1. 列出 data/ 目录所有时间戳备份：ls -lt data/datastore.*.php data/*.save.php
2. 选取升级操作发生**之前**时间戳最近的一份作为恢复源
3. 同时覆盖 datastore.php 和 config（注意格式匹配）
4. 删除 updates.txt
5. 完整重新升级
6. 对比升级前后的书签数量（后台 Tools 页可见）验证完整性
```

### 12.5 升级前的最佳实践建议

虽然代码没有自动执行这些步骤，但升级前手动做可大幅降低风险：

```bash
# 1. 全量快照备份（最重要）
cp -a data/ data_backup_$(date +%Y%m%d_%H%M%S)/

# 2. 记录当前状态
php -r '
  require "vendor/autoload.php";
  $conf = new Shaarli\Config\ConfigManager();
  echo "Config IO: " . get_class($conf->getConfigIO()) . "\n";
  echo "Bookmark count: TODO \n";
' > data/upgrade_pre_check.txt

# 3. 确保磁盘空间
df -h data/

# 4. 延长 PHP 超时（防止大规模迁移中途超时）
# 在 php.ini 临时设置：
# max_execution_time = 300
# memory_limit = 512M

# 5. 单 Tab 登录执行升级（避免并发竞态）
```

---

## 十三、锁降级无锁路径：LockAcquireException 降级后双进程并发改写的完整后果

### 13.1 锁降级机制源码解析

[BookmarkIO::synchronized()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L152-L159) 是所有文件 I/O 的互斥保护入口：

```php
protected function synchronized(callable $function): void
{
    try {
        $this->mutex->synchronized($function);
    } catch (LockAcquireException $exception) {
        $function();  // ← 降级：无锁直接执行
    }
}
```

**降级触发条件**（FlockMutex 抛出 LockAcquireException 的场景）：

FlockMutex 构造参数为 `new FlockMutex(fopen(SHAARLI_MUTEX_FILE, 'r'), 2)`（[ContainerBuilder.php#L100](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/container/ContainerBuilder.php#L100)），第二个参数 `2` 是**超时秒数**。LockAcquireException 在以下情况被抛出：

| 场景 | 触发原因 |
|------|----------|
| 共享主机 `/tmp` 挂载为 noexec | flock() 系统调用被禁用 |
| NFS 挂载目录 | flock() 在某些 NFS 配置下不可靠 |
| 文件描述符耗尽 | `fopen()` 返回 false → flock(null) 异常 |
| 超时 2 秒仍未获取锁 | 另一进程持锁超过 2 秒 |

降级后，`$function()` 被直接调用，**等同于完全没有并发保护**。

此外，**updates.txt 的读写完全不走 synchronized()**。[UpdaterUtils::writeUpdatesFile()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/updater/UpdaterUtils.php#L33-L43) 直接调用 `file_put_contents()`，没有任何锁保护——无论锁是否正常工作。

### 13.2 降级无锁后双进程并发写 updates.txt 的推演

[ShaarliMiddleware::runUpdates()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/ShaarliMiddleware.php#L67-L83) 的执行流程：

```
进程 A                                   进程 B
─────────────────────────────────────    ─────────────────────────────────────
readUpdatesFile() → [M1, M2]
                                         readUpdatesFile() → [M1, M2]
update() 执行 M3 → 成功
                                         update() 执行 M3 → 成功
writeUpdatesFile([M1, M2, M3])
                                         writeUpdatesFile([M1, M2, M3])
```

**竞态窗口 1：writeUpdatesFile 的 file_put_contents 交叉写入**

`file_put_contents()` 在默认模式下**不是原子操作**。它等价于 `fopen → fwrite → fclose`。如果两个进程的 fwrite 交叉执行：

```
进程 A fwrite("M1;M2;M3")     →  磁盘可能的内容：
进程 B fwrite("M1;M2;M3")         "M1;M2;M3M1;M2;M3"  ← 交叉拼接
                                   或
                                   "M1;M2;M3"          ← 完整覆盖（如果 B 在 A fclose 后才 fopen）
```

**后果分析**：

| 交叉结果 | 下次 readUpdatesFile 解析 | 后续 update() 行为 |
|----------|--------------------------|-------------------|
| `"M1;M2;M3"` | `['M1', 'M2', 'M3']` | 正常，M3 不再执行 |
| `"M1;M2;M3M1;M2;M3"` | `['M1', 'M2', 'M3M1', 'M2', 'M3']` | `M3M1` 不匹配任何方法名，跳过；`M2`、`M3` 在 doneUpdates 中，M2 被跳过，但**尾部多出的方法名会被忽略**（因为它们是无效方法名，`startsWith('updateMethod')` 检查不通过） |
| `"M1;M2;M3;M1;M2;M3"` | `['M1', 'M2', 'M3', 'M1', 'M2', 'M3']` | 有重复条目但 in_array 仍能正确匹配，M3 不再执行 |
| `""`（极端：两进程 fopen 时刻重叠，均 truncate 后只写了部分） | `[]` | **所有方法重新执行**——回到全量重试，靠各方法幂等守卫保护 |

结论：updates.txt 竞态的**最坏后果**是进度记录丢失（文件被截断为空），导致下次请求全量重试。不会导致方法被跳过（因为进度只会丢失不会凭空增加）。

### 13.3 降级无锁后双进程并发写 datastore 的推演

[BookmarkIO::write()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L114-L142) 的关键操作：

```php
$data = base64_encode(gzdeflate(serialize($links)));  // ← 构造完整字符串
$data = self::$phpPrefix . $data . self::$phpSuffix;
$this->synchronized(function () use ($data) {
    file_put_contents($this->datastore, $data);        // ← 降级后无锁
});
```

**降级场景下双进程写推演**：

```
进程 A                                  进程 B
──────────────────────────────────      ──────────────────────────────────
serialize(BookmarkArray_1) → $data_A
                                        serialize(BookmarkArray_2) → $data_B
file_put_contents(datastore, $data_A)
                                        file_put_contents(datastore, $data_B)
```

PHP 的 `file_put_contents()` 默认使用 `w` 模式（truncate + write），其底层实现：

1. `open(path, O_WRONLY|O_CREAT|O_TRUNC)` → 截断文件
2. `write(fd, data)` → 写入新内容
3. `close(fd)` → 关闭

**交叉时序推演**：

```
时序  进程 A                          进程 B                          datastore 文件内容
────  ──────────────────────────────  ──────────────────────────────  ─────────────────────
T1    open(O_TRUNC) → 截断为空                                        (空)
T2    write($data_A) 开始写入                                         $data_A 前半段
T3                                     open(O_TRUNC) → 截断为空       (空) ← A 的写入被丢弃
T4                                     write($data_B) 完成            $data_B 完整
T5    write($data_A) 继续（fd 仍有效）                                 $data_B + $data_A 残余？
```

实际上 T3 的 `open(O_TRUNC)` 会独立操作文件 inode，T2 和 T4 的 write 操作的文件偏移量各自独立。最终文件内容取决于**谁最后 close**：

| 最后 close 的进程 | 文件内容 | 可读性 |
|-------------------|----------|--------|
| 进程 A | $data_A 完整（进程 A 的 fd 偏移量独立，不受 B 的 truncate 影响） | ✅ 可正常 unserialize |
| 进程 B | $data_B 完整 | ✅ 可正常 unserialize |
| 交叉写（极低概率，依赖内核调度） | $data_A 前段 + $data_B 后段（混合内容） | ❌ unserialize 失败 |

**混合内容场景的具体后果**：

1. 下次请求 `BookmarkIO::read()` 执行 `gzinflate(base64_decode(...))` 时：
   - 如果 base64_decode 得到非法二进制 → `gzinflate()` 返回 false 或抛出 Warning
   - `unserialize(false)` → 返回 false
   - `empty(false)` 为 true → 根据 `filesize > 100` 判断：
     - 文件大（混合后一般仍大于 100 字节）→ 抛出 `NotWritableDataStoreException`
     - 文件小 → 抛出 `EmptyDataStoreException`

2. **进入 BookmarkFileService 构造函数的 catch 分支**：
   - `EmptyDataStoreException` → 创建空 BookmarkArray → `$this->save()` → **用空数据覆盖混合文件** → ❗ **数据全部丢失**
   - `NotWritableDataStoreException` → 不被 catch → 异常冒泡到中间件 → 500 错误页面 → 用户看到错误提示

**关键结论**：锁降级下 datastore 并发写的最坏后果不是"数据混乱"，而是**空数据覆盖导致数据全丢**。因为 `EmptyDataStoreException` 分支会自动用空的 BookmarkArray 执行 save，把已损坏但可恢复的混合文件彻底覆盖为空数据。

### 13.4 降级无锁下升级期间业务写入的竞态

除了双进程同时升级外，另一个危险场景是**一个进程在升级写 datastore，另一个进程在正常业务写 datastore**。

在正常流程中，升级发生在 ShaarliMiddleware（请求早期），业务写入发生在 Controller 中（请求晚期）。但 HTTP 请求是独立的 PHP 进程，两个不同请求的时间线可以完全重叠：

```
请求 A（升级）：Middleware → update() → BookmarkIO::write(迁移后数据)
请求 B（业务）：Middleware(已登录，升级已完成) → Controller → addBookmark → BookmarkIO::write(含新书签的数据)
```

如果锁正常，B 的 write 会等 A 的 write 完成。但降级后：

```
T0  A: 读取旧格式 datastore（遗留数组）
T1  B: 读取新格式 datastore（BookmarkArray）— 假设部分已迁移
T2  A: write(完整迁移后的 BookmarkArray)     ← 不含 B 新增的书签
T3  B: write(含新增书签的 BookmarkArray)      ← 基于 T1 的快照
```

结果：B 的写入覆盖 A 的写入。因为 B 的快照更新（包含新增书签），**但 B 可能不包含 A 在迁移中修正的数据**（如 URL 格式修正、sticky 字段补全等）。实际影响取决于迁移方法的具体操作。

### 13.5 锁降级竞态风险总表

| 竞态对象 | 降级后保护 | 最坏后果 | 数据恢复可能性 |
|----------|------------|----------|----------------|
| updates.txt | 无保护 | 进度丢失，全量重试（幂等方法安全） | ✅ 可自动恢复 |
| datastore (双升级进程) | 无保护 | 混合写入 → unserialize 失败 → EmptyDataStoreException → **空数据覆盖** | ❌ **不可恢复**（除非有时间戳备份） |
| datastore (升级 vs 业务) | 无保护 | 迁移修正数据丢失，仅保留业务快照 | ⚠️ 部分可恢复（丢失的是迁移修正） |
| config.json.php | 无保护 | 进程 B 用默认值覆盖 A 的真实配置 | ❌ 需从 config.save.php 手动恢复 |

---

## 十四、datastore 缺失或为空时的边界判定全路径

### 14.1 BookmarkFileService 构造函数中的四层分支

位于 [BookmarkFileService.php#L62-L105](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkFileService.php#L62-L105)：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     __construct() 入口                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  条件 1: !isLoggedIn && hide_public_links                          │
│    → $this->bookmarks = new BookmarkArray()                        │
│    → 跳过一切读取和迁移                                             │
│                                                                     │
│  条件 2: 正常访问（登录/公开可见）                                   │
│    → bookmarksIO->read()                                           │
│      │                                                              │
│      ├─ 正常: 返回 BookmarkArray                                   │
│      │   → instanceof 检查通过 → 正常流程                          │
│      │                                                              │
│      ├─ 正常: 返回 array (遗留格式)                                │
│      │   → instanceof 检查失败 → migrate() + exit()                │
│      │                                                              │
│      ├─ DatastoreNotInitializedException                           │
│      │   → $this->bookmarks = new BookmarkArray()                  │
│      │   → isLoggedIn? → initialize() → 写入示例书签               │
│      │   → !isLoggedIn? → 仅内存空数组，不写入磁盘                  │
│      │                                                              │
│      ├─ EmptyDataStoreException                                    │
│      │   → $this->bookmarks = new BookmarkArray()                  │
│      │   → isLoggedIn? → save() → 写入空 BookmarkArray 到磁盘     │
│      │   → !isLoggedIn? → 仅内存空数组                             │
│      │                                                              │
│      └─ NotWritableDataStoreException                              │
│          → 不被 catch → 异常冒泡 → 500 错误页                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 14.2 BookmarkIO::read() 异常触发条件精确定义

[BookmarkIO::read()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L75-L104) 的异常判定逻辑：

```php
public function read()
{
    if (! file_exists($this->datastore)) {       // ← 判定 1
        throw new DatastoreNotInitializedException();
    }

    if (!is_writable($this->datastore)) {        // ← 判定 2
        throw new NotWritableDataStoreException($this->datastore);
    }

    $content = null;
    $this->synchronized(function () use (&$content) {
        $content = file_get_contents($this->datastore);
    });

    $links = unserialize(gzinflate(base64_decode(
        substr($content, strlen(self::$phpPrefix), -strlen(self::$phpSuffix))
    )));

    if (empty($links)) {                          // ← 判定 3
        if (filesize($this->datastore) > 100) {   // ← 判定 4
            throw new NotWritableDataStoreException($this->datastore);
        }
        throw new EmptyDataStoreException();
    }

    return $links;
}
```

**异常判定决策树**：

| 磁盘状态 | file_exists | is_writable | unserialize 结果 | empty? | filesize | 抛出异常 |
|----------|-------------|-------------|------------------|--------|----------|----------|
| 文件不存在 | false | — | — | — | — | **DatastoreNotInitializedException** |
| 文件存在但不可写 | true | false | — | — | — | **NotWritableDataStoreException** |
| 文件存在，内容为空字符串 | true | true | `unserialize(false)` → false | true | 0 | **EmptyDataStoreException** |
| 文件存在，仅有 PHP 包裹无内容 | true | true | `unserialize(gzinflate(base64_decode('')))` → false/Warning | true | <100 | **EmptyDataStoreException** |
| 文件存在，反序列化得到空数组 | true | true | `[]` | true | <100 | **EmptyDataStoreException** |
| 文件存在，反序列化得到空 BookmarkArray | true | true | `BookmarkArray(count=0)` | **true**（empty 对无属性对象返回 true） | <100 | **EmptyDataStoreException** |
| 文件存在，反序列化损坏但文件 > 100 字节 | true | true | false 或 PHP Warning | true | >100 | **NotWritableDataStoreException** |
| 文件存在，正常 BookmarkArray | true | true | BookmarkArray(count>0) | false | — | **正常返回** |
| 文件存在，正常遗留数组 | true | true | array(count>0) | false | — | **正常返回** |

### 14.3 关键边界场景分析

#### 边界 1：空 BookmarkArray 误判为 EmptyDataStoreException

PHP 的 `empty()` 函数对**任何没有属性的对象**返回 true。`BookmarkArray` 对象在 `count=0` 时，`empty($bookmarkArray)` 返回 true。这意味着：

- 如果用户手动清空了所有书签（通过 Web 界面删除最后一个），save() 写入了一个包含空 BookmarkArray 的 datastore
- 下次 read() 时，`unserialize()` 返回空 BookmarkArray，`empty()` 判定为 true
- 抛出 `EmptyDataStoreException`
- BookmarkFileService 创建**新的空 BookmarkArray** → `save()` 写入

**后果**：功能上无损失（空数据还是空数据），但**空 BookmarkArray 对象被重新创建一次**。如果用户的空 BookmarkArray 有某些特殊属性（理论上不会，因为所有属性在构造函数中初始化），可能丢失。实际影响为零。

#### 边界 2：datastore 不存在 → DatastoreNotInitializedException → initialize()

`DatastoreNotInitializedException` 触发后，BookmarkFileService 调用 `$this->initialize()`，由 [BookmarkInitializer](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkInitializer.php#L39-L114) 创建 3 条示例书签（2 条私有 + 1 条公开），然后 `save()` 写入新格式的 BookmarkArray。

**与 LegacyLinkDB::check() 的关键区别**：

| 对比项 | BookmarkFileService (新) | LegacyLinkDB::check() (旧) |
|--------|--------------------------|----------------------------|
| 触发条件 | datastore 文件不存在 | datastore 文件不存在 |
| 初始内容 | 3 条 Bookmark 对象 | 2 条关联数组 |
| 存储格式 | BookmarkArray (FORMAT_C) | 遗留数组 (FORMAT_B，含 id/created) |
| 写入方式 | BookmarkIO::write()（FlockMutex） | FileUtils::writeFlatDB()（无锁） |
| 读写锁 | 有 synchronized | 无 |

这意味着：如果用户首次安装后直接走新版路径，datastore 一开始就是 FORMAT_C（BookmarkArray），永远不会触发遗留迁移。只有从旧版本升级的用户才会遇到 `instanceof BookmarkArray` 为 false 的情况。

#### 边界 3：datastore 存在但内容被截断 → NotWritableDataStoreException 冒泡

如果 datastore 文件存在但内容被截断（如磁盘满导致写了一半），`unserialize()` 返回 false，且 `filesize > 100`（因为文件包含部分 base64 内容），会抛出 `NotWritableDataStoreException`。

此异常**不被 BookmarkFileService 的 catch 捕获**（catch 仅处理 `EmptyDataStoreException | DatastoreNotInitializedException`），会冒泡到 ShaarliMiddleware → 最终显示 500 错误页。

**这是正确的安全行为**：截断文件不应该被静默覆盖为空数据，而应该让用户知晓并手动从备份恢复。

#### 边界 4：datastore 文件存在但权限不足

`is_writable()` 返回 false → 直接抛出 `NotWritableDataStoreException`，不尝试读取。这是**防御性设计**：如果无法写入，读取后也无法 save()，不如尽早失败。

### 14.4 「缺失/为空」边界是否误触发遗留迁移的判定表

| 磁盘状态 | BookmarkIO::read() 返回 | instanceof BookmarkArray | 走遗留迁移? | 走空初始化? | 实际路径 |
|----------|------------------------|------------------------|------------|------------|----------|
| 文件不存在 | DatastoreNotInitializedException | — | ❌ | ✅ initialize() | 新建 3 条示例书签 |
| 文件为空（0 字节） | EmptyDataStoreException | — | ❌ | ✅ save() 空数组 | 空数据 |
| 文件仅含 PHP 包裹 | EmptyDataStoreException | — | ❌ | ✅ save() 空数组 | 空数据 |
| 反序列化得到空 array | EmptyDataStoreException | — | ❌ | ✅ save() 空数组 | 空数据 |
| 反序列化得到空 BookmarkArray | EmptyDataStoreException | — | ❌ | ✅ save() 空数组 | 空数据 |
| 反序列化得到非空 array | 原始 array 对象 | **false** | ✅ migrate() | ❌ | 遗留迁移 |
| 反序列化得到非空 BookmarkArray | BookmarkArray 对象 | **true** | ❌ | ❌ | 正常流程 |
| 反序列化失败且文件 > 100B | NotWritableDataStoreException | — | ❌ | ❌ | 500 错误页 |
| 文件不可写 | NotWritableDataStoreException | — | ❌ | ❌ | 500 错误页 |

**核心结论**：只要 datastore 文件存在且内容可解析，`BookmarkIO::read()` 会正常返回反序列化结果。此时**唯一的迁移判定依据是 `instanceof BookmarkArray`**。「缺失」和「为空」两种边界都走**空数据初始化**路径，绝不走遗留迁移——这是正确的，因为遗留迁移的前提是有旧数据需要迁移，空数据无需迁移。

---

## 十五、从破损或截断的时间戳备份回滚引发的二次损坏

### 15.1 备份文件的物理结构

所有 datastore 文件（包括备份）使用相同的物理包装：

```
<?php /* <base64(gzdeflate(serialize(data)))> */ ?>
```

由 [FileUtils::writeFlatDB()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/helper/FileUtils.php#L37-L51) 和 [BookmarkIO::write()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L114-L142) 写入。

### 15.2 破损类型与回滚后果

#### 破损类型 A：文件末尾被截断（最常见）

磁盘满时 `file_put_contents()` 可能写了一部分就被中断。文件结构变为：

```
<?php /* S7QysKquBQA=...（截断，缺少尾部 */ ?>）
```

回滚操作：`cp datastore.YYYYMMDDHHmmss.php datastore.php`

**二次损坏链**：

```
1. cp 覆盖 datastore.php 为截断的备份文件
2. BookmarkIO::read() 执行：
   $content = file_get_contents(datastore.php)
   substr($content, strlen('<?php /* '), -strlen(' */ ?>'))
   ── 当文件末尾缺少 ' */ ?>' 时：
      - strlen(' */ ?>') = 6
      - substr 从末尾删 6 字符，但实际内容不足 → 返回更短的字符串或空字符串
   base64_decode(更短字符串) → 可能得到部分二进制
   gzinflate(部分二进制) → false 或 PHP Warning
   unserialize(false) → false
3. empty(false) === true → filesize(datastore.php) 取决于截断位置：
   - 大于 100 字节 → NotWritableDataStoreException → 500 错误页
   - 小于 100 字节 → EmptyDataStoreException → save() 空数据 → ❗ 原始数据被空数据覆盖
```

**结论**：如果截断的备份文件小于 100 字节，回滚后**不仅无法恢复数据，还会触发空数据初始化流程，把唯一可能通过手动修复的截断文件也覆盖掉**。

#### 破损类型 B：base64 内容中间截断

文件有完整的 PHP 包裹头尾，但中间的 base64 字符串被截断：

```
<?php /* S7QysKquBQA=...（中间截断）... */ ?>
```

回滚后的后果链：

```
1. substr 正常剥离 PHP 包裹头尾
2. base64_decode(截断的 base64) → 得到不完整二进制（PHP 会忽略无效尾部）
3. gzinflate(不完整二进制) → false 或 PHP Warning
4. unserialize(false) → false
5. empty(false) === true
6. filesize > 100 → NotWritableDataStoreException → 500 错误页
```

这种情况下系统**不会自动覆盖数据**，因为 `filesize > 100` 触发的是 `NotWritableDataStoreException`，冒泡为 500 错误。用户仍可手动修复。

#### 破损类型 C：文件内容全部为零/空格

某些文件系统错误可能导致文件被填零：

```
<?php /* AAAAAAAAAAAAAA... */ ?>  （base64 的全零编码）
```

后果链：

```
1. substr 剥离 PHP 包裹
2. base64_decode → 二进制全零
3. gzinflate(全零) → false
4. unserialize(false) → false
5. empty(true) + filesize 取决于填充量 → 通常是 NotWritableDataStoreException
```

不会触发空数据覆盖，但数据已不可恢复。

#### 破损类型 D：序列化对象版本不兼容

备份文件完整可读，但 `serialize()` 的对象定义与当前代码不兼容（如类名变更、属性删除）。这种情况在跨主版本备份中更常见，详见第十六章。

### 15.3 回滚前的安全校验步骤

在执行 `cp backup datastore.php` 之前，应先验证备份文件的完整性：

```bash
# 步骤 1：检查文件大小
ls -la data/datastore.YYYYMMDDHHmmss.php
# 如果文件 < 50 字节，几乎可以确定是空的或损坏的

# 步骤 2：验证 PHP 包裹完整性
head -c 9 data/datastore.YYYYMMDDHHmmss.php | xxd
# 期望输出: 3c3f706870202f2a20  (<?php /* )

tail -c 7 data/datastore.YYYYMMDDHHmmss.php | xxd
# 期望输出: 202a2f203f3e0a     ( */ ?>\n)

# 步骤 3：验证反序列化
php -r '
  $file = "data/datastore.YYYYMMDDHHmmss.php";
  $content = file_get_contents($file);
  if (strlen($content) < 20) { echo "ERROR: file too short\n"; exit(1); }
  $prefix = "<?php /* ";
  $suffix = " */ ?>";
  $payload = substr($content, strlen($prefix), -strlen($suffix));
  $binary = @base64_decode($payload, true);
  if ($binary === false) { echo "ERROR: base64 decode failed\n"; exit(1); }
  $inflated = @gzinflate($binary);
  if ($inflated === false) { echo "ERROR: gzinflate failed\n"; exit(1); }
  $data = @unserialize($inflated);
  if ($data === false) { echo "ERROR: unserialize failed\n"; exit(1); }
  $type = gettype($data);
  if ($type === "object") $type = get_class($data);
  echo "OK: type=$type, count=" . (is_countable($data) ? count($data) : "N/A") . "\n";
'
# 期望输出: OK: type=Shaarli\Bookmark\BookmarkArray, count=XXX
#    或:    OK: type=array, count=XXX

# 步骤 4：仅在步骤 3 通过后才执行回滚
cp data/datastore.YYYYMMDDHHmmss.php data/datastore.php
```

### 15.4 破损备份回滚二次损坏风险总表

| 破损类型 | 自动后果 | 是否触发空数据覆盖 | 手动可恢复性 |
|----------|----------|-------------------|-------------|
| 尾部截断（文件 < 100B） | EmptyDataStoreException → save() 空数据 | ❗ **是** | ❌ **不可恢复** |
| 尾部截断（文件 > 100B） | NotWritableDataStoreException → 500 | 否 | ⚠️ 可能可部分修复（base64 截断前的数据） |
| 中间截断（有完整包裹） | NotWritableDataStoreException → 500 | 否 | ❌ 不可恢复（gzinflate 要求完整输入） |
| 全零/乱码填充 | NotWritableDataStoreException → 500 | 否 | ❌ 不可恢复 |
| 完整但版本不兼容 | unserialize 异常或对象不完整 | 否（大概率 > 100B → NotWritable） | ⚠️ 取决于版本差异 |

---

## 十六、跨主版本备份兼容性判断

### 16.1 数据格式的版本演进与兼容性

Shaarli 的数据文件通过 `serialize()` 持久化，PHP 的序列化格式天然绑定**类名和属性结构**。跨主版本恢复备份时，核心风险在于序列化对象定义的变更。

| 版本区间 | 数据格式 | 类名/结构变更 | 恢复兼容性 |
|----------|----------|--------------|------------|
| v0.5 → v0.8 | FORMAT_A (日期主键数组) | 无类名，纯关联数组 | ✅ 完全兼容（LegacyLinkDB 可读） |
| v0.8 → v0.9 | FORMAT_B (整数主键数组) | 无类名，纯关联数组 | ✅ 完全兼容（LegacyLinkDB 可读） |
| v0.9 → v0.12 | FORMAT_B → FORMAT_C | `array` → `BookmarkArray` + `Bookmark` | ⚠️ 需经过迁移链 |
| v0.12 → v0.13+ | FORMAT_C (BookmarkArray) | Bookmark 属性可能增减 | ⚠️ 取决于具体变更 |

### 16.2 PHP 序列化的前向/后向兼容规则

PHP `unserialize()` 对对象的处理：

| 场景 | unserialize 行为 | 后果 |
|------|-----------------|------|
| 类存在，属性增加 | 新属性取默认值 | ✅ 安全 |
| 类存在，属性删除 | `__unserialize` 忽略多余属性（如果有自定义逻辑）；否则触发 `Undefined property` Notice | ⚠️ 可能丢失数据 |
| 类不存在 | 创建 `__PHP_Incomplete_Class` 对象 | ❌ 不可用，后续 `instanceof` 检查全失败 |
| 类重命名 | 等同于类不存在 | ❌ 同上 |
| 属性重命名 | 旧属性名值丢失，新属性取默认值 | ⚠️ 数据丢失 |

### 16.3 Bookmark 类的属性变更历史

Bookmark 对象在不同版本间属性逐步增加：

| 属性 | 引入版本 | 默认值 | 恢复旧备份时行为 |
|------|----------|--------|-----------------|
| `id` | v0.9 | null | 始终存在 |
| `shortUrl` | v0.9 | '' | 始终存在 |
| `url` | v0.9 | '' | 始终存在 |
| `title` | v0.9 | '' | 始终存在 |
| `description` | v0.9 | '' | 始终存在 |
| `tags` | v0.9 | [] | 始终存在 |
| `thumbnail` | v0.9 | '' | 始终存在 |
| `sticky` | v0.12 | false | 旧备份无此属性 → unserialize 后 `Undefined property` Notice → **不会导致 fatal 错误** |
| `created` | v0.9 | null | 始终存在 |
| `updated` | v0.9 | null | 始终存在 |
| `private` | v0.9 | false | 始终存在 |
| `additionalContent` | 后期 | [] | 旧备份无此属性 → Notice |

**结论**：由于属性**只增不删**，用旧版本的 BookmarkArray 备份恢复到新版本时，缺失的属性会被 PHP 自动忽略（触发 Notice 但不致命），新代码访问这些属性时使用默认值。功能上不会崩溃，但**缺失的属性值会回退为默认值**（如 sticky 全变 false、thumbnail 全变空）。

### 16.4 跨版本恢复的完整兼容性矩阵

| 备份格式 → 恢复目标 | FORMAT_A 备份 | FORMAT_B 备份 | FORMAT_C (旧) 备份 | FORMAT_C (新) 备份 |
|---------------------|--------------|--------------|-------------------|-------------------|
| **FORMAT_A 环境** | ✅ 直接可用 | ❌ ID 不兼容 | ❌ 类不存在 | ❌ 类不存在 |
| **FORMAT_B 环境** | ✅ LegacyLinkDB 兼容 | ✅ 直接可用 | ❌ 类不存在 | ❌ 类不存在 |
| **FORMAT_C (旧) 环境** | ⚠️ 需迁移 | ⚠️ 需迁移 | ✅ 直接可用 | ⚠️ 新属性缺失 |
| **FORMAT_C (新) 环境** | ⚠️ 需迁移 | ⚠️ 需迁移 | ✅ 属性默认值填充 | ✅ 直接可用 |

### 16.5 跨版本恢复操作步骤

#### 场景 A：FORMAT_A/B 备份恢复到 FORMAT_C 环境

```
步骤 1：验证备份完整性（见 15.3 的校验步骤）

步骤 2：确认备份格式类型
        php -r '
          // ... 读取并反序列化 ...
          $type = gettype($data);
          if ($type === "object") $type = get_class($data);
          echo $type . "\n";
        '
        # 输出 "array" → FORMAT_A 或 FORMAT_B

步骤 3：用备份覆盖 datastore.php
        cp data/datastore.YYYYMMDDHHmmss.php data/datastore.php

步骤 4：删除 updates.txt 中相关的迁移记录
        # 删掉 updateMethodDatastoreIds 和 updateMethodMigrateDatabase
        # 或者直接删除整个 updates.txt 触发完整重跑
        rm -f data/updates.txt

步骤 5：浏览器登录 Shaarli
        - BookmarkFileService 检测到 array（非 BookmarkArray）→ 触发 migrate()
        - LegacyUpdater 自动执行所有未记录的迁移方法
        - 包括 DatastoreIds + MigrateDatabase → 自动完成格式转换

步骤 6：验证书签数量和内容完整性
```

#### 场景 B：FORMAT_C 旧版备份恢复到新版 FORMAT_C 环境

```
步骤 1：验证备份完整性

步骤 2：直接覆盖
        cp data/datastore.YYYYMMDDHHmmss_1.php data/datastore.php

步骤 3：无需删除 updates.txt
        - BookmarkFileService 检测到 BookmarkArray（旧版）→ instanceof 通过
        - 但某些新属性缺失 → 代码中访问时触发 Notice
        - 不影响核心功能，只是缺失属性使用默认值

步骤 4（可选）：重新运行属性补全升级方法
        # 删除 updates.txt 中对应方法的记录，触发重跑
        # 例如删掉 updateMethodSetSticky 让它重新为所有书签添加 sticky=false
```

#### 场景 C：FORMAT_C 备份恢复到 FORMAT_B 环境（版本降级）

**不推荐，且无法自动完成**。需要：

```
1. 在新版环境中导出为 Netscape Bookmark HTML 格式（Tools → Export）
2. 在旧版环境中导入
3. 或手动编写转换脚本将 BookmarkArray → 关联数组
```

### 16.6 备份兼容性的设计缺陷与改进建议

| 当前设计 | 问题 | 改进建议 |
|----------|------|----------|
| 使用 PHP serialize() | 绑定类定义，跨版本不兼容 | 改用 JSON 或版本化的序列化格式 |
| 无版本标记 | 无法判断备份来自哪个版本 | 在 datastore 头部写入版本号 |
| 属性只增不删但无迁移映射 | 旧备份恢复后缺失属性无补偿 | 在 Bookmark::__unserialize() 中补全默认值 |
| 备份文件名不含格式标记 | 无法从文件名判断 FORMAT_A/B/C | 命名中加入格式标识，如 `datastore.v2.YYYYMMDDHHmmss.php` |

---

## 十七、锁文件不可用时的完整回落路径：SHAARLI_MUTEX_FILE 异常全链路

### 17.1 SHAARLI_MUTEX_FILE 的定义与作用

[SHAARLI_MUTEX_FILE](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/init.php#L63) 定义在 `init.php` 第 63 行：

```php
define('SHAARLI_MUTEX_FILE', __FILE__);
```

它就是 `init.php` 文件自身。所有需要互斥保护的地方都用这个文件作为 flock 的目标：

- [ContainerBuilder.php#L100](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/container/ContainerBuilder.php#L100) - Web 端 bookmarkService
- [ApiMiddleware.php#L150](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/api/ApiMiddleware.php#L150) - API 端 LinkDb
- [ApplicationUtils.php#L251](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/helper/ApplicationUtils.php#L251) - 健康检查

使用 `init.php` 自身作为锁文件是巧妙的设计：
- 保证文件一定存在（代码已经在运行了）
- 不需要额外创建锁文件
- 所有进程共享同一个锁目标

### 17.2 fopen(SHAARLI_MUTEX_FILE, 'r') 返回 false 的触发场景

理论上 `init.php` 一定存在且可读（因为 PHP 正在执行它），但以下极端场景会导致 `fopen` 失败：

| 场景 | 原因 | fopen 返回值 |
|------|------|-------------|
| `chmod 000 init.php` | 文件权限被意外修改为不可读 | `false` + PHP Warning |
| `open_basedir` 限制 | PHP 配置禁止访问该路径 | `false` + Warning |
| 文件句柄耗尽 | 系统文件描述符达到上限 | `false` + Warning |
| init.php 被删除（运行时） | 极端人为操作 | `false` + Warning |
| 磁盘 IO 错误 | 硬件故障 | `false` + Warning |

### 17.3 异常传播完整链路

当 `fopen(SHAARLI_MUTEX_FILE, 'r')` 返回 `false` 时，异常传播路径为：

```
ContainerBuilder 构造 bookmarkService
    │
    ├─ new FlockMutex(fopen(SHAARLI_MUTEX_FILE, 'r'), 2)
    │      │
    │      └─ fopen 返回 false → FlockMutex 内部保存了 false 作为 file handle
    │         （构造函数不抛异常，只是保存参数）
    │
    └─ BookmarkFileService 构造 → 正常完成（此时还没用到锁）
           │
           ▼
首次调用 BookmarkIO::write() / read()
    │
    ├─ $this->synchronized(function () { ... })
    │     │
    │     └─ $this->mutex->synchronized($function)
    │           │
    │           ├─ flock(false, LOCK_EX)  ← 传入 false 作为 file handle
    │           │     └─ PHP Warning: flock(): supplied resource is not a valid stream resource
    │           │     └─ 返回 false
    │           │
    │           └─ malkusch/lock 检测到 flock 失败 → 抛出 LockAcquireException
    │
    └─ BookmarkIO::synchronized() 捕获 LockAcquireException
          │
          └─ 直接执行 $function() → 降级为无锁执行
```

**关键要点**：
- FlockMutex 的**构造函数不会抛异常**，它只是保存传入的 file handle
- 异常在**第一次实际使用锁**时（调用 `synchronized()`）才抛出
- BookmarkIO 的 read() 和 write() 都经过 synchronized()，所以两者都会降级
- **BookmarkIO 构造函数的 NoMutex 兜底**只在 `$mutex === null` 时生效，fopen 返回 false 不属于 null，所以走的是 synchronized() 内部的 LockAcquireException 捕获路径，而不是 NoMutex 路径

### 17.4 锁完全失效后的系统行为全景

当锁完全不可用时（持续抛 LockAcquireException），整个系统的并发保护降级为：

| 组件 | 原保护 | 降级后 | 后果 |
|------|--------|--------|------|
| datastore 读取 | synchronized 内 file_get_contents | 直接 file_get_contents | 读操作本身无害，可能读到正在写入的半截数据 |
| datastore 写入 | synchronized 内 file_put_contents | 直接 file_put_contents | ❗ 并发写可能产生混合内容 → 数据损坏 |
| updates.txt 读写 | 无保护（本来就没锁） | 无保护 | 同第十章分析，进度可能丢失 |
| config 写入 | 无保护 | 无保护 | 并发写可能配置损坏 |
| history 写入 | 无保护 | 无保护 | 历史记录可能混乱 |

### 17.5 ApplicationUtils::checkDatastoreMutex() 健康检查

[ApplicationUtils::checkDatastoreMutex()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/helper/ApplicationUtils.php#L249-L261) 提供了锁健康检查：

```php
public static function checkDatastoreMutex(): array
{
    $mutex = new FlockMutex(fopen(SHAARLI_MUTEX_FILE, 'r'), 2);
    try {
        $mutex->synchronized(function () {
            return true;
        });
    } catch (LockAcquireException $e) {
        $errors[] = t('Lock can not be acquired on the datastore. You might encounter concurrent access issues.');
    }
    return $errors ?? [];
}
```

这个检查应该在安装/升级后通过 Tools 页面运行，但**日常访问不会自动触发**。锁失效时用户不会收到任何警告，只会在并发场景下遇到数据损坏。

### 17.6 锁降级的设计权衡

| 设计选择 | 优点 | 缺点 |
|----------|------|------|
| 捕获 LockAcquireException 后无锁执行 | 兼容共享主机（很多共享主机不支持 flock），保证基本可用性 | 并发场景下数据可能损坏 |
| 使用 init.php 作为锁文件 | 无需额外文件，一定存在 | 文件本身被误操作改权限时锁失效 |
| 不自动检测锁失效 | 实现简单 | 用户无感知，出问题难排查 |

---

## 十八、半截 datastore 文件的 read 判定链路全解析

### 18.1 产生半截文件的典型场景

`file_put_contents()` 使用 `w` 模式（O_TRUNC + O_WRONLY），写入过程中进程意外终止会产生半截文件：

| 场景 | 原因 | 半截程度 |
|------|------|----------|
| PHP `max_execution_time` 超时 | 大数据量写入时超时被强制终止 | 取决于写入速度，通常写了一部分 |
| 内存耗尽 `exit` | 大 datastore 的 serialize/gzdeflate 内存不足 | 可能完全没写（内存错误发生在构造数据时） |
| `kill -9` 进程 | 管理员强制杀进程 | 写了一部分 |
| 服务器断电 | 硬件故障 | 写了一部分或完全没写 |
| 磁盘满 | `disk_free_space` 检查后磁盘又被其他进程占用 | 写了一部分后失败 |

注意：`BookmarkIO::write()` 内部有 `checkDiskSpace()` 检查（[BookmarkIO.php#L133-L135](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L133-L135)），但检查和写入之间有 TOCTOU 竞态窗口（时间差），且只在 synchronized 内部检查。

### 18.2 半截文件的三种形态

根据截断位置的不同，半截文件分为三种形态：

#### 形态 A：仅 PHP 头部 + 部分 base64（尾部截断）

```
<?php /* S7QysKquBQA=...（后半段缺失）
```
- 缺少 ` */ ?>` 尾部
- 文件大小：几字节到几千字节不等

#### 形态 B：完整 PHP 包裹 + 中间 base64 截断（极罕见）

```
<?php /* S7QysKquBQA=...（中间断了）... */ ?>
```
- 头尾完整但中间 base64 数据缺了一块
- 通常只在极特殊的并发写交叉场景出现（两个进程的 write 交错）

#### 形态 C：空文件 / 仅 PHP 包裹

```
<?php /*  */ ?>
```
- truncate 之后立即被杀，还没开始写
- 或只写了极少量数据

### 18.3 BookmarkIO::read() 对三种形态的判定链路

完整判定流程（基于 [BookmarkIO::read()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L75-L104)）：

```
file_exists(datastore)
  │
  ├─ 否 → DatastoreNotInitializedException
  │
  └─ 是
      │
      is_writable(datastore)
        │
        ├─ 否 → NotWritableDataStoreException
        │
        └─ 是
            │
            file_get_contents → $content
              │
              substr($content, 9, -6)  // 剥 PHP 包裹
                │
                ├─ 如果尾部缺 " */ ?>": substr 从末尾取 -6 实际截取了有效数据
                │   → base64 字符串短了 6 字节（或者更多，如果 truncate 严重）
                │
                base64_decode($payload)
                  │
                  ├─ 正常 base64 尾部被截断 → 返回部分二进制 + PHP Notice
                  ├─ 严重损坏 → 返回 false
                  │
                  gzinflate($binary)
                    │
                    ├─ 数据流不完整 → false + PHP Warning
                    │
                    unserialize($inflated)
                      │
                      ├─ false 或损坏数据 → false + PHP Warning
                      │
                      empty($links)
                        │
                        ├─ true
                        │   │
                        │   filesize(datastore) > 100
                        │     │
                        │     ├─ 是 → NotWritableDataStoreException
                        │     └─ 否 → EmptyDataStoreException
                        │
                        └─ false → 正常返回（极小概率：半截数据刚好能反序列化）
```

### 18.4 三种半截形态的判定结果对照表

| 半截形态 | filesize | substr 结果 | base64_decode | gzinflate | unserialize | empty | filesize > 100 | 最终异常 |
|----------|----------|------------|---------------|-----------|-------------|-------|---------------|----------|
| 形态 A 轻微截断（几百字节） | >100B | base64 末尾少 6 字节 | 部分二进制 | false | false | true | 是 | **NotWritableDataStoreException** |
| 形态 A 严重截断（<100B） | <100B | 极短 base64 | false/部分 | false | false | true | 否 | **EmptyDataStoreException** ❗ |
| 形态 B 中间截断 | >100B | 完整长度 base64 但内容缺 | 不完整二进制 | false | false | true | 是 | **NotWritableDataStoreException** |
| 形态 C 空/仅包裹 | 9~16B | 空/空 | false | false | false | true | 否 | **EmptyDataStoreException** ❗ |

### 18.5 关键风险点：小半截文件触发空数据覆盖

**最危险的场景**：datastore 只写了几十字节就崩溃了（文件 < 100 字节）。

后果链：
```
1. 半截文件 < 100 字节
2. BookmarkIO::read() → EmptyDataStoreException
3. BookmarkFileService 构造函数 catch 住
4. $this->bookmarks = new BookmarkArray()   ← 空数组
5. 因为 isLoggedIn → $this->save()            ← 把空数据写回磁盘！
6. ❗ 原本只是部分损坏、可能还有办法手动恢复的半截文件
   被一份空的 BookmarkArray 彻底覆盖了
```

这是一个**危险的设计**：`EmptyDataStoreException` 的处理逻辑假设"空数据 = 需要初始化"，但实际上空/小文件也可能是**写入失败导致的损坏**。

对比：`NotWritableDataStoreException` 不被 catch，会冒泡为 500 错误，用户能看到异常，不会触发自动覆盖。

### 18.6 阈值 100 字节的来源与问题

`filesize > 100` 的判断（[BookmarkIO.php#L97](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L97)）意图是区分：
- 真正的空 datastore（新建的，只有 PHP 包裹壳）→ 初始化
- 损坏的 datastore（应该有数据但读不出来）→ 报错

但 100 字节是个**粗略阈值**：
- 空 BookmarkArray 序列化后约 60-80 字节（base64 + PHP 包裹）
- 只有 1-2 条书签的小型 datastore 可能也只有几百字节
- 如果恰好 datastore 本身就很小，截断后可能落到 100 字节以下 → 误触发空数据覆盖

更安全的设计应该用**校验和**或**版本标记**来区分"正常空数据"和"损坏数据"，而不是靠文件大小猜测。

---

## 十九、恢复备份期间的 UI 侧竞态与用户操作陷阱

### 19.1 典型场景：用户手忙脚乱恢复备份

**场景还原**：用户升级出问题了，按照文档说明用备份恢复。操作序列：

```
T0: 用户在浏览器 Tab1 看到错误页面
T1: 用户 SSH 到服务器，执行 cp data/datastore.20240115143022.php data/datastore.php
T2: cp 命令执行中...（大文件可能需要几秒）
T3: 用户不耐烦，切回浏览器按了 F5 刷新
T4: PHP 请求进入 → BookmarkFileService 构造
     → 读到一个正在被 cp 的半截 datastore
T5: cp 命令完成（但文件已经被 PHP 读了一半）
```

### 19.2 并发路径一：cp 过程中读触发空数据覆盖

这是最危险的路径，发生在以下条件同时满足时：

1. 备份文件较大（cp 需要时间）
2. 用户在 cp 完成前刷新了页面
3. 读到的半截数据 < 100 字节（恰好 cp 刚开始不久）

```
时间线（cp 命令 vs HTTP 请求）：

进程 A（cp 命令）             进程 B（PHP 请求）
───────────────────────      ───────────────────────
T0: fopen(datastore, w)
T1: write 前 50 字节
                              T2: 浏览器刷新，PHP 启动
                              T3: file_exists → true
                              T4: is_writable → true
                              T5: file_get_contents → 读到 50 字节半截
                              T6: unserialize → false
                              T7: empty → true + filesize=50 < 100
                              T8: throw EmptyDataStoreException
                              T9: BookmarkFileService catch → 初始化空 BookmarkArray
                              T10: save() → 写入空数据（覆盖了 A 正在写的文件）
T11: 继续写剩下的数据
      → 写的是半截文件？不，A 的 fd 是独立的，继续从自己的 offset 写
      → 但 B 的 save() 已经 truncate 并重写了文件
      → A 的后续写入会追加？不，A 是 w 模式，有自己的 inode？
      → 实际：如果 B 的 save() 是新的 fopen(w)，则文件 inode 不变，内容被替换
         A 持有的旧 fd 仍然指向同一个文件（同一个 inode），
         但 B 已经 truncate 了文件，A 的 write 会从 offset 50 开始写
         → 最终文件 = 前 50 字节空数据 + A 的后半段备份数据
         → 完全混乱，无法恢复
```

**最终后果**：备份文件和原始 datastore 都被破坏，两边都不完整。

### 19.3 并发路径二：备份是旧格式 → 双 Tab 并发迁移

如果用户恢复的是**遗留数组格式**的备份（FORMAT_A/B），恢复后的首次访问会触发 `migrate()` + `exit()`：

```
用户操作：
T0: 恢复遗留格式备份
T1: Tab1 刷新 → 触发 migrate() → 显示 "Please reload the page"
T2: 用户还没看到提示，在 Tab2 又刷新了一下
T3: Tab2 也触发 migrate() → 两个进程同时迁移
```

**竞态分析**（锁正常时）：
- 迁移过程中有多次 datastore 读写
- BookmarkIO 的 write 有 FlockMutex 保护
- 但 migrate() 整体不是原子的（先读、再转换、再写）
- 两个进程可能都读到旧格式数据、各自转换、各自写回
- 由于输入相同、转换是纯函数，最终写回内容一致 → 安全

**但锁降级时**：
- 两个进程的 write 无锁交错 → 可能产生混合内容
- 同第十三章分析的 datastore 并发写后果

### 19.4 并发路径三：迁移中用户不断刷新

`migrate()` 最后用 `exit()` 终止请求（[BookmarkFileService.php#L96-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkFileService.php#L96-L99)）：

```php
exit(
    'Your data store has been migrated, please reload the page.' . PHP_EOL .
    'If this message keeps showing up, please delete data/updates.txt file.'
);
```

用户看到这段纯文本提示后可能会：
1. 立即按 F5 刷新 → 正常（迁移已完成，instanceof 通过）
2. 但如果迁移**还没完成**就 exit 了？—— 不，migrate() 是同步执行的，exit() 在最后

但有一种危险情况：
- 迁移过程中某个 updateMethod 失败（返回 false）
- 但 migrate() 不检查返回值，照常用 `exit()` 输出提示
- 用户刷新 → 再次迁移 → 再次失败 → 无限循环
- 文档提示 "If this message keeps showing up, please delete data/updates.txt file"
- 但删除 updates.txt 会**重置所有升级进度**，包括已经成功的方法
- 已经成功的方法可能因幂等性不好（如 escape 方法）导致数据损坏

### 19.5 UI 侧竞态防护建议

代码层面目前**没有任何防护**，需要用户手动遵守：

1. **恢复备份前先停服务**：
   ```bash
   sudo systemctl stop php-fpm
   cp backup.dat datastore.php
   sudo systemctl start php-fpm
   ```
   
2. **单 Tab 操作**：升级/恢复期间只开一个浏览器 Tab

3. **看到迁移提示后等 3 秒再刷新**：确保 PHP 进程完全退出

4. **恢复后先验证再使用**：
   ```bash
   php -r '
     // 验证 datastore 完整性
     $c = file_get_contents("data/datastore.php");
     $d = substr($c, 9, -6);
     $data = @unserialize(@gzinflate(@base64_decode($d)));
     echo $data ? "OK, count=" . count($data) . "\n" : "FAIL\n";
   '
   ```

---

## 二十、降级方向兼容性：新版备份恢复到旧版本环境

### 20.1 降级恢复的典型场景

| 场景 | 原因 | 风险等级 |
|------|------|----------|
| 升级后功能不满意，想回滚到旧版本 | 用户操作 | 中 |
| 新版有 Bug，紧急降级 | 生产事故 | 高 |
| 开发环境用新版，生产环境用旧版，数据同步 | 部署问题 | 中 |
| 备份是新版导出的，需要在旧版环境中恢复 | 备份混用 | 高 |

### 20.2 PHP serialize 的降级方向规则

旧版本代码 `unserialize()` 新版数据时：

| 差异类型 | 行为 | 后果 |
|----------|------|------|
| 新版多了属性 | 属性保留在对象中，旧代码不访问 | ⚠️ 数据保留但不使用，下次 save 时会被序列化回去（形成"幽灵属性"） |
| 新版少了属性 | 旧代码访问时 → `Undefined property` Notice | ❌ 功能异常，但不 fatal |
| 类名变更 | `__PHP_Incomplete_Class` 对象 | ❌ 完全不可用，instanceof 全失败 |
| 类被删除 | 同上 | ❌ 同上 |
| 属性类型变更（如 array → object） | 保留原始类型 | ⚠️ 旧代码按旧类型使用可能出错 |

Shaarli 的历史版本中，Bookmark 类的演变以**属性增加**为主，类名不变，属性类型基本稳定。所以降级的主要风险是**幽灵属性**和**新属性默认值丢失**。

### 20.3 Bookmark 类属性的降级兼容性表

| 属性 | 存在版本 | 降级到旧版后的行为 |
|------|----------|-------------------|
| `id` | 全部版本 | 正常 |
| `title` | 全部版本 | 正常 |
| `url` | 全部版本 | 正常 |
| `description` | 全部版本 | 正常 |
| `tags` | 全部版本 | 正常（数组类型稳定） |
| `private` | 全部版本 | 正常 |
| `thumbnail` | 全部版本 | 正常 |
| `created` | v0.9+ | 正常（DateTime 对象） |
| `updated` | v0.9+ | 正常（DateTime 对象） |
| `shortUrl` | v0.9+ | 正常 |
| `sticky` | v0.12+ | 旧版无此属性 → 旧代码 `$bookmark->sticky` 触发 Undefined property Notice，但不致命 |
| `additionalContent` | 后期版本 | 幽灵属性，旧代码不访问 → 无功能影响，但下次 save 时会被序列化保留 |

### 20.4 BookmarkArray 降级兼容性

BookmarkArray 内部结构更稳定，主要属性（`$bookmarks`、`$ids`、`$keys`、`$urls`、`$position`）从引入以来变化不大。

**降级风险点**：
- 如果新版 BookmarkArray 增加了新属性，降级后这些属性作为幽灵属性存在
- 旧版的 reorder() / 搜索等方法使用的是已知属性，不受幽灵属性影响
- 但如果新版**修改了** `$bookmarks` 的内部结构（如从数组改成别的），会出问题（历史上没发生过）

### 20.5 配置文件的降级兼容性

配置降级比数据降级更危险，因为配置格式有明确的版本跳跃（PHP → JSON）：

| 备份格式 | 恢复到旧版环境 | 后果 |
|----------|---------------|------|
| JSON 配置（config.json.php） | 只支持 PHP 配置的旧版 | ConfigManager 检测不到 `.php` 文件 → 用 JSON 格式加载 → 能正常读写 JSON → ⚠️ 但旧版代码可能依赖某些已删除/重命名的配置键 |
| PHP 配置（config.php） | 新版环境 | 正常，新版兼容 PHP 格式 | ✅ 安全（有 ConfigPhp + LEGACY_KEYS_MAPPING） |
| JSON 配置含新键 | 旧版 | 旧版 ConfigJson 能读，但旧代码不认识这些键 → 被忽略 | ⚠️ 新功能配置丢失 |

**最严重的降级配置风险**：新版中某些配置键被重命名或合并，降级后旧版代码找不到对应的键 → 功能异常甚至错误。

### 20.6 降级恢复的操作步骤

#### 安全降级流程

```
步骤 1：在新版环境中导出为 Netscape HTML 格式（通用格式）
        Tools → Export → 下载 .html 文件
        （这是最安全的降级方式，不依赖 serialize）

步骤 2：在新版环境中导出配置为 PHP 格式（如果旧版只支持 PHP）
        或手动将 JSON 配置转换为 PHP 格式

步骤 3：部署旧版本代码

步骤 4：删除旧 data 目录，干净安装

步骤 5：在旧版中导入 Netscape HTML 书签

步骤 6：手动重新配置设置
```

#### 直接用 datastore 降级的风险操作（不推荐）

```
步骤 1：确认两个版本之间的 Bookmark 类差异
        - 类名是否相同？
        - 属性是只增不减吗？
        - 是否有 __sleep/__wakeup 魔术方法？

步骤 2：备份当前旧版的 data 目录

步骤 3：将新版 datastore 覆盖过去

步骤 4：访问验证
        - 首页是否正常加载？
        - 书签数量对不对？
        - 打开单条书签有没有错误？
        - 保存/编辑功能是否正常？

步骤 5：如果异常，立即回滚
```

### 20.7 降级兼容性风险总表

| 降级对象 | 风险等级 | 主要风险 | 恢复难度 |
|----------|----------|----------|----------|
| datastore (FORTC → FORMAT_C) | ⚠️ 中低 | 新属性丢失，幽灵属性残留 | 低（通常能用） |
| datastore (FORTC → FORMAT_B) | ❌ 高 | 类不存在 → __PHP_Incomplete_Class | 高（需格式转换） |
| config (JSON → PHP) | ❌ 高 | 旧版不支持 JSON 格式 | 中（需手动转换） |
| updates.txt | ⚠️ 中 | 旧版不认新版的方法名，全部重跑 | 低（幂等方法安全） |
| page cache | ✅ 低 | 缓存失效，重新生成 | 无 |
| history.php | ⚠️ 中 | 历史记录格式可能不兼容 | 低（不影响核心功能） |

---

## 二十一、权限受限下 flock 失败：SELinux/AppArmor 限制的 LockAcquireException 链路

### 21.1 SELinux/AppArmor 阻断 flock 的触发场景

在启用了强制访问控制（MAC）的 Linux 系统上，即使文件存在且 Unix 权限（chmod）正确，flock 系统调用仍可能被策略拒绝：

| 场景 | SELinux/AppArmor 行为 | flock() 返回值 |
|------|----------------------|----------------|
| PHP 进程运行在 `httpd_t` 域，init.php 文件标记为 `httpd_sys_content_t`（只读内容） | SELinux 拒绝 `httpd_t` 域对 `httpd_sys_content_t` 类型文件的 `lock` 权限 | `false` + PHP Warning + `/var/log/audit/audit.log` 中出现 AVC 拒绝记录 |
| AppArmor 配置文件仅允许 PHP `read` 访问 `/var/www/shaarli/`，未授予 `lock` 能力 | AppArmor 拒绝 flock 系统调用 | `false` + PHP Warning + `/var/log/syslog` 或 `journalctl` 中出现 DENIED 记录 |
| Docker/容器化部署中 seccomp 配置过滤了 flock 系统调用 | 系统调用被 EACCES 或 EPERM 拦截 | `false` + Warning |
| PHP-FPM 运行在受限 user namespace 中 | 锁的文件描述符传递被命名空间隔离破坏 | `false` 或异常行为 |

### 21.2 异常传播完整链路（与 fopen=false 的差异）

SELinux/AppArmor 拒绝的情况与 fopen 返回 false 的路径有**关键差异**：

```
SELinux/AppArmor 场景链路：

ContainerBuilder 构造 bookmarkService
    │
    ├─ fopen(SHAARLI_MUTEX_FILE, 'r')
    │     │
    │     └─ SELinux 允许 read → 返回有效 resource（≠ false）
    │
    ├─ new FlockMutex($valid_resource, 2)  ← FlockMutex 拿到了真实文件句柄
    │
    └─ BookmarkFileService 构造 → 正常完成
           │
           ▼
首次调用 BookmarkIO::write()
    │
    ├─ $this->synchronized($callback)
    │     │
    │     └─ $this->mutex->synchronized($callback)
    │           │
    │           ├─ flock($valid_resource, LOCK_EX)
    │           │     │
    │           │     └─ SELinux 拒绝系统调用
    │           │        → flock() 返回 false
    │           │        → PHP Warning: flock(): unable to lock file
    │           │        → 审计日志记录 AVC 拒绝
    │           │
    │           └─ malkusch/lock 检测到 flock 返回 false
    │              → 2 秒内重试（可配置超时）
    │              → 超时后 → 抛出 LockAcquireException
    │
    └─ BookmarkIO::synchronized() 捕获 LockAcquireException
          │
          └─ 直接执行 $callback → 无锁降级执行
```

**与 fopen=false 的关键区别**：
- fopen=false：FlockMutex 构造时就拿到 false，**第一次 flock 调用会立即失败**（flock(false) 报错）
- SELinux 拒绝：FlockMutex 构造时拿到有效 resource，**flock 尝试真实执行**，会经历 2 秒的重试等待后才抛异常

这意味着 SELinux 限制下的**每次文件写入都会多等 2 秒**（超时时间），对用户体验影响更大。

### 21.3 2 秒超时重试期间的行为

malkusch/lock 的 FlockMutex 在内部会循环重试 flock，直到超过 2 秒超时：

```
时序（SELinux 拒绝场景）：

T0: synchronized() 被调用
T1: flock($fh, LOCK_EX) → false (SELinux denied)
T2: usleep(100ms)
T3: flock($fh, LOCK_EX) → false (SELinux denied)
T4: usleep(100ms)
    ... (循环约 20 次，共 2 秒)
T20:最后一次 flock 尝试 → false
T21:抛出 LockAcquireException
T22:BookmarkIO::synchronized() 捕获
T23:无锁执行 callback
```

总耗时约 2 秒。如果有**多个并发请求**，每个请求都会独立经历这 2 秒超时，最终都降级为无锁执行，实际上**扩大了并发窗口**（因为每个请求都等了 2 秒，让更多请求有机会进入写路径）。

### 21.4 升级流程在 SELinux 限制下的表现

升级涉及多次 datastore 和 config 写入，每次都会触发：

```
runUpdates()
  │
  ├─ updater->update()
  │     │
  │     ├─ updateMethodConfigToJson
  │     │     └─ conf->write() → 无锁（2s 超时降级）
  │     │
  │     ├─ updateMethodDatastoreIds
  │     │     └─ linkDB->save() → 无锁（2s 超时降级）
  │     │
  │     ├─ updateMethodMigrateDatabase
  │     │     └─ BookmarkIO->write() → 无锁（2s 超时降级）
  │     │
  │     └─ ...
  │
  └─ writeUpdatesFile()
        └─ 本来就无锁
```

**总耗时估算**：每个写操作 2 秒超时，如果有 N 个迁移方法涉及写入，总耗时 ≈ 2N 秒。例如 3 个写入方法 → 6 秒。如果 PHP `max_execution_time` 配置为 30 秒，这个延时还能忍受；但如果是 5 秒，则会导致 PHP 执行超时中断，与第十八章分析的半截文件场景叠加。

### 21.5 检测与排查方法

判断是否因 SELinux/AppArmor 导致锁失效：

```bash
# 方法 1：检查 SELinux 状态
getenforce
# 输出 Enforcing → SELinux 强制模式下可能拒绝
# 输出 Permissive → SELinux 只记录不拒绝
# 输出 Disabled → 无影响

# 方法 2：查看审计日志
ausearch -m avc -ts recent | grep init.php
# 或
grep "flock\|lock" /var/log/audit/audit.log | tail -20

# 方法 3：检查 AppArmor 日志
grep "DENIED" /var/log/syslog | grep php
aa-status   # 查看 AppArmor 状态和加载的配置文件

# 方法 4：临时切换 SELinux 为 Permissive 验证
setenforce 0
# 重试升级，如果速度变快且不报错 → 确认是 SELinux 问题

# 方法 5：正确的修复方式（而非关闭 SELinux）
# 为 init.php 设置正确的 SELinux 上下文：
semanage fcontext -a -t httpd_sys_rw_content_t /var/www/shaarli/init.php
restorecon -v /var/www/shaarli/init.php
# 或者允许 HTTPD 进程加锁：
setsebool -P httpd_unified 1
```

### 21.6 与 fopen=false 场景的后果对比表

| 对比项 | fopen 返回 false | SELinux/AppArmor 拒绝 flock |
|--------|------------------|------------------------------|
| FlockMutex 构造时状态 | 存储 false | 存储有效 resource |
| 异常抛出时机 | 第一次 synchronized() 立即 | 2 秒超时后 |
| 每次写操作的额外耗时 | <1ms（立即降级） | ≈2000ms（超时等待） |
| 升级总耗时影响 | 可忽略 | 每个写操作 +2 秒 |
| 是否可能触发 PHP max_execution_time | 否 | 可能（写操作多时超时中断 → 半截文件） |
| 审计日志 | 无记录 | AVC / DENIED 记录 |
| 降级后的并发行为 | 一致（都是无锁） | 一致（都是无锁） |
| 误触发半截文件概率 | 低 | **高**（超时导致执行超时中断） |

---

## 二十二、解压乱码后对象层产生的内存错误链路

### 22.1 解压乱码的三种来源

datastore 解压乱码（gzinflate 返回非预期数据）的典型触发场景：

| 来源 | 具体表现 | gzinflate 行为 |
|------|----------|----------------|
| datastore 文件物理损坏 | 位翻转、磁盘坏块、截断 | 返回 false 或部分乱码二进制 |
| base64 解码时字符集问题 | 文件包含非法 base64 字符 | base64_decode 返回 false 或不完整二进制 |
| gzip 数据 CRC 校验失败 | 数据在传输/存储中被篡改 | gzinflate 返回 false 或 PHP Warning |
| 版本不兼容的序列化数据 | 跨版本 unserialize 产生乱码对象 | gzinflate 成功但 unserialize 返回错误类型 |
| 恶意构造的压缩炸弹 | gzinflate 解压后产出 GB 级数据 | 内存耗尽 → PHP Fatal error |

### 22.2 完整错误传播链路

从文件读取到对象层内存错误的六级链路：

```
第 1 层：文件 I/O
  file_get_contents(datastore.php)
    │
    └─ 成功 → $content = 完整或半截文件内容

第 2 层：PHP 包裹剥离
  substr($content, 9, -6)
    │
    ├─ 文件过短 → 返回 false 或空字符串
    └─ 正常 → 提取 base64 payload

第 3 层：Base64 解码
  base64_decode($payload, true)  ← 注意第二个参数 true 要求严格模式
    │
    ├─ 含非法字符 → 返回 false + PHP Warning
    └─ 正常 → 二进制压缩数据

第 4 层：GZIP 解压（最容易出内存问题的一层）
  gzinflate($binary)
    │
    ├─ 数据损坏 → false + Warning
    ├─ 压缩炸弹 → 解压出数 GB 数据 → 内存耗尽
    │   → PHP Fatal error:  Allowed memory size of XXX bytes exhausted
    │   → 进程立即终止，不执行后续代码
    │   → file_put_contents 可能未完成 → 半截文件
    │
    └─ 正常 → 序列化字符串

第 5 层：PHP 反序列化
  unserialize($inflated)
    │
    ├─ 数据格式错误 → false + Warning
    ├─ 对象类不存在 → __PHP_Incomplete_Class 对象
    │   → 后续 instanceof BookmarkArray → false
    │   → 触发 migrate() 路径
    │   → LegacyLinkDB 尝试再次读取 → 同样得到 __PHP_Incomplete_Class
    │   → foreach 遍历 __PHP_Incomplete_Class 失败 → PHP Warning
    │
    ├─ 属性类型错误 → 对象构造失败
    │   → 如 DateTime 属性被反序列化为字符串
    │   → 后续访问 $bookmark->getCreated()->format()
    │   → Fatal error: Call to a member function format() on string
    │
    └─ 正常 → BookmarkArray 或 array

第 6 层：对象层操作
  instanceof 检查 / BookmarkFilter / reorder 等
    │
    ├─ instanceof BookmarkArray → false（得到 array 或其他对象）
    │   → migrate() 路径 → LegacyUpdater
    │
    ├─ 对象属性不完整
    │   → 访问不存在的属性 → Undefined property Notice
    │   → Bookmark::validate() 检查 id/shortUrl/created → InvalidBookmarkException
    │
    ├─ DateTime 属性被反序列化为错误类型
    │   -> Fatal error: Uncaught Error: Call to a member function format() on bool
    │
    └─ 正常 → 无错误
```

### 22.3 压缩炸弹（Zip Bomb）的内存破坏路径

这是最危险的解压乱码场景。`gzinflate()` 对输入数据不做内存上限检查：

```
恶意构造的输入：
  极小的 gzip 压缩数据（几百字节）
  解压后产出 > 2 GB 的序列化字符串

执行路径：
  T1: gzinflate($payload) → 开始分配内存
  T2: memory_limit = 128M → 分配到 128MB 时
  T3: PHP Fatal error: Allowed memory size of 134217728 bytes exhausted
  T4: 进程立即终止（register_shutdown_function 可能执行但不可靠）
  T5: 如果此时正处于 write() 的 synchronized 回调中
      → file_put_contents 未执行或只执行了一部分
      → 半截文件 → 触发第十八章的空数据覆盖风险
```

Shaarli 代码中**没有对 gzinflate 输出大小做任何限制**，也没有 `ini_set('memory_limit')` 的临时提升或保护。

### 22.4 反序列化失败后的分支走向

[BookmarkIO::read()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/bookmark/BookmarkIO.php#L75-L104) 对 unserialize 返回值的处理：

```php
$links = unserialize(gzinflate(base64_decode(...)));

if (empty($links)) {
    if (filesize($this->datastore) > 100) {
        throw new NotWritableDataStoreException($this->datastore);
    }
    throw new EmptyDataStoreException();
}

return $links;
```

**反序列化不同失败结果的分支走向**：

| unserialize 返回值 | empty() | filesize > 100 | 抛出异常 | 后续行为 |
|-------------------|---------|---------------|----------|----------|
| `false` | true | >100 | NotWritableDataStoreException | 500 错误页 |
| `false` | true | <100 | EmptyDataStoreException | 初始化空 BookmarkArray → save() 覆盖 |
| `null` | true | — | 同上 | 同上 |
| `0` / `""` / `[]` | true | — | 同上 | 同上 |
| `__PHP_Incomplete_Class` 对象 | false（对象非空） | — | — | 返回该对象 → instanceof BookmarkArray → false → migrate() |
| `stdClass` 对象 | false | — | — | 返回该对象 → instanceof BookmarkArray → false → migrate() |
| `array`（遗留格式） | false（非空数组） | — | — | 返回数组 → instanceof BookmarkArray → false → migrate() |
| `BookmarkArray`（正常） | false（非空或空对象） | — | — | 返回对象 → instanceof 通过 |

### 22.5 __PHP_Incomplete_Class 进入 migrate() 的二次损坏

最危险的一种乱码是：文件格式完好但 serialize 数据引用了不存在的类，产生 `__PHP_Incomplete_Class` 对象。

后果链：

```
1. unserialize() 返回 __PHP_Incomplete_Class 对象
2. empty(object) → false
3. 返回给 BookmarkFileService
4. !instanceof BookmarkArray → true
5. 调用 migrate()
6. LegacyUpdater 实例化 LegacyLinkDB
7. LegacyLinkDB 再次读取同一个 datastore.php → 同样得到 __PHP_Incomplete_Class
8. LegacyLinkDB 构造函数中 foreach($this->links as $key => &$link)
   → __PHP_Incomplete_Class 实现了 Iterator 吗？
   → PHP 7+ 下会触发 Warning: Invalid argument supplied for foreach()
   → 返回空数据
9. LegacyUpdater 操作空数据
10. linkDB->save() → 把空数据写回 datastore.php
11. ❗ 原始乱码但可修复的文件被空数据覆盖
```

这是一条**隐蔽的二次损坏路径**：乱码数据不是直接被判为空，而是先被判为「非 BookmarkArray 的对象」→ 触发遗留迁移 → 迁移中再次失败 → 空数据覆盖。

### 22.6 Bookmark 对象内部 DateTime 属性的类型错误

如果乱码导致 Bookmark 对象的 `$created` 属性从 DateTime 变成了字符串或布尔值：

```php
// Bookmark 正常使用时
$bookmark->getCreated()->format('Y-m-d');

// 如果 $created 被反序列化为 false
// → Fatal error: Uncaught Error: Call to a member function format() on bool
// → 进程立即终止
// → 此时如果有写操作未完成 → 半截文件
```

这类错误属于 **E_ERROR / Fatal error**，不能被 try/catch 捕获（PHP 7+ 下 Error 实现了 Throwable 可以被捕获，但 Shaarli 的中间件只捕获 UnauthorizedException，其他异常走 ErrorController，但 Fatal error 在触发 autoload 前可能已终止）。

---

## 二十三、浏览器缓存与旧状态请求：Cache-Control、PageCache 和升级状态一致性

### 23.1 三层缓存架构

Shaarli 的响应缓存分为三层，需要在升级时确保全部失效：

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: HTTP 协议级缓存头（init.php 全局设置）          │
│  Cache-Control: no-store, no-cache, must-revalidate     │
│  Pragma: no-cache                                        │
│  Last-Modified: 当前时间                                  │
├─────────────────────────────────────────────────────────┤
│  Layer 2: 服务端页面缓存 PageCacheManager                │
│  data/pagecache/sha1(url).cache 文件                     │
│  主要缓存 RSS/ATOM Feed 和 Daily 页面                    │
├─────────────────────────────────────────────────────────┤
│  Layer 3: RainTPL 模板缓存                               │
│  tmp/rain-tpl-cache/ 下编译后的 PHP 模板                 │
│  每次请求都检查模板文件更新时间                           │
└─────────────────────────────────────────────────────────┘
```

### 23.2 HTTP 缓存头的全局设置

[init.php#L82-L86](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/init.php#L82-L86) 在每次请求开始时发送严格的禁用缓存头：

```php
header("Last-Modified: " . gmdate("D, d M Y H:i:s") . " GMT");
header("Cache-Control: no-store, no-cache, must-revalidate");
header("Cache-Control: post-check=0, pre-check=0", false);
header("Pragma: no-cache");
```

**各项含义**：
- `no-store`：浏览器和任何中间代理**都不得存储**响应的任何部分
- `no-cache`：可以存储但**使用前必须向服务器验证**（即强制发送 If-Modified-Since）
- `must-revalidate`：缓存过期后**必须向服务器验证**，不能直接使用过期副本
- `post-check=0, pre-check=0`：IE 专用的缓存控制扩展，同样禁用缓存
- `Pragma: no-cache`：HTTP/1.0 兼容

这些头的设置意味着：**理论上浏览器不会缓存任何 Shaarli 页面**。每次访问都会重新请求服务器。

### 23.3 Service Worker 存在性确认

代码库全局搜索无 Service Worker 相关引用（无 `navigator.serviceWorker.register`、无 `sw.js` 文件、无 Service Worker 相关 HTML 标签）。

**结论**：Shaarli 不使用 Service Worker 进行离线缓存。升级期间的缓存一致性问题**不涉及 SW 层面**，仅需考虑浏览器标准 HTTP 缓存和服务端 PageCache。

### 23.4 服务端 PageCache 的工作机制

[PageCacheManager](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/render/PageCacheManager.php) 管理基于文件的页面缓存：

**写入路径**（以 Feed 为例，[FeedController.php#L54](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/controller/visitor/FeedController.php#L54)）：
```php
$cache = $this->container->pageCacheManager->getCachePage($pageUrl);
// ... 生成 RSS 内容 $content ...
$cache->cache($content);  // file_put_contents(data/pagecache/sha1(url).cache, $content)
```

**读取路径**（[CachedPage::cachedVersion()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/feed/CachedPage.php#L48-L67)）：
```php
public function cachedVersion()
{
    if (!$this->shouldBeCached) return null;       // 登录用户不缓存
    if (!is_file($this->filename)) return null;    // 缓存文件不存在
    // ... DatePeriod 有效期检查 ...
    return file_get_contents($this->filename);     // 返回缓存内容
}
```

关键设计：**已登录用户（`isLoggedIn=true`）的请求完全不使用 PageCache**，只有匿名访客会命中缓存。

### 23.5 升级时的缓存失效路径

[ShaarliMiddleware::runUpdates()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/front/ShaarliMiddleware.php#L74-L82) 在升级成功后：

```php
if (!empty($newUpdates)) {
    $this->container->updater->writeUpdates(...);
    $this->container->pageCacheManager->invalidateCaches();  // ← 清除服务端缓存
}
```

`invalidateCaches()` 最终调用 `purgeCachedPages()`：
```php
array_map('unlink', glob($this->pageCacheDir . '/*.cache'));  // 删除所有 .cache 文件
```

**缓存失效的覆盖范围**：

| 缓存层 | 升级时是否自动失效 | 失效方式 |
|--------|-------------------|----------|
| HTTP 浏览器缓存 | ✅ 是（设计上） | no-store/no-cache 头本身阻止缓存 |
| 服务端 PageCache (.cache 文件) | ✅ 是 | invalidateCaches() 删除 |
| RainTPL 模板编译缓存 | ❌ **否** | 依赖文件 mtime 自动检测 |

### 23.6 实际场景中的缓存不一致风险

虽然 HTTP 头设置了 no-store，但在以下场景仍可能出现缓存不一致：

#### 场景 1：用户在升级前已打开页面

```
T0: 用户浏览器打开 Shaarli 首页（已登录）
T1: 管理员在服务器上替换代码 + 复制备份
T2: 用户浏览器 Tab 未关闭，JavaScript 轮询或用户点击链接
T3: 由于浏览器设置了 no-cache，会发送请求
     → 但用户 Cookie 中的 session 仍有效
     → ShaarliMiddleware::runUpdates() 执行
     → 检测到数据格式是旧的 → 触发 migrate()
     → exit("Please reload")
T4: 用户看到纯文本提示
```

这种情况下**不会显示旧内容**，因为每次请求都会回到服务器。

#### 场景 2：升级期间匿名访客访问 Feed

```
T0: 管理员开始升级（替换代码中）
T1: 匿名访客请求 /feed/atom
T2: 旧代码（部分文件未替换完）生成 RSS → 写入 PageCache
T3: 代码替换完成 + 升级执行 + invalidateCaches() → 删除 .cache
T4: 缓存被清除，新请求生成新格式 Feed → 正常
```

这种场景无缓存残留，因为 invalidateCaches() 在升级最后执行。

#### 场景 3：HTTP 代理/CDN 缓存

如果用户在 Shaarli 前面部署了 Varnish、Cloudflare 等反向代理缓存：
- 即使 Shaarli 发送了 no-cache，代理配置可能忽略这些头并自行缓存
- 升级后旧的响应可能被代理继续提供给访客
- **需要在代理层面额外执行缓存清除**

### 23.7 RainTPL 模板缓存的版本一致性

RainTPL 编译模板到 `tmp/rain-tpl-cache/*.rain.php`。它的缓存校验机制是**比较模板文件 mtime**：如果模板文件的修改时间晚于编译缓存文件，就重新编译。

升级时替换了模板文件（tpl/ 目录），文件 mtime 更新 → RainTPL 自动检测并重新编译。**这层不需要手动清除**。

但如果升级时用 `cp -a`（保留时间戳）复制模板文件，mtime 可能不变 → RainTPL 认为缓存有效 → 旧模板与新代码不兼容 → 模板报错。此时需要手动删除 tmp/rain-tpl-cache/ 下所有文件。

---

## 二十四、插件元数据与代码的版本兼容性：降级加载实现

### 24.1 插件系统的四层结构

每个 Shaarli 插件由四层文件组成，每层在跨版本时的兼容性风险不同：

```
plugins/<plugin_name>/
├── <plugin_name>.php     // 主代码文件（含 hook_* 函数）
├── <plugin_name>.meta    // INI 格式元数据（描述、参数定义）
├── <plugin_name>.css     // 可选：CSS 样式
└── <plugin_name>.js      // 可选：JavaScript
```

元数据示例（[demo_plugin.meta](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/plugins/demo_plugin/demo_plugin.meta)）：
```ini
description="A demo plugin covering all use cases..."
parameters="DEMO_PLUGIN_PARAMETER;DEMO_PLUGIN_OTHER_PARAMETER"
parameter.DEMO_PLUGIN_PARAMETER="This is a parameter..."
parameter.DEMO_PLUGIN_OTHER_PARAMETER="Other demo parameter"
```

### 24.2 插件加载的完整流程与错误容忍

[PluginManager::load()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/plugin/PluginManager.php#L84-L107) 的容错设计：

```
load($authorizedPlugins)
    │
    ├─ glob(plugins/*, GLOB_ONLYDIR) 扫描插件目录
    │
    ├─ 遍历 authorizedPlugins 中每个配置启用的插件名
    │     │
    │     ├─ array_search 检查目录是否存在
    │     │   └─ 不存在 → continue（静默跳过）
    │     │
    │     └─ loadPlugin($dir, $pluginName)
    │           │
    │           ├─ 目录不存在 → PluginFileNotFoundException
    │           │   → catch → error_log → continue
    │           │
    │           ├─ <plugin>.php 不存在 → PluginFileNotFoundException
    │           │   → catch → error_log → continue
    │           │
    │           ├─ include_once <plugin>.php
    │           │   └─ 任何 Throwable（语法错误、类不存在、依赖缺失）
    │           │      → catch，错误消息加入 $this->errors
    │           │      → 插件不加入 loadedPlugins
    │           │
    │           ├─ <plugin>_init() 调用
    │           │   └─ 任何 Throwable
    │           │      → catch，错误消息加入 $this->errors
    │           │
    │           └─ <plugin>_register_routes() 调用
    │               └─ PluginInvalidRouteException
    │                  → 不 catch → 冒泡到上层
    │
    └─ 全部完成，不抛异常（单个插件失败不影响整体）
```

**核心设计原则：单个插件失败不影响 Shaarli 主体运行**。即使大部分插件加载失败，核心功能仍可用。

### 24.3 元数据 getPluginsMeta() 的版本兼容处理

[PluginManager::getPluginsMeta()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/plugin/PluginManager.php#L230-L270) 读取 .meta 文件时的容错：

```php
foreach ($dirs as $pluginDir) {
    $plugin = basename($pluginDir);
    $metaFile = $pluginDir . $plugin . '.meta';
    if (!is_file($metaFile) || !is_readable($metaFile)) {
        continue;  // ← .meta 文件缺失或不可读 → 静默跳过该插件
    }

    $metaData[$plugin] = parse_ini_file($metaFile);  // ← parse_ini_file 返回 false 时
    // ... 后续代码假设 $metaData[$plugin] 是数组
    // 如果 .meta 文件损坏，parse_ini_file 返回 false
    // → 后续 $metaData[$plugin]['order'] 触发 Warning: Illegal string offset 'order'
    // → 但不致命，页面仍能显示
```

**降级兼容行为**：

| .meta 文件状态 | getPluginsMeta() 行为 | 对用户的影响 |
|----------------|----------------------|-------------|
| 文件不存在 | 插件不出现在插件管理页列表中 | 插件仍可能正常工作（.php 正常加载），只是后台看不到配置项 |
| 文件存在但 parse_ini_file 失败 | 触发 PHP Warning，插件以错误元数据形式出现 | 后台插件列表显示异常，但不影响前端功能 |
| 参数缺失 `parameters=` 行 | `$params = []`，插件无参数配置 | 可用默认值或直接不配置参数 |
| 旧版本新增参数未在新版中定义 | 参数被保留在配置中但不被旧版插件代码使用 | 无影响，忽略多余参数 |
| 新版删除了旧版中存在的参数 | 参数定义不存在但配置中有值 | parse_ini_file 不会报错，参数值被保留但不被使用 |

### 24.4 插件配置的存储与跨版本兼容

插件配置存储在 ConfigManager 的 `plugins.*` 命名空间下：

```json
{
  "plugins": {
    "ENABLED": ["wallabag", "qrcode"],
    "WALLABAG_URL": "https://wallabag.example.com",
    "PIWIK_URL": "https://piwik.example.com"
  }
}
```

跨版本时的兼容性：

- **插件已删除但配置仍存在**：配置项保留在 JSON 中，不被任何代码读取 → 占用空间但无影响
- **新版插件增加新参数**：旧配置中无该参数 → 插件代码需处理 `conf->get()` 返回 null 的情况 → 通常有默认值兜底
- **旧版插件使用了新版已删除的核心 hook**：`function_exists($hookFunction)` 检查返回 false → hook 不执行 → 插件功能降级但不报错
- **新版插件的 init() 依赖新版核心 API**：init() 抛出异常 → 被 catch → 插件不加载 → 错误消息记录在 `$this->errors`

### 24.5 新版插件降级到旧版的三种失败模式

| 失败模式 | 触发条件 | 用户可见表现 | 数据影响 |
|----------|----------|-------------|----------|
| **模式 1：静默不加载** | 新版插件使用了旧版不存在的 hook 名称、或依赖旧版没有的核心类 | 插件不工作，后台插件列表可能不显示 | 配置保留，不损坏 |
| **模式 2：错误日志记录** | 新版插件的 `_init()` 函数使用了旧版不存在的 API | 页面正常显示，但 error_log 中有异常记录 | 配置保留 |
| **模式 3：管理页异常** | 新版 .meta 文件含旧版 parse_ini_file 无法解析的语法 | 后台插件管理页显示 PHP Warning | 不影响前端功能 |
| **模式 4（最严重）：模板/资源引用错误** | 新版插件的 .php 中 render 了旧版不存在的模板文件 | 调用 hook 的页面抛异常 → ErrorController 500 | 可能导致页面无法访问，但核心数据不损坏 |

### 24.6 插件系统的降级恢复步骤

如果升级后因插件不兼容导致页面 500：

```bash
# 步骤 1：通过配置文件禁用所有插件
cd data/
# 编辑 config.json.php，将 plugins.ENABLED 设置为空数组
# 或直接重命名 plugins 目录
mv plugins plugins_disabled

# 步骤 2：验证核心功能恢复
curl -I http://shaarli.example.com/
# 期望 HTTP 200

# 步骤 3：逐个启用插件，定位不兼容的插件
# 在 plugins_disabled 中逐个移回 plugins/ 目录
# 每移动一个测试一次页面访问

# 步骤 4：对不兼容插件降级处理
# - 检查插件目录是否有旧版本可用
# - 或修改 <plugin>_init() 中的新版 API 调用为旧版等价
# - 或在 plugin 代码中添加版本检查

# 步骤 5：检查 PluginManager 错误
# 在浏览器开发者模式或页面 HTML 源码中查找
# "plugin incompatibility" 字样的错误消息
```

### 24.7 插件 hook 执行的防御性设计

[PluginManager::executeHooks()](file:///d:/fz/0601-1/solo-dogfeeding/code/80-Shaarli/application/plugin/PluginManager.php#L118-L150) 的容错确保单个插件 hook 失败不影响其他插件：

```php
foreach ($this->loadedPlugins as $plugin) {
    $hookFunction = $this->buildHookName($hook, $plugin);

    if (function_exists($hookFunction)) {
        try {
            $data = call_user_func($hookFunction, $data, $this->conf);
        } catch (\Throwable $e) {
            // 单个插件 hook 异常 → 记录错误，继续执行下一个插件
            $error = $plugin . t(' [plugin incompatibility]: ') . $e->getMessage();
            $this->errors = array_unique(array_merge($this->errors, [$error]));
        }
    }
    // 函数不存在 → 静默跳过，不报错
}
```

这意味着：即使有 9 个插件正常、1 个插件 hook 抛异常，**数据仍会被正确传递**给后续插件，不会中断整个 hook 链。
