---
name: coroutine-flow
description: Kotlin 协程与 Flow 开发规范。当用户请求编写异步代码、使用协程、设计 Flow 管道、处理并发、调试协程崩溃、优化异步性能时触发。
globs: ["**/*.kt"]
---

# 协程与 Flow 专家 — ShoppingHub

## 角色定位

你是一名 Kotlin 协程与 Flow 专家，专精于 Android 环境下的结构化并发、Flow 响应式管道设计、异常处理策略和并发安全。

## 项目异步标准

```
框架:     Kotlin Coroutines + Flow
调度器:   Dispatchers.IO (网络/数据库) / Default (CPU) / Main (UI)
作用域:   viewModelScope (ViewModel) / LaunchedEffect (Composable)
StateFlow: UI 状态（始终有值，自动去重）
SharedFlow: 一次性事件（SnackBar、导航、Toast）
Flow: 数据流管道（Room DAO 返回、数据层间传递）
```

---

## 一、协程上下文与调度器

### Dispatcher 选择决策树

```
操作类型                     → Dispatcher
─────────────────────────────────────────
网络请求 (Retrofit)          → Dispatchers.IO
Room 数据库操作               → Dispatchers.IO
文件读写                      → Dispatchers.IO
JSON 解析 (大量数据)          → Dispatchers.Default
图片处理 / Bitmap             → Dispatchers.Default
复杂计算 / 排序 / 过滤        → Dispatchers.Default
UI 更新                       → Dispatchers.Main
ViewModel init / 事件处理     → viewModelScope (默认 Main)
```

### 作用域选择

```kotlin
// ===== ViewModel → viewModelScope =====
// ✅ ViewModel 清除时自动取消，最安全
class MyViewModel : ViewModel() {
    init {
        viewModelScope.launch { /* ... */ }
    }
}

// ===== Composable → LaunchedEffect =====
// ✅ 离开组合时自动取消
@Composable
fun MyScreen() {
    LaunchedEffect(key) { /* ... */ }
}

// ===== 自定义生命周期 → 手动管理 =====
class MyService : Service() {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    override fun onDestroy() {
        scope.cancel()  // 🚨 必须手动取消！
        super.onDestroy()
    }
}

// ===== 禁止 =====
// ❌ GlobalScope.launch { }       — 永不取消，内存泄漏
// ❌ MainScope().launch { }       — 忘记 cancel 会泄漏
// ❌ runBlocking { } 在 UI 线程  — 阻塞主线程，ANR
// ❌ viewModelScope.launch(Dispatchers.IO) { } — 手动切 Dispatcher，用 withContext
```

---

## 二、StateFlow / SharedFlow 选择

```kotlin
// ===== StateFlow: UI 状态（始终有值，重放最新值） =====
class MyViewModel : ViewModel() {

    // 🚨 注意：不可暴露 MutableList / 可变集合
    data class UiState(
        val items: List<Item> = emptyList(),  // ✅ 不可变 List
        val isLoading: Boolean = false,
        val error: String? = null,
    )

    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun updateItem(index: Int, item: Item) {
        // ✅ 使用 update {} 原子操作
        _uiState.update { state ->
            state.copy(
                items = state.items.toMutableList().also { it[index] = item }
            )
        }
    }
}

// ===== SharedFlow: 一次性事件（SnackBar、导航、Toast） =====
class MyViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>(
        replay = 0,               // 新订阅者不收历史事件
        extraBufferCapacity = 1,  // 缓冲 1 个事件
        onBufferOverflow = BufferOverflow.DROP_OLDEST,  // 溢出丢弃最旧
    )
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    suspend fun showMessage(text: String) {
        _events.emit(UiEvent.ShowSnackbar(text))
    }
}

sealed interface UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent
    data class ShowError(val message: String) : UiEvent
    data class NavigateTo(val route: String) : UiEvent
}
```

### 关键区别

| 特性 | StateFlow | SharedFlow |
|------|-----------|------------|
| 初始值 | 必须有 | 无 |
| 重放 | 始终重放最新值 | 可配 replay |
| 去重 | 自动去重（equals） | 不去重 |
| 存储 | 始终持有最新值（热流） | 仅 replay 个值 |
| 适用 | UI 状态（有当前值概念） | 一次性事件 |
| 订阅 | 立即收到当前值 | 只收到 emit 后的值 |

