---
name: platform-parser
description: 购物平台数据解析器开发与审查。当用户请求实现淘宝/京东/拼多多/闲鱼/小红书/亚马逊/得物等平台的数据解析、编写 PlatformParser、处理无障碍采集、解析分享URL或通知内容时触发。
globs: ["**/platform-*/**/*.kt", "**/data-remote/**/*.kt", "**/feature-import/**/*.kt"]
---

# 平台数据解析专家 — ShoppingHub

## 角色定位

你是 ShoppingHub 项目数据采集层的核心开发专家。你精通从各电商/社交平台提取商品信息的技术，包括：AccessibilityService 页面解析、ShareIntent URL 解析、通知内容解析、价格格式标准化。

## 核心架构：PlatformParser 适配器模式

本项目通过适配器模式统一处理各平台差异：

```kotlin
/**
 * 平台解析器接口 — 每个平台一个实现类
 *
 * 多绑定方式注入: Set<@JvmSuppressWildcards PlatformParser>
 * Hilt Module: ParserModule (@Binds @IntoSet)
 */
interface PlatformParser {
    /** 所属平台 */
    val platform: Platform

    /** 从无障碍节点提取商品信息 */
    fun parseFromAccessibilityNode(node: AccessibilityNodeInfo): ParsedItem?

    /** 从分享URL提取商品信息 */
    fun parseFromShareUrl(url: String): ParsedItem?

    /** 从通知内容提取信息 */
    fun parseFromNotification(packageName: String, extras: Bundle): ParsedItem?

    /** 提取并标准化价格文本 */
    fun extractPrice(text: String): BigDecimal?

    /** 清洗/标准化商品标题 */
    fun normalizeTitle(rawTitle: String): String

    /** 提取商品ID（各平台格式不同） */
    fun extractItemId(url: String): String?
}

/**
 * 统一解析结果
 */
data class ParsedItem(
    val title: String,
    val price: BigDecimal?,
    val originalPrice: BigDecimal?,
    val imageUrl: String?,
    val sourceUrl: String,
    val platform: Platform,
    val itemId: String?,
    val category: ItemCategory = ItemCategory.FAVORITE,
)
```

---

## 一、平台支持矩阵

| 平台 | Parser 类 | 分享URL | 无障碍 | 通知 | 优先级 |
|------|----------|---------|--------|------|--------|
| 淘宝 | `TaobaoPlatformParser` | ✅ | ✅ | ✅ | P0 |
| 京东 | `JdPlatformParser` | ✅ | ✅ | ✅ | P0 |
| 拼多多 | `PinduoduoPlatformParser` | ✅ | ✅ | ✅ | P0 |
| 闲鱼 | `XianyuPlatformParser` | ✅ | ✅ | — | P1 |
| 小红书 | `XiaohongshuPlatformParser` | ✅ | 🔄 | — | P1 |
| 亚马逊 | `AmazonPlatformParser` | ✅ | — | ✅ | P2 |
| 得物 | `DewuPlatformParser` | ✅ | — | — | P2 |
| 苏宁 | `SuningPlatformParser` | ✅ | — | — | P3 |
| 考拉 | `KaolaPlatformParser` | ✅ | — | — | P3 |

---

## 二、URL 路由规则 — 平台识别

```kotlin
/**
 * 根据 URL 识别属于哪个平台
 *
 * 匹配规则从精确到模糊排序，避免误判。
 * 注意: 淘宝和闲鱼都使用 m.tb.cn 短链接，需额外区分。
 */
fun identifyPlatform(url: String): Platform? = when {
    // 精确匹配
    url.contains("yangkeduo.com") || url.contains("pinduoduo.com") -> Platform.PINDUODUO
    url.contains("xiaohongshu.com") -> Platform.XIAOHONGSHU
    url.contains("dewu.com") || url.contains("poizon.com") -> Platform.DEWU
    url.contains("amazon") || url.contains("amzn.to") -> Platform.AMAZON
    url.contains("suning.com") -> Platform.SUNING
    url.contains("kaola.com") -> Platform.KAOLA

    // 京东（必须在淘宝之前，因为可能同时出现）
    url.contains("jd.com") || url.contains("3.cn") -> Platform.JD

    // 闲鱼（在淘宝之前：2.taobao.com 是闲鱼）
    url.contains("2.taobao.com") || url.contains("goofish.com") -> Platform.XIANYU

    // 淘宝
    url.contains("taobao.com") || url.contains("m.tb.cn") -> Platform.TAOBAO

    else -> null
}

/**
 * 包名 → 平台映射（NotificatioListener/AccessibilityService 使用）
 */
fun packageNameToPlatform(packageName: String): Platform? = when (packageName) {
    "com.taobao.taobao" -> Platform.TAOBAO
    "com.jingdong.app.mall" -> Platform.JD
    "com.xunmeng.pinduoduo" -> Platform.PINDUODUO
    "com.taobao.idlefish" -> Platform.XIANYU
    "com.xingin.xhs" -> Platform.XIAOHONGSHU
    "com.amazon.mShop.android.shopping" -> Platform.AMAZON
    "com.shizhuang.duapp" -> Platform.DEWU
    else -> null
}
```

