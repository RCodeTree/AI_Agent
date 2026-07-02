---
name: compose-page-builder
description: Android Compose 页面脚手架生成 — 基于 MVVM + Clean Architecture，自动生成 Screen、ViewModel、UiState、Navigation 完整代码骨架
---

# compose-page-builder

Android Jetpack Compose 页面快速生成技能。当你需要新建一个页面（Screen）时，本技能提供完整的页面脚手架生成流程。

**触发词：** "创建页面"、"新建 Screen"、"写一个XX页"、"生成XX页面"、"添加XX功能模块"

---

## 工作流程

### Step 1: 识别页面类型

根据用户需求，判断页面属于哪种类型：

| 页面类型 | 特征 | 典型场景 |
|----------|------|----------|
| **List（列表页）** | 展示数据集合，支持滚动/筛选 | 收藏列表、商品流、搜索结果 |
| **Detail（详情页）** | 展示单个实体完整信息 | 商品详情、订单详情、平台统计 |
| **Form（表单页）** | 用户输入/编辑数据 | 手动录入、编辑信息、设置项 |
| **Dashboard（仪表盘）** | 多区块数据概览 | 首页、数据统计、个人中心 |
| **Settings（设置页）** | 配置项/开关/选项 | 设置、偏好管理、权限配置 |
| **Onboarding（引导页）** | 分步引导/HorizontalPager | 新手引导、功能介绍 |

### Step 2: 确定模块归属

将页面归入对应的 feature 模块：

```
feature-home/        → 首页、仪表盘
feature-collection/  → 收藏管理（列表+详情）
feature-cart/        → 购物车聚合
feature-order/       → 已购订单
feature-wishlist/    → 愿望清单
feature-price-alert/ → 降价提醒
feature-search/      → 全局搜索
feature-settings/    → 设置
feature-import/      → 数据导入/手动录入
feature-onboarding/  → 新手引导
```

如果页面属于全新功能领域，建议创建新的 `feature-{name}` 模块。

### Step 3: 创建导航路由

在 `feature-{name}/navigation/{Name}Navigation.kt` 中定义路由：

```kotlin
package com.youlikehub.feature.{name}.navigation

import androidx.navigation.NavController
import androidx.navigation.NavGraphBuilder
import androidx.navigation.compose.composable
import kotlinx.serialization.Serializable

@Serializable
data object {Name}Route

// 带参数的路由（按需）：
// @Serializable
// data class {Name}DetailRoute(val itemId: String)

fun NavGraphBuilder.{name}Screen(
    onNavigateBack: () -> Unit,
    // 按需添加其他导航回调
) {
    composable<{Name}Route> {
        {Name}Screen(
            onNavigateBack = onNavigateBack,
        )
    }
}

fun NavController.navigateTo{Name}() {
    navigate({Name}Route) {
        popUpTo(graph.findStartDestination().id) { saveState = true }
        launchSingleTop = true
        restoreState = true
    }
}
```

### Step 4: 定义 UI State

在 `feature-{name}/ui/{Name}ViewModel.kt` 同级（或独立文件）定义 UiState：

```kotlin
import com.youlikehub.core.model.UiModel  // 通用 UI 展示模型

sealed interface {Name}UiState {
    data object Loading : {Name}UiState

    data class Success(
        val data: List<{Name}UiModel>,  // 或其他页面数据
        val isRefreshing: Boolean = false,
    ) : {Name}UiState

    data class Error(
        val message: String,
    ) : {Name}UiState
}

data class {Name}UiModel(
    val id: String,
    val title: String,
    val subtitle: String? = null,
    val imageUrl: String? = null,
    // 按页面需求添加字段——只放 UI 展示需要的数据，不放 domain 实体
)
```

**规则：**
- UiModel 是 UI 层专用展示模型，**不是** domain entity
- 日期、价格等已格式化为 String
- 包含平台图标 resId、显示名称等 UI 专用字段

### Step 5: 编写 ViewModel

