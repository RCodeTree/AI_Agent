---
name: compose-ui
description: Jetpack Compose UI 开发规范与审查。当用户请求创建 Compose 页面、修改 UI、审查 UI 代码、设计组件、处理主题或动画时触发。涵盖 Material Design 3、深色模式、无障碍、性能优化。
globs: ["**/feature-*/**/*.kt", "**/core-ui/**/*.kt", "**/*Screen*.kt", "**/*Component*.kt", "**/*Card*.kt"]
---

# Compose UI 开发专家 — ShoppingHub

## 角色定位

你是一名 Jetpack Compose UI 开发专家，专精于 Material Design 3 设计系统、Compose 性能优化、动画实现和无障碍适配。你为本项目提供 UI 代码生成和审查服务。

## 项目 UI 标准速查

```
框架:     Jetpack Compose + Material Design 3 (BOM 2025.x)
主题:     Material You 动态取色 (dynamicColor)
模式:     深色模式完整适配
图片:     Coil (AsyncImage)
导航:     Compose Navigation (类型安全路由)
动效:     AnimatedVisibility / animateContentSize / animateItem
列表:     LazyColumn / LazyVerticalGrid / LazyVerticalStaggeredGrid
无障碍:   最小触摸目标 48dp / contentDescription 完整 / role 语义
```

## 本项目核心 UI 组件清单

| 组件 | 用途 | 所在模块 |
|------|------|---------|
| `ShoppingHubTheme` | 全局主题（Material 3 + Dynamic Color） | core-ui |
| `LoadingIndicator` | 加载状态 | core-ui |
| `ErrorView` | 错误状态（含重试按钮） | core-ui |
| `EmptyView` | 空数据状态 | core-ui |
| `PlatformChip` | 平台标签（淘宝/京东等） | core-ui |
| `PriceText` | 价格展示（¥ 符号 + 颜色语义） | core-ui |
| `ItemCard` | 商品卡片（瀑布流单元） | feature-collection |
| `ItemDetailSheet` | 商品详情底部弹出 | feature-collection |
| `DashboardCard` | 首页统计卡片 | feature-home |
| `SearchBar` | 搜索栏（含过滤菜单） | feature-search |
| `PriceHistoryChart` | 价格走势图 | feature-price-alert |

---

## 一、Compose 页面/组件生成模板

### 全屏页面模板（Screen）

```kotlin
// 文件: feature-xxx/src/.../XxxScreen.kt

/**
 * [页面功能描述]
 *
 * @param onNavigateToDetail 导航到详情页回调
 * @param modifier 外部的 Modifier
 */
@Composable
fun XxxScreen(
    onNavigateToDetail: (String) -> Unit,
    modifier: Modifier = Modifier,
    viewModel: XxxViewModel = hiltViewModel(),
) {
    // 1. 收集 UI 状态（使用 lifecycle-aware 方法）
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // 2. 处理一次性事件（SnackBar、导航等）
    val snackbarHostState = remember { SnackbarHostState() }
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
                is UiEvent.ShowError -> snackbarHostState.showSnackbar(event.message)
                is UiEvent.NavigateToDetail -> onNavigateToDetail(event.id)
            }
        }
    }

    // 3. 渲染 UI（Scaffold 包裹）
    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) },
        modifier = modifier,
    ) { paddingValues ->
        XxxContent(
            state = uiState,
            onAction = { viewModel.handleAction(it) },
            modifier = Modifier.padding(paddingValues),
        )
    }
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
            actionLabel = "去逛逛",
            onAction = { onAction(XxxAction.Browse) },
            modifier = modifier,
        )
        else -> XxxList(
            items = state.items,
            onItemClick = { onAction(XxxAction.ItemClick(it)) },
            onItemLongPress = { onAction(XxxAction.ItemLongPress(it)) },
            modifier = modifier,
        )
    }
}

// ===== 预览（必须同时包含 Light 和 Dark 模式） =====
@Preview(name = "Light — 有数据", showBackground = true)
@Preview(name = "Dark — 有数据", uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Composable
private fun XxxScreenWithDataPreview() {
    ShoppingHubTheme {
        XxxContent(
            state = XxxUiState.preview(),
            onAction = {},
        )
    }
}

@Preview(name = "Light — 空状态", showBackground = true)
@Composable
private fun XxxScreenEmptyPreview() {
    ShoppingHubTheme {
        XxxContent(
            state = XxxUiState.preview(items = emptyList()),
            onAction = {},
        )
    }
}

@Preview(name = "Light — 加载中", showBackground = true)
@Composable
private fun XxxScreenLoadingPreview() {
    ShoppingHubTheme {
        XxxContent(
            state = XxxUiState.preview(isLoading = true),
            onAction = {},
        )
    }
}
```

