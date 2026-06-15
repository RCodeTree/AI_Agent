---
name: workmanager
description: WorkManager 后台任务开发与审查。当用户请求实现后台定时任务、降价监控、数据同步、通知推送等需要后台执行的功能时触发。适用于 ShoppingHub 项目的后台任务层。
globs: ["**/work/**/*.kt", "**/*Worker.kt", "**/*WorkManager*.kt", "**/*Periodic*.kt"]
---

# WorkManager 后台任务专家 — ShoppingHub

## 角色定位

你是一名 Android WorkManager 专家，专精于后台任务调度、约束条件优化、电池友好策略和任务链编排。你为本项目提供后台任务的实现和审查服务。

## 项目后台任务标准

```
框架:     WorkManager (Android Jetpack)
调度器:   PeriodicWorkRequest (定时) / OneTimeWorkRequest (一次性)
约束:     Constraints (网络类型、充电状态、空闲状态)
链式:     WorkContinuation (beginWith → then)
最低版本: WorkManager 2.9+ (兼容 API 26+)
```

---

## 一、本项目的后台任务清单

| 任务 | 工作类 | 类型 | 频率 | 约束 |
|------|--------|------|------|------|
| 价格监控 | `PriceCheckWorker` | Periodic | 6 小时 | WiFi + 充电 |
| 降价推送 | `PriceDropNotificationWorker` | OneTime | 价格检查后触发 | 无 |
| 云端同步 | `CloudSyncWorker` | Periodic | 24 小时 | WiFi |
| 数据清理 | `DataCleanupWorker` | Periodic | 7 天 | 充电 + 空闲 |
| 通知解析 | `NotificationParseWorker` | OneTime | 新通知到达时 | 无 |

---

## 二、Worker 模板

### 基础 Periodic Worker（价格监控）

```kotlin
/**
 * 定期检查所有可监控商品的价格变化
 *
 * 触发频率: 每 6 小时（最短 15 分钟，由系统弹性调度）
 * 约束条件: WiFi 网络 + 设备充电中
 */
@HiltWorker
class PriceCheckWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val checkPriceUseCase: CheckPriceUseCase,
    private val notificationHelper: PriceDropNotificationHelper,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // 1. 从输入获取参数（可选）
        val targetPlatform = inputData.getString("platform")?.let { Platform.valueOf(it) }

        return try {
            // 2. 执行价格检查
            val priceDrops = checkPriceUseCase(targetPlatform)

            // 3. 处理降价商品（推送通知）
            if (priceDrops.isNotEmpty()) {
                notificationHelper.showPriceDropNotification(priceDrops)
            }

            // 4. 返回结果
            val outputData = workDataOf(
                "checked_count" to priceDrops.size,
                "drops_found" to priceDrops.count { it.isDropped },
            )
            Result.success(outputData)

        } catch (e: Exception) {
            // 5. 根据异常类型决定是否重试
            if (runAttemptCount < 3 && e is IOException) {
                Result.retry()  // 网络异常，稍后重试
            } else {
                Result.failure()  // 其他异常，放弃
            }
        }
    }
}
```

### 基础 OneTime Worker（降价推送）

```kotlin
/**
 * 推送降价通知给用户
 * 由 PriceCheckWorker 完成后通过 WorkContinuation 触发
 */
@HiltWorker
class PriceDropNotificationWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val notificationManager: NotificationManagerCompat,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val itemId = inputData.getString("item_id") ?: return Result.failure()
        val title = inputData.getString("title") ?: "商品"
        val oldPrice = inputData.getString("old_price")
        val newPrice = inputData.getString("new_price")

        notificationManager.showPriceDrop(itemId, title, oldPrice, newPrice)
        return Result.success()
    }
}
```

### 长时间运行 Worker（数据同步）

```kotlin
/**
 * 云端数据同步 Worker
 * 使用 setForeground() 确保系统不杀死同步任务
 */
@HiltWorker
class CloudSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncEngine: SyncEngine,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // 设置为前台任务（显示同步进行中的通知）
        setForeground(createForegroundInfo())

        return try {
            val result = syncEngine.sync()
            val outputData = workDataOf(
                "synced_count" to result.syncedCount,
                "conflicts" to result.conflicts.size,
            )
            Result.success(outputData)
        } catch (e: Exception) {
            Result.retry()
        }
    }

    private fun createForegroundInfo(): ForegroundInfo {
        val notification = NotificationCompat.Builder(applicationContext, CHANNEL_SYNC)
            .setContentTitle("正在同步数据")
            .setContentText("同步收藏数据到云端...")
            .setSmallIcon(R.drawable.ic_sync)
            .setOngoing(true)
            .build()
        return ForegroundInfo(SYNC_NOTIFICATION_ID, notification)
    }

    companion object {
        const val CHANNEL_SYNC = "sync_channel"
        const val SYNC_NOTIFICATION_ID = 1001
    }
}
```

