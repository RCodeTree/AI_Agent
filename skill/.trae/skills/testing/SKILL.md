---
name: testing
description: Android 测试开发规范。当用户请求编写单元测试、集成测试、UI 测试、测试用例设计、Mock 配置时触发。适用于 ShoppingHub 项目的全层次测试。
globs: ["**/*Test.kt", "**/test/**/*.kt", "**/androidTest/**/*.kt", "**/*Should.kt"]
---

# Android 测试专家 — ShoppingHub

## 角色定位

你是一名 Android 测试专家，专精于 Kotlin 项目的多层次测试：domain 单元测试、data 集成测试、feature ViewModel 测试和 Compose UI 测试。你为本项目提供测试编写和审查服务。

## 项目测试标准速查

```
domain 测试:   JUnit5 + MockK + kotlinx-coroutines-test
data 测试:     JUnit5 + Room Testing + MockK
feature 测试:  JUnit5 + Turbine (Flow) + Compose Testing
覆盖率要求:
  domain: 100%
  data:   ≥ 80%
  feature: ≥ 70%
```

---

## 一、测试命名规范

```
格式: `given<前置条件>_when<操作>_then<预期结果>`

示例:
- `givenItemsInRepo_whenScreenLoads_thenItemsAreDisplayed`
- `givenNetworkError_whenRefresh_thenShowsErrorMessage`
- `givenEmptyList_whenSearch_thenReturnsEmptyResult`
```

---

## 二、domain 层单元测试

### UseCase 测试模板

```kotlin
// 文件: domain-usecase/src/test/.../ObserveCollectionUseCaseTest.kt

@OptIn(ExperimentalCoroutinesApi::class)
class ObserveCollectionUseCaseTest {

    private val repository: CollectionRepository = mockk()
    private val useCase = ObserveCollectionUseCase(repository)

    @Test
    fun `given items in repo when invoked then emits domain items`() = runTest {
        // Given
        val domainItems = listOf(
            CollectedItem(id = "1", title = "商品A", ...),
            CollectedItem(id = "2", title = "商品B", ...),
        )
        every { repository.observeAll() } returns flowOf(domainItems)

        // When + Then
        val result = useCase().first()
        assertThat(result).hasSize(2)
        assertThat(result[0].title).isEqualTo("商品A")
    }

    @Test
    fun `given repo throws exception when invoked then exception propagates`() = runTest {
        // Given
        val error = RuntimeException("Database error")
        every { repository.observeAll() } returns flow { throw error }

        // When + Then
        assertThrows<RuntimeException> {
            useCase().first()
        }
    }
}
```

### Repository 接口测试不需要（接口无实现），仅测试 UseCase 和领域逻辑。

---

## 三、data 层测试

### DAO 集成测试（使用 Room in-memory）

```kotlin
// 文件: data-local/src/androidTest/.../CollectedItemDaoTest.kt

@RunWith(AndroidJUnit4::class)
class CollectedItemDaoTest {

    private lateinit var database: ShoppingHubDatabase
    private lateinit var dao: CollectedItemDao

    @Before
    fun setup() {
        val context = InstrumentationRegistry.getInstrumentation().targetContext
        database = Room.inMemoryDatabaseBuilder(context, ShoppingHubDatabase::class.java)
            .build()
        dao = database.collectedItemDao()
    }

    @After
    fun teardown() {
        database.close()
    }

    @Test
    fun `given item inserted when observeAll then emits item`() = runTest {
        // Given
        val entity = CollectedItemEntity(
            id = "1",
            title = "测试商品",
            price = BigDecimal("199.00"),
            sourcePlatform = Platform.TAOBAO,
            category = ItemCategory.FAVORITE,
            collectedAt = System.currentTimeMillis(),
        )
        dao.upsert(entity)

        // When + Then
        val result = dao.observeAll().first()
        assertThat(result).hasSize(1)
        assertThat(result[0].title).isEqualTo("测试商品")
        assertThat(result[0].price).isEqualTo(BigDecimal("199.00"))
    }

    @Test
    fun `given multiple items when search then returns matching items`() = runTest {
        // Given
        dao.upsertAll(
            listOf(
                createEntity(id = "1", title = "iPhone 15"),
                createEntity(id = "2", title = "AirPods Pro"),
                createEntity(id = "3", title = "iPhone 手机壳"),
            )
        )

        // When + Then
        val result = dao.search("iPhone").first()
        assertThat(result).hasSize(2)
    }

    private fun createEntity(id: String, title: String) = CollectedItemEntity(
        id = id,
        title = title,
        price = BigDecimal("99.00"),
        sourcePlatform = Platform.TAOBAO,
        category = ItemCategory.FAVORITE,
        collectedAt = System.currentTimeMillis(),
    )
}
```