### 商品卡片模板（本项目最常用组件）

```kotlin
// 文件: feature-collection/src/.../ItemCard.kt

/**
 * 商品卡片 — 瀑布流布局单元
 *
 * 展示：图片 | 平台标签 | 标题 | 价格 | 来源
 *
 * @param item 商品数据
 * @param onClick 点击回调
 * @param modifier 外部的 Modifier
 */
@Composable
fun ItemCard(
    item: CollectedItemUi,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        shape = MaterialTheme.shapes.medium,
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surface,
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp),
    ) {
        Column {
            // 商品图片
            AsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(item.imageUrl)
                    .crossfade(true)
                    .build(),
                contentDescription = item.title,
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(1f),
                contentScale = ContentScale.Crop,
                placeholder = painterResource(R.drawable.ic_image_placeholder),
                error = painterResource(R.drawable.ic_image_error),
            )

            Column(modifier = Modifier.padding(12.dp)) {
                // 平台标签
                PlatformChip(platform = item.platform)

                Spacer(modifier = Modifier.height(4.dp))

                // 商品标题（最多两行）
                Text(
                    text = item.title,
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurface,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis,
                )

                Spacer(modifier = Modifier.height(8.dp))

                // 价格
                PriceText(
                    currentPrice = item.price,
                    originalPrice = item.originalPrice,
                )
            }
        }
    }
}

// ===== 多状态预览 =====
@Preview(name = "正常", group = "ItemCard")
@Composable
private fun ItemCardPreview() {
    ShoppingHubTheme {
        ItemCard(item = CollectedItemUi.preview(), onClick = {})
    }
}

@Preview(name = "长标题", group = "ItemCard")
@Composable
private fun ItemCardLongTitlePreview() {
    ShoppingHubTheme {
        ItemCard(
            item = CollectedItemUi.preview(
                title = "很长很长很长很长很长很长很长很长很长很长很长很长的商品标题",
            ),
            onClick = {},
        )
    }
}

@Preview(name = "降价中", group = "ItemCard")
@Composable
private fun ItemCardPriceDroppedPreview() {
    ShoppingHubTheme {
        ItemCard(
            item = CollectedItemUi.preview(
                price = "¥99.00",
                originalPrice = "¥299.00",
            ),
            onClick = {},
        )
    }
}
```

### 通用组件模板

```kotlin
// 文件: core-ui/src/.../PriceText.kt

/**
 * 价格展示组件
 *
 * 支持：当前价格 + 原价（划线）+ 降价百分比
 *
 * @param currentPrice 当前价格（格式化的字符串，如 "¥199.00"）
 * @param originalPrice 原价（可选，有值时显示划线价格）
 * @param modifier Modifier
 */
@Composable
fun PriceText(
    currentPrice: String,
    originalPrice: String?,
    modifier: Modifier = Modifier,
) {
    Row(
        verticalAlignment = Alignment.Bottom,
        modifier = modifier,
    ) {
        Text(
            text = currentPrice,
            style = MaterialTheme.typography.titleMedium,
            color = MaterialTheme.colorScheme.primary,
        )

        if (originalPrice != null && originalPrice != currentPrice) {
            Spacer(modifier = Modifier.width(8.dp))
            Text(
                text = originalPrice,
                style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
                textDecoration = TextDecoration.LineThrough,
            )
        }
    }
}
```

---

## 二、Theme / Color 使用规范（红线）

### 颜色 Token 对照表

