---
name: compose-component-cookbook
description: Android Compose 组件配方手册 — 常用 UI 组件的标准化实现模式，含 Material 3 适配、状态处理、无障碍支持
---

# compose-component-cookbook

可复用的 Jetpack Compose 组件模式与配方。当你需要编写单个 UI 组件时，参考本手册中的标准实现模式。

**触发词：** "写一个XX组件"、"封装XX"、"实现一个XX"、"XX怎么用Compose写"

---

## 组件编写总则

任何组件都遵循以下契约：

```kotlin
@Composable
fun ComponentName(
    // 1. 数据参数
    data: ComponentData,
    // 2. 事件回调
    onClick: () -> Unit = {},
    // 3. Modifier（第一个可选参数或紧跟数据参数）
    modifier: Modifier = Modifier,
    // 4. 样式变体（按需）
    style: ComponentStyle = ComponentStyle.Default,
) {
    // 实现中：
    // - 颜色使用 MaterialTheme.colorScheme.xxx
    // - 字体使用 MaterialTheme.typography.xxx
    // - 形状使用 MaterialTheme.shapes.xxx
    // - contentDescription 不为 null
}
```

**铁律：**
- 始终接受 `modifier` 参数——让调用方控制布局
- 颜色从 `MaterialTheme.colorScheme` 取，绝不硬编码
- 图片/图标必须有 `contentDescription`
- 提供至少一个 `@Preview` 函数

---

## 一、信息展示组件

### 1.1 商品卡片（CollectionItemCard）

列表/网格中展示收藏商品的卡片：

```kotlin
@Composable
fun CollectionItemCard(
    item: CollectionItemUiModel,
    onClick: () -> Unit,
    onFavoriteToggle: (() -> Unit)? = null,
    modifier: Modifier = Modifier,
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        shape = MaterialTheme.shapes.medium,
    ) {
        Column {
            // 商品图片
            AsyncImage(
                model = item.imageUrl,
                contentDescription = item.title,
                contentScale = ContentScale.Crop,
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(1f),
                placeholder = painterResource(R.drawable.placeholder_item),
                error = painterResource(R.drawable.placeholder_item),
            )
            // 信息区
            Column(modifier = Modifier.padding(12.dp)) {
                // 平台标签
                Surface(
                    color = MaterialTheme.colorScheme.secondaryContainer,
                    shape = MaterialTheme.shapes.small,
                ) {
                    Text(
                        text = item.platformName,
                        style = MaterialTheme.typography.labelSmall,
                        color = MaterialTheme.colorScheme.onSecondaryContainer,
                        modifier = Modifier.padding(horizontal = 8.dp, vertical = 2.dp),
                    )
                }
                Spacer(modifier = Modifier.height(4.dp))

                // 标题（最多2行）
                Text(
                    text = item.title,
                    style = MaterialTheme.typography.bodyMedium,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis,
                )
                Spacer(modifier = Modifier.height(4.dp))

                // 价格行
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Text(
                        text = item.priceDisplay,
                        style = MaterialTheme.typography.titleMedium,
                        color = MaterialTheme.colorScheme.error,
                    )
                    if (item.originalPriceDisplay != null) {
                        Spacer(modifier = Modifier.width(6.dp))
                        Text(
                            text = item.originalPriceDisplay,
                            style = MaterialTheme.typography.bodySmall,
                            color = MaterialTheme.colorScheme.onSurfaceVariant,
                            textDecoration = TextDecoration.LineThrough,
                        )
                    }
                }
            }
        }
    }
}
```

### 1.2 价格标签（PriceTag）

```kotlin
enum class PriceTagStyle { Default, Compact, Large }

@Composable
fun PriceTag(
    currentPrice: String,
    originalPrice: String? = null,
    modifier: Modifier = Modifier,
    style: PriceTagStyle = PriceTagStyle.Default,
) {
    val (currentStyle, originalStyle) = when (style) {
        PriceTagStyle.Default -> MaterialTheme.typography.titleMedium to MaterialTheme.typography.bodySmall
        PriceTagStyle.Compact -> MaterialTheme.typography.bodyMedium to MaterialTheme.typography.labelSmall
        PriceTagStyle.Large -> MaterialTheme.typography.headlineSmall to MaterialTheme.typography.bodyMedium
    }

    Row(modifier = modifier, verticalAlignment = Alignment.Bottom) {
        Text(
            text = currentPrice,
            style = currentStyle,
            color = MaterialTheme.colorScheme.error,
        )
        if (originalPrice != null) {
            Spacer(modifier = Modifier.width(6.dp))
            Text(
                text = originalPrice,
                style = originalStyle,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
                textDecoration = TextDecoration.LineThrough,
            )
        }
    }
}
```

### 1.3 平台图标（PlatformIcon）

