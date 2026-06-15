# ShoppingHub — Android 项目规则（Trae 专用）

## 项目身份

**ShoppingHub** 是一款跨平台购物收藏聚合 Android 原生应用，使用 Kotlin 开发。核心功能是将用户分散在淘宝、京东、拼多多、闲鱼、小红书、亚马逊、得物等平台的收藏夹、购物车、已购订单、愿望清单统一汇集管理。

---

## 一、技术栈（不可自行变更）

| 层级 | 技术方案 | 红线 |
|------|----------|------|
| 语言 | **Kotlin 2.0+** | 禁止 Java 为主语言，仅允许底层模块 Java/Kotlin 混编 |
| UI | **Jetpack Compose + Material Design 3** | 禁止 XML View 体系，禁止混用 View + Compose |
| 架构 | **MVVM + Clean Architecture** | 严格分层，依赖方向不可逆 |
| DI | **Hilt** (Dagger 封装) | 禁止手动构造依赖、禁止 Koin 等其他 DI 框架 |
| 数据库 | **Room** (KSP 编译器) | 禁止直接使用 SQLiteOpenHelper |
| 网络 | **Retrofit + OkHttp + Kotlin Serialization** | 禁止 Gson、FastJson |
| 图片 | **Coil** (Compose 原生) | 禁止 Glide、Picasso |
| 异步 | **Kotlin Coroutines + Flow** | 禁止 RxJava、AsyncTask、Thread 裸用 |
| 后台 | **WorkManager** | 禁止 AlarmManager、JobScheduler 裸用 |
| 导航 | **Compose Navigation** (类型安全路由) | 禁止 Fragment 导航 |
| OCR | **ML Kit Text Recognition** | Google 官方方案 |
| 云同步 | **Firebase Firestore** (可选) | — |
| 崩溃收集 | **Firebase Crashlytics** | — |
| 构建 | **Gradle KTS + Version Catalog** | 禁止 Groovy 构建脚本、禁止硬编码依赖版本 |

---

## 二、版本约束

| 配置项 | 设定值 | 说明 |
|--------|--------|------|
| `minSdk` | **26** (Android 8.0) | 覆盖 95%+ 活跃设备 |
| `compileSdk` | **35** (Android 15) | 编译时适配最新 SDK |
| `targetSdk` | **35** | Google Play 上架要求 |
| `kotlin` | **2.0+** | K2 编译器（稳定） |
| `AGP` | **8.7+** | Android Gradle Plugin |
| `compose-bom` | **2025.x** | 通过 BOM 统一管理 Compose 版本 |

### 关键兼容性处理

- **NotificationListenerService** — API 19+ 支持，本项目 minSdk 26 完全覆盖
- **AccessibilityService** — 需适配各厂商 ROM 权限引导页（小米/OPPO/vivo/华为各有不同）
- **Picture-in-Picture** — API 26+ 刚好覆盖
- **快捷方式(Shortcuts)** — API 26+
- **存储访问** — 使用 SAF (Storage Access Framework) 替代传统 File API

---

## 三、Clean Architecture 分层依赖（强制规则）

```
app → feature/* → domain → core
              ↘ data → domain → core
              ↘ platform → core
```

### 各层约束

| 层 | 职责 | 约束 | 允许的依赖 |
|----|------|------|-----------|
| **domain** | 业务核心 | **纯 Kotlin 模块，零 Android 框架依赖**。不可出现 `android.*`、`Context`、`ViewModel`、`LiveData`、`androidx.*` | 仅 Kotlin 标准库 + Coroutines |
| **data** | 数据协调 | 实现 domain 定义的 Repository 接口 | domain、core |
| **feature** | UI + 交互 | **只能依赖 domain**，不可直接依赖 data 层的具体实现类 | domain、core-ui、core-common |
| **core** | 基础能力 | 各子模块按需单向依赖，不可形成循环 | 同层其他 core 子模块（需单向） |
| **app** | 组装入口 | 唯一可依赖所有模块的入口，负责 DI 组装、Application 初始化 | 所有模块 |
| **platform** | 系统服务 | 封装 Android 系统级 API（无障碍、通知、分享） | core |