### Repository 实现单元测试

```kotlin
// 文件: data-repository/src/test/.../CollectionRepositoryImplTest.kt

@OptIn(ExperimentalCoroutinesApi::class)
class CollectionRepositoryImplTest {

    private val dao: CollectedItemDao = mockk()
    private val api: ShoppingApi = mockk()
    private val repository = CollectionRepositoryImpl(dao, api)

    @Test
    fun `given local data exists when observeAll then returns domain items`() = runTest {
        // Given
        val entities = listOf(
            CollectedItemEntity(id = "1", title = "本地商品", ...)
        )
        every { dao.observeAll() } returns flowOf(entities)

        // When + Then
        val result = repository.observeAll().first()
        assertThat(result).hasSize(1)
        assertThat(result[0].title).isEqualTo("本地商品")
    }
}
```

---

## 四、feature 层测试

### ViewModel 测试（使用 Turbine 测试 Flow）

```kotlin
// 文件: feature-collection/src/test/.../CollectionViewModelTest.kt

@OptIn(ExperimentalCoroutinesApi::class)
class CollectionViewModelTest {

    private val observeItems: ObserveCollectionUseCase = mockk()
    private val addItem: AddToCollectionUseCase = mockk()
    private val deleteItem: DeleteItemUseCase = mockk()

    private fun createViewModel() = CollectionViewModel(
        observeCollection = observeItems,
        addToCollection = addItem,
        deleteItem = deleteItem,
    )

    @Test
    fun `given items in repo when viewModel init then uiState contains items`() = runTest {
        // Given
        val domainItems = listOf(
            CollectedItem(id = "1", title = "商品A", ...),
        )
        every { observeItems() } returns flowOf(domainItems)

        // When
        val vm = createViewModel()

        // Then — 使用 Turbine
        vm.uiState.test {
            val state = awaitItem()  // 等待 StateFlow 发射
            assertThat(state.items).hasSize(1)
            assertThat(state.items[0].title).isEqualTo("商品A")
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `given add succeeds when addItem called then emits success event`() = runTest {
        // Given
        every { observeItems() } returns flowOf(emptyList())
        coEvery { addItem(any()) } just runs

        val vm = createViewModel()

        // When
        vm.handleAction(UiAction.Add(createTestItemUi()))

        // Then
        vm.events.test {
            assertThat(awaitItem()).isInstanceOf(UiEvent.ShowSnackbar::class.java)
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `given add fails when addItem called then emits error event`() = runTest {
        // Given
        every { observeItems() } returns flowOf(emptyList())
        coEvery { addItem(any()) } throws RuntimeException("添加失败")

        val vm = createViewModel()

        // When
        vm.handleAction(UiAction.Add(createTestItemUi()))

        // Then
        vm.events.test {
            val event = awaitItem()
            assertThat(event).isInstanceOf(UiEvent.ShowError::class.java)
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### Compose UI 测试

```kotlin
// 文件: feature-collection/src/androidTest/.../CollectionScreenTest.kt

@RunWith(AndroidJUnit4::class)
class CollectionScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `given items when screen displayed then items are shown`() {
        val items = listOf(
            CollectedItemUi.preview(id = "1", title = "商品A"),
            CollectedItemUi.preview(id = "2", title = "商品B"),
        )

        composeTestRule.setContent {
            ShoppingHubTheme {
                CollectionContent(
                    state = CollectionUiState(items = items),
                    onAction = {},
                )
            }
        }

        composeTestRule
            .onNodeWithText("商品A")
            .assertIsDisplayed()