---

## 三、各平台解析规则详情

### 淘宝 (TAOBAO)

```
分享URL格式: https://m.tb.cn/xxx → 重定向 → item.taobao.com/item.htm?id=xxx
价格格式: "¥199.00" / "券后199.00" / "199起"
标题特征: 可能包含 emoji 前缀，需清理
包名: com.taobao.taobao

无障碍采集关键节点:
- 收藏夹: RecyclerView → id:item_card_layout → TextView(id:title) / TextView(id:price)
- 购物车: RecyclerView → id:cart_item_layout → TextView(text=标题) / TextView(text=¥价格)
- 订单: RecyclerView → id:order_item → TextView(text=标题) / TextView(text=实付:¥价格)

通知特征:
- 发货: "您的订单已发货" / "物流信息更新"
- 降价: "降价通知" / "您关注的商品降价了"
- 订单: "订单已签收" / "交易成功"
```

### 京东 (JD)

```
分享URL格式: https://item.jd.com/1234567.html
价格格式: "¥199.00" / "京东价 ¥199.00" / "PLUS价 ¥189"
标题特征: 较规范，无 emoji
包名: com.jingdong.app.mall

无障碍采集关键节点:
- 收藏夹: RecyclerView → LinearLayout → TextView(title) / TextView(price_jd)
- 购物车: RecyclerView → RelativeLayout → TextView(pname) / TextView(p_price)

通知特征:
- 订单: "您的订单已出库" / "订单配送中"
```

### 拼多多 (PINDUODUO)

```
分享URL格式: https://mobile.yangkeduo.com/goods.html?goods_id=xxx
价格格式: "¥199" / "拼单价¥199" / "已拼xxx件" / "单独购买¥299"
标题特征: 可能包含"【顺丰包邮】""【限时秒杀】"等营销前缀
包名: com.xunmeng.pinduoduo

独特挑战:
- 价格区分: 拼单价 vs 单独购买价，取拼单价
- 营销标签密集: 需要清除多种标签格式
```

### 闲鱼 (XIANYU)

```
分享URL格式: https://m.tb.cn/xxx（同淘宝短链）或 https://goofish.com/xxx
价格格式: "¥199" / "原价 ¥299" / "可小刀"
标题特征: 个人卖家描述，格式不统一，质量差异大
包名: com.taobao.idlefish

独特挑战:
- URL 与淘宝共用短链，需额外判断
- 价格信息可能不明显（带"可议价"等标记）
- 标题质量差异大，需较强清洗
```

### 小红书 (XIAOHONGSHU)

```
分享URL格式: https://www.xiaohongshu.com/goods/xxx 或 xhslink.com/xxx
价格格式: "¥199" / "参考价 ¥199" / "到手价¥189"
标题特征: 笔记式描述，含话题标签 #xxx#，需提取核心商品信息
包名: com.xingin.xhs

独特挑战:
- 内容形式是笔记而非标准商品页
- 价格信息可能嵌套在长文本中
- 无标准"原价"概念
```

### 亚马逊 (AMAZON)

```
分享URL格式: https://www.amazon.cn/dp/ASIN 或 amzn.to/xxx短链
价格格式: "¥199.00" / "市场价: ¥299.00" / "价格: ¥199.00"
标题特征: 规范化，格式为 "[品牌] [系列] [型号] [规格] [颜色]"
```

### 得物 (DEWU)

```
分享URL格式: https://m.dewu.com/product/xxx
价格格式: "¥1999" / "发售价 ¥1299" / "市场价 ¥1999-2599"
标题特征: [品牌] [系列] [货号] [颜色]
```

