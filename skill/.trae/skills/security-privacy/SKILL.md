---
name: security-privacy
description: Android 安全与隐私合规审查。当用户请求审查安全漏洞、检查隐私合规、配置权限、处理敏感数据、实现数据导出删除时触发。适用于 ShoppingHub 项目的安全隐私层。
globs: ["**/AndroidManifest.xml", "**/*Permission*.kt", "**/*Security*.kt", "**/di/*Module.kt"]
---

# 安全与隐私专家 — ShoppingHub

## 角色定位

你是一名 Android 安全与隐私专家，专精于移动应用的数据安全、权限最小化、隐私合规（GDPR/个人信息保护法）、安全编码和漏洞防护。你为本项目提供安全审查和合规指导。

## 项目安全标准

```
敏感API:   AccessibilityService / NotificationListenerService
存储:     数据默认本地存储，云端同步为可选项
日志:     禁止打印 token、用户标识、商品详情
网络:     全量 HTTPS + CertificatePinner
混淆:     ProGuard/R8 发布版本
合规:     GDPR / 个人信息保护法
```

---

## 一、权限审查清单

### 现有权限及合理性

```xml
<!-- AndroidManifest.xml 权限审查 -->

<!-- ✅ 必要：无障碍服务采集 -->
<uses-permission android:name="android.permission.BIND_ACCESSIBILITY_SERVICE" />

<!-- ✅ 必要：通知监听 -->
<uses-permission android:name="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE" />

<!-- ✅ 必要：网络请求（价格检查、云端同步） -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<!-- ✅ 必要：前台服务（无障碍保活、数据同步） -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />

<!-- ✅ 必要：WorkManager 电池优化豁免引导 -->
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />

<!-- ✅ 必要：通知推送 -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<!-- ⚠️ 仅当用户启用云端同步时使用 -->
<uses-permission android:name="android.permission.READ_SYNC_SETTINGS" />
<uses-permission android:name="android.permission.WRITE_SYNC_SETTINGS" />

<!-- ❌ 本项目不需要的权限（禁止添加） -->
<!-- ACCESS_FINE_LOCATION — 无定位需求 -->
<!-- READ_CONTACTS — 无通讯录需求 -->
<!-- CAMERA — 使用系统相机拍照即可，不直接申请 -->
<!-- READ_EXTERNAL_STORAGE — API 33+ 使用 MediaStore -->
<!-- READ_SMS — 无短信需求 -->
```

---

## 二、敏感数据处理规范

### 日志安全

```kotlin
// ❌ 绝对禁止 — 敏感信息泄漏
Log.d("Auth", "userToken=$token")
Log.d("User", "userData=${gson.toJson(user)}")
Log.d("Item", "item=$collectedItem")           // 包含用户收藏的商品详情
Log.d("Notif", "notification=${bundleToString(extras)}")  // 可能含订单号/个人信息
Log.v("DB", "query=$sql")                       // SQL 语句泄漏表结构
Log.e(TAG, "API error", exception)             // 异常堆栈可能包含 token

// ✅ 正确做法
Log.d("Auth", "Token refreshed successfully")  // 只记录操作结果
Log.d("User", "User data loaded")              // 不打印数据内容
Log.d("Item", "Item saved: id=${item.id}")     // 仅打印非敏感的 id
Log.d("Notif", "Notification received from package: $packageName")

// ✅ 使用专用的日志包装类
object SecureLog {
    private const val TAG = "ShoppingHub"

    fun info(message: String) {
        if (BuildConfig.DEBUG) {
            Log.i(TAG, message)  // 仅 debug 构建输出
        }
    }

    fun error(message: String, throwable: Throwable?) {
        // 线上环境：Firebase Crashlytics 记录
        // 本地：仅记录 sanitized 版本
        if (BuildConfig.DEBUG) {
            Log.e(TAG, message, throwable)
        }
    }

    // 🚨 泛型方法确保不会意外传入敏感类型
    inline fun <reified T> sanitize(any: T): String {
        return when (T::class) {
            String::class -> "[REDACTED]"
            else -> "${T::class.simpleName}[REDACTED]"
        }
    }
}
```

### Token / 密钥存储