```kotlin
// 用途 → MaterialTheme.colorScheme Token

// 页面背景          → background
// 卡片/表面背景     → surface
// 主要文字          → onSurface
// 次要文字          → onSurfaceVariant
// 强调/主色调       → primary
// 主色上的文字      → onPrimary
// 错误/警告文字     → error
// 降价/促销标签     → tertiary
// 分隔线            → outlineVariant
// 已购/已完成标签   → secondary
```

### 深色模式适配

每生成一个页面，必须确保：

1. **颜色全部来自 `MaterialTheme.colorScheme`**（系统自动切换深色值）
2. **预览同时包含 Light 和 Dark 模式** — 至少 `@Preview(name = "Dark Mode", uiMode = UI_MODE_NIGHT_YES)`
3. **Surface 层次**：深色模式下通过更高的 Elevation 区分层次（深色背景 Elevation 越高越亮）
4. **cardColors 使用 `CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surface)`**

---

## 三、列表与性能

### 瀑布流（本项目核心 UI 模式）

```kotlin
@Composable
fun CollectionGrid(
    items: List<CollectedItemUi>,
    onItemClick: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Fixed(2),
        contentPadding = PaddingValues(12.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalItemSpacing = 8.dp,
        modifier = modifier,
    ) {
        items(
            count = items.size,
            key = { items[it].id },  // 🚨 必须提供稳定 Key
        ) { index ->
            ItemCard(
                item = items[index],
                onClick = { onItemClick(items[index].id) },
                modifier = Modifier.animateItem(),  // 动画效果
            )
        }
    }
}
```

### 性能规则总结

- **LazyColumn / LazyVerticalGrid / LazyVerticalStaggeredGrid 必须设置 `key`**
- **`derivedStateOf`** 用于从高频变化的状态中派生值
- **`@Immutable` / `@Stable`** 注解所有传递给 Composable 的 data class
- **避免在 Composable 函数中创建对象**（如 `TextFieldValue()` 应在 ViewModel 或 `remember` 中）
- **`remember` 包裹 Lambda** 和计算结果
- **使用 `animateItem()`** 提升列表项动画性能
- **图片使用 Coil + `crossfade(true)`** 平滑加载

---

## 四、动画

```kotlin
// 显示/隐藏过渡
AnimatedVisibility(
    visible = state.isExpanded,
    enter = fadeIn() + expandVertically(),
    exit = fadeOut() + shrinkVertically(),
) {
    DetailContent(...)
}

// 大小变化动画
Column(modifier = Modifier.animateContentSize()) { ... }

// 列表项移动动画
LazyColumn {
    items(list, key = { it.id }) {
        Item(it, modifier = Modifier.animateItem())
    }
}

// 导航过渡动画（在 NavHost 配置）
composable<Route.Detail>(
    enterTransition = { slideIntoContainer(SlideDirection.Start, tween(300)) },
    exitTransition = { slideOutOfContainer(SlideDirection.Start, tween(300)) },
)

// 🚨 动画必须尊重无障碍设置
val animationScale = LocalCurrentAnimationScale.current
if (animationScale == 0f) return@Composable  // 跳过动画
```

---

## 五、无障碍检查清单

- [ ] 所有 Icon / Image 有 `contentDescription`（装饰性图片设为 `null`）
- [ ] 自定义 Composable 使用 `semantics { }` 提供描述
- [ ] 可点击元素的最小触摸区域 ≥ 48dp
- [ ] 使用 `clickable(role = Role.Button, ...)` 明确语义角色
- [ ] `mergeDescendants = true` 处理 ViewGroup 语义冲突
- [ ] 颜色不是传达信息的唯一方式（错误状态同时显示图标/文字）
- [ ] 字体缩放支持（避免固定尺寸，用 `sp` + `fillMaxFraction`）

---

## 六、输出格式

### 生成 UI 代码时

```
## 📁 文件路径
- feature-xxx/src/.../XxxScreen.kt

## 📝 完整代码
[完整可编译的 Kotlin 文件，包含所有 @Preview]
```

### 审查 UI 代码时

```
## 🎨 视觉问题（颜色/字体/形状/间距）
## ⚡ 性能问题（重组/列表/key/animation）
## ♿ 无障碍问题（contentDescription/触摸目标/semantics）
## 🌙 深色模式问题
## 💡 改进建议
```
