---
name: compose-data-bridge
description: Android Compose UI ↔ 数据层桥接 — ViewModel/UseCase/Repository 模式、Flow 收集、状态管理、错误处理、Room/Retrofit 集成
---

# compose-data-bridge

连接 Jetpack Compose UI 层与数据层（Room + Retrofit + DataStore）的桥接技能。当 UI 页面已搭建完成，需要接入真实数据时使用。

**触发词：** "接入数据"、"连接API"、"添加Repository"、"数据层怎么设计"、"UseCase怎么写"、"Room怎么用"

---

## 核心数据流

```
Composable (Screen)
    ↑ collectAsStateWithLifecycle()
ViewModel
    ↑ StateFlow<UiState>
UseCase (domain 层，纯业务逻辑)
    ↑ Flow<List<DomainModel>>
Repository (data 层实现 domain 接口)
    ↑ Flow / suspend
DataSource (Room DAO / Retrofit API / DataStore)
```

**关键约束：**
- UI 层只依赖 `domain` 和 `core`，**不直接依赖 `data`**
- `domain` 层纯 Kotlin，**不依赖 Android 框架**
- Repository 接口定义在 `domain`，实现在 `data`

---

## Step 1: 定义领域模型（domain-model）

```kotlin
// domain/domain-model/src/main/java/com/youlikehub/domain/model/CollectionItem.kt

data class CollectionItem(
    val id: String,
    val title: String,
    val price: java.math.BigDecimal?,
    val originalPrice: java.math.BigDecimal?,
    val imageUrl: String?,
    val sourcePlatform: Platform,
    val sourceUrl: String?,
    val category: ItemCategory,
    val status: ItemStatus,
    val collectedAt: java.time.Instant,
    val lastCheckedAt: java.time.Instant?,
    val isFavorite: Boolean,
)

enum class Platform {
    TAOBAO, JD, PINDUODUO, XIANYU, XIAOHONGSHU,
    AMAZON, DEWU, SUNING, KAOLA, OTHER,
}

enum class ItemCategory {
    FAVORITE, CART, PURCHASED, WISHLIST, LIKE,
}

enum class ItemStatus {
    AVAILABLE, SOLD_OUT, PRICE_DROP, UNKNOWN,
}
```

## Step 2: 定义 Repository 接口（domain-repository）

```kotlin
// domain/domain-repository/src/main/java/com/youlikehub/domain/repository/CollectionRepository.kt

import kotlinx.coroutines.flow.Flow

interface CollectionRepository {
    /** 获取最近收藏 */
    fun getRecentItems(limit: Int = 20): Flow<List<CollectionItem>>

    /** 按平台筛选 */
    fun getItemsByPlatform(platform: Platform): Flow<List<CollectionItem>>

    /** 按分类筛选 */
    fun getItemsByCategory(category: ItemCategory): Flow<List<CollectionItem>>

    /** 全文搜索 */
    fun searchItems(query: String): Flow<List<CollectionItem>>

    /** 获取单个商品 */
    suspend fun getItemById(id: String): CollectionItem?

    /** 插入/更新商品 */
    suspend fun upsertItem(item: CollectionItem)

    /** 删除商品 */
    suspend fun deleteItem(id: String)

    /** 切换收藏状态 */
    suspend fun toggleFavorite(id: String)

    /** 获取各平台统计 */
    fun getPlatformStats(): Flow<List<PlatformStats>>
}

data class PlatformStats(
    val platform: Platform,
    val count: Int,
)
```

## Step 3: 实现 UseCase（domain-usecase）

```kotlin
// domain/domain-usecase/src/main/java/com/youlikehub/domain/usecase/GetRecentItemsUseCase.kt

import javax.inject.Inject

class GetRecentItemsUseCase @Inject constructor(
    private val repository: CollectionRepository,
) {
    /** operator invoke 使调用方可以像函数一样使用 */
    operator fun invoke(limit: Int = 20): Flow<List<CollectionItem>> =
        repository.getRecentItems(limit)
}
```

**UseCase 设计原则：**
- 每个 UseCase 只做一件事（单一职责）
- 通过 `operator fun invoke()` 提供函数式调用
- 可以组合多个 Repository 的数据
- 可以包含业务校验/转换逻辑

```kotlin
// 复杂 UseCase 示例：获取降价商品
class GetPriceDropItemsUseCase @Inject constructor(
    private val repository: CollectionRepository,
) {
    operator fun invoke(): Flow<List<CollectionItem>> =
        repository.getRecentItems()
            .map { items ->
                items.filter { it.status == ItemStatus.PRICE_DROP }
                    .sortedByDescending { it.lastCheckedAt }
            }
}
```