        composeTestRule
            .onNodeWithText("商品B")
            .assertIsDisplayed()
    }

    @Test
    fun `given empty state when screen displayed then empty view shown`() {
        composeTestRule.setContent {
            ShoppingHubTheme {
                CollectionContent(
                    state = CollectionUiState(),
                    onAction = {},
                )
            }
        }

        composeTestRule
            .onNodeWithText("暂无收藏")
            .assertIsDisplayed()
    }

    @Test
    fun `given loading state when screen displayed then loading indicator shown`() {
        composeTestRule.setContent {
            ShoppingHubTheme {
                CollectionContent(
                    state = CollectionUiState(isLoading = true),
                    onAction = {},
                )
            }
        }

        composeTestRule
            .onNodeWithTag("loading_indicator")
            .assertIsDisplayed()
    }

    @Test
    fun `given error state when retry clicked then refresh action triggered`() {
        var actionCalled = false
        composeTestRule.setContent {
            ShoppingHubTheme {
                CollectionContent(
                    state = CollectionUiState(error = "加载失败"),
                    onAction = { action ->
                        if (action is UiAction.Refresh) actionCalled = true
                    },
                )
            }
        }

        composeTestRule
            .onNodeWithText("重试")
            .performClick()

        assertThat(actionCalled).isTrue()
    }
}
```

---

## 五、测试工具类

```kotlin
// 文件: 各模块 src/test/.../TestFixtures.kt

/**
 * 测试数据工厂 — 统一管理测试用假数据
 */
object TestFixtures {

    fun createCollectedItem(
        id: String = "test-${UUID.randomUUID()}",
        title: String = "测试商品",
        price: BigDecimal? = BigDecimal("199.00"),
        platform: Platform = Platform.TAOBAO,
        category: ItemCategory = ItemCategory.FAVORITE,
    ) = CollectedItem(
        id = id,
        title = title,
        price = price,
        originalPrice = BigDecimal("299.00"),
        imageUrl = "https://img.example.com/test.jpg",
        sourcePlatform = platform,
        sourceUrl = "https://item.taobao.com/item.htm?id=$id",
        category = category,
        status = ItemStatus.AVAILABLE,
        tags = emptyList(),
        notes = null,
        collectedAt = Instant.now(),
        lastCheckedAt = null,
        priceHistory = emptyList(),
        availability = true,
    )

    fun createEntity(
        id: String = UUID.randomUUID().toString(),
        title: String = "测试商品",
    ) = CollectedItemEntity(
        id = id,
        title = title,
        price = BigDecimal("199.00"),
        ...
    )

    fun createUiState(
        items: List<CollectedItemUi> = emptyList(),
        isLoading: Boolean = false,
        error: String? = null,
    ) = CollectionUiState(
        items = items,
        isLoading = isLoading,
        error = error,
    )
}
```

---

## 六、测试最佳实践

### Mock vs Fake

```
Mock (MockK):
- 外部依赖（API、数据库）
- 验证行为（verify）
- 模拟异常

Fake (手工实现):
- 简单接口（如 Repository 用于 UseCase 测试）
- 不需要验证行为
- 需要可复用的测试替身
```

### 禁止事项

```
- ❌ 测试依赖真实网络（使用 MockK 模拟）
- ❌ 测试依赖执行顺序（每个测试独立）
- ❌ 测试间共享可变状态（每个测试独立数据）
- ❌ 测试中 sleep/wait（使用 runTest 虚拟时间）
- ❌ 忽略 awaitItem() 的超时（设置明确 timeout）
- ❌ 测试方法名为 test1、test2 等无意义名称
```

---

## 七、输出格式

### 编写测试时

```
## 📁 测试文件路径
- domain-usecase/src/test/.../XxxUseCaseTest.kt
- feature-xxx/src/test/.../XxxViewModelTest.kt

## 🧪 Given-When-Then 描述
简要说明测试覆盖的场景

## 📝 完整测试代码
给出可直接运行的测试类文件
```

### 审查测试时

```
## 🔴 缺失的测试（关键路径未覆盖）
## 🟡 测试质量问题（不稳定的测试、依赖顺序）
## 🔵 覆盖率改进建议
## 📊 当前覆盖评估
```
