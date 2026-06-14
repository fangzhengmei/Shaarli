# 渲染与缓存深度补充分析

> 本文档订正前序分析中的若干事实偏差，并完整解析每日 RSS 缓存、render_feed 钩子差异等核心细节。

---

## 一、每日 RSS 入口与 validityPeriod 时间窗口缓存

### 1.1 路由与入口

每日 RSS 的路由定义见 [index.php#L125](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/index.php#L125)：

```php
$this->get('/daily-rss', '\Shaarli\Front\Controller\Visitor\DailyController:rss')->setName('rss');
```

> ⚠️ 注意：主 Feed（RSS/ATOM）和每日 RSS 共享 Slim 路由名 `rss`，但实际是不同控制器的不同方法。

### 1.2 三种粒度的每日 RSS

每日 RSS 支持三种时间粒度，通过查询参数（隐式键名）区分，见 [DailyPageHelper::extractRequestedType()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/helper/DailyPageHelper.php#L26-L35)：

| 粒度 | URL 格式 | 日期格式 | RSS 条目数（天数） | validityPeriod 窗口 |
|------|---------|---------|-------------------|--------------------|
| day   | `/daily-rss` 或 `/daily-rss?day=20201016` | `Ymd`（年月日） | 30 天 | 当日 00:00:00 → 23:59:59 |
| week  | `/daily-rss?week=202041` | `YW`（年+周号） | 26 周（~6 个月） | 当周周一 00:00 → 周日 23:59 |
| month | `/daily-rss?month=202010` | `Ym`（年月） | 12 个月 | 当月 1 号 00:00 → 月末 23:59 |

### 1.3 DailyPageHelper::getCacheDatePeriodByType() 详解

核心代码 [DailyPageHelper.php#L231-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/helper/DailyPageHelper.php#L231-L240)：

```php
public static function getCacheDatePeriodByType(string $type, DateTimeImmutable $requested = null): DatePeriod
{
    $requested = $requested ?? new DateTimeImmutable();

    return new DatePeriod(
        static::getStartDateTimeByType($type, $requested),   // 起始
        new \DateInterval('P1D'),                            // 间隔（1天，仅为构造 DatePeriod 所需）
        static::getEndDateTimeByType($type, $requested)      // 结束
    );
}
```

**关键点**：
- `DatePeriod` 在这里并非用于"生成一段时间内的每天"，而是作为一个携带起止时间的容器
- `DateInterval('P1D')` 只是构造参数要求，实际在判断缓存有效性时不会用到间隔
- 实际读取的是 `$period->getStartDate()` 和 `$period->getEndDate()`

### 1.4 validityPeriod 在 CachedPage 中的检查逻辑

代码见 [CachedPage.php#L53-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/feed/CachedPage.php#L53-L66)：

```php
public function cachedVersion()
{
    if (!$this->shouldBeCached) {
        return null;
    }

    if (!is_file($this->filename)) {
        return null;
    }

    // ↓↓↓ validityPeriod 时间窗口检查 ↓↓↓
    if ($this->validityPeriod !== null) {
        $cacheDate = \DateTime::createFromFormat('U', (string) filemtime($this->filename));
        if (
            $cacheDate < $this->validityPeriod->getStartDate()
            || $cacheDate > $this->validityPeriod->getEndDate()
        ) {
            return null;  // 缓存文件的 mtime 不在该时间段内 → 视为过期
        }
    }

    return file_get_contents($this->filename);
}
```

**检查逻辑**：
```
缓存文件的修改时间（filemtime）
    ├── 早于 period 开始时间 → 无效（跨周期了，需要重新生成）
    ├── 晚于 period 结束时间 → 无效（应该不存在这种情况，除非未来文件）
    └── 在 period 范围内     → 有效，直接返回缓存
```

### 1.5 具体场景示意（以 Day 粒度为例）

假设今天是 **2020-10-16（周五）**：

1. **10-16 上午 9:00** 首次请求 `/daily-rss`
   - `DatePeriod`：2020-10-16 00:00 → 2020-10-16 23:59
   - 缓存文件不存在，重新生成，缓存写入后 filemtime = 09:00
   - ✅ 09:00 在 [00:00, 23:59] 范围内 → 缓存有效

2. **10-16 下午 3:00** 再次请求
   - 缓存 filemtime = 09:00，仍在当日范围内 → ✅ 返回缓存

3. **10-17 上午 9:00** 请求（第二天了）
   - `DatePeriod`：2020-10-17 00:00 → 2020-10-17 23:59
   - 缓存 filemtime = 10-16 09:00
   - ❌ 10-16 09:00 < 10-17 00:00 → **时间窗口自动过期**，返回 null，重新生成
   - 新缓存 filemtime = 10-17 09:00 → 循环往复

### 1.6 主 Feed 与每日 RSS 的缓存对比

| 维度 | 主 Feed（FeedController） | 每日 RSS（DailyController:rss） |
|------|--------------------------|--------------------------------|
| 入口 | `/feed/atom`、`/feed/rss` | `/daily-rss` |
| 缓存键 | `sha1(page_url)` | `sha1(page_url)`（含 `?week`/`?month` 区分） |
| validityPeriod | ❌ **无**，仅依赖 invalidateCaches() 显式清除 | ✅ **有**，按 day/week/month 粒度自动过期 |
| 过期机制 | 被动等待：书签变更/登出/清缓存 | 主动到期：缓存文件 mtime 不在时间窗口内即自动失效 |
| 未完结条目过滤 | 无（包含最新所有） | ✅ 有：`new DateTime() < $endDateTime` 时跳过当前未结束周期 |

### 1.7 每日 RSS 的"未完结周期"过滤

除了缓存层的 validityPeriod，每日 RSS 还有一层**业务层过滤**，见 [DailyController.php#L127-L130](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/DailyController.php#L127-L130)：

```php
// We only want the RSS entry to be published when the period is over.
if (new DateTime() < $endDateTime) {
    continue;  // 跳过当前尚未结束的日期/周/月
}
```

**目的**：确保 RSS 阅读器不会接收到内容还在变动的条目。比如今天还没结束，今天的每日条目可能还会新增书签，因此不在 Feed 中发布。

---

## 二、主 Feed 与每日 RSS 的 render_feed 钩子差异

### 2.1 主 Feed（FeedController）- 触发 render_feed

代码见 [FeedController.php#L49](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/FeedController.php#L49)：

```php
$data = $this->container->feedBuilder->buildData($feedType, $request->getParams());
$this->executePageHooks('render_feed', $data, 'feed.' . $feedType);  // ✅ 触发
$this->assignAllView($data);
$content = $this->render('feed.' . $feedType);
```

**特征**：
- 明确调用 `executePageHooks('render_feed', ...)`
- 模板参数 `target` = `'feed.atom'` 或 `'feed.rss'`
- 插件可以注入 `$data['feed_plugins_header'][]`

### 2.2 每日 RSS（DailyController:rss）- **不触发 render_feed**

代码见 [DailyController.php#L150-L160](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/visitor/DailyController.php#L150-L160)：

```php
$this->assignAllView([
    'title' => $this->container->conf->get('general.title', 'Shaarli'),
    'index_url' => $indexUrl,
    'page_url' => $pageUrl,
    'hide_timestamps' => $this->container->conf->get('privacy.hide_timestamps', false),
    'days' => $dataPerDay,
    'type' => $type,
    'localizedType' => $this->translateType($type),
]);

$rssContent = $this->render(TemplatePage::DAILY_RSS);  // ⚠️ 直接 render，没有 executePageHooks
```

**特征**：
- ❌ **不调用**任何 `executePageHooks()`（包括 render_feed）
- 只走 `executeDefaultHooks()`（render_includes / render_header / render_footer），但这三个是通用 HTML 页面钩子，对 RSS XML 模板基本无意义
- DailyController 的方法注释也明确说明：`"This RSS feed cannot be filtered and does not trigger plugins yet."`

### 2.3 差异对照表

| 维度 | 主 Feed（/feed/rss、/feed/atom） | 每日 RSS（/daily-rss） |
|------|----------------------------------|----------------------|
| `render_feed` 钩子 | ✅ 触发，target=`feed.rss`/`feed.atom` | ❌ 不触发 |
| `executeDefaultHooks()` | ✅ 走 render（在 `ShaarliVisitorController::render()` 内） | ✅ 同左（但对 RSS 模板无效） |
| `feed_plugins_header` 占位符 | ✅ 可被插件填充 | ❌ 即使模板中有占位符也没有数据 |
| `_PAGE_` 元数据注入 | `feed.atom` / `feed.rss` | `dailyrss` |
| 插件可影响 | 内容、Header | 几乎无法影响（因为 render_feed 不触发） |

### 2.4 模板层的呼应

主 Feed 模板（[feed.rss.html](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/tpl/default/feed.rss.html#L13)、[feed.atom.html](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/tpl/default/feed.atom.html#L11)）都有：

```html
{loop="$feed_plugins_header"}
  {$value}
{/loop}
```

每日 RSS 模板（`dailyrss.html`）虽然也可能包含类似占位符，但 **DailyController 不触发钩子，所以数组始终为空**。

---

## 三、FileUtils::clearFolder 默认 exclude 行为订正

### 3.1 之前的归因（不准确）

> "exclude：排除的文件名数组（**默认排除 `.htaccess`**）"

### 3.2 订正：方法本身无默认排除

[FileUtils::clearFolder()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/helper/FileUtils.php#L95-L129) 签名：

```php
public static function clearFolder(string $path, bool $selfDelete, array $exclude = []): bool
```

**方法本身的默认值是 `[]`（空数组），没有默认排除任何文件。**

### 3.3 `.htaccess` 排除来自调用方

是 [ServerController::clearCache()](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ServerController.php#L71-L97) 在调用时显式传入的：

```php
public function clearCache(Request $request, Response $response): Response
{
    $exclude = ['.htaccess'];   // ← 调用方手动指定，不是 clearFolder 的默认值

    // ... 选择 $folders ...

    foreach ($folders as $folder) {
        FileUtils::clearFolder($folder, false, $exclude);  // ← 作为参数传入
    }
}
```

### 3.4 完整事实

| 维度 | 事实 |
|------|------|
| `FileUtils::clearFolder()` 默认 exclude | `[]`（空数组，不排除任何文件） |
| `.htaccess` 何时被排除 | 仅在 `ServerController::clearCache()` 这个手动清缓存场景下 |
| 其他调用方如果不传 exclude | 会删除 `.htaccess`，可能导致访问规则丢失 |
| PageCacheManager::purgeCachedPages() | 使用 `glob('*.cache')` 只匹配 .cache 后缀，不经过 FileUtils，天然保留其他文件 |

---

## 四、配置保存触发缓存失效的条件订正

### 4.1 之前的归因（不准确）

> "保存配置后**且配置文件真的被修改了**"

### 4.2 订正：无条件触发，写入成功即触发

代码见 [ConfigureController.php#L114-L117](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/front/controller/admin/ConfigureController.php#L114-L117)：

```php
try {
    $this->container->conf->write($this->container->loginManager->isLoggedIn());
    $this->container->history->updateSettings();
    $this->container->pageCacheManager->invalidateCaches();  // ← 只要 write() 没抛异常就执行，无前置判断
} catch (Throwable $e) {
    // 异常分支：不执行清缓存
}
```

**ConfigManager::write() 内部也没有"是否实际修改过"的检查**，见 [ConfigManager.php#L214-L241](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/application/config/ConfigManager.php#L214-L241)：

```php
public function write($isLoggedIn)
{
    // 权限校验 + 必填字段校验 ...（都通过才往下走）

    return $this->configIO->write($this->getConfigFileExt(), $this->loadedConfig);
    // ↑ 直接写入，不对比新旧内容是否相同
}
```

**ConfigIO 实现（ConfigJson / ConfigPhp）** 也不做 diff，直接 `file_put_contents` 覆盖。

### 4.3 完整事实

| 条件 | 是否触发 invalidateCaches() |
|------|---------------------------|
| 用户未登录 | ❌ （write() 抛 UnauthorizedConfigException） |
| 必填字段缺失 | ❌ （write() 抛 MissingFieldConfigException） |
| 文件写入失败 | ❌ （write() 抛 IOException） |
| 文件写入成功（即使内容完全相同） | ✅ **触发** |
| 配置内容完全没变（用户点了"保存"但没改） | ✅ **触发** |

**结论**：ConfigureController 保存配置后，只要 `ConfigIO::write()` 不抛异常，就**无条件**触发 `invalidateCaches()`，**不判断配置是否实际变化**。

---

## 五、RainTPL include 编译中 tpl_dir_temp 死赋值订正

### 5.1 之前的描述（不准确）

之前的分析中只是转录了代码片段，未指出这行赋值的实际问题：

```php
// 普通 include
'$tpl_dir_temp = self::$tpl_dir;' .
'$tpl->assign( $this->var );' .
'$tpl->draw( ... );'

// 带缓存 include
'$tpl_dir_temp = self::$tpl_dir;' .
'$tpl->assign( $this->var );' .
'$tpl->draw( ... );'
```

### 5.2 订正：tpl_dir_temp 是**死赋值（Dead Write）**

代码位置：[rain.tpl.class.php#L422](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L422) 和 [rain.tpl.class.php#L432](file:///d:/fz/0601-1/solo-dogfeeding/code/79-Shaarli/inc/rain.tpl.class.php#L432)

**在 include 编译出的 PHP 代码中**：

```php
<?php 
    $tpl = new RainTpl;
    $tpl_dir_temp = self::$tpl_dir;     // ← 写入变量
    $tpl->assign( $this->var );
    $tpl->draw( dirname("includes") . ( substr("includes",-1,1) != "/" ? "/" : "" ) . basename("includes") );
    // $tpl_dir_temp 后续再也没有被读取 → 死赋值
?>
```

**证据**：全局搜索 `tpl_dir_temp` 仅出现在这两处写入点，**完全没有任何读取操作**。

### 5.3 可能的原始意图（推测）

RainTPL 可能原本打算在 `draw()` 结束后恢复 tpl_dir：

```php
// 可能原本的设计是：
$tpl_dir_temp = self::$tpl_dir;   // 保存旧值
// ... 可能在某处修改 self::$tpl_dir ...
self::$tpl_dir = $tpl_dir_temp;   // 恢复旧值 ← 但这行从未实现过
```

但由于 Shaarli 场景下 `tpl_dir` 是全局静态值，不随 include 而改变，因此这行保存/恢复代码被**部分删除但清理不彻底**，留下了死赋值。

### 5.4 实际影响

- **无运行时语义影响**：变量赋值后不读取，PHP 执行结果等价于不写这行
- **微小性能开销**：每次 include 多一次变量赋值操作，可忽略
- **代码可读性问题**：阅读编译后代码时可能产生困惑

---

## 附录：订正点汇总

| # | 订正主题 | 原理解释 | 结论 |
|---|---------|---------|------|
| 1 | 每日 RSS 缓存 | 使用 DatePeriod 做 validityPeriod 时间窗口，缓存 mtime 不在窗口内自动过期 | 非被动失效，有主动到期机制 |
| 2 | render_feed 钩子 | 仅 FeedController（主 RSS/ATOM）触发；DailyController（每日 RSS）完全不触发 | 两者不等价 |
| 3 | FileUtils::clearFolder 默认 exclude | 方法默认值是 `[]` 空数组；`.htaccess` 排除来自 ServerController 调用方显式传入 | 非方法默认行为 |
| 4 | Configure 保存缓存失效条件 | 只要 ConfigIO::write() 不抛异常就无条件 invalidateCaches()；不判断内容是否真的变化 | 非"且配置文件真的被修改" |
| 5 | tpl_dir_temp | RainTPL include 编译出的代码中 `$tpl_dir_temp = self::$tpl_dir` 只写不读，是历史残留死赋值 | 无实际语义影响 |
