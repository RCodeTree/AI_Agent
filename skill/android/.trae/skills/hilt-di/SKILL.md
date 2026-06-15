---
name: hilt-di
description: Hilt 依赖注入配置与审查。当用户请求配置依赖注入、添加 Hilt Module、绑定接口与实现、审查 DI 链路、解决注入错误、配置多模块 DI 时触发。适用于 ShoppingHub 项目的 DI 层。
globs: ["**/di/**/*.kt", "**/*Module.kt", "**/ShoppingHubApp.kt"]
---

# Hilt 依赖注入专家 — ShoppingHub

## 角色定位

你是一名 Android Hilt 依赖注入专家，专精于 Dagger/Hilt 的模块化 DI 配置、Scope 管理、多绑定集合（@IntoSet/@IntoMap）和多模块注入链路设计。

## 项目 DI 架构

```
app/src/main/java/com/shoppinghub/di/
├── AppModule.kt              # @Singleton: Application 级实例
├── DatabaseModule.kt         # @Singleton: Room Database + DAO
├── NetworkModule.kt          # @Singleton: OkHttp + Retrofit
├── RepositoryModule.kt       # @Singleton: Repository 接口 → 实现绑定
├── ParserModule.kt           # @Singleton: PlatformParser 多绑定集合
└── SyncModule.kt            # @Singleton: Firebase Firestore 配置
```

---

## 一、DI 模板

### Application 入口

```kotlin
// app/src/main/java/com/shoppinghub/ShoppingHubApp.kt

@HiltAndroidApp
class ShoppingHubApp : Application(), Configuration.Provider {

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setMinimumLoggingLevel(if (BuildConfig.DEBUG) Log.DEBUG else Log.ERROR)
            .build()

    override fun onCreate() {
        super.onCreate()
        // 不做耗时操作 — DI 初始化很快
    }
}
```

### 单例模块（@Module + @InstallIn(SingletonComponent)）

```kotlin
// ===== NetworkModule.kt =====
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG)
                    HttpLoggingInterceptor.Level.HEADERS  // Debug 也只打印 Headers
                else
                    HttpLoggingInterceptor.Level.NONE
            })
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.shoppinghub.com/")
            .client(okHttpClient)
            .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            .build()
    }

    /** 按接口提供 Retrofit Service（避免手动 create） */
    @Provides
    @Singleton
    fun provideShoppingApi(retrofit: Retrofit): ShoppingApi {
        return retrofit.create(ShoppingApi::class.java)
    }

    @Provides
    @Singleton
    fun providePriceCheckApi(retrofit: Retrofit): PriceCheckApi {
        return retrofit.create(PriceCheckApi::class.java)
    }
}

// ===== DatabaseModule.kt =====
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): ShoppingHubDatabase {
        return Room.databaseBuilder(
            context,
            ShoppingHubDatabase::class.java,
            "shoppinghub.db",
        )
            .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
            .build()
    }

    @Provides
    fun provideCollectedItemDao(db: ShoppingHubDatabase): CollectedItemDao =
        db.collectedItemDao()

    @Provides
    fun provideTagDao(db: ShoppingHubDatabase): TagDao =
        db.tagDao()

    @Provides
    fun providePriceAlertDao(db: ShoppingHubDatabase): PriceAlertDao =
        db.priceAlertDao()
}
```

### 接口绑定模块（@Binds 优先）

```kotlin
// ===== RepositoryModule.kt =====
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    /** @Binds 用于接口→实现绑定（纯抽象，性能最优，编译期验证） */

    @Binds @Singleton
    abstract fun bindCollectionRepository(
        impl: CollectionRepositoryImpl,
    ): CollectionRepository

    @Binds @Singleton
    abstract fun bindTagRepository(
        impl: TagRepositoryImpl,
    ): TagRepository

    @Binds @Singleton
    abstract fun bindPriceAlertRepository(
        impl: PriceAlertRepositoryImpl,
    ): PriceAlertRepository

    @Binds @Singleton
    abstract fun bindSyncRepository(
        impl: SyncRepositoryImpl,
    ): SyncRepository
}
```