```kotlin
@Composable
fun PlatformIcon(
    platform: Platform,
    modifier: Modifier = Modifier,
    size: Dp = 24.dp,
) {
    val icon = when (platform) {
        Platform.TAOBAO -> R.drawable.ic_platform_taobao
        Platform.JD -> R.drawable.ic_platform_jd
        Platform.PINDUODUO -> R.drawable.ic_platform_pinduoduo
        Platform.XIANYU -> R.drawable.ic_platform_xianyu
        Platform.XIAOHONGSHU -> R.drawable.ic_platform_xiaohongshu
        Platform.AMAZON -> R.drawable.ic_platform_amazon
        Platform.DEWU -> R.drawable.ic_platform_dewu
        else -> R.drawable.ic_platform_other
    }

    val label = when (platform) {
        Platform.TAOBAO -> "淘宝"
        Platform.JD -> "京东"
        Platform.PINDUODUO -> "拼多多"
        Platform.XIANYU -> "闲鱼"
        Platform.XIAOHONGSHU -> "小红书"
        Platform.AMAZON -> "亚马逊"
        Platform.DEWU -> "得物"
        else -> "其他平台"
    }

    Icon(
        painter = painterResource(id = icon),
        contentDescription = label,
        modifier = modifier.size(size),
        tint = Color.Unspecified, // 保留图标原始颜色
    )
}
```

---

## 二、状态组件

### 2.1 加载指示器（LoadingIndicator）

```kotlin
@Composable
fun LoadingIndicator(
    modifier: Modifier = Modifier,
    message: String? = "加载中...",
) {
    Box(
        modifier = modifier.fillMaxSize(),
        contentAlignment = Alignment.Center,
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            CircularProgressIndicator(
                color = MaterialTheme.colorScheme.primary,
            )
            if (message != null) {
                Spacer(modifier = Modifier.height(16.dp))
                Text(
                    text = message,
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                )
            }
        }
    }
}
```

### 2.2 错误内容（ErrorContent）

```kotlin
@Composable
fun ErrorContent(
    message: String,
    onRetry: (() -> Unit)? = null,
    modifier: Modifier = Modifier,
    illustration: @Composable (() -> Unit)? = null,
) {
    Box(
        modifier = modifier.fillMaxSize(),
        contentAlignment = Alignment.Center,
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            modifier = Modifier.padding(32.dp),
        ) {
            // 可自定义插图，默认显示错误图标
            if (illustration != null) {
                illustration()
            } else {
                Icon(
                    imageVector = Icons.Outlined.ErrorOutline,
                    contentDescription = null,
                    modifier = Modifier.size(64.dp),
                    tint = MaterialTheme.colorScheme.error.copy(alpha = 0.6f),
                )
            }
            Spacer(modifier = Modifier.height(16.dp))
            Text(
                text = message,
                style = MaterialTheme.typography.bodyLarge,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
                textAlign = TextAlign.Center,
            )
            if (onRetry != null) {
                Spacer(modifier = Modifier.height(16.dp))
                OutlinedButton(onClick = onRetry) {
                    Text("重试")
                }
            }
        }
    }
}
```

### 2.3 空内容（EmptyContent）

```kotlin
@Composable
fun EmptyContent(
    title: String,
    description: String? = null,
    action: @Composable (() -> Unit)? = null,
    modifier: Modifier = Modifier,
    illustration: @Composable (() -> Unit)? = null,
) {
    Box(
        modifier = modifier.fillMaxSize(),
        contentAlignment = Alignment.Center,
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            modifier = Modifier.padding(32.dp),
        ) {
            if (illustration != null) {
                illustration()
            } else {
                Icon(
                    imageVector = Icons.Outlined.Inbox,
                    contentDescription = null,
                    modifier = Modifier.size(72.dp),
                    tint = MaterialTheme.colorScheme.onSurfaceVariant.copy(alpha = 0.3f),
                )
            }
            Spacer(modifier = Modifier.height(16.dp))
            Text(
                text = title,
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onSurface,
            )
            if (description != null) {
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text = description,
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                    textAlign = TextAlign.Center,
                )
            }
            action?.let {
                Spacer(modifier = Modifier.height(16.dp))
                it()
            }
        }
    }
}
```

---

## 三、交互组件