### 模块详细索引

```
项目根目录/
├── app/                          # Application + MainActivity + DI 入口
│   └── src/main/java/com/shoppinghub/
│       ├── ShoppingHubApp.kt     # @HiltAndroidApp
│       ├── MainActivity.kt       # 单 Activity 入口（@AndroidEntryPoint）
│       └── di/                   # Hilt DI 模块（AppModule、DatabaseModule、NetworkModule、RepositoryModule）
│
├── core/                         # 核心基础层（不依赖任何业务模块）
│   ├── core-common/              # 通用工具、扩展函数、常量、Result 封装、CoroutineDispatchers
│   ├── core-network/             # Retrofit/OkHttp 配置、拦截器链、AuthInterceptor
│   ├── core-data/                # Room 基类、数据源接口、SyncableDao
│   ├── core-ui/                  # 主题（ShoppingHubTheme）、通用 Composable（LoadingIndicator、ErrorView、EmptyView）
│   ├── core-model/               # 通用数据模型（跨模块共享）、Platform 枚举
│   └── core-datastore/           # DataStore 偏好存储封装（用户设置、Token）
│
├── data/                         # 数据层
│   ├── data-local/               # Room DAO、Entity、Database、Migration
│   ├── data-remote/              # 各平台 API 接口（Retrofit Service）、DTO
│   └── data-repository/          # Repository 实现（协调 local + remote，离线优先）
│
├── domain/                       # 领域层（纯 Kotlin，无 Android 依赖）
│   ├── domain-model/             # 领域模型、值对象、Platform/ItemCategory/ItemStatus 枚举
│   ├── domain-usecase/           # UseCase（一个类一个用例，operator fun invoke）
│   └── domain-repository/        # Repository 接口定义（Flow 查询 + suspend 写入）
│
├── feature/                      # 功能模块（按业务领域拆分）
│   ├── feature-home/             # 首页仪表盘（平台统计卡片、降价汇总、最近添加）
│   ├── feature-collection/       # 收藏管理（瀑布流列表、详情页、编辑、删除、批量操作）
│   ├── feature-cart/             # 购物车聚合
│   ├── feature-order/            # 已购订单
│   ├── feature-wishlist/         # 愿望清单
│   ├── feature-price-alert/      # 降价提醒（阈值设置、提醒列表）
│   ├── feature-search/           # 全局搜索（全文搜索 + 多维度筛选）
│   ├── feature-settings/         # 设置（权限管理、同步设置、关于）
│   ├── feature-onboarding/       # 新手引导
│   └── feature-import/           # 数据导入（多渠道采集入口：分享/无障碍/手动/OCR）
│
├── platform/                     # 平台适配（系统服务）
│   ├── platform-accessibility/   # AccessibilityService（批量采集）
│   ├── platform-notification/    # NotificationListenerService（订单/物流/降价通知）
│   └── platform-share/           # ShareIntent 接收（IntentFilter ACTION_SEND）
│
├── sync/                         # 同步引擎
│   └── sync-engine/              # 云端同步（Firebase Firestore）、冲突解决（LWW + 用户介入）
│
└── build-logic/                  # 构建逻辑
    └── convention/               # 自定义 Gradle Convention Plugins
```

---

## 四、Kotlin 编码规范

### 命名约定

