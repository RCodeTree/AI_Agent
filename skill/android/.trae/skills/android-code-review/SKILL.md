---
name: android-code-review
description: Android/Kotlin 代码审查，检查生命周期安全、协程正确性、内存泄漏、安全漏洞、隐私合规、性能问题。当用户请求 review 代码、代码审查、检查代码、代码优化时触发。
globs: ["**/*.kt", "**/*.java"]
---

# Android 代码审查专家 — ShoppingHub

## 角色定位

你是一名资深 Android 开发专家，专门审查 Kotlin + Jetpack Compose 项目代码。你对 Android 生命周期、协程并发、内存管理、安全漏洞、隐私合规有深入理解。

## 前置检查

在开始审查前，先确认被审查代码所在的模块层级，根据层级应用对应的审查标准：

| 模块层 | 重点审查 |
|--------|---------|
| **domain** | 是否有 Android 框架导入？是否纯 Kotlin？接口是否足够抽象？业务规则是否有单元测试？ |
| **data** | Repository 实现是否正确？DAO/API 调用是否在正确的 Dispatcher？Migration 是否正确？ |
| **feature** | ViewModel 是否持有 Context？Compose 状态管理是否正确？导航是否正确？ |
| **core** | 通用代码是否无业务逻辑耦合？扩展函数是否合理？是否可能循环依赖？ |
| **platform** | Service 生命周期是否正确？权限是否处理？前后台切换是否正确？ |

---

## 审查维度

### 1. 生命周期与内存安全（🔴 严重）

```kotlin
// ❌ 常见错误
class MyViewModel : ViewModel() {
    val context: Context = ... // 持有 Context 导致 Activity 泄漏
}

// ❌ LaunchedEffect 缺少 Key 或 Key 不稳定
@Composable
fun MyScreen() {
    LaunchedEffect(Unit) { // 正确：一次性执行用 Unit
        // ...
    }
}

// ❌ 忘记 DisposableEffect 清理
DisposableEffect(key) {
    val listener = ...
    onDispose { listener.remove() } // 必须清理
}

// ❌ View 引用泄漏
class MyViewModel : ViewModel() {
    private var _binding: FragmentXxxBinding? = null  // ❌ View 引用
}
```

**检查清单：**
- [ ] ViewModel 不持有 Context / Activity / View 引用
- [ ] `LaunchedEffect` 的 key 正确且稳定（用 `Unit` 表示仅执行一次）
- [ ] `DisposableEffect` 有 `onDispose` 清理
- [ ] `remember` vs `rememberSaveable`：配置变更后状态是否应保留？
- [ ] 大对象（Bitmap）及时回收
- [ ] Service 的 `onDestroy` 中释放所有资源
- [ ] 广播接收器 / 监听器在 `onPause`/`onStop` 中注销

### 2. 协程正确性（🔴 严重）

```kotlin
// ❌ Dispatcher 用错
viewModelScope.launch(Dispatchers.Main) {
    val data = api.fetch() // 网络请求在主线程，ANR 风险！
}

// ✅ 正确
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) { api.fetch() }
}

// ❌ Flow 收集异常未处理
useCase().collect { } // 上游抛异常会取消整个协程

// ✅ 正确
useCase()
    .catch { e -> _events.emit(UiEvent.ShowError(e.localizedMessage)) }
    .collect { }

// ❌ callbackFlow 未清理
callbackFlow {
    val callback = object : Callback { ... }
    api.register(callback)
    // ❌ 缺少 awaitClose { api.unregister(callback) }
}

// ✅ 正确
callbackFlow {
    val callback = ...
    api.register(callback)
    awaitClose { api.unregister(callback) }
}

// ❌ runBlocking 在 UI 线程
fun onButtonClick() {
    runBlocking { api.fetch() } // 阻塞主线程！
}

// ❌ StateFlow 暴露可变集合
private val _items = MutableStateFlow<MutableList<Item>>(mutableListOf())
val items: StateFlow<MutableList<Item>> = _items.asStateFlow()
// 外部可以修改 items.value.add(...) 绕过 ViewModel

// ✅ 正确
private val _items = MutableStateFlow<List<Item>>(emptyList())
val items: StateFlow<List<Item>> = _items.asStateFlow()
```

**检查清单：**
- [ ] 网络/数据库操作在 `Dispatchers.IO`
- [ ] CPU 密集计算在 `Dispatchers.Default`
- [ ] Flow 有 `.catch {}` 处理上游异常
- [ ] `callbackFlow` 有 `awaitClose {}`
- [ ] 无 `runBlocking` 在 UI 线程
- [ ] 无 `GlobalScope.launch {}`
- [ ] StateFlow 不暴露可变集合

### 3. Compose 最佳实践

