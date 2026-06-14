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