## Step 4: 实现 Repository（data-repository）

```kotlin
// data/data-repository/src/main/java/com/youlikehub/data/repository/CollectionRepositoryImpl.kt

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import javax.inject.Inject

class CollectionRepositoryImpl @Inject constructor(
    private val localDataSource: CollectionLocalDataSource,
    private val remoteDataSource: CollectionRemoteDataSource,
) : CollectionRepository {

    override fun getRecentItems(limit: Int): Flow<List<CollectionItem>> =
        localDataSource.getRecentItems(limit)
            .map { entities -> entities.map { it.toDomainModel() } }

    override fun searchItems(query: String): Flow<List<CollectionItem>> =
        localDataSource.searchItems(query)
            .map { entities -> entities.map { it.toDomainModel() } }

    override suspend fun getItemById(id: String): CollectionItem? =
        localDataSource.getItemById(id)?.toDomainModel()

    override suspend fun upsertItem(item: CollectionItem) {
        localDataSource.upsertItem(item.toEntity())
    }

    override suspend fun deleteItem(id: String) {
        localDataSource.deleteItem(id)
    }

    override suspend fun toggleFavorite(id: String) {
        localDataSource.toggleFavorite(id)
    }

    // ... 其他方法
}
```

## Step 5: Room DAO（data-local）

```kotlin
// data/data-local/src/main/java/com/youlikehub/data/local/dao/CollectionItemDao.kt

import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface CollectionItemDao {

    @Query("SELECT * FROM collection_items ORDER BY collected_at DESC LIMIT :limit")
    fun getRecentItems(limit: Int): Flow<List<CollectionItemEntity>>

    @Query("SELECT * FROM collection_items WHERE platform = :platform ORDER BY collected_at DESC")
    fun getItemsByPlatform(platform: String): Flow<List<CollectionItemEntity>>

    @Query("SELECT * FROM collection_items WHERE title LIKE '%' || :query || '%'")
    fun searchItems(query: String): Flow<List<CollectionItemEntity>>

    @Query("SELECT * FROM collection_items WHERE id = :id")
    suspend fun getItemById(id: String): CollectionItemEntity?

    @Upsert
    suspend fun upsertItem(item: CollectionItemEntity)

    @Query("DELETE FROM collection_items WHERE id = :id")
    suspend fun deleteItem(id: String)

    @Query("UPDATE collection_items SET is_favorite = NOT is_favorite WHERE id = :id")
    suspend fun toggleFavorite(id: String)

    @Query("""
        SELECT platform, COUNT(*) as count 
        FROM collection_items 
        GROUP BY platform 
        ORDER BY count DESC
    """)
    fun getPlatformStats(): Flow<List<PlatformStatsEntity>>
}
```

## Step 6: Room Entity 与 Domain Model 映射

```kotlin
// data/data-local/.../entity/CollectionItemEntity.kt

@Entity(tableName = "collection_items")
data class CollectionItemEntity(
    @PrimaryKey val id: String,
    val title: String,
    val price: BigDecimal?,
    val originalPrice: BigDecimal?,
    val imageUrl: String?,
    val platform: String,        // Platform.name
    val sourceUrl: String?,
    val category: String,        // ItemCategory.name
    val status: String,          // ItemStatus.name
    val collectedAt: Long,       // epoch millis
    val lastCheckedAt: Long?,
    val isFavorite: Boolean,
)

// 映射函数
fun CollectionItemEntity.toDomainModel() = CollectionItem(
    id = id,
    title = title,
    price = price,
    originalPrice = originalPrice,
    imageUrl = imageUrl,
    sourcePlatform = Platform.valueOf(platform),
    sourceUrl = sourceUrl,
    category = ItemCategory.valueOf(category),
    status = ItemStatus.valueOf(status),
    collectedAt = Instant.ofEpochMilli(collectedAt),
    lastCheckedAt = lastCheckedAt?.let { Instant.ofEpochMilli(it) },
    isFavorite = isFavorite,
)

fun CollectionItem.toEntity() = CollectionItemEntity(
    id = id,
    title = title,
    price = price,
    originalPrice = originalPrice,
    imageUrl = imageUrl,
    platform = sourcePlatform.name,
    sourceUrl = sourceUrl,
    category = category.name,
    status = status.name,
    collectedAt = collectedAt.toEpochMilli(),
    lastCheckedAt = lastCheckedAt?.toEpochMilli(),
    isFavorite = isFavorite,
)
```

## Step 7: ViewModel 数据接入（两种模式）

### 模式 A：一次性加载（StateFlow + viewModelScope）

适用于：详情页、表单页等不需要持续监听的场景

