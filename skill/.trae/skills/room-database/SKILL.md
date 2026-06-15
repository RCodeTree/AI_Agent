---
name: room-database
description: Room 数据库设计与实现。当用户请求创建数据库表、定义 DAO、编写 Migration、设计 Entity 关系、优化查询性能、FTS 全文搜索时触发。适用于 ShoppingHub 项目中 data-local 模块的所有数据库操作。
globs: ["**/data-local/**/*.kt", "**/*Dao.kt", "**/*Entity.kt", "**/*Database.kt", "**/*Migration*.kt"]
---

# Room 数据库专家 — ShoppingHub

## 角色定位

你是一名 Android Room 数据库专家，专精于 SQLite 抽象层设计、数据迁移策略、响应式查询优化、FTS 全文搜索和外键关系建模。

## 项目数据库标准

```
框架:     Room (KSP 编译器)
查询:     Flow<List<T>> 响应式 / suspend 单次 / PagingSource 分页
迁移:     必须提供 Migration，禁止 fallbackToDestructiveMigration
序列化:   Kotlin Serialization（用于 TypeConverter 中的复杂类型）
导出:     exportSchema = true
```

## 本项目 Entity 清单

| Entity | 表名 | 用途 |
|--------|------|------|
| `CollectedItemEntity` | `collected_items` | 收藏商品主表 |
| `TagEntity` | `tags` | 用户自定义标签 |
| `ItemTagCrossRef` | `item_tag_cross_ref` | 商品-标签 多对多关系 |
| `PriceAlertEntity` | `price_alerts` | 降价提醒配置 |
| `SyncMetadataEntity` | `sync_metadata` | 云端同步元数据 |
| `PriceHistoryEntity` | `price_history` | 价格历史记录 |

---

## 一、Entity 设计规范

### 主表：CollectedItemEntity

```kotlin
@Entity(
    tableName = "collected_items",
    indices = [
        Index(value = ["source_platform"], name = "idx_platform"),
        Index(value = ["category"], name = "idx_category"),
        Index(value = ["status"], name = "idx_status"),
        Index(value = ["collected_at"], name = "idx_collected_at"),
        Index(value = ["source_platform", "category"], name = "idx_platform_category"),
        Index(value = ["source_url"], name = "idx_source_url"),
    ],
)
data class CollectedItemEntity(
    @PrimaryKey
    @ColumnInfo(name = "id")
    val id: String,

    @ColumnInfo(name = "title")
    val title: String,

    @ColumnInfo(name = "price")
    val price: BigDecimal?,  // TypeConverter 转 String

    @ColumnInfo(name = "original_price")
    val originalPrice: BigDecimal?,

    @ColumnInfo(name = "image_url")
    val imageUrl: String?,

    @ColumnInfo(name = "source_platform")
    val sourcePlatform: Platform,  // TypeConverter: enum → String

    @ColumnInfo(name = "source_url")
    val sourceUrl: String?,

    @ColumnInfo(name = "category")
    val category: ItemCategory,

    @ColumnInfo(name = "status")
    val status: ItemStatus,

    @ColumnInfo(name = "collected_at")
    val collectedAt: Long,  // epoch millis

    @ColumnInfo(name = "last_checked_at")
    val lastCheckedAt: Long?,

    @ColumnInfo(name = "availability")
    val availability: Boolean,

    @ColumnInfo(name = "notes", defaultValue = "")
    val notes: String = "",

    @ColumnInfo(name = "tags_json")
    val tags: List<String> = emptyList(),  // TypeConverter: List → JSON String

    @ColumnInfo(name = "price_history_json")
    val priceHistory: List<PricePoint> = emptyList(),  // TypeConverter: List → JSON String
)

// ===== TypeConverter 示例 =====

/** 平台枚举 ↔ 数据库字符串 */
class PlatformConverter {
    @TypeConverter
    fun fromPlatform(value: Platform): String = value.name

    @TypeConverter
    fun toPlatform(value: String): Platform = Platform.valueOf(value)
}

/** 价格历史 ↔ JSON（使用 kotlinx.serialization） */
class PriceHistoryConverter {
    private val json = Json { ignoreUnknownKeys = true }

    @TypeConverter
    fun fromList(value: List<PricePoint>): String = json.encodeToString(value)

    @TypeConverter
    fun toList(value: String): List<PricePoint> =
        try {
            json.decodeFromString(value)
        } catch (e: Exception) {
            emptyList()
        }
}
```

### 标签表 + 多对多关系