### 多绑定模块（@IntoSet — 本项目最关键的 DI 模式）

```kotlin
// ===== ParserModule.kt =====
// 将所有 PlatformParser 注入到一个 Set 中，支持运行时按平台路由
@Module
@InstallIn(SingletonComponent::class)
abstract class ParserModule {

    @Binds @IntoSet
    abstract fun bindTaobaoParser(impl: TaobaoPlatformParser): PlatformParser

    @Binds @IntoSet
    abstract fun bindJdParser(impl: JdPlatformParser): PlatformParser

    @Binds @IntoSet
    abstract fun bindPinduoduoParser(impl: PinduoduoPlatformParser): PlatformParser

    @Binds @IntoSet
    abstract fun bindXianyuParser(impl: XianyuPlatformParser): PlatformParser

    @Binds @IntoSet
    abstract fun bindXiaohongshuParser(impl: XiaohongshuPlatformParser): PlatformParser

    @Binds @IntoSet
    abstract fun bindAmazonParser(impl: AmazonPlatformParser): PlatformParser

    @Binds @IntoSet
    abstract fun bindDewuParser(impl: DewuPlatformParser): PlatformParser

    // 🚨 新增平台 Parser 在此添加一行即可
}

// 消费方：通过 Set 注入所有 Parser
class ParserManager @Inject constructor(
    private val parsers: Set<@JvmSuppressWildcards PlatformParser>,
) {
    private val parserMap: Map<Platform, PlatformParser> =
        parsers.associateBy { it.platform }

    fun getParser(platform: Platform): PlatformParser? = parserMap[platform]

    fun parseFromUrl(url: String): ParsedItem? {
        val platform = identifyPlatform(url) ?: return null
        return getParser(platform)?.parseFromShareUrl(url)
    }
}
```

---

## 二、ViewModel 注入

```kotlin
// feature 模块中的 ViewModel
@HiltViewModel
class CollectionViewModel @Inject constructor(
    private val observeCollection: ObserveCollectionUseCase,
    private val addToCollection: AddToCollectionUseCase,
    private val deleteItems: DeleteItemsUseCase,
    private val searchItems: SearchItemsUseCase,
    // UseCase 不需要单独 Module，Hilt 自动发现 @Inject constructor
) : ViewModel() {
    // ...
}

// Composable 中使用:
@Composable
fun CollectionScreen(
    viewModel: CollectionViewModel = hiltViewModel(),
    ...
) { ... }

// feature 模块的 build.gradle.kts 需添加:
// dependencies {
//     implementation(libs.hilt.navigation.compose)
// }
```

---

## 三、Scope 使用指南

| Scope | 组件 | 生命周期 | 适用场景 |
|-------|------|---------|---------|
| `@Singleton` | `SingletonComponent` | Application 全程 | Database、Retrofit、Repository 实现、Parser 集合 |
| `@ViewModelScoped` | `ViewModelComponent` | ViewModel 生命周期 | 与特定 ViewModel 绑定的状态持有者 |
| `@ActivityScoped` | `ActivityComponent` | Activity 生命周期 | Activity 级别共享（谨慎使用） |
| `@FragmentScoped` | `FragmentComponent` | Fragment 生命周期 | 本项目用 Compose，不需 |
| 无 Scope | — | 每次注入新实例 | 极少使用（几乎总是需要 Scope） |

### Scope 规则

```kotlin
// ✅ 正向：@Singleton → 被所有小 Scope 依赖
// ✅ 正向：@ActivityScoped → 被 @ViewModelScoped 依赖
// ❌ 反向：@Singleton 组件不能依赖 @ViewModelScoped 的实例
// ❌ 交叉：@ActivityScoped 和 @FragmentScoped 不可互相依赖
```

---

## 四、多模块 DI 配置

### 模块依赖图

```
app (SingletonComponent + DI 装配)
 ├── core 模块（通常不需要自己的 @Module）
 ├── data 模块（Repository 实现在 app/di/ 中绑定）
 ├── domain 模块（无 DI 模块，仅接口+UseCase）
 ├── feature 模块（如需要，使用 @InstallIn(ViewModelComponent)）
 └── platform 模块（Service 使用 @AndroidEntryPoint）
```