```kotlin
// ===== 类命名 =====
// Repository 接口（domain-repository 层）
interface CollectionRepository { }

// Repository 实现（data-repository 层）
class CollectionRepositoryImpl @Inject constructor(
    private val dao: CollectedItemDao,
    private val api: ShoppingApi,
) : CollectionRepository { }

// UseCase（domain-usecase 层）— 一个 UseCase 只做一件事
class ObserveCollectionUseCase @Inject constructor(...)
class AddToFavoritesUseCase @Inject constructor(...)
class DeleteItemUseCase @Inject constructor(...)
class SearchItemsUseCase @Inject constructor(...)

// ViewModel（feature 层）
@HiltViewModel
class CollectionViewModel @Inject constructor(...) : ViewModel() { }

// Room Entity（data-local 层）
@Entity(tableName = "collected_items")
data class CollectedItemEntity(...)

// 网络 DTO（data-remote 层）
@Serializable
data class ProductDetailDto(...)

// ===== 文件命名 =====
// ViewModel:            XxxViewModel.kt
// 全屏页面:              XxxScreen.kt
// Composable 组件:       XxxCard.kt / XxxRow.kt / XxxChip.kt / XxxSheet.kt
// UseCase:              动词+名词.kt（如 ImportFromShareUseCase.kt、CheckPriceDropUseCase.kt）
// Repository 接口:       XxxRepository.kt
// Repository 实现:       XxxRepositoryImpl.kt
// Room DAO:              XxxDao.kt
// Room Entity:           XxxEntity.kt
// Hilt Module:           XxxModule.kt
// Platform Parser:       XxxPlatformParser.kt
```

### 安全编码（红线）

```kotlin
// ❌ 禁止 — 会直接拒绝代码
val name = user!!.name                    // 禁止 !! 非空断言操作符
GlobalScope.launch { }                    // 禁止 GlobalScope（永不取消，内存泄漏）
runBlocking { }                           // 禁止在主线程用 runBlocking（阻塞 UI，ANR）
Log.d("TAG", "token=$userToken")          // 禁止日志打印敏感信息（token、PII、用户数据）
const val API_KEY = "sk-xxxxxx"           // 禁止硬编码密钥 / Token
MutableStateFlow<MutableList<T>>(...)     // 禁止 StateFlow 暴露可变集合
viewModelScope.launch(Dispatchers.IO) { } // 禁止在 ViewModel 中手动切换 Dispatcher（用 withContext）

// ✅ 正确
val name = user?.name ?: return            // 安全调用 + 提前返回
viewModelScope.launch { }                 // 使用恰当的 CoroutineScope
withContext(Dispatchers.IO) { }           // 在 suspend 函数内切换调度器
// 敏感信息不记录到日志
```

### 扩展函数优先

```kotlin
// ❌ 工具类模式（Java 风格）
object PriceUtils {
    fun formatPrice(price: BigDecimal?): String { ... }
    fun isPriceDrop(old: BigDecimal?, new: BigDecimal?): Boolean { ... }
}

// ✅ Kotlin 扩展函数
fun BigDecimal?.toPriceString(): String = this?.let {
    "¥${it.setScale(2, RoundingMode.HALF_UP)}"
} ?: "价格未知"

fun BigDecimal?.isLowerThan(other: BigDecimal?): Boolean =
    this != null && other != null && this < other
```

### Kotlin 惯用写法

```kotlin
// ✅ when 表达式（密封类必须穷举）
val text = when (platform) {
    Platform.TAOBAO -> "淘宝"
    Platform.JD -> "京东"
    Platform.PINDUODUO -> "拼多多"
    Platform.XIANYU -> "闲鱼"
    Platform.XIAOHONGSHU -> "小红书"
    Platform.AMAZON -> "亚马逊"
    Platform.DEWU -> "得物"
    Platform.OTHER -> "其他"
}

// ✅ data class + copy()
val updated = state.copy(
    items = state.items + newItem,
    isLoading = false,
)

// ✅ 作用域函数按语义使用
// let: 非空执行        — value?.let { process(it) }
// apply: 对象配置      — OkHttpClient.Builder().apply { ... }.build()
// run: 计算并返回      — val result = run { ... }
// also: 副作用         — .also { Log.d(TAG, "processed: $it") }

// ✅ 延迟加载
private val json: Json by lazy {
    Json { ignoreUnknownKeys = true; coerceInputValues = true }
}
```

---

## 五、Compose UI 规范

### 页面架构模板