```kotlin
@Entity(
    tableName = "tags",
    indices = [Index(value = ["name"], unique = true)],
)
data class TagEntity(
    @PrimaryKey
    @ColumnInfo(name = "id")
    val id: String,

    @ColumnInfo(name = "name")
    val name: String,

    @ColumnInfo(name = "color")
    val color: Int,  // ARGB 颜色值

    @ColumnInfo(name = "created_at")
    val createdAt: Long,
)

/** 商品-标签 多对多关联表 */
@Entity(
    tableName = "item_tag_cross_ref",
    primaryKeys = ["item_id", "tag_id"],
    foreignKeys = [
        ForeignKey(
            entity = CollectedItemEntity::class,
            parentColumns = ["id"],
            childColumns = ["item_id"],
            onDelete = ForeignKey.CASCADE,
        ),
        ForeignKey(
            entity = TagEntity::class,
            parentColumns = ["id"],
            childColumns = ["tag_id"],
            onDelete = ForeignKey.CASCADE,
        ),
    ],
)
data class ItemTagCrossRef(
    @ColumnInfo(name = "item_id") val itemId: String,
    @ColumnInfo(name = "tag_id") val tagId: String,
)
```

---

## 二、DAO 设计规范

```kotlin
@Dao
interface CollectedItemDao {

    // ===== 查询（响应式） =====
    @Query("SELECT * FROM collected_items ORDER BY collected_at DESC")
    fun observeAll(): Flow<List<CollectedItemEntity>>

    @Query("SELECT * FROM collected_items WHERE source_platform = :platform ORDER BY collected_at DESC")
    fun observeByPlatform(platform: Platform): Flow<List<CollectedItemEntity>>

    @Query("SELECT * FROM collected_items WHERE category = :category ORDER BY collected_at DESC")
    fun observeByCategory(category: ItemCategory): Flow<List<CollectedItemEntity>>

    // ===== 全文搜索（LIKE 模式） =====
    @Query("""
        SELECT * FROM collected_items
        WHERE title LIKE '%' || :query || '%'
        ORDER BY collected_at DESC
    """)
    fun search(query: String): Flow<List<CollectedItemEntity>>

    // ===== 单次查询 =====
    @Query("SELECT * FROM collected_items WHERE id = :id")
    suspend fun getById(id: String): CollectedItemEntity?

    @Query("SELECT id FROM collected_items WHERE source_url = :url LIMIT 1")
    suspend fun findIdByUrl(url: String): String?

    // ===== 写入 =====
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsert(item: CollectedItemEntity)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsertAll(items: List<CollectedItemEntity>)

    // ===== 更新 =====
    @Query("""
        UPDATE collected_items
        SET price = :price,
            original_price = :originalPrice,
            last_checked_at = :checkedAt,
            availability = :availability,
            status = :status
        WHERE id = :id
    """)
    suspend fun updatePrice(
        id: String,
        price: BigDecimal?,
        originalPrice: BigDecimal?,
        checkedAt: Long,
        availability: Boolean,
        status: ItemStatus,
    )

    // ===== 删除 =====
    @Query("DELETE FROM collected_items WHERE id = :id")
    suspend fun deleteById(id: String)

    @Query("DELETE FROM collected_items WHERE id IN (:ids)")
    suspend fun deleteByIds(ids: List<String>)

    // ===== 统计 =====
    @Query("SELECT COUNT(*) FROM collected_items")
    fun observeCount(): Flow<Int>

    @Query("""
        SELECT source_platform, COUNT(*) as cnt
        FROM collected_items
        GROUP BY source_platform
    """)
    fun observeCountByPlatform(): Flow<List<PlatformCount>>
}

/** 统计结果（无需是 Entity，列名匹配即可） */
data class PlatformCount(
    val source_platform: Platform,
    val cnt: Int,
)
```

### 标签关联查询

```kotlin
@Dao
interface TagDao {

    @Query("SELECT * FROM tags ORDER BY name ASC")
    fun observeAll(): Flow<List<TagEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsert(tag: TagEntity)

    @Delete
    suspend fun delete(tag: TagEntity)

    /** 获取某个商品的所有标签 */
    @Query("""
        SELECT t.* FROM tags t
        INNER JOIN item_tag_cross_ref ref ON t.id = ref.tag_id
        WHERE ref.item_id = :itemId
    """)
    fun observeTagsForItem(itemId: String): Flow<List<TagEntity>>
}
```

---

## 三、Database 类规范

```kotlin
@Database(
    entities = [
        CollectedItemEntity::class,
        TagEntity::class,
        ItemTagCrossRef::class,
        PriceAlertEntity::class,
        SyncMetadataEntity::class,
    ],
    version = 1,
    exportSchema = true,  // 🚨 必须导出 Schema
)
@TypeConverters(
    PlatformConverter::class,
    ItemCategoryConverter::class,
    ItemStatusConverter::class,
    PriceHistoryConverter::class,
)
abstract class ShoppingHubDatabase : RoomDatabase() {
    abstract fun collectedItemDao(): CollectedItemDao
    abstract fun tagDao(): TagDao
    abstract fun priceAlertDao(): PriceAlertDao
    abstract fun syncMetadataDao(): SyncMetadataDao
}
```

---

## 四、Migration 规范（红线）