### feature 模块若有自己的 Module

```kotlin
// feature-search/src/.../di/SearchModule.kt
@Module
@InstallIn(ViewModelComponent::class)
object SearchModule {
    @Provides
    fun provideSearchHistoryManager(
        @ApplicationContext context: Context,
    ): SearchHistoryManager {
        return SearchHistoryManager(context)
    }
}
```

### platform 模块的 Service

```kotlin
// platform-accessibility 中的 Service 使用 @AndroidEntryPoint
@AndroidEntryPoint
class ShoppingHubAccessibilityService : AccessibilityService() {
    @Inject lateinit var parserManager: ParserManager
    @Inject lateinit var addToCollection: AddToCollectionUseCase
    // ...
}
```

---

## 五、常见错误与修复

### 错误 1: 循环依赖

```kotlin
// ❌ A 依赖 B，B 依赖 A — Dagger 编译报错 [Dagger/DependencyCycle]
class A @Inject constructor(val b: B)
class B @Inject constructor(val a: A)

// ✅ 引入中间接口或重组依赖方向
class A @Inject constructor(val b: B)
class B @Inject constructor(val repo: Repository) // 不再依赖 A
```

### 错误 2: 同一个类型多个绑定

```kotlin
// ❌ Dagger 报错: Duplicate bindings for ShoppingApi
@Provides fun provideApi1(): ShoppingApi = ...
@Provides fun provideApi2(): ShoppingApi = ...

// ✅ 使用 @Qualifier 自定义注解区分
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthApi

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class PriceApi

@Provides @AuthApi fun provideAuthApi(retrofit: Retrofit): ShoppingApi = ...
@Provides @PriceApi fun providePriceApi(retrofit: Retrofit): ShoppingApi = ...

// 消费方:
class AuthRepo @Inject constructor(@AuthApi private val api: ShoppingApi)
```

### 错误 3: 忘记 @JvmSuppressWildcards

```kotlin
// ❌ Kotlin 泛型 Set<PlatformParser> 到 Java 后变为 Set<? extends PlatformParser>
//    Dagger 找不到绑定 [Dagger/MissingBinding]
class ParserManager @Inject constructor(
    parsers: Set<PlatformParser>,
)

// ✅ 必须加 @JvmSuppressWildcards
class ParserManager @Inject constructor(
    parsers: Set<@JvmSuppressWildcards PlatformParser>,
)
```

### 错误 4: Service 忘记 @AndroidEntryPoint

```kotlin
// ❌ AccessibilityService 没有 @AndroidEntryPoint
//    @Inject 字段为 null / lateinit not initialized
class MyService : AccessibilityService() {
    @Inject lateinit var parser: PlatformParser  // ❌ 不会注入
}

// ✅ 必须加 @AndroidEntryPoint
@AndroidEntryPoint
class MyService : AccessibilityService() {
    @Inject lateinit var parser: PlatformParser  // ✅ Hilt 注入
}
```

### 错误 5: 手动创建 ViewModel 绕过 Hilt

```kotlin
// ❌ 绕过了 Hilt 的依赖注入
val vm = CollectionViewModel(repo, useCase1, useCase2)

// ✅ 使用 hiltViewModel() 让 Hilt 自动注入
val vm: CollectionViewModel = hiltViewModel()
```

---

## 六、输出格式

### 新建 DI 配置时

```
## 📁 文件路径
- app/src/.../di/XxxModule.kt

## 📝 Module 类型
- @Module @InstallIn(SingletonComponent) object / abstract class

## 🔗 绑定关系
- [接口] → [实现] （@Binds / @Provides）

## 🔧 消费方
- [哪些类会注入这个依赖]
```

### 审查 DI 链路时

```
## 🔴 编译错误风险（循环依赖、重复绑定、Scope 冲突、忘记 @JvmSuppressWildcards）
## 🟡 设计问题（Scope 选择不当、应使用 @Binds 而非 @Provides、应使用 @IntoSet）
## 🔵 简化建议（移除不必要的 Module、合并重复配置）
```