```kotlin
/**
 * [页面功能描述]
 *
 * @param onNavigateToDetail 导航回调
 * @param modifier 外部的 Modifier
 */
@Composable
fun XxxScreen(
    onNavigateToDetail: (String) -> Unit,
    modifier: Modifier = Modifier,
    viewModel: XxxViewModel = hiltViewModel(),
) {
    // 1. 收集 UI 状态（必须使用 lifecycle-aware 方法）
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // 2. 处理一次性事件（SnackBar、导航等）
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is XxxEvent.ShowError -> { /* SnackBar */ }
                is XxxEvent.NavigateToDetail -> onNavigateToDetail(event.id)
            }
        }
    }

    // 3. 渲染 UI（委托给纯 UI Composable）
    XxxContent(
        state = uiState,
        onAction = { viewModel.handleAction(it) },
        modifier = modifier,
    )
}

/**
 * 纯 UI 内容组件（无 ViewModel 依赖，可独立 @Preview）
 */
@Composable
private fun XxxContent(
    state: XxxUiState,
    onAction: (XxxAction) -> Unit,
    modifier: Modifier = Modifier,
) {
    when {
        state.isLoading -> LoadingIndicator(modifier = modifier)
        state.error != null -> ErrorView(
            message = state.error,
            onRetry = { onAction(XxxAction.Retry) },
            modifier = modifier,
        )
        state.isEmpty -> EmptyView(
            icon = Icons.Outlined.BookmarkBorder,
            message = "暂无收藏",
            modifier = modifier,
        )
        else -> XxxList(
            items = state.items,
            onItemClick = { onAction(XxxAction.ItemClick(it)) },
            modifier = modifier,
        )
    }
}

// ===== 预览（必须同时包含 Light 和 Dark 模式） =====
@Preview(name = "Light Mode", showBackground = true)
@Preview(name = "Dark Mode", uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Composable
private fun XxxScreenPreview() {
    ShoppingHubTheme {
        XxxContent(
            state = XxxUiState.preview(),
            onAction = {},
        )
    }
}
```

### 强制规则

| 规则 | 说明 |
|------|------|
| **禁止硬编码颜色** | 必须使用 `MaterialTheme.colorScheme.xxx`，不可写 `Color(0xFF...)` |
| **禁止硬编码字体** | 必须使用 `MaterialTheme.typography.xxx`，不可写 `fontSize = 16.sp` |
| **禁止硬编码形状** | 必须使用 `MaterialTheme.shapes.xxx`，不可写 `RoundedCornerShape(12.dp)` |
| **深色模式适配** | 所有页面必须在深色模式下可读，Preview 同时包含 Light/Dark |
| **collectAsStateWithLifecycle** | Screen 层必须使用，不可用 `collectAsState()` |
| **LazyColumn/LazyGrid 必须设 key** | `items(list, key = { it.id })` |
| **单 Composable ≤ 200 行** | 超出必须拆分为多个子 Composable |
| **contentDescription** | 所有 Icon/Image 必须设置（装饰性图片用 `null`） |
| **Touch target ≥ 48dp** | Material Design 无障碍标准 |
| **Modifier 作为第一个可选参数** | `fun MyComposable(modifier: Modifier = Modifier, ...)` |
| **Lambda 参数用 remember 包裹** | `val onClick = remember { { id: String -> ... } }` |
| **@Immutable / @Stable 注解** | 所有 Composable 使用的 data class 必须加 `@Immutable` |
| **derivedStateOf** | 从高频变化的状态中派生值，减少重组 |

### 性能规则

```kotlin
// ✅ 列表性能
LazyVerticalStaggeredGrid(
    columns = StaggeredGridCells.Fixed(2),
    ...
) {
    items(
        items = items,
        key = { it.id },               // 🚨 必须有稳定 Key
    ) { item ->
        ItemCard(
            item = item,
            onClick = { onItemClick(item.id) },
            modifier = Modifier.animateItem(),  // 列表动画
        )
    }
}

// ✅ 避免不必要的重组：用 @Immutable 标注数据类
@Immutable
data class CollectedItemUi(
    val id: String,
    val title: String,
    val price: String,
    val platform: Platform,
    val imageUrl: String?,
)
```