### 3.1 搜索栏（SearchBar）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppSearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    onSearch: (String) -> Unit,
    placeholder: String = "搜索...",
    modifier: Modifier = Modifier,
) {
    SearchBar(
        query = query,
        onQueryChange = onQueryChange,
        onSearch = onSearch,
        active = false,
        onActiveChange = { },
        placeholder = { Text(placeholder) },
        modifier = modifier.fillMaxWidth(),
    ) {
        // 搜索建议/历史记录区域
        // 仅当 active=true 时显示，按需实现
    }
}
```

### 3.2 筛选标签组（FilterChipGroup）

```kotlin
@Composable
fun <T> FilterChipGroup(
    items: List<T>,
    selectedItem: T?,
    onItemSelected: (T) -> Unit,
    modifier: Modifier = Modifier,
    label: (T) -> String = { it.toString() },
) {
    LazyRow(
        modifier = modifier,
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        contentPadding = PaddingValues(horizontal = 16.dp),
    ) {
        items(items) { item ->
            FilterChip(
                selected = item == selectedItem,
                onClick = { onItemSelected(item) },
                label = { Text(label(item)) },
            )
        }
    }
}
```

### 3.3 确认对话框（ConfirmDialog）

```kotlin
@Composable
fun ConfirmDialog(
    title: String,
    message: String,
    confirmText: String = "确认",
    dismissText: String = "取消",
    onConfirm: () -> Unit,
    onDismiss: () -> Unit,
    isDestructive: Boolean = false,  // 删除等危险操作为 true
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(title) },
        text = { Text(message) },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text(
                    text = confirmText,
                    color = if (isDestructive) MaterialTheme.colorScheme.error
                            else MaterialTheme.colorScheme.primary,
                )
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text(dismissText)
            }
        },
    )
}
```

---

## 四、图片加载组件

### 4.1 AsyncImage 封装

```kotlin
@Composable
fun AppAsyncImage(
    model: Any?,
    contentDescription: String?,
    modifier: Modifier = Modifier,
    contentScale: ContentScale = ContentScale.Crop,
    placeholderRes: Int = R.drawable.placeholder_item,
    errorRes: Int = R.drawable.placeholder_item,
    crossfade: Boolean = true,
) {
    AsyncImage(
        model = model,
        contentDescription = contentDescription,
        contentScale = contentScale,
        modifier = modifier,
        placeholder = remember {
            Coil3Painter(placeholderRes)
        },
        error = remember {
            Coil3Painter(errorRes)
        },
    )
}
```

---

## 五、Section 模式

### 5.1 分区标题（SectionHeader）

```kotlin
@Composable
fun SectionHeader(
    title: String,
    modifier: Modifier = Modifier,
    actionLabel: String? = null,
    onActionClick: (() -> Unit)? = null,
) {
    Row(
        modifier = modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 8.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Text(
            text = title,
            style = MaterialTheme.typography.titleMedium,
        )
        if (actionLabel != null && onActionClick != null) {
            TextButton(onClick = onActionClick) {
                Text(actionLabel)
            }
        }
    }
}
```

---

## 六、YouLikeHub 业务组件

### 6.1 平台统计卡片（PlatformStatsCard）

```kotlin
@Composable
fun PlatformStatsCard(
    platformName: String,
    platformIcon: Int,
    itemCount: Int,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Card(
        onClick = onClick,
        modifier = modifier.width(140.dp),
        shape = MaterialTheme.shapes.medium,
    ) {
        Column(
            modifier = Modifier.padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
        ) {
            Icon(
                painter = painterResource(id = platformIcon),
                contentDescription = null,
                modifier = Modifier.size(32.dp),
                tint = Color.Unspecified,
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = "$itemCount",
                style = MaterialTheme.typography.headlineSmall,
            )
            Text(
                text = "件商品",
                style = MaterialTheme.typography.labelSmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
            )
            Text(
                text = platformName,
                style = MaterialTheme.typography.labelMedium,
            )
        }
    }
}
```

### 6.2 降价提醒卡片（PriceAlertCard）

```kotlin
@Composable
fun PriceAlertCard(
    itemTitle: String,
    currentPrice: String,
    originalPrice: String,
    dropPercent: Int,
    platformName: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.errorContainer.copy(alpha = 0.3f),
        ),
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            // 降价幅度标签
            Surface(
                color = MaterialTheme.colorScheme.error,
                shape = MaterialTheme.shapes.small,
            ) {
                Text(
                    text = "↓$dropPercent%",
                    style = MaterialTheme.typography.labelLarge,
                    color = MaterialTheme.colorScheme.onError,
                    modifier = Modifier.padding(horizontal = 8.dp, vertical = 4.dp),
                )
            }
            Spacer(modifier = Modifier.width(12.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = itemTitle,
                    style = MaterialTheme.typography.bodyMedium,
                    maxLines = 1,
                    overflow = TextOverflow.Ellipsis,
                )
                PriceTag(
                    currentPrice = currentPrice,
                    originalPrice = originalPrice,
                    style = PriceTagStyle.Compact,
                )
            }
        }
    }
}
```

---

## 预览规范

每个组件至少提供 Light + Dark 两种 Preview：

```kotlin
@Preview(name = "Light", group = "ComponentName", showBackground = true)
@Preview(name = "Dark", group = "ComponentName", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
private fun PreviewComponentName() {
    YouLikeHubTheme {
        ComponentName(
            data = ComponentData.Companion.preview(),
            onClick = {},
        )
    }
}
```

---

## 组件开发检查清单

- [ ] 接受 `modifier` 参数且正确传递到根 Composable
- [ ] 颜色使用 `MaterialTheme.colorScheme.xxx`
- [ ] 字体使用 `MaterialTheme.typography.xxx`
- [ ] 图标/图片有 `contentDescription`（装饰性图标用 `null`）
- [ ] 支持 `enabled` 状态（交互组件）
- [ ] 至少提供 Light + Dark Preview
- [ ] 不硬编码尺寸——用 `fillMaxWidth()`、`weight()`、`aspectRatio()` 等弹性布局
- [ ] 点击区域不小于 48dp × 48dp（Material 可访问性标准）

---

*技能版本：v1.0 | 基于 YouLikeHub 项目 | 技术栈：Jetpack Compose + Material 3 + Coil 3*