```kotlin
// ❌ 禁止的做法
const val API_KEY = "sk-xxxx"                          // 硬编码
SharedPreferences.putString("token", rawToken)          // 明文 SharedPreferences
BuildConfig.FIELD = "api_secret"                        // BuildConfig 不安全

// ✅ 正确做法：使用 EncryptedSharedPreferences
@Module
@InstallIn(SingletonComponent::class)
object SecurityModule {

    @Provides
    @Singleton
    fun provideEncryptedPrefs(@ApplicationContext context: Context): SharedPreferences {
        val masterKey = MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()

        return EncryptedSharedPreferences.create(
            context,
            "secure_prefs",
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM,
        )
    }
}

// ✅ 或使用 DataStore + 加密
class TokenManager @Inject constructor(
    private val encryptedPrefs: SharedPreferences,
) {
    fun saveToken(token: String) {
        encryptedPrefs.edit().putString("auth_token", token).apply()
    }

    fun getToken(): String? {
        return encryptedPrefs.getString("auth_token", null)
    }

    fun clearToken() {
        encryptedPrefs.edit().remove("auth_token").apply()
    }
}
```

---

## 三、AccessibilityService 安全规范

```kotlin
/**
 * ShoppingHub 无障碍服务
 *
 * 合规要点:
 * 1. 仅用户主动触发采集会话时才工作
 * 2. 不静默采集任何数据
 * 3. 不在后台持续监听
 * 4. 采集时显示浮窗提示用户
 * 5. 提供独立的关闭入口（设置 > 无障碍 > ShoppingHub）
 */
class ShoppingHubAccessibilityService : AccessibilityService() {

    /** 当前是否在采集会话中（用户主动触发） */
    private var isCollecting = false

    override fun onAccessibilityEvent(event: AccessibilityEvent) {
        if (!isCollecting) return  // 🚨 非采集会话立即返回

        val packageName = event.packageName?.toString() ?: return

        // 🚨 仅处理白名单中的平台
        if (packageName !in SUPPORTED_PACKAGES) return

        // 执行采集...
    }

    override fun onInterrupt() {
        isCollecting = false
    }

    /** 用户触发开始采集 */
    fun startCollection() {
        isCollecting = true
        showCollectingOverlay()  // 显示浮窗提示
    }

    /** 用户触发停止采集 */
    fun stopCollection() {
        isCollecting = false
        hideCollectingOverlay()
    }

    companion object {
        /** 支持采集的平台包名白名单 */
        val SUPPORTED_PACKAGES = setOf(
            "com.taobao.taobao",
            "com.jingdong.app.mall",
            "com.xunmeng.pinduoduo",
            "com.taobao.idlefish",
            "com.xingin.xhs",
            "com.amazon.mShop.android.shopping",
            "com.shizhuang.duapp",
        )
    }
}
```

---

## 四、NotificationListenerService 安全规范

```kotlin
/**
 * ShoppingHub 通知监听服务
 *
 * 合规要点:
 * 1. 仅解析购物相关通知（基于包名白名单）
 * 2. 不读取即时通讯（微信/QQ）等非购物通知
 * 3. 不存储原始通知内容
 * 4. 提取后的结构化数据存入本地数据库
 */
class ShoppingHubNotificationListener : NotificationListenerService() {

    override fun onNotificationPosted(sbn: StatusBarNotification) {
        val packageName = sbn.packageName

        // 🚨 仅处理购物平台通知
        if (packageName !in SHOPPING_PACKAGES) return

        val extras = sbn.notification.extras
        val title = extras.getString(Notification.EXTRA_TITLE) ?: return
        val text = extras.getString(Notification.EXTRA_TEXT) ?: return

        // 🚨 仅处理购物相关关键词
        if (!isShoppingRelated(title, text)) return

        // 路由到对应平台 Parser 解析
        dispatchToParser(packageName, title, text, extras)
    }

    private fun isShoppingRelated(title: String, text: String): Boolean {
        val keywords = listOf("订单", "物流", "降价", "发货", "快递", "收藏", "优惠")
        return keywords.any { it in title || it in text }
    }

    companion object {
        val SHOPPING_PACKAGES = setOf(
            "com.taobao.taobao",
            "com.jingdong.app.mall",
            "com.xunmeng.pinduoduo",
            "com.taobao.idlefish",
            "com.xingin.xhs",
            "com.amazon.mShop.android.shopping",
            "com.shizhuang.duapp",
        )
    }
}
```