---

## 六、协程与 Flow 规范

### Dispatcher 选择

| 操作类型 | Dispatcher |
|----------|-----------|
| 网络请求 (Retrofit) | `Dispatchers.IO` |
| Room 数据库操作 | `Dispatchers.IO` |
| 文件读写 | `Dispatchers.IO` |
| JSON 解析 (大量数据) | `Dispatchers.Default` |
| 图片处理 / Bitmap | `Dispatchers.Default` |
| 复杂计算 / 排序 / 过滤 | `Dispatchers.Default` |
| UI 更新 | `Dispatchers.Main`（flowOn 自动切换） |

### ViewModel 模板

```kotlin
@HiltViewModel
class MyViewModel @Inject constructor(
    private val observeItems: ObserveItemsUseCase,
    private val addItem: AddItemUseCase,
    private val deleteItem: DeleteItemUseCase,
) : ViewModel() {

    data class UiState(
        val items: List<ItemUi> = emptyList(),
        val isLoading: Boolean = false,
        val error: String? = null,
    )

    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun handleAction(action: UiAction) {
        when (action) {
            is UiAction.Add -> addItem(action.item)
            is UiAction.Delete -> deleteItem(action.id)
            is UiAction.Refresh -> refresh()
        }
    }

    private fun addItem(item: ItemUi) {
        viewModelScope.launch {
            try {
                addItem(item.toDomain())
                _events.emit(UiEvent.ShowSnackbar("添加成功"))
            } catch (e: Exception) {
                _events.emit(UiEvent.ShowError(e.localizedMessage))
            }
        }
    }
}

sealed interface UiAction {
    data class Add(val item: ItemUi) : UiAction
    data class Delete(val id: String) : UiAction
    data object Refresh : UiAction
}

sealed interface UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent
    data class ShowError(val message: String) : UiEvent
    data class NavigateTo(val route: String) : UiEvent
}
```

### Flow 规则

- **Repository 查询** 返回 `Flow<List<T>>`（Room 原生支持）
- **Repository 写入** 使用 `suspend` 函数
- **StateFlow** 用于 UI 状态（始终有值，自动去重）
- **SharedFlow** 用于一次性事件（SnackBar、导航、Toast）
- **callbackFlow { }** 内必须调用 `awaitClose { }` 释放资源
- **多个 Flow 组合** 使用 `combine()` / `flatMapLatest()`，不用嵌套 collect
- **异常处理**：Repository 层传播异常，ViewModel 层 `.catch {}` 转换为用户可读消息
- **禁止** `GlobalScope.launch {}`、`runBlocking {}`、`MutableStateFlow<MutableList<T>>`

---

## 七、Room 数据库规范

- **禁止 `fallbackToDestructiveMigration()`**：生产环境必须提供 Migration
- **`exportSchema = true`**：导出 Schema 以便生成 Migration 和追踪变更
- **DAO 查询** 返回 `Flow<List<T>>` 以支持响应式更新
- **DAO 写入** 使用 `suspend` 函数
- **TypeConverter** 必须注册所有非基本类型字段（枚举、BigDecimal、List、复杂对象）
- **外键关系** 必须通过 `ForeignKey` 注解声明
- **索引** 为常用查询条件建立单列或复合索引
- **时间存储** 统一使用 `Long`（epoch millis），不存 `Date` 或 `Instant`
- **Migration** 每个版本变更一个 Migration 对象（`MIGRATION_N_N+1`），不得修改已发布的 Migration

---

## 八、安全与隐私（合规红线）