```kotlin
@HiltViewModel
class ItemDetailViewModel @Inject constructor(
    private val getItemById: GetItemByIdUseCase,
    private val deleteItem: DeleteItemUseCase,
    savedStateHandle: SavedStateHandle,  // 从导航参数获取 id
) : ViewModel() {

    private val itemId: String = savedStateHandle.toRoute<ItemDetailRoute>().itemId

    private val _uiState = MutableStateFlow<ItemDetailUiState>(ItemDetailUiState.Loading)
    val uiState: StateFlow<ItemDetailUiState> = _uiState.asStateFlow()

    init {
        loadItem()
    }

    fun onRetry() = loadItem()
    fun onDelete() { viewModelScope.launch { /* 删除逻辑 */ } }

    private fun loadItem() {
        viewModelScope.launch {
            _uiState.value = ItemDetailUiState.Loading
            try {
                val item = getItemById(itemId)
                if (item != null) {
                    _uiState.value = ItemDetailUiState.Success(
                        item = item.toDetailUiModel(),
                    )
                } else {
                    _uiState.value = ItemDetailUiState.Error("商品不存在")
                }
            } catch (e: Exception) {
                _uiState.value = ItemDetailUiState.Error(
                    e.message ?: "加载失败"
                )
            }
        }
    }
}
```

### 模式 B：响应式流（Flow 自动更新）

适用于：列表页、仪表盘等需要实时反映数据库变化的场景

```kotlin
@HiltViewModel
class CollectionListViewModel @Inject constructor(
    private val getRecentItems: GetRecentItemsUseCase,
    private val toggleFavorite: ToggleFavoriteUseCase,
) : ViewModel() {

    val uiState: StateFlow<CollectionListUiState> = getRecentItems()
        .map { items ->
            if (items.isEmpty()) {
                CollectionListUiState.Empty
            } else {
                CollectionListUiState.Success(
                    items = items.map { it.toListUiModel() },
                )
            }
        }
        .catch { e ->
            emit(CollectionListUiState.Error(e.message ?: "加载失败"))
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),  // 5秒无订阅者后停止
            initialValue = CollectionListUiState.Loading,
        )

    fun onToggleFavorite(id: String) {
        viewModelScope.launch {
            try {
                toggleFavorite(id)
            } catch (e: Exception) {
                // 错误已通过 Flow catch 处理
                // 如需 Toast 通知，可通过 Channel/SharedFlow 发送一次性事件
            }
        }
    }
}
```

## Step 8: Domain → UI Model 转换

```kotlin
// 在 ViewModel 文件或独立的 mapper 文件中
// 永远不要让 Composable 接触 domain model

fun CollectionItem.toListUiModel() = CollectionItemUiModel(
    id = id,
    title = title,
    priceDisplay = price?.let { "¥${"%.2f".format(it)}" } ?: "暂无价格",
    originalPriceDisplay = originalPrice?.let { "¥${"%.2f".format(it)}" },
    imageUrl = imageUrl,
    platformIcon = sourcePlatform.iconRes,
    platformName = sourcePlatform.displayName,
    isFavorite = isFavorite,
    collectedDate = collectedAt.toDisplayString(),
    statusLabel = status.displayName,
)

// 格式化扩展函数
private fun Instant.toDisplayString(): String {
    val formatter = java.time.format.DateTimeFormatter
        .ofPattern("yyyy-MM-dd HH:mm")
        .withZone(java.time.ZoneId.systemDefault())
    return formatter.format(this)
}

private val Platform.iconRes: Int @DrawableRes get() = when (this) {
    Platform.TAOBAO -> R.drawable.ic_platform_taobao
    Platform.JD -> R.drawable.ic_platform_jd
    // ...
    else -> R.drawable.ic_platform_other
}

private val Platform.displayName: String get() = when (this) {
    Platform.TAOBAO -> "淘宝"
    Platform.JD -> "京东"
    // ...
    else -> "其他"
}
```

---

## 错误处理策略

### 分层错误处理

```kotlin
// 1. DataSource 层：抛出具体异常
sealed class DataException(message: String, cause: Throwable? = null) : Exception(message, cause) {
    class NetworkError(cause: Throwable? = null) : DataException("网络连接失败", cause)
    class ServerError(code: Int, message: String) : DataException("服务器错误 ($code): $message")
    class NotFound(id: String) : DataException("未找到记录: $id")
    class Unknown(cause: Throwable? = null) : DataException("未知错误", cause)
}

// 2. Repository 层：捕获并转换为领域异常（可选，或直接透传）
// 3. ViewModel 层：捕获所有异常，转换为用户可读的错误消息
```

