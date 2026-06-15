---
name: navigation
description: Compose Navigation 导航设计与实现。当用户请求设置页面路由、配置导航图、处理深层链接、实现页面转场动画、类型安全导航时触发。适用于 ShoppingHub 项目的导航系统。
globs: ["**/navigation/**/*.kt", "**/*NavGraph*.kt", "**/*Route*.kt", "**/*Navigation*.kt"]
---

# Compose Navigation 专家 — ShoppingHub

## 角色定位

你是一名 Jetpack Compose Navigation 专家，专精于类型安全路由、深层链接处理、导航动画和返回栈管理。你为本项目提供导航设计和实现服务。

## 项目导航标准

```
框架:     Compose Navigation (类型安全路由)
依赖:     androidx.navigation:navigation-compose
集成:     hilt-navigation-compose (Hilt ViewModel 支持)
最低版本: Navigation 2.8+ (类型安全路由正式版)
```

---

## 一、路由定义（类型安全）

```kotlin
// 文件: feature-collection/src/.../CollectionRoute.kt

import kotlinx.serialization.Serializable

/**
 * 收藏模块 — 路由定义
 *
 * 使用 @Serializable 实现类型安全的参数传递
 */
sealed interface CollectionRoute {

    /** 收藏列表页 */
    @Serializable
    data object List : CollectionRoute

    /** 商品详情页 */
    @Serializable
    data class Detail(val itemId: String) : CollectionRoute

    /** 编辑页（可选参数用默认值） */
    @Serializable
    data class Edit(
        val itemId: String,
        val autoFocusField: String? = null,
    ) : CollectionRoute

    /** 批量选择模式 */
    @Serializable
    data object BatchSelect : CollectionRoute
}
```

### 全应用路由汇总

```kotlin
// 文件: app/src/.../navigation/ShoppingHubRoutes.kt

@Serializable
sealed interface AppRoute {
    @Serializable data object Home : AppRoute
    @Serializable data object Collection : AppRoute
    @Serializable data object Cart : AppRoute
    @Serializable data object Orders : AppRoute
    @Serializable data object Wishlist : AppRoute
    @Serializable data object Search : AppRoute
    @Serializable data object Settings : AppRoute
}

@Serializable
sealed interface SettingsRoute {
    @Serializable data object Main : SettingsRoute
    @Serializable data object Privacy : SettingsRoute
    @Serializable data object About : SettingsRoute
}
```

---

## 二、NavHost 配置

```kotlin
// 文件: app/src/.../navigation/ShoppingHubNavHost.kt

@Composable
fun ShoppingHubNavHost(
    navController: NavHostController,
    modifier: Modifier = Modifier,
) {
    NavHost(
        navController = navController,
        startDestination = AppRoute.Home,
        modifier = modifier,
    ) {
        // ===== 底栏页面 =====
        composable<AppRoute.Home> {
            HomeScreen(
                onNavigateToCollection = {
                    navController.navigate(AppRoute.Collection)
                },
                onNavigateToItemDetail = { itemId ->
                    navController.navigate(CollectionRoute.Detail(itemId))
                },
            )
        }

        composable<AppRoute.Collection> {
            CollectionScreen(
                onItemClick = { itemId ->
                    navController.navigate(CollectionRoute.Detail(itemId))
                },
                onBatchSelect = {
                    navController.navigate(CollectionRoute.BatchSelect)
                },
            )
        }

        // ===== 详情页（非底栏） =====
        composable<CollectionRoute.Detail>(
            enterTransition = {
                slideIntoContainer(SlideDirection.Start, tween(300))
            },
            exitTransition = {
                slideOutOfContainer(SlideDirection.Start, tween(300))
            },
            popEnterTransition = {
                slideIntoContainer(SlideDirection.End, tween(300))
            },
            popExitTransition = {
                slideOutOfContainer(SlideDirection.End, tween(300))
            },
        ) { backStackEntry ->
            val route: CollectionRoute.Detail = backStackEntry.toRoute()
            ItemDetailScreen(
                itemId = route.itemId,
                onNavigateBack = { navController.popBackStack() },
                onNavigateToEdit = {
                    navController.navigate(CollectionRoute.Edit(route.itemId))
                },
            )
        }

        // ===== 编辑页 =====
        composable<CollectionRoute.Edit> { backStackEntry ->
            val route: CollectionRoute.Edit = backStackEntry.toRoute()
            EditItemScreen(
                itemId = route.itemId,
                autoFocusField = route.autoFocusField,
                onNavigateBack = { navController.popBackStack() },
                onSaved = { navController.popBackStack() },
            )
        }
    }
}
```

---

## 三、底部导航栏

