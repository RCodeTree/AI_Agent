# YouLikeHub Skills — Android Compose UI 开发技能包

为「YouLikeHub」购物收藏聚合 App 量身定制的 AI 辅助开发技能包。基于 **Kotlin + Jetpack Compose + Material 3 + MVVM + Clean Architecture**，覆盖 UI 页面设计、组件编写、主题配置、数据层接入全流程。

---

## 技能目录

| 技能 | 目录 | 用途 | 触发场景 |
|------|------|------|----------|
| **compose-page-builder** | `compose-page-builder/` | 页面脚手架生成 | "创建一个XX页面"、"新建XX Screen" |
| **compose-component-cookbook** | `compose-component-cookbook/` | 组件模式与配方 | "写一个XX组件"、"封装一个XX" |
| **compose-theme-config** | `compose-theme-config/` | 主题与设计令牌 | "配置主题"、"设置颜色/字体"、"深色模式" |
| **compose-data-bridge** | `compose-data-bridge/` | UI ↔ 数据层桥接 | "接入数据"、"连接API"、"ViewModel怎么写" |

---

## 快速开始

### Trae IDE（首选）

将整个 `you-like-hub-skills/` 目录复制到项目的 `.trae/skills/` 下：

```
cp -r you-like-hub-skills/* /your-project/.trae/skills/
```

Trae 会自动加载每个子目录下的 `SKILL.md` 作为独立技能。在对话中提及相关关键词时，对应技能会自动激活。

### Cursor

将各 `SKILL.md` 的内容合并或分拆放入 `.cursor/rules/`：

```
cursor-project/
├── .cursor/
│   └── rules/
│       ├── compose-page-builder.mdc
│       ├── compose-component-cookbook.mdc
│       ├── compose-theme-config.mdc
│       └── compose-data-bridge.mdc
```

### GitHub Copilot

将核心规范提取为 `.github/copilot-instructions.md`：

```markdown
<!-- 将各 SKILL.md 中的「架构分层规则」「命名约定」「禁止模式」部分提取到这里 -->
```

### Claude / Claude Code

将 `you-like-hub-skills/` 放入项目的 `.claude/` 目录，或通过 `/init` 命令生成 `CLAUDE.md` 时引用这些技能文件。

### Windsurf

将规范放入 `.windsurfrules` 或项目根目录的 rules 文件中。

### 通用 Agent / IDE

将对应 `SKILL.md` 的内容作为 system prompt 或项目上下文传入。建议按需加载而非全部注入。

---

## 技术栈

| 维度 | 选型 | 版本 |
|------|------|------|
| 语言 | Kotlin | 2.0+ |
| UI 框架 | Jetpack Compose | BOM 2025.x+ |
| 设计系统 | Material Design 3 (Material You) | latest stable |
| 架构模式 | MVVM + Clean Architecture | — |
| 依赖注入 | Hilt | — |
| 导航 | Compose Navigation (Type-safe) | — |
| 图片加载 | Coil 3 | — |
| 异步 | Kotlin Coroutines + Flow | — |
| 本地存储 | Room + DataStore | — |
| minSdk | 26 (Android 8.0) | — |
| targetSdk | 35 (Android 15) | — |

---

## 架构分层

```
app ──► feature/* ──► domain ──► core
                │
                ▼
              data ──► domain ──► core
```

| 层级 | 职责 | 可依赖 |
|------|------|--------|
| **feature** | Screen、ViewModel、UiState | domain, core-ui, core-model |
| **domain** | UseCase、Repository 接口 | core-model（纯 Kotlin） |
| **data** | Repository 实现、DAO、Remote | domain, core |
| **core** | 通用工具、UI 基础组件 | 无业务依赖 |

---

## 项目结构约定

```
feature-{name}/
├── navigation/
│   └── {Name}Navigation.kt      # Route + NavGraphBuilder 扩展
├── ui/
│   ├── {Name}Screen.kt          # Screen（外层注入 + Content 纯函数）
│   ├── {Name}ViewModel.kt       # ViewModel + UiState 定义
│   └── components/              # 页面内复用组件
└── di/                          # 模块级 Hilt Module（按需）
```

---

## 命名约定速查

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| Screen Composable | `{Feature}Screen` | `HomeScreen` |
| UI State | `{Feature}UiState` | `HomeUiState` |
| ViewModel | `{Feature}ViewModel` | `HomeViewModel` |
| 导航路由 | `{Feature}Route` | `HomeRoute` |
| 子组件 | 描述性名词 | `PlatformStatsCard` |
| 导航扩展 | `NavGraphBuilder.{name}()` | `homeScreen()` |

---

## 各技能文件说明

### compose-page-builder

```
compose-page-builder/
├── SKILL.md                       # 页面生成主技能
└── templates/
    ├── BaseScreen.kt.tmpl         # 基础 Screen 骨架
    ├── BaseViewModel.kt.tmpl      # 基础 ViewModel 骨架
    ├── BaseUiState.kt.tmpl        # 基础 UiState 骨架
    ├── BaseNavigation.kt.tmpl     # 基础 Navigation 骨架
    ├── ListPage.kt.tmpl           # 列表页模板
    ├── DetailPage.kt.tmpl         # 详情页模板
    ├── FormPage.kt.tmpl           # 表单页模板
    ├── DashboardPage.kt.tmpl      # 仪表盘页模板
    └── SettingsPage.kt.tmpl       # 设置页模板
```

### compose-component-cookbook

```
compose-component-cookbook/
└── SKILL.md                       # 组件模式与配方（单文件，按组件类型分节）
```

### compose-theme-config

```
compose-theme-config/
└── SKILL.md                       # 主题配置（颜色/字体/形状/深色模式）
```

### compose-data-bridge

```
compose-data-bridge/
└── SKILL.md                       # 数据层桥接（ViewModel ↔ UseCase ↔ Repository）
```

---

## 使用建议

1. **新建项目时**：先用 `compose-theme-config` 搭建主题，再用 `compose-page-builder` 逐个生成页面
2. **日常开发时**：让 AI 按需加载对应技能，避免一次性加载所有技能导致上下文溢出
3. **组件复用**：编写页面时参考 `compose-component-cookbook` 进行组件抽取
4. **数据接入**：页面 UI 稳定后，用 `compose-data-bridge` 对接数据层

---

*技能包版本：v1.0 | 适配项目：YouLikeHub | 技术栈：Kotlin 2.0 + Jetpack Compose + Material 3*