```kotlin
// ❌ 绝对禁止
Room.databaseBuilder(context, AppDatabase::class.java, "db")
    .fallbackToDestructiveMigration()  // 🔴 生产环境数据全丢！
    .build()

// ✅ 正确：每个版本变更一个 Migration
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("""
            ALTER TABLE collected_items
            ADD COLUMN notes TEXT NOT NULL DEFAULT ''
        """)
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // 创建新表
        db.execSQL("""
            CREATE TABLE IF NOT EXISTS price_alerts (
                id TEXT PRIMARY KEY NOT NULL,
                item_id TEXT NOT NULL,
                target_price REAL,
                is_enabled INTEGER NOT NULL DEFAULT 1,
                created_at INTEGER NOT NULL,
                FOREIGN KEY (item_id) REFERENCES collected_items(id) ON DELETE CASCADE
            )
        """)
        db.execSQL("CREATE INDEX IF NOT EXISTS idx_alert_item ON price_alerts(item_id)")
    }
}

// 注册
Room.databaseBuilder(context, ShoppingHubDatabase::class.java, "shoppinghub.db")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

### Migration 编写规则

1. **每个版本变更一个 Migration 对象**，从版本 N 到 N+1
2. **不得修改已发布的 Migration**（历史记录只读）
3. **ALTER TABLE 仅支持 ADD COLUMN**（SQLite 限制）
4. **新增 NOT NULL 列必须带 DEFAULT 值**
5. **复杂变更**：创建新表 → 迁移数据 → 删除旧表 → 重命名新表
6. **迁移后必须有自动化测试**

### Migration 测试

```kotlin
@Test
fun migration_1_2_preservesData() {
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        ShoppingHubDatabase::class.java,
    )

    // 创建 v1 数据库并插入测试数据
    val db1 = helper.createDatabase(TEST_DB, 1).apply {
        execSQL("""
            INSERT INTO collected_items (id, title, price, collected_at, ...)
            VALUES ('1', 'Test Item', '199.00', ${System.currentTimeMillis()}, ...)
        """)
        close()
    }

    // 迁移到 v2
    val db2 = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2)

    // 验证数据完整性
    db2.query("SELECT * FROM collected_items WHERE id = '1'").use { cursor ->
        assertThat(cursor.moveToFirst()).isTrue()
        assertThat(cursor.getString(cursor.getColumnIndexOrThrow("notes")))
            .isEqualTo("")
    }
}
```

---

## 五、FTS4 全文搜索（数据量 > 1000 条时）

```kotlin
/**
 * FTS4 虚拟表 — 加速全文搜索
 * 与 collected_items 主表通过 content= 参数保持同步
 */
@Entity(tableName = "items_fts")
@Fts4(contentEntity = CollectedItemEntity::class)
data class ItemFts(
    @ColumnInfo(name = "title") val title: String,
    // FTS 仅索引搜索需要的列
)

// DAO
@Dao
interface ItemFtsDao {
    @Query("SELECT rowid FROM items_fts WHERE items_fts MATCH :query")
    suspend fun search(query: String): List<Long>
}

// 联合查询
@Query("""
    SELECT * FROM collected_items
    WHERE id IN (
        SELECT id FROM items_fts WHERE items_fts MATCH :query
    )
    ORDER BY collected_at DESC
""")
fun fullTextSearch(query: String): Flow<List<CollectedItemEntity>>
```

---

## 六、查询性能优化

```kotlin
// ✅ 复合索引 — 覆盖常用组合查询
// 已在 @Entity indices 中声明

// ✅ 避免 N+1 查询 — 用 INNER JOIN 一次获取
@Query("""
    SELECT ci.* FROM collected_items ci
    INNER JOIN item_tag_cross_ref ref ON ci.id = ref.item_id
    WHERE ref.tag_id = :tagId
    ORDER BY ci.collected_at DESC
""")
fun observeByTag(tagId: String): Flow<List<CollectedItemEntity>>

// ✅ 分页查询 — 配合 Paging 3
@Query("""
    SELECT * FROM collected_items
    ORDER BY collected_at DESC
    LIMIT :limit OFFSET :offset
""")
suspend fun getPaged(limit: Int, offset: Int): List<CollectedItemEntity>

// ✅ COUNT 查询别在主线程 — Room 自动在后台线程
@Query("SELECT COUNT(*) FROM collected_items")
suspend fun count(): Int
```

---

## 七、输出格式

### 创建 Entity/DAO/Database 时

```
## 📁 文件路径
- data/data-local/src/.../entity/XxxEntity.kt
- data/data-local/src/.../dao/XxxDao.kt

## 📝 Entity 设计
[表名、列、索引、外键]

## 📝 DAO 接口
[查询方法、写入方法]

## 🔗 与其他表的关系
[外键关系、关联查询]
```

### 编写 Migration 时

```
## 🔢 版本: N → N+1
## 📝 变更内容
[新增表/列、删除表、数据迁移逻辑]

## 🧪 Migration 测试
[测试代码]
```

### 审查数据库代码时

```
## 🔴 数据丢失风险
## 🟡 性能问题（缺少索引、N+1 查询、全表扫描）
## 🔵 设计建议（字段类型优化、索引建议、FTS 升级建议）
```