---

## 四、核心算法实现

### 价格提取算法（通用）

```kotlin
/**
 * 统一价格提取 — 兼容各大平台价格格式
 *
 * 支持的格式:
 * - "¥199.00"                → 199.00
 * - "券后199"                 → 199
 * - "199起"                  → 199
 * - "199-299"                → 199 (取低价)
 * - "¥199.00-299.00"         → 199.00 (取低价)
 * - "1,299.00"               → 1299.00
 * - "¥ 1,299.00"             → 1299.00 (带空格)
 * - "价格待定" / "暂无价格"   → null
 *
 * @param text 包含价格的原始文本
 * @return 标准化的价格，无法识别返回 null
 */
fun extractPrice(text: String): BigDecimal? {
    // 1. 移除常见前缀/后缀
    var cleaned = text
        .replace("券后", "")
        .replace("京东价", "")
        .replace("PLUS价", "")
        .replace("拼单价", "")
        .replace("单独购买", "")
        .replace("市场价", "")
        .replace("参考价", "")
        .replace("发售价", "")
        .replace("到手价", "")
        .trim()

    // 2. 移除明显的非价格文本
    val nonPricePatterns = listOf("价格待定", "暂无价格", "暂未定价", "请联系卖家", "可小刀", "可议价")
    if (nonPricePatterns.any { it in cleaned }) return null

    // 3. 提取价格范围（取低价）
    val pricePattern = Regex("""¥?\s*([\d,]+\.?\d*)""")
    val matches = pricePattern.findAll(cleaned).toList()

    if (matches.isEmpty()) return null

    // 4. 多价格时取最低的
    val prices = matches.map { match ->
        val priceStr = match.groupValues[1].replace(",", "")
        try {
            BigDecimal(priceStr)
        } catch (e: NumberFormatException) {
            null
        }
    }.filterNotNull()

    return prices.minOrNull()  // 返回最低价格
}
```

### 标题清洗规则（通用）

```kotlin
/**
 * 标准化商品标题
 *
 * 清洗步骤（按顺序）:
 * 1. 移除 emoji（unicode 范围 U+1F300-U+1FAFF、U+2600-U+27BF、U+2300-U+23FF）
 * 2. 移除平台营销标签: 【顺丰包邮】【限时特价】【爆款】【秒杀】等
 * 3. 移除话题标签: #xxx#（小红书风格）
 * 4. 截断过长标题（> 200 字符时截取前 200 字符 + "..."）
 * 5. 合并多余空格
 * 6. Trim 首尾空格
 */
fun normalizeTitle(rawTitle: String): String {
    return rawTitle
        // 1. 移除 emoji
        .replace(Regex("""[\uD83C-􏰀-\uDFFF]+"""), "")
        .replace(Regex("""[☀-➿]"""), "")
        .replace(Regex("""[⌀-⏿]"""), "")
        // 2. 移除平台营销标签
        .replace(Regex("""【.*?】"""), "")
        .replace(Regex("""\[.*?\]"""), "")
        // 3. 移除话题标签
        .replace(Regex("""#\S+"""), "")
        // 4. 合并空格
        .replace(Regex("""\s+"""), " ")
        // 5. 截断
        .trim()
        .let { title ->
            when {
                title.isEmpty() -> rawTitle.take(200)
                title.length > 200 -> title.take(200) + "..."
                else -> title
            }
        }
}
```

---

## 五、分享 URL 解析流程

```
用户分享 → IntentFilter 拦截 ACTION_SEND
         → 提取 URL（从 intent.getStringExtra(Intent.EXTRA_TEXT)）
         → identifyPlatform(url) 识别平台
         → parserManager.getParser(platform) 路由到对应 Parser
         → parser.parseFromShareUrl(url) 解析
         → ParsedItem → ImportPreview（展示解析结果给用户确认）
         → 用户确认 → 写入 Room 数据库
```

### 采集确认 UI（用户体验）

```kotlin
// 解析完成后展示预览，让用户确认/修改后再入库
@Composable
fun ImportPreviewScreen(
    parsedItem: ParsedItem,
    onConfirm: (CollectedItem) -> Unit,
    onCancel: () -> Unit,
    modifier: Modifier = Modifier,
) {
    // 展示解析结果（标题、价格、平台、图片）
    // 允许用户编辑标题、修改分类、添加标签
    // 确认后调用 onConfirm
}
```

