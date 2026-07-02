---
name: compose-theme-config
description: Android Compose 主题与设计令牌配置 — Material 3 颜色/字体/形状系统、深色模式、Material You 动态取色
---

# compose-theme-config

Material Design 3 主题系统配置技能。用于设置应用的全局视觉风格——颜色方案、排版系统、形状定义、深色/浅色模式，以及 Material You 动态取色。

**触发词：** "配置主题"、"设置颜色"、"字体怎么配"、"深色模式"、"Material You"

---

## 快速配置流程

### Step 1: 确定配色策略

根据应用定位选择：

| 策略 | 适用场景 | 实现方式 |
|------|----------|----------|
| **Material You 动态取色** | 个性化优先，跟随系统壁纸 | `dynamicLightColorScheme()` / `dynamicDarkColorScheme()` |
| **自定义品牌色** | 有明确品牌色/设计规范 | `lightColorScheme()` / `darkColorScheme()` 手动定义 |
| **混合策略** | Android 12+ 用动态取色，旧版本用品牌色 | 运行时判断 `Build.VERSION.SDK_INT` |

**YouLikeHub 推荐：混合策略**——Android 12+ 使用 Material You 动态取色，旧版本回退到品牌蓝紫色方案。

### Step 2: 定义颜色令牌

在 `core-ui` 模块中定义：

```kotlin
// core-ui/src/main/java/com/youlikehub/core/ui/theme/Color.kt

import androidx.compose.ui.graphics.Color

// ========== 品牌色 ==========
// 主色 —— 蓝紫调，传达「智能」与「信赖」
val BrandPrimary = Color(0xFF6750A4)
val BrandOnPrimary = Color(0xFFFFFFFF)
val BrandPrimaryContainer = Color(0xFFEADDFF)
val BrandOnPrimaryContainer = Color(0xFF21005D)

// 辅色 —— 青色，用于强调与标签
val BrandSecondary = Color(0xFF625B71)
val BrandOnSecondary = Color(0xFFFFFFFF)
val BrandSecondaryContainer = Color(0xFFE8DEF8)
val BrandOnSecondaryContainer = Color(0xFF1D192B)

// 第三色 —— 暖橙，用于降价/促销/警示
val BrandTertiary = Color(0xFF7D5260)
val BrandOnTertiary = Color(0xFFFFFFFF)
val BrandTertiaryContainer = Color(0xFFFFD8E4)
val BrandOnTertiaryContainer = Color(0xFF31111D)

// ========== 语义色 ==========
val BrandError = Color(0xFFB3261E)
val BrandOnError = Color(0xFFFFFFFF)
val BrandErrorContainer = Color(0xFFF9DEDC)
val BrandOnErrorContainer = Color(0xFF410E0B)

// ========== 中性色（深色主题） ==========
val DarkSurface = Color(0xFF1C1B1F)
val DarkOnSurface = Color(0xFFE6E1E5)
val DarkSurfaceVariant = Color(0xFF49454F)
val DarkOnSurfaceVariant = Color(0xFFCAC4D0)
val DarkBackground = Color(0xFF1C1B1F)
val DarkOnBackground = Color(0xFFE6E1E5)

// ========== 中性色（浅色主题） ==========
val LightSurface = Color(0xFFFFFBFE)
val LightOnSurface = Color(0xFF1C1B1F)
val LightSurfaceVariant = Color(0xFFE7E0EC)
val LightOnSurfaceVariant = Color(0xFF49454F)
val LightBackground = Color(0xFFFFFBFE)
val LightOnBackground = Color(0xFF1C1B1F)
```

### Step 3: 定义颜色方案

```kotlin
// core-ui/src/main/java/com/youlikehub/core/ui/theme/ColorScheme.kt

import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme

val BrandLightColorScheme = lightColorScheme(
    primary = BrandPrimary,
    onPrimary = BrandOnPrimary,
    primaryContainer = BrandPrimaryContainer,
    onPrimaryContainer = BrandOnPrimaryContainer,
    secondary = BrandSecondary,
    onSecondary = BrandOnSecondary,
    secondaryContainer = BrandSecondaryContainer,
    onSecondaryContainer = BrandOnSecondaryContainer,
    tertiary = BrandTertiary,
    onTertiary = BrandOnTertiary,
    tertiaryContainer = BrandTertiaryContainer,
    onTertiaryContainer = BrandOnTertiaryContainer,
    error = BrandError,
    onError = BrandOnError,
    errorContainer = BrandErrorContainer,
    onErrorContainer = BrandOnErrorContainer,
    surface = LightSurface,
    onSurface = LightOnSurface,
    surfaceVariant = LightSurfaceVariant,
    onSurfaceVariant = LightOnSurfaceVariant,
    background = LightBackground,
    onBackground = LightOnBackground,
)

val BrandDarkColorScheme = darkColorScheme(
    primary = Color(0xFFD0BCFF),
    onPrimary = Color(0xFF381E72),
    primaryContainer = Color(0xFF4F378B),
    onPrimaryContainer = Color(0xFFEADDFF),
    secondary = Color(0xFFCCC2DC),
    onSecondary = Color(0xFF332D41),
    secondaryContainer = Color(0xFF4A4458),
    onSecondaryContainer = Color(0xFFE8DEF8),
    tertiary = Color(0xFFEFB8C8),
    onTertiary = Color(0xFF492532),
    tertiaryContainer = Color(0xFF633B48),
    onTertiaryContainer = Color(0xFFFFD8E4),
    error = Color(0xFFF2B8B5),
    onError = Color(0xFF601410),
    errorContainer = Color(0xFF8C1D18),
    onErrorContainer = Color(0xFFF9DEDC),
    surface = DarkSurface,
    onSurface = DarkOnSurface,
    surfaceVariant = DarkSurfaceVariant,
    onSurfaceVariant = DarkOnSurfaceVariant,
    background = DarkBackground,
    onBackground = DarkOnBackground,
)
```