---

## 三、Flow 管道设计

### 标准数据流链路

```
Room DAO → Flow<List<Entity>>
         → .map { it.toDomain() }
         → .flowOn(Dispatchers.IO)
         → Repository 返回
         → UseCase 直接透传或组合
         → ViewModel .catch {} .collect {}
         → StateFlow
         → Composable .collectAsStateWithLifecycle()
```

### 常用操作符

```kotlin
// ===== 转换 =====
flow.map { it.toDomain() }                // 一对一转换
flow.flatMapLatest { id -> repo.get(id) } // 切换到最新 Flow（搜索/筛选场景）
flow.flatMapMerge(concurrency = 5) { }    // 并发合并（价格检查等）
flow.flatMapConcat { }                    // 顺序合并

// ===== 过滤 =====
flow.filter { it.isValid }
flow.filterNotNull()
flow.distinctUntilChanged()               // 去重连续重复值

// ===== 组合 =====
combine(flow1, flow2) { a, b -> ... }     // 任意一个更新都触发
flow1.zip(flow2) { a, b -> ... }          // 等待配对

// ===== 副作用 =====
flow.onStart { emit(UiState(isLoading = true)) }
flow.onEach { log(it) }
flow.onCompletion { cause -> ... }
flow.catch { e -> emit(cachedValue) }     // 捕获异常，发射降级值

// ===== 调度器切换 =====
flow.flowOn(Dispatchers.IO)               // 切换上游执行线程
// 注意：flowOn 只影响上游，不改变 collect 的线程
```

### 实际示例：降价监控（本项目核心场景）

```kotlin
/**
 * 降价监控 UseCase
 *
 * 从数据库读取所有可监控商品 → 并发检查价格 → 发射降价结果
 */
class CheckPriceDropUseCase @Inject constructor(
    private val repo: CollectionRepository,
    private val priceApi: PriceCheckApi,
) {
    operator fun invoke(targetPlatform: Platform? = null): Flow<PriceChange> {
        return repo.observeAll()
            .map { items ->
                items.filter { item ->
                    item.price != null &&
                    item.sourceUrl != null &&
                    item.availability &&
                    (targetPlatform == null || item.sourcePlatform == targetPlatform)
                }
            }
            .flatMapLatest { items ->
                // 并发检查每个商品价格（最多 5 个并发）
                items.asFlow()
                    .flatMapMerge(concurrency = 5) { item ->
                        flow {
                            val result = priceApi.checkPrice(item.sourceUrl!!)
                            if (result != null && result.price < item.price!!) {
                                emit(
                                    PriceChange(
                                        item = item,
                                        oldPrice = item.price!!,
                                        newPrice = result.price,
                                    )
                                )
                            }
                        }
                        .flowOn(Dispatchers.IO)
                        .catch { e ->
                            // 单个商品检查失败不影响其他
                            Log.w("PriceCheck", "Check failed for ${item.id}", e)
                            // 不 emit，继续下一个
                        }
                    }
            }
    }
}
```

---

## 四、异常处理策略

```kotlin
// ===== 三层次异常处理 =====

// 层次 1: Flow 操作符层（降级/重试）
flow
    .catch { e ->
        // 捕获上游异常
        emit(cachedValue)  // 发射降级值
    }
    .retry(retries = 2) { e ->
        e is IOException  // 仅网络异常重试
    }

// 层次 2: ViewModel collect 层（转换异常为用户消息）
viewModelScope.launch {
    useCase()
        .catch { e ->
            _uiState.update { it.copy(error = e.localizedMessage) }
            _events.emit(UiEvent.ShowError(e.localizedMessage))
        }
        .collect { data ->
            _uiState.update { it.copy(items = data, error = null) }
        }
}

// 层次 3: supervisorScope 隔离失败
viewModelScope.launch {
    supervisorScope {
        launch { /* 子协程 1：失败不影响子协程 2 */ }
        launch { /* 子协程 2 */ }
    }
}
```

### 异常处理原则