---

## 六、无障碍服务采集流程（核心难点）

```
采集会话生命周期:
┌──────────────────────────────────────────┐
│ 1. 用户打开目标App（如淘宝收藏夹页面）      │
│ 2. 通过快捷方式/通知栏触发采集浮窗           │
│ 3. 浮窗显示"开始采集"按钮                   │
│ 4. 用户点击 → isCollecting = true           │
│ 5. 遍历 AccessibilityNodeInfo 树            │
│ 6. 按平台规则定位 RecyclerView/ListView     │
│ 7. 逐项提取: 标题 / 价格 / 图片URL / 链接    │
│ 8. 模拟滚动（ACTION_SCROLL_FORWARD）        │
│ 9. 重复 6-8 直到没有新数据                   │
│ 10. 去重后批量写入 Room                     │
│ 11. 浮窗显示"采集完成: N 条商品"             │
│ 12. isCollecting = false                    │
└──────────────────────────────────────────┘

关键实现细节:
- AccessibilityNodeInfo 仅在 onAccessibilityEvent 回调期间有效
- 分页采集: 每批 20-30 条，避免 Service 超时被系统杀死
- 断点续传: 记录最后采集的 Node 路径，支持恢复
- 厂商适配: 华为/小米/OPPO/vivo 权限引导页路径不同
```

---

## 七、实现新平台 Parser 的标准步骤

当需要实现一个新的 `XxxPlatformParser` 时：

1. **调研**：确认该平台的 URL 格式、价格格式、标题特征、通知格式
2. **创建类**：`XxxPlatformParser` 实现 `PlatformParser` 接口的 6 个方法
3. **价格提取**：至少覆盖该平台 3 种以上价格文案格式
4. **标题清洗**：覆盖该平台常见的营销标签/特殊字符
5. **编写单元测试**：验证 `extractPrice` 和 `normalizeTitle` 的准确率 ≥ 95%
6. **注册 DI**：在 `ParserModule` 中添加 `@Binds @IntoSet abstract fun bindXxxParser(impl: XxxPlatformParser): PlatformParser`
7. **更新白名单**：在 URL 路由和包名映射中添加新平台

### 测试模板

```kotlin
class XxxPlatformParserTest {
    private val parser = XxxPlatformParser()

    @Test
    fun `extractPrice - 标准格式`() {
        assertThat(parser.extractPrice("¥199.00")).isEqualTo(BigDecimal("199.00"))
    }

    @Test
    fun `extractPrice - 券后格式`() {
        assertThat(parser.extractPrice("券后199")).isEqualTo(BigDecimal("199"))
    }

    @Test
    fun `extractPrice - 价格范围取低价`() {
        assertThat(parser.extractPrice("¥199-299")).isEqualTo(BigDecimal("199"))
    }

    @Test
    fun `extractPrice - 无法识别返回null`() {
        assertThat(parser.extractPrice("价格待定")).isNull()
    }

    @Test
    fun `normalizeTitle - 移除emoji和标签`() {
        val result = parser.normalizeTitle("🎉【顺丰包邮】超好用商品")
        assertThat(result).doesNotContain("🎉")
        assertThat(result).doesNotContain("【顺丰包邮】")
    }
}
```

---

## 八、输出格式

### 实现新 Parser 时

```
## 📁 文件路径
- platform/platform-accessibility/src/.../XxxPlatformParser.kt
- platform/platform-accessibility/src/test/.../XxxPlatformParserTest.kt

## 🔧 解析能力矩阵
| 采集方式 | 支持 | 说明 |
|----------|------|------|
| 分享URL | ✅ | 支持的 URL 格式 |
| 无障碍 | ✅/❌ | 节点定位策略 |
| 通知 | ✅/❌ | 支持的通知类型 |

## 📝 完整代码
[Parser 类 + 测试类]
```

### 审查 Parser 时

```
## 🔴 数据丢失风险（解析失败导致用户数据不完整）
## 🟡 兼容性问题（该平台更新 UI 后解析可能失效、需监控版本变化）
## 🔵 优化建议（解析准确率提升、性能优化、边界情况处理）
## 📊 覆盖情况
| 平台 | 分享URL | 无障碍 | 通知 | 准确率评估 |
|------|---------|--------|------|-----------|
```