```kotlin
package com.youlikehub.feature.{name}.ui

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class {Name}ViewModel @Inject constructor(
    private val getItems: GetItemsUseCase,
) : ViewModel() {

    private val _uiState = MutableStateFlow<{Name}UiState>({Name}UiState.Loading)
    val uiState: StateFlow<{Name}UiState> = _uiState.asStateFlow()

    init {
        loadData()
    }

    fun onRefresh() {
        loadData()
    }

    fun onItemClick(itemId: String) {
        // 通过事件通知 Screen 层导航（如果需要）
        // 或直接在 ViewModel 中暴露事件 Flow
    }

    fun onRetry() {
        loadData()
    }

    private fun loadData() {
        viewModelScope.launch {
            _uiState.value = {Name}UiState.Loading
            try {
                val items = getItems()
                _uiState.value = {Name}UiState.Success(
                    data = items.map { it.toUiModel() },
                )
            } catch (e: Exception) {
                _uiState.value = {Name}UiState.Error(
                    message = e.message ?: "加载失败"
                )
            }
        }
    }

    // domain model → UI model 转换
    private fun DomainModel.toUiModel() = {Name}UiModel(
        id = this.id,
        title = this.title,
        subtitle = this.subtitle,
        imageUrl = this.imageUrl,
    )
}
```

**规则：**
- 使用 `@HiltViewModel` + `@Inject`，禁止 `AndroidViewModel`
- `MutableStateFlow` 设为 `private`，通过 `asStateFlow()` 暴露只读版本
- 用户操作通过 public 方法暴露（`onXxx` 命名）
- 数据转换在 ViewModel 中完成，不让 Composable 接触 domain model
- 使用 `viewModelScope` 管理协程生命周期

### Step 6: 编写 Screen Composable

Screen 函数分为两层：外层处理注入和状态收集，内层（Content）是纯函数：

```kotlin
package com.youlikehub.feature.{name}.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.youlikehub.core.ui.components.LoadingIndicator
import com.youlikehub.core.ui.components.ErrorContent
import com.youlikehub.core.ui.components.EmptyContent

@Composable
fun {Name}Screen(
    onNavigateBack: () -> Unit,
    onItemClick: (String) -> Unit = {},
    modifier: Modifier = Modifier,
    viewModel: {Name}ViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    {Name}ScreenContent(
        uiState = uiState,
        onNavigateBack = onNavigateBack,
        onItemClick = onItemClick,
        onRefresh = { viewModel.onRefresh() },
        onRetry = { viewModel.onRetry() },
        modifier = modifier,
    )
}

@Composable
private fun {Name}ScreenContent(
    uiState: {Name}UiState,
    onNavigateBack: () -> Unit,
    onItemClick: (String) -> Unit,
    onRefresh: () -> Unit,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("{页面标题}") },
                navigationIcon = {
                    // 如果非首页，显示返回按钮
                },
            )
        },
        modifier = modifier,
    ) { paddingValues ->
        Box(modifier = Modifier.padding(paddingValues).fillMaxSize()) {
            when (uiState) {
                is {Name}UiState.Loading -> LoadingIndicator()
                is {Name}UiState.Error -> ErrorContent(
                    message = uiState.message,
                    onRetry = onRetry,
                )
                is {Name}UiState.Success -> {
                    if (uiState.data.isEmpty()) {
                        EmptyContent(
                            title = "暂无内容",
                            description = "这里还没有数据",
                        )
                    } else {
                        // 主要 UI 内容
                        {Name}MainContent(
                            items = uiState.data,
                            onItemClick = onItemClick,
                        )
                    }
                }
            }
        }
    }
}

@Composable
private fun {Name}MainContent(
    items: List<{Name}UiModel>,
    onItemClick: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    // 实际的内容区域
    // 根据页面类型选择 LazyColumn / LazyVerticalGrid / Column 等
}
```