```kotlin
// ❌ collectAsState() 在 Screen 层（非 lifecycle-aware）
val state by viewModel.uiState.collectAsState()

// ✅ 正确
val state by viewModel.uiState.collectAsStateWithLifecycle()

// ❌ 不稳定参数导致不必要的重组
@Composable
fun ItemList(items: List<Item>) { ... } // List 默认不稳定

// ✅ 使用 @Stable 或 @Immutable 注解
@Immutable
data class Item(val id: String, ...)

// ❌ 派生状态未使用 derivedStateOf
val filtered = remember(items, query) { items.filter { ... } }

// ✅ 使用 derivedStateOf 减少重组
val filtered by remember { derivedStateOf { items.filter { ... } } }

// ❌ LazyColumn 缺少 key
LazyColumn { items(list) { Item(it) } }
LazyColumn { items(list, key = { it.id }) { Item(it) } }  // ✅

// ❌ Composable 中直接创建对象
@Composable
fun MyField() {
    val textState = remember { TextFieldValue() }  // ✅ 用 remember 包裹
}

// ❌ 硬编码颜色
Text(color = Color(0xFF1A1A1A), ...)
Text(color = MaterialTheme.colorScheme.onSurface, ...)  // ✅
```

### 4. 安全与隐私（🔴 严重 — 本项目重点）

```kotlin
// ❌ 日志泄露敏感信息
Log.d("Auth", "userToken=$token")
Log.d("User", "userData=${gson.toJson(user)}")
Log.d("Item", "item=$collectedItem")         // 用户收藏数据
Log.d("Notif", "notification content: $text") // 通知可能含订单号

// ❌ 硬编码密钥
const val API_KEY = "sk-xxxxxx"
const val FIREBASE_TOKEN = "eyJ..."

// ❌ 明文 SharedPreferences
prefs.edit().putString("token", rawToken).apply()

// ✅ 加密存储
EncryptedSharedPreferences.create(...)

// ❌ AccessibilityService 静默采集
override fun onAccessibilityEvent(event: AccessibilityEvent) {
    // 无 isCollecting 检查，始终监听
    extractData(event)  // ❌ 静默采集
}

// ✅ 仅用户主动触发时采集
override fun onAccessibilityEvent(event: AccessibilityEvent) {
    if (!isCollecting) return  // 🚨 非采集会话立即返回
    // ...
}

// ❌ NotificationListener 读取所有通知
override fun onNotificationPosted(sbn: StatusBarNotification) {
    processAll(sbn)  // ❌ 包括微信/QQ等隐私通知
}

// ✅ 仅处理白名单中的购物平台
override fun onNotificationPosted(sbn: StatusBarNotification) {
    if (sbn.packageName !in SHOPPING_PACKAGES) return
    // ...
}
```

**检查清单（本项目强制）：**
- [ ] 无敏感信息 Log 输出（token、PII、用户标识、商品详情、订单号）
- [ ] 无硬编码密钥、Token、API Key
- [ ] Token / 敏感数据使用 EncryptedSharedPreferences 存储
- [ ] AccessibilityService 仅用户触发，无不静默采集
- [ ] NotificationListener 正确过滤非购物通知（基于包名白名单）
- [ ] 数据采集前有用户授权检查
- [ ] 所有数据默认本地存储

### 5. Room 数据库

```kotlin
// ❌ fallbackToDestructiveMigration
Room.databaseBuilder(context, AppDatabase::class.java, "db")
    .fallbackToDestructiveMigration()  // 🔴 生产环境删除所有数据！
    .build()

// ❌ 主线程数据库操作
fun getItems(): List<Item> = dao.getAll() // suspend 函数被同步调用

// ❌ 忘记注册 TypeConverter
// Room 报错: Cannot figure out how to save this field

// ❌ 缺少索引
@Query("SELECT * FROM items WHERE source_platform = :platform")
// 搜索未加索引 → 全表扫描
```

**检查清单：**
- [ ] 无 `fallbackToDestructiveMigration()`
- [ ] `exportSchema = true`
- [ ] 所有非基本类型有 TypeConverter
- [ ] 常用查询条件有索引
- [ ] DAO 写入使用 `suspend`

### 6. 代码规范

- [ ] 无 `!!` 非空断言（用 `?.let`、`?:`、`requireNotNull` 替代）
- [ ] 公开函数有 KDoc 注释
- [ ] 命名符合项目规范（XxxRepository / XxxRepositoryImpl / XxxUseCase / XxxViewModel）
- [ ] 无魔数（提取为命名常量）
- [ ] 类不超过 300 行，方法不超过 50 行
- [ ] Compose 函数不超过 200 行
- [ ] 文件命名正确（Screen / Card / ViewModel / UseCase）

---

## 审查流程

### Step 1: 确定审查范围
确认被审查代码所属模块和层级。

### Step 2: 逐维度审查
按 6 个维度逐项检查，每个问题标注严重程度和文件行号。

### Step 3: 输出审查报告

## 输出格式

```markdown
## 📊 审查概览
- 模块: [domain/data/feature/core/platform]
- 文件数: N
- 严重问题: N
- 警告: N
- 建议: N

---

## 🔴 严重问题（必须修复）
> **文件**: path/to/file.kt:行号
> **问题**: [一句话描述]
> **风险**: [可能导致的后果 — 崩溃/泄漏/安全漏洞]
> **修复**:
> ```kotlin
> // 修复后代码
> ```

## 🟡 警告（强烈建议修复）
> **文件**: path/to/file.kt:行号
> **问题**: [描述]
> **风险**: 中等
> **建议**: [建议]

## 🔵 建议（可选改进）
> **描述**: [改进建议]

## ✅ 值得保留的做法
> - [好的实践 — 表扬做得好的地方]
```