1. **所有数据采集功能** 必须用户明确授权后才可开启，并提供独立的关闭入口
2. **AccessibilityService** 不可静默采集，仅用户主动触发采集会话时工作
3. **NotificationListenerService** 仅解析购物相关通知（基于包名白名单过滤），忽略其他
4. **所有数据** 默认仅本地存储，云端同步为可选项
5. **敏感信息**（token、用户标识、商品详情、个人信息）禁止写入 Log
6. **禁止硬编码** 密钥、Token、API Key — 使用环境变量或安全存储
7. **网络通信** 全量 HTTPS，OkHttp 配置 certificatePinner
8. **提供完整的数据导出与删除功能** — 符合 GDPR/个人信息保护法
9. **隐私政策** 中清晰说明数据采集范围、使用目的、存储方式
10. **ProGuard/R8** 混淆发布版本，防止逆向工程

---

## 九、测试标准

| 层 | 测试类型 | 框架 | 覆盖率要求 |
|----|----------|------|-----------|
| **domain** | 单元测试 | JUnit5 + MockK | **100%** |
| **data** | Repository 单元测试 + DAO 集成测试 | JUnit5 + Room Testing + MockK | ≥ 80% |
| **feature** | ViewModel 单元测试 + Compose UI 测试 | JUnit5 + Compose Testing + Turbine | ≥ 70% |

- **Flow 测试** 使用 **Turbine** 库
- **Compose 测试** 使用 **Compose Testing** 库
- **测试命名**：``given<前置条件>_when<操作>_then<预期结果>``
- **测试模式**：Given-When-Then
- **禁止** 测试依赖网络（全部 Mock）、测试依赖执行顺序、测试间共享可变状态

---

## 十、Git 工作流

```
main                    # 生产分支（受保护，仅通过 PR 合入）
  └── develop           # 开发集成分支
       ├── feature/<module>/<desc>   # 功能分支，如 feature/feature-collection/waterfall-layout
       ├── fix/<issue>-<desc>        # 修复分支，如 fix/123-crash-on-import
       ├── refactor/<desc>           # 重构分支
       └── chore/<desc>              # 构建/CI/依赖更新
```

- **Commit 信息**：中文，格式 `<type>: <简述>`，如 `feat: 添加收藏夹瀑布流布局`、`fix: 修复无障碍服务在小米ROM上的崩溃`
- **Type**：`feat` / `fix` / `refactor` / `docs` / `test` / `chore` / `perf`
- **PR 合并前** 必须通过：lint（ktlint） + test（全量）+ build（assembleDebug）
- **禁止** force push 到 main/develop、直接 commit 到 main

---

## 十一、Trae AI 协作规则（重要）

### 使用 Trae 时的行为守则

1. **所有回答使用中文** — 技术术语可用英文，但解释和描述必须使用中文
2. **关键逻辑和容易产生歧义的地方添加中文注释** — 代码注释优先中文
3. **生成超过 30 行的类时，先考虑是否可以拆分或抽象** — 遵循单一职责原则
4. **禁止修改与当前任务无关的代码** — 不要顺便 "修复" 未要求的代码
5. **生成新代码前先确认该代码应放在哪个模块** — 参考本文件 §三 模块索引
6. **涉及架构决策时，先说明理由再编码** — 让开发者理解 "为什么"
7. **UI 代码生成后必须附带 `@Preview` 预览函数** — 同时包含 Light 和 Dark 模式
8. **所有公开函数必须添加 KDoc 注释** — 说明参数和返回值
9. **生成代码后，主动指出可能需要修改的地方** — 如 TODO、需要配置的常量、未实现的依赖
10. **当不确定 API 用法时，优先使用 Android 官方文档推荐方式**，而非猜测

### Trae Agent 调用 Skill 时的通用规则

- **遵循本文件 §一 的技术栈红线**，不得提议替代方案（如不要建议 "用 Flutter 会不会更好"）
- **遵循 §三 的分层约束**，生成的代码必须放在正确的模块
- **遵循 §四 的命名约定**，类名、文件名必须符合规范
- **遵循 §八 的安全红线**，不得生成违反隐私规则的代码

---

*文档版本：v2.0 | 编制日期：2026-06-15 | 状态：已评审*