**规则：**
- Screen 函数总是接受 `modifier` 和导航回调作为参数
- 使用 `collectAsStateWithLifecycle()` 而非 `collectAsState()`
- 所有三种状态（Loading / Error / Empty / Success）都要处理
- Content 函数是纯函数，不注入 ViewModel，方便预览

### Step 7: 注册导航

在 `app` 模块的 `MainActivity.kt`（或顶层 `NavHost`）中注册：

```kotlin
NavHost(
    navController = navController,
    startDestination = HomeRoute,
) {
    homeScreen(...)
    // 新增：
    {name}Screen(
        onNavigateBack = { navController.popBackStack() },
    )
}
```

### Step 8: 添加 Preview

```kotlin
@Preview(name = "Light", showBackground = true)
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
private fun Preview{Name}Screen() {
    YouLikeHubTheme {
        {Name}ScreenContent(
            uiState = {Name}UiState.Success(
                data = listOf(
                    {Name}UiModel(id = "1", title = "示例标题", subtitle = "副标题"),
                ),
            ),
            onNavigateBack = {},
            onItemClick = {},
            onRefresh = {},
            onRetry = {},
        )
    }
}
```

---

## 页面类型专属模式

### 列表页（ListPage）

```kotlin
@Composable
fun ListContent(
    items: List<UiModel>,
    onItemClick: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 160.dp),
        modifier = modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp),
    ) {
        items(items = items, key = { it.id }) { item ->
            ItemCard(item = item, onClick = { onItemClick(item.id) })
        }
    }
}
```

**要点：**
- 始终提供 `key` 参数
- 使用 `GridCells.Adaptive` 实现响应式
- 支持 pull-to-refresh（`pullToRefresh` modifier 或 `PullToRefreshBox`）
- 预留在列表顶部/底部分页加载逻辑

### 详情页（DetailPage）

```kotlin
@Composable
fun DetailContent(
    item: DetailUiModel,
    onBack: () -> Unit,
    onShare: () -> Unit,
    onDelete: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
    ) {
        // Hero Image
        AsyncImage(model = item.imageUrl, ...)
        // 标题区域
        Text(item.title, style = MaterialTheme.typography.headlineMedium)
        // 信息区域
        // 操作按钮
        // 相关内容
    }
}
```

**要点：**
- 内容可滚动：`Modifier.verticalScroll(rememberScrollState())`
- TopAppBar 中包含分享/删除等操作
- 使用 `ModalBottomSheet` 展示删除确认等危险操作

### 表单页（FormPage）

```kotlin
@Composable
fun FormContent(
    formState: FormUiState,
    onFieldChange: (FormField, String) -> Unit,
    onSubmit: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp),
    ) {
        OutlinedTextField(
            value = formState.title,
            onValueChange = { onFieldChange(FormField.TITLE, it) },
            label = { Text("标题") },
            isError = formState.titleError != null,
            supportingText = formState.titleError?.let { { Text(it) } },
        )
        // ... 更多字段

        Button(
            onClick = onSubmit,
            enabled = formState.isValid,
            modifier = Modifier.fillMaxWidth(),
        ) {
            Text("提交")
        }
    }
}
```

**要点：**
- Label 在上方，不在 placeholder
- 错误信息在输入框下方
- 提交按钮在键盘上方（`imePadding()`）
- 使用 `rememberSaveable` 保留表单状态

### 仪表盘页（DashboardPage）

```kotlin
@Composable
fun DashboardContent(
    stats: List<StatsUiModel>,
    recentItems: List<ItemUiModel>,
    alerts: List<AlertUiModel>,
    modifier: Modifier = Modifier,
) {
    LazyColumn(
        modifier = modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp),
    ) {
        // 统计卡片区
        item { StatsSection(stats = stats) }
        // 最近添加
        item { SectionHeader("最近添加") }
        items(recentItems, key = { it.id }) { item ->
            RecentItemRow(item = item)
        }
        // 降价提醒
        item { SectionHeader("降价提醒") }
        items(alerts, key = { it.id }) { alert ->
            AlertCard(alert = alert)
        }
    }
}
```