---

## 三、约束条件配置

```kotlin
// ===== 价格检查：WiFi + 充电 =====
val priceCheckConstraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.UNMETERED)  // 仅 WiFi
    .setRequiresCharging(true)                       // 充电中
    .setRequiresBatteryNotLow(true)                  // 电量充足
    .build()

// ===== 数据清理：充电 + 设备空闲 =====
val cleanupConstraints = Constraints.Builder()
    .setRequiresCharging(true)
    .setRequiresDeviceIdle(true)    // API 23+，设备空闲
    .build()

// ===== 轻量同步：仅联网 =====
val syncConstraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)  // 任意网络
    .build()
```

---

## 四、任务调度

### 在 Application 中初始化

```kotlin
@HiltAndroidApp
class ShoppingHubApp : Application(), Configuration.Provider {

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setMinimumLoggingLevel(if (BuildConfig.DEBUG) Log.DEBUG else Log.ERROR)
            .build()

    override fun onCreate() {
        super.onCreate()
        schedulePeriodicTasks()
    }

    private fun schedulePeriodicTasks() {
        val workManager = WorkManager.getInstance(this)

        // 价格检查: 每 6 小时
        val priceCheckRequest = PeriodicWorkRequestBuilder<PriceCheckWorker>(
            repeatInterval = 6,         // 间隔 6 小时
            repeatIntervalTimeUnit = TimeUnit.HOURS,
            flexTimeInterval = 30,      // 弹性时间 30 分钟
            flexTimeIntervalUnit = TimeUnit.MINUTES,
        )
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.UNMETERED)
                    .setRequiresCharging(true)
                    .build()
            )
            .setBackoffCriteria(
                BackoffPolicy.EXPONENTIAL,
                WorkRequest.MIN_BACKOFF_MILLIS,
                TimeUnit.MILLISECONDS,
            )
            .build()

        // 唯一任务：避免重复调度
        workManager.enqueueUniquePeriodicWork(
            "price_check",
            ExistingPeriodicWorkPolicy.KEEP,  // 已有则保留
            priceCheckRequest,
        )

        // 云端同步: 每 24 小时
        val syncRequest = PeriodicWorkRequestBuilder<CloudSyncWorker>(
            repeatInterval = 24,
            repeatIntervalTimeUnit = TimeUnit.HOURS,
        )
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.UNMETERED)
                    .build()
            )
            .build()

        workManager.enqueueUniquePeriodicWork(
            "cloud_sync",
            ExistingPeriodicWorkPolicy.KEEP,
            syncRequest,
        )
    }
}
```

---

## 五、链式任务编排

```kotlin
// 场景: 价格检查 → 降价推送（串行链）
class PriceCheckManager @Inject constructor(
    private val workManager: WorkManager,
) {
    fun startPriceCheckWithNotification() {
        val checkRequest = OneTimeWorkRequestBuilder<PriceCheckWorker>()
            .setConstraints(Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
            )
            .build()

        val notifyRequest = OneTimeWorkRequestBuilder<PriceDropNotificationWorker>()
            .setInputData(
                workDataOf("item_id" to "...", "title" to "商品标题", "new_price" to "199")
            )
            .build()

        // 串行：先检查 → 再推送
        workManager
            .beginWith(checkRequest)
            .then(notifyRequest)
            .enqueue()
    }
}
```

---

## 六、观察任务状态

```kotlin
@Composable
fun SyncStatusIndicator(modifier: Modifier = Modifier) {
    val context = LocalContext.current
    val workManager = remember { WorkManager.getInstance(context) }

    val workInfo by workManager
        .getWorkInfosForUniqueWorkLiveData("cloud_sync")
        .observeAsState()

    val isRunning = workInfo?.any { it.state == WorkInfo.State.RUNNING } == true

    if (isRunning) {
        LinearProgressIndicator(modifier = modifier)
    }
}
```

---

## 七、电池优化引导

```kotlin
/**
 * 引导用户将 App 加入电池优化白名单
 * 确保后台任务不被系统杀死
 */
fun Context.requestBatteryOptimizationExemption() {
    val powerManager = getSystemService<PowerManager>()!!
    if (!powerManager.isIgnoringBatteryOptimizations(packageName)) {
        val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
            data = Uri.parse("package:$packageName")
        }
        startActivity(intent)
    }
}
```

---

## 八、输出格式

### 实现 Worker 时

给出完整的 Worker 类 + 约束配置 + 调度注册代码。

### 审查后台任务时

```
## 🔴 严重问题（任务被杀死、电池耗尽、死循环）
## 🟡 设计问题（约束不合理、重试策略不当、频率过高）
## 🔵 优化建议（延迟执行、合并任务、降低频率）
```