### Step 4: 定义排版系统

```kotlin
// core-ui/src/main/java/com/youlikehub/core/ui/theme/Type.kt

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val YouLikeHubTypography = Typography(
    // 大标题：页面顶部主标题
    headlineLarge = TextStyle(
        fontWeight = FontWeight.Bold,
        fontSize = 28.sp,
        lineHeight = 36.sp,
        letterSpacing = 0.sp,
    ),
    // 中标题：区块标题
    headlineMedium = TextStyle(
        fontWeight = FontWeight.Bold,
        fontSize = 24.sp,
        lineHeight = 32.sp,
        letterSpacing = 0.sp,
    ),
    // 小标题
    headlineSmall = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 20.sp,
        lineHeight = 28.sp,
        letterSpacing = 0.sp,
    ),
    // 大标题（列表/卡片内）
    titleLarge = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 20.sp,
        lineHeight = 28.sp,
        letterSpacing = 0.sp,
    ),
    // 中标题（Section Header）
    titleMedium = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.15.sp,
    ),
    // 小标题
    titleSmall = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.1.sp,
    ),
    // 正文（大）
    bodyLarge = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp,
    ),
    // 正文（中）—— 默认正文
    bodyMedium = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.25.sp,
    ),
    // 正文（小）
    bodySmall = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 12.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.4.sp,
    ),
    // 标签（大）—— 按钮文字
    labelLarge = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.1.sp,
    ),
    // 标签（中）
    labelMedium = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 12.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp,
    ),
    // 标签（小）—— Chip 内文字、角标
    labelSmall = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 11.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp,
    ),
)
```

### Step 5: 定义形状系统

```kotlin
// core-ui/src/main/java/com/youlikehub/core/ui/theme/Shape.kt

import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Shapes
import androidx.compose.ui.unit.dp

val YouLikeHubShapes = Shapes(
    // 小组件：Chip、TextField、Snackbar
    small = RoundedCornerShape(8.dp),
    // 中组件：Card、Dialog、BottomSheet
    medium = RoundedCornerShape(12.dp),
    // 大组件：ModalDrawer、LargeCard
    large = RoundedCornerShape(16.dp),
)
```

### Step 6: 组装 Theme Composable

```kotlin
// core-ui/src/main/java/com/youlikehub/core/ui/theme/Theme.kt

import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.dynamicDarkColorScheme
import androidx.compose.material3.dynamicLightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalContext

@Composable
fun YouLikeHubTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,     // 是否启用 Material You 动态取色
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        // Android 12+ 且启用动态取色 → 使用系统壁纸色
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        // 回退到品牌自定义色
        darkTheme -> BrandDarkColorScheme
        else -> BrandLightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = YouLikeHubTypography,
        shapes = YouLikeHubShapes,
        content = content,
    )
}
```

### Step 7: 在 Application/Activity 中应用

```kotlin
// app/MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            YouLikeHubTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background,
                ) {
                    AppNavHost()
                }
            }
        }
    }
}
```

---

## 颜色使用规则

| 场景 | 正确用法 | 禁止用法 |
|------|----------|----------|
| 主色元素 | `MaterialTheme.colorScheme.primary` | `Color(0xFF6750A4)` |
| 页面背景 | `MaterialTheme.colorScheme.background` | `Color.White` |
| 卡片背景 | `MaterialTheme.colorScheme.surface` | 自定义色 |
| 标题文字 | `MaterialTheme.colorScheme.onSurface` | `Color.Black` |
| 次要文字 | `MaterialTheme.colorScheme.onSurfaceVariant` | `Color.Gray` |
| 错误/降价 | `MaterialTheme.colorScheme.error` | `Color.Red` |
| 分割线 | `MaterialTheme.colorScheme.outlineVariant` | 自定义灰 |

---

## 深色模式适配

### 图片在深色模式下的处理

```kotlin
// 商品图片不随主题变化
AsyncImage(
    model = item.imageUrl,
    contentDescription = item.title,
    // 不设置 colorFilter，保持原始色彩
)

// 装饰性图标可跟随主题
Icon(
    painter = painterResource(id = R.drawable.ic_decor),
    contentDescription = null,
    tint = MaterialTheme.colorScheme.onSurfaceVariant,
)
```

### 卡片在深色模式下的处理

```kotlin
Card(
    colors = CardDefaults.cardColors(
        // 使用 surface 或 surfaceVariant，自动适配深浅模式
        containerColor = MaterialTheme.colorScheme.surface,
    ),
) { ... }
```

---

## 主题配置检查清单

- [ ] Color.kt 中定义了完整的 Light 和 Dark 颜色令牌
- [ ] Typography 覆盖了全部 15 个 TextStyle
- [ ] Shapes 定义了 small / medium / large 三档
- [ ] Theme composable 正确处理了 Material You 和品牌色回退
- [ ] 没有任何 Composable 中硬编码颜色
- [ ] 深色模式下文字对比度 ≥ 4.5:1
- [ ] `contentDescription` 不为 null（除装饰性元素外）
- [ ] 提供至少一个 Light + Dark 的完整页面 Preview

---

*技能版本：v1.0 | 基于 YouLikeHub 项目 | 设计系统：Material Design 3*