**要点：**
- 使用 `LazyColumn` + `item{}` + `items{}` 混合布局
- 不同区块之间用 `SectionHeader` 分隔
- 统计卡片支持横向滑动（`LazyRow`）
- 预留下拉刷新和骨架屏

### 设置页（SettingsPage）

```kotlin
@Composable
fun SettingsContent(
    settings: SettingsUiState,
    onToggleDarkMode: (Boolean) -> Unit,
    onToggleNotification: (Boolean) -> Unit,
    onNavigateTo: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    LazyColumn(modifier = modifier.fillMaxSize()) {
        // 外观设置组
        item { SettingsGroupHeader("外观") }
        item {
            SwitchSettingItem(
                title = "深色模式",
                subtitle = "使用深色主题",
                checked = settings.isDarkMode,
                onCheckedChange = onToggleDarkMode,
            )
        }
        // 通知设置组
        item { SettingsGroupHeader("通知") }
        item {
            SwitchSettingItem(
                title = "降价提醒",
                subtitle = "商品降价时推送通知",
                checked = settings.priceAlertEnabled,
                onCheckedChange = onToggleNotification,
            )
        }
        // 关于组
        item { SettingsGroupHeader("关于") }
        item {
            ClickSettingItem(
                title = "隐私政策",
                onClick = { onNavigateTo("privacy") },
            )
        }
    }
}
```

**要点：**
- 设置项归类分组，组间留白
- Switch / Click / Select 三种设置项模式
- 使用 `DataStore` 持久化设置状态
- 危险操作（如清除数据）用红色文字 + 确认对话框

---

## 质量检查清单

### 架构合规
- [ ] Screen 分为外层（注入）和 Content 层（纯函数）
- [ ] UiState 使用 `sealed interface` 包含 Loading / Success / Error
- [ ] ViewModel 只暴露 `StateFlow`，不暴露 `MutableStateFlow`
- [ ] UI 展示数据使用 UiModel，不直接使用 domain entity
- [ ] 导航回调作为 lambda 参数传入

### Compose 最佳实践
- [ ] 列表项提供 `key` 参数
- [ ] 使用 `collectAsStateWithLifecycle()` 收集 Flow
- [ ] 副作用使用 `LaunchedEffect` / `DisposableEffect`
- [ ] 所有 Composable 接受 `modifier` 参数
- [ ] 颜色使用 `MaterialTheme.colorScheme.xxx`
- [ ] 至少提供 Light 和 Dark 两种 Preview

### 状态覆盖
- [ ] Loading 状态有加载指示器
- [ ] Error 状态有错误信息和重试按钮
- [ ] Empty 状态有引导性空态插图/文案
- [ ] Success 状态有实际内容

### 预备数据层接入
- [ ] ViewModel 通过 UseCase 获取数据（可先用 mock）
- [ ] ViewModel 中有 `toUiModel()` 转换函数
- [ ] 错误处理覆盖网络异常

---

## 禁止模式

| 反模式 | 正确做法 |
|--------|----------|
| Composable 中 `remember { mutableStateOf() }` 持有业务状态 | 状态由 ViewModel 通过 UiState 提供 |
| 直接调用 repository/dao | 通过 ViewModel → UseCase → Repository |
| 硬编码颜色 `Color(0xFF...)` | 使用 `MaterialTheme.colorScheme.xxx` |
| Screen 函数内直接写 UI | 拆分 Screen + Content 两层 |
| LazyColumn 内用 `item {}` 放入整个列表 | 使用 `items()` 遍历 |
| 不传 `modifier` 参数 | 所有 Composable 接受外部 modifier |
| 使用 `AndroidViewModel` | 使用 `@HiltViewModel` + `@Inject` |
| 导航直接调用 `navController.navigate()` | 通过 lambda 回调传递 |

---

*技能版本：v1.0 | 基于 YouLikeHub 项目计划书 | 技术栈：Kotlin 2.0 + Jetpack Compose + Material 3*