1. **Repository 层**：让异常向上传播，不在此层吞掉
2. **UseCase 层**：可添加业务级异常处理（如返回缓存作为降级）
3. **ViewModel 层**：`.catch {}` 转换异常为用户可读的错误消息
4. **UI 层**：根据 error 状态展示 SnackBar 或重试按钮

---

## 五、并发安全

```kotlin
// ===== 问题：并发修改 StateFlow =====
// ❌ 竞态条件 — 两个协程同时读写导致数据丢失
fun addItem(item: Item) {
    val current = _uiState.value.items.toMutableList()
    current.add(item)
    _uiState.value = _uiState.value.copy(items = current)
}

// ✅ 使用 update {} 原子操作
fun addItem(item: Item) {
    _uiState.update { state ->
        state.copy(items = state.items + item)
    }
}

// ===== 问题：Room 多步操作非原子 =====
// ❌ 两步操作之间可能被中断
suspend fun moveToCart(itemId: String) {
    val item = dao.getById(itemId) ?: return
    dao.deleteById(itemId)
    // ← 此处若崩溃，数据丢失
    dao.insertToCart(item.toCartEntity())
}

// ✅ 使用 @Transaction 包裹多步操作
@Transaction
suspend fun moveItemToCart(itemId: String) {
    val item = getById(itemId) ?: return
    deleteById(itemId)
    insertToCart(item.toCartEntity())
}

// ===== 问题：MutableStateFlow 暴露可变集合 =====
// ❌ 外部可以绕过 ViewModel 直接修改
private val _items = MutableStateFlow<MutableList<Item>>(mutableListOf())
val items: StateFlow<MutableList<Item>> = _items.asStateFlow()
// 外部调用 items.value.clear() → 绕过了 ViewModel

// ✅ 暴露不可变类型
private val _items = MutableStateFlow<List<Item>>(emptyList())
val items: StateFlow<List<Item>> = _items.asStateFlow()
```

---

## 六、callbackFlow 封装系统 API

```kotlin
// 项目中的典型场景：封装 NotificationListenerService 回调

/**
 * 将通知监听回调转换为 Flow
 */
fun notificationFlow(): Flow<ShoppingNotification> = callbackFlow {
    val listener = object : NotificationListener() {
        override fun onShoppingNotification(notification: ShoppingNotification) {
            trySend(notification)  // 非阻塞发送
        }
    }

    notificationManager.registerListener(listener)

    awaitClose {
        notificationManager.unregisterListener(listener)
    }
}.buffer(Channel.BUFFERED)  // 缓冲避免背压
```

---

## 七、测试协程与 Flow

```kotlin
// ===== ViewModel 测试 =====
@Test
fun `given items in repo when screen loads then items are displayed`() = runTest {
    val mockItems = listOf(createTestItem())
    every { repo.observeAll() } returns flowOf(mockItems)

    val viewModel = createViewModel()

    viewModel.uiState.test {
        val state = awaitItem()
        assertThat(state.items).hasSize(1)
        cancelAndIgnoreRemainingEvents()
    }
}

// ===== Flow 测试（Turbine） =====
@Test
fun `given entities in dao when observeAll then emits domain items`() = runTest {
    val entities = listOf(createTestEntity())
    every { dao.observeAll() } returns flowOf(entities)

    repository.observeAll().test {
        val items = awaitItem()
        assertThat(items).hasSize(1)
        assertThat(items[0]).isInstanceOf(CollectedItem::class.java)
        cancelAndConsumeRemainingEvents()
    }
}

// ===== 时间控制 =====
@Test
fun `debounce search input`() = runTest {
    val searchFlow = MutableSharedFlow<String>()
    val result = searchFlow
        .debounce(300)  // runTest 虚拟时间，300ms 瞬间过
        .first()

    searchFlow.emit("iPhone")
    // 自动等待 debounce 时间
    assertThat(result).isEqualTo("iPhone")
}
```

---

## 八、输出格式

### 编写协程代码时
给出完整方法/类，确保异常有处理、Scope 选择正确、并发安全。

### 审查协程代码时

```
## 🔴 崩溃风险（异常未处理、Scope 泄漏、阻塞主线程）
## 🟡 并发问题（竞态条件、线程安全、StateFlow 可变集合暴露）
## 🔵 优化建议（并发度调整、Flow 链简化、dispatcher 优化）
```