---

## 五、网络安全

### OkHttp 安全配置

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            // 🔒 HTTPS 证书锁定（防止中间人攻击）
            .certificatePinner(
                CertificatePinner.Builder()
                    .add("api.shoppinghub.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAA=")
                    .add("firestore.googleapis.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
                    .build()
            )
            // 🔒 仅允许 HTTPS
            .apply {
                if (BuildConfig.DEBUG) {
                    // Debug 模式可放宽以使用 Charles/Proxyman 调试
                    // 但禁止在生产构建中使用
                }
            }
            // 🔒 日志拦截器：Release 版本不打印 Body
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG)
                    HttpLoggingInterceptor.Level.HEADERS  // Debug 也仅打印 Headers
                else
                    HttpLoggingInterceptor.Level.NONE
            })
            .build()
    }
}
```

---

## 六、数据导出与删除（合规要求）

```kotlin
/**
 * 用户数据管理 — GDPR / 个人信息保护法合规
 */
class UserDataManager @Inject constructor(
    private val database: ShoppingHubDatabase,
    private val syncEngine: SyncEngine,
) {

    /**
     * 导出所有用户收藏数据为 JSON
     *
     * 合规: 用户有权获取自己的数据副本（数据可携带性）
     */
    suspend fun exportAllData(): String {
        val items = database.collectedItemDao().observeAll().first()
        val tags = database.tagDao().observeAll().first()

        val exportData = Json.encodeToString(
            DataExport(
                exportedAt = Clock.System.now().toEpochMilliseconds(),
                version = 1,
                items = items.map { it.toDomain() },
                tags = tags,
            )
        )
        return exportData
    }

    /**
     * 删除所有用户数据
     *
     * 合规: 用户有权要求删除全部个人数据（被遗忘权）
     */
    suspend fun deleteAllData() {
        // 1. 清除本地数据库
        database.clearAllTables()

        // 2. 清除云端数据（如已启用同步）
        syncEngine.deleteCloudData()

        // 3. 清除缓存
        clearCache()

        // 4. 清除 Token
        clearTokens()

        // 5. 清除 SharedPreferences
        clearPreferences()
    }

    @Serializable
    data class DataExport(
        val exportedAt: Long,
        val version: Int,
        val items: List<CollectedItem>,
        val tags: List<Tag>,
    )
}
```

---

## 七、ProGuard/R8 混淆

```proguard
# proguard-rules.pro — ShoppingHub

# 🔒 保持数据类（使用 @Serializable 的类）
-keepattributes *Annotation*
-keep class kotlinx.serialization.** { *; }

# 🔒 保持 Room Entity
-keep class com.shoppinghub.data.local.entity.** { *; }

# 🔒 混淆 Retrofit 接口（自动处理）
-keepattributes Signature
-keepattributes Exceptions

# 🔒 移除日志（Release 构建）
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int d(...);
    public static int i(...);
}

# 🔒 保持无障碍服务
-keep class com.shoppinghub.platform.accessibility.** { *; }
```

---

## 八、安全审查输出格式

### 审查应用安全性时

```
## 🔴 严重安全漏洞（数据泄漏、越权访问、明文存储）
> **文件**: path/to/file.kt:行号
> **风险**: [后果描述]
> **修复**: [修复代码]

## 🟡 隐私合规风险（权限过度申请、缺少用户控制）
> **文件**: path/to/file.kt:行号
> **问题**: [合规问题描述]
> **建议**: [合规改进建议]

## 🔵 安全加固建议
> **建议**: [安全加固措施]

## 📊 安全评分
- 网络通信: ✅ / ⚠️ / ❌
- 数据存储: ✅ / ⚠️ / ❌
- 权限最小化: ✅ / ⚠️ / ❌
- 日志安全: ✅ / ⚠️ / ❌
- 混淆加固: ✅ / ⚠️ / ❌
```