```kotlin
// 文件: app/src/.../navigation/BottomNavBar.kt

/**
 * 底部导航栏目标定义
 */
enum class BottomNavDestination(
    val route: AppRoute,
    val label: String,
    val selectedIcon: ImageVector,
    val unselectedIcon: ImageVector,
) {
    HOME(
        route = AppRoute.Home,
        label = "首页",
        selectedIcon = Icons.Filled.Home,
        unselectedIcon = Icons.Outlined.Home,
    ),
    COLLECTION(
        route = AppRoute.Collection,
        label = "收藏",
        selectedIcon = Icons.Filled.Bookmarks,
        unselectedIcon = Icons.Outlined.Bookmarks,
    ),
    CART(
        route = AppRoute.Cart,
        label = "购物车",
        selectedIcon = Icons.Filled.ShoppingCart,
        unselectedIcon = Icons.Outlined.ShoppingCart,
    ),
    ORDERS(
        route = AppRoute.Orders,
        label = "订单",
        selectedIcon = Icons.Filled.Receipt,
        unselectedIcon = Icons.Outlined.Receipt,
    ),
    PROFILE(
        route = AppRoute.Settings,
        label = "我的",
        selectedIcon = Icons.Filled.Person,
        unselectedIcon = Icons.Outlined.Person,
    ),
}

@Composable
fun ShoppingHubBottomBar(
    navController: NavHostController,
    modifier: Modifier = Modifier,
) {
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route

    NavigationBar(modifier = modifier) {
        BottomNavDestination.entries.forEach { destination ->
            val isSelected = currentRoute == destination.route::class.qualifiedName

            NavigationBarItem(
                selected = isSelected,
                onClick = {
                    navController.navigate(destination.route) {
                        // 弹出到起始目的地，避免重复叠加
                        popUpTo(navController.graph.findStartDestination().id) {
                            saveState = true
                        }
                        launchSingleTop = true
                        restoreState = true
                    }
                },
                icon = {
                    Icon(
                        imageVector = if (isSelected) destination.selectedIcon
                                      else destination.unselectedIcon,
                        contentDescription = destination.label,
                    )
                },
                label = { Text(destination.label) },
            )
        }
    }
}
```

---

## 四、深层链接（Deep Link）

### URL 映射定义

```kotlin
// 支持的深层链接格式:
// shoppinghub://item/{itemId}           → 商品详情页
// shoppinghub://import?url={shareUrl}   → 数据导入页
// shoppinghub://search?q={query}        → 搜索页
// https://shoppinghub.com/item/{itemId} → Web URL 映射（可选）
```

### NavGraph 注册深层链接

```kotlin
composable<CollectionRoute.Detail>(
    deepLinks = listOf(
        navDeepLink<CollectionRoute.Detail>(
            basePath = "shoppinghub://item",
        ),
        // Web URL 映射
        navDeepLink<CollectionRoute.Detail>(
            basePath = "https://shoppinghub.com/item",
        ),
    ),
) { backStackEntry ->
    val route: CollectionRoute.Detail = backStackEntry.toRoute()
    ItemDetailScreen(itemId = route.itemId, ...)
}
```

### AndroidManifest 配置

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="shoppinghub"
            android:host="item" />
    </intent-filter>
</activity>
```

---

## 五、跨模块导航（推荐方式）

```kotlin
// ✅ 接口回调方式（避免 feature 模块间直接依赖）
// 在 app 模块组装导航回调：

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    // 全局导航回调接口
    val globalNavigator = remember(navController) {
        object : GlobalNavigator {
            override fun navigateToItem(itemId: String) {
                navController.navigate(CollectionRoute.Detail(itemId))
            }
            override fun navigateToSearch(query: String) {
                navController.navigate(SearchRoute.Result(query))
            }
            override fun navigateBack() {
                navController.popBackStack()
            }
        }
    }

    CompositionLocalProvider(
        LocalGlobalNavigator provides globalNavigator,
    ) {
        ShoppingHubNavHost(navController = navController)
    }
}

// ✅ 在任意 feature 模块中使用:
@Composable
fun SomeScreen() {
    val navigator = LocalGlobalNavigator.current
    // navigator.navigateToItem("item-123")
}
```

---

## 六、返回栈管理

```kotlin
// ===== 场景 1: 登录后返回之前页面 =====
navController.navigate(LoginRoute) {
    // 不 pop，等登录成功后手动处理
}

// 登录成功后:
navController.popBackStack()  // 关闭登录页
navController.navigate(SettingsRoute)  // 导航到目标页

// ===== 场景 2: 从详情页回到首页（跨多层返回）=====
navController.navigate(AppRoute.Home) {
    popUpTo(navController.graph.findStartDestination().id) {
        inclusive = true  // 包括首页本身也弹出重建
    }
}

// ===== 场景 3: 设置 → 隐私 → 关于 → 返回设置 =====
// Compose Navigation 自动维护返回栈，popBackStack() 即可
navController.popBackStack()
```

---

## 七、导航与 ViewModel 集成

```kotlin
// ✅ 使用 hiltViewModel() 在 Navigation 目的地内获取 ViewModel
composable<CollectionRoute.Detail> { backStackEntry ->
    val route: CollectionRoute.Detail = backStackEntry.toRoute()

    // 使用 backStackEntry 作为 ViewModelStoreOwner
    val viewModel: ItemDetailViewModel = hiltViewModel(backStackEntry)

    ItemDetailScreen(
        itemId = route.itemId,
        viewModel = viewModel,
        ...
    )
}

// ✅ 共享 ViewModel（同一 NavGraph 内）
// 使用父级的 backStackEntry
composable<CollectionRoute.Detail> { backStackEntry ->
    val parentEntry = remember(backStackEntry) {
        navController.getBackStackEntry(CollectionRoute.List)
    }
    val sharedViewModel: CollectionSharedViewModel = hiltViewModel(parentEntry)

    ...
}
```

---

## 八、输出格式

### 设计导航时

```
## 🗺️ 导航图
- 列出所有路由和它们的参数

## 📁 文件清单
- app/src/.../navigation/ShoppingHubNavHost.kt
- feature-xxx/src/.../XxxRoute.kt

## 🔀 页面流转
- [描述页面之间的跳转关系]
```

### 审查导航时

```
## 🔴 导航问题（返回栈泄漏、深层链接冲突）
## 🟡 设计改进（路由组织、参数类型优化、动画）
## 🔵 用户体验建议
```