### ViewModel 中的统一错误处理

```kotlin
private fun loadData() {
    viewModelScope.launch {
        _uiState.value = UiState.Loading
        try {
            val data = useCase()
            _uiState.value = UiState.Success(data = data.toUiModels())
        } catch (e: DataException.NetworkError) {
            _uiState.value = UiState.Error("网络连接失败，请检查网络后重试")
        } catch (e: DataException.ServerError) {
            _uiState.value = UiState.Error(e.message ?: "服务器开小差了，稍后再试")
        } catch (e: Exception) {
            _uiState.value = UiState.Error("加载失败: ${e.message}")
        }
    }
}
```

---

## 开发阶段 Mock 数据策略

在实际数据源开发完成前，使用假数据驱动 UI 开发：

```kotlin
// 开发用假数据
object MockData {
    val sampleItems = listOf(
        CollectionItemUiModel(
            id = "mock-1",
            title = "Apple AirPods Pro (第二代) 无线降噪耳机",
            priceDisplay = "¥1,799.00",
            originalPriceDisplay = "¥1,999.00",
            imageUrl = null,
            platformIcon = 0,
            platformName = "京东",
            isFavorite = true,
            collectedDate = "2026-07-01 14:30",
            statusLabel = "在售",
        ),
        // ... 更多假数据
    )
}

// 在 ViewModel init 中：
// 开发阶段直接使用 mock，后续替换为 UseCase 调用
init {
    _uiState.value = UiState.Success(data = MockData.sampleItems)
}
```

### 使用 Hilt 环境切换

```kotlin
// 创建 BuildConfig 控制的 Repository 实现
class CollectionRepositoryFake @Inject constructor() : CollectionRepository {
    override fun getRecentItems(limit: Int): Flow<List<CollectionItem>> = flow {
        emit(MockData.domainItems.take(limit))
    }
    // ...
}

// Hilt Module 中按 buildType 绑定
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindCollectionRepository(
        impl: CollectionRepositoryImpl,  // 或 CollectionRepositoryFake
    ): CollectionRepository
}
```

---

## DataStore 偏好存储

```kotlin
// core-datastore 中封装
class UserPreferencesDataSource @Inject constructor(
    private val dataStore: DataStore<Preferences>,
) {
    val isDarkMode: Flow<Boolean> = dataStore.data.map { prefs ->
        prefs[IS_DARK_MODE_KEY] ?: false
    }

    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { prefs ->
            prefs[IS_DARK_MODE_KEY] = enabled
        }
    }

    companion object {
        private val IS_DARK_MODE_KEY = booleanPreferencesKey("is_dark_mode")
    }
}
```

---

## Hilt DI 模块配置

```kotlin
// 各级 DI 模块

// data/data-local/di/LocalDataModule.kt
@Module
@InstallIn(SingletonComponent::class)
object LocalDataModule {
    @Provides @Singleton
    fun provideDatabase(@ApplicationContext context: Context): YouLikeHubDatabase =
        Room.databaseBuilder(context, YouLikeHubDatabase::class.java, "youlikehub.db")
            .fallbackToDestructiveMigration()  // 开发阶段，正式版需 Migration
            .build()

    @Provides fun provideCollectionItemDao(db: YouLikeHubDatabase) = db.collectionItemDao()
}

// data/data-repository/di/RepositoryModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds @Singleton
    abstract fun bindCollectionRepository(impl: CollectionRepositoryImpl): CollectionRepository
}

// domain/domain-usecase/di/UseCaseModule.kt
@Module
@InstallIn(SingletonComponent::class)
object UseCaseModule {
    // UseCase 是无状态对象，可以直接用 @Provides 或让 Hilt 通过 @Inject constructor 构造
    // (GetRecentItemsUseCase 已有 @Inject constructor，无需在此显式声明)
}
```

---

## 数据层检查清单

- [ ] Repository 接口定义在 `domain-repository` 模块
- [ ] Repository 实现在 `data-repository` 模块
- [ ] ViewModel 通过 UseCase 访问数据，不直接调用 Repository
- [ ] domain model 与 Room Entity 分离，通过映射函数转换
- [ ] ViewModel 中 domain model → UI model 的转换使用扩展函数
- [ ] Room DAO 方法返回 `Flow<>` 以实现响应式更新
- [ ] 写操作使用 `suspend` 函数
- [ ] 错误处理覆盖 NetworkError / ServerError / Unknown
- [ ] 开发阶段有 Mock 数据可用
- [ ] Hilt Module 正确绑定各层依赖

---

*技能版本：v1.0 | 基于 YouLikeHub 项目 | 数据层：Room + Retrofit + DataStore + Hilt*
