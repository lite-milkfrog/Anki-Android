# AnkiDroid Fork — 可安装主题、扩展中心与内嵌 Agent 可行性评估 (2026-10-12)

> **身份**：独立 Fork 的研究与设计合同，不是已实现功能，也不是官方 AnkiDroid 立场。
> **真源**：`D:\AgentData\10_Workspaces\ankidroid-ui-lab`，`eval/ui-plugin-market`；官方基线 `153ee7521b598049218c56ad8401948018fc18e5`。父级报告：`docs/evaluation/ANKIDROID_UI_ADDONS_2026-10-12.md`。
> **阶段**：RESEARCH / SOURCE VERIFIED + ENV CHECKED；无 Gradle assemble、无截图、无 Android 运行/性能证明。正式发布门禁未打开。
> **范围边界**：App 全局原生 UI（选项、设置、主页、菜单、控件、图标）；主题**不是**卡片 HTML/CSS 模板。保留用户 Prism 模板与 FSRS、AnkiWeb 数据路径。

## 0. 总结和决策

**有条件可做，建议分期而不是一次重构。** 核心路线：官方主题为默认且可恢复 → 声明式全局主题包 → 统一扩展中心（Themes / Reviewer JS / Skills / Integrations）→ 以 **Kotlin App 内部受控工具网关** 集成 Agent（Koog 做第一优先的兼容性试验）→ 复用 Google AI Edge Gallery 的 mobile Skills/文件导入/MCP/权限交互思路，**不照搬整个 Gallery App**。云端 OpenAI-compatible 模型为默认；设备端推理按兼容性和体积作为可选适配器。禁止任意 Python 桌面 Anki Add-on 原样安装的虚假兼容承诺。

## 1. 源码与环境已观察的事实

- `AnkiDroid/src/main/java/com/ichi2/themes/Themes.kt` 由 `PrefsRepository` 决定 day/night/follow-system 并调用 `context.setTheme(currentTheme.styleResId)`，官方主题采用静态 Android 资源。原 XML 如 `res/values/theme_light.xml` 含颜色、drawable 和 navigation/popup/reviewer attributes。Compose 桥 `com/ichi2/compose/theme/Theme.kt` 将宿主 XML 颜色映射为 Material3 `ColorScheme`。结论：任意用户运行时修改 `R.style` **不可通过一个 JSON 改文件就直接完成**；需实现运行时 Token Resolver/ThemeOverlay + 组件适配；一些老 View 需逐页改造。
- `DeckPicker.kt` 长且业务密集；`HomeScreenNavigation.kt` 已有底部导航，但使用开发配置门禁；`Preferences.kt` 以 `PreferenceFragmentCompat` 构建。建议先主题中间层和两三个代表页面，不改记忆算法调度。
- `api/src/main/java/com/ichi2/anki/api/AddContentApi.kt` 已提供 note/deck/model 的写入、查询和媒体导入 API。嵌入式 Agent 在相同 App 进程更应调用内部稳定 ViewModel/service 命令而不是操作私有 SQLite 或借助无障碍点击按钮。没有现成“任意设置”通用 Agent API，需逐个域梳理 ToolGateway 并做回归。
- `AnkiDroid/src/main/java/com/ichi2/anki/jsaddons/AddonData.kt` 有 npm-like 元数据/版本校验；`TgzPackageExtract.kt` 有 tgz 包提取；`AnkiDroidJsAPIConstants.kt` 标记 JS API 0.0.3；在 `src` 中对 parse/extract 的使用主要是定义及测试，并不能把它称为现成插件市场。官方需求 #7959 仍开放、标注 2.26 milestone（milestone 不等于发布承诺）。
- Windows 检查：JDK 17、Git、ADB、Android SDK platforms 33/34/36，build tools 30/33/36、emulator.exe 存在，AVD 有 `CloudCLIRemote_API_36_1`、`Medium_Phone_API_36.1`；`adb devices -l` 当时为空。**模拟器存在不等于 APK 已安装或运行通过**。
- 用户当前选择的制卡能力：`card-forge-v2` v2.5.1；默认原版 Prism + Lumina/Nebula/Classic 卡片皮肤；规范为 `prism-deck/v1` JSON，通过 `prism_builder.py` 生成带 GUID、Model、真实 `章节::题型` 多级子牌组的 APKG。不要把 Prism **卡面主题**和这里新定义的**应用 UI 主题**混为一谈。现有 `decks-learning` Skill 是 Obsidian Decks Markdown/FSRS 工作流，互不替代。

## 2. 主题格式（提案 `.ankitheme` v1）

**格式选择**：ZIP 包，扩展名 `.ankitheme`；包内 `manifest.json` + `tokens.json` + 可选 `icons/*.webp` 或 `icons/*.png`。JSON Schema 校验；以 theme ID + version 标识；限定 theme scope = `app-ui`；明示 `cards` 不在范围内。默认仅静态声明式，禁止 JS、DEX、Jar、shell 脚本、任意 Android XML、远端字体下载；导入时验证最大文件数、总体解压大小、压缩比、哈希、路径。

建议包结构：

```text
calm-blue.ankitheme
  manifest.json
  tokens.json
  icons/
    settings.webp
    deck.webp
```

`manifest.json` 示例：

```json
{
  "format": "ankidroid-app-theme/1",
  "id": "cn.milkfrog.calm-blue",
  "name": "Calm Blue",
  "version": "1.0.0",
  "scope": "app-ui",
  "minHostVersion": "2.25",
  "author": "Milkfrog",
  "iconPreset": "material-rounded",
  "assets": { "settings": "icons/settings.webp", "deck": "icons/deck.webp" }
}
```

`tokens.json` 示例（需通过 token 名称白名单）：

```json
{
  "light": {
    "colors": {"primary": "#2863A3", "onPrimary": "#FFFFFF", "surface": "#FAFBFD", "onSurface": "#18212B", "outline": "#687887"},
    "shape": {"cardRadiusDp": 16, "dialogRadiusDp": 20},
    "density": "comfortable",
    "typography": {"scale": 1.0}
  },
  "dark": {
    "colors": {"primary": "#A5C9FF", "onPrimary": "#102D50", "surface": "#12161C", "onSurface": "#E1E9F4", "outline": "#88929E"},
    "shape": {"cardRadiusDp": 16, "dialogRadiusDp": 20},
    "density": "comfortable",
    "typography": {"scale": 1.0}
  }
}
```

**应用机制**：优先将可动态应用的 Compose/自绘 Android View 属性从 `ThemeTokenResolver` 取得；不能无损覆盖旧 XML 的 UI 则由兼容层映射原始样式或明确标注 unsupported；每页/组件适配按真实屏幕覆盖矩阵验证，不能声称首版已经全局覆盖。Android SVG 不是任意 runtime drawable 原生支持的格式；优先 WebP/PNG 或内置 VectorDrawable ID，SVG 仅通过安全转换/白名单后另行支持。

**官方主题恢复**：在主题列表置顶 `Official (Follow System/Day/Night/Plain/Black/E-ink ... 按真实官方选项)`，激活 `source=official` 删除所有临时覆盖，保留原始 `AppTheme` + `DayTheme` + `NightTheme` 设置；重新创建 Activity/重新加载可兼容组件；设置损坏时开机 safe-mode 直达原版。第三方主题可复制、预览、启用、卸载、导入/导出，无法删除内置官方资源。设置几个内置图标预设，如 Material Rounded/Outlined/Mono，以逻辑图标名引用，不暴露资源 ID。主题切换允许同时预览亮/暗/系统与字体放大、平板和简体中文。

**Agent Theme Author Skill**：原生预装 `SKILL.md`（文本级指导）要求收集目标风格、产出 token JSON、validate contrast/touch target/字体缩放、渲染首页+设置+弹窗双主题预览、比较截图、再由用户批准 `theme.apply`。`theme.restore_official` 为单独安全工具，**不依赖模型成功响应**。Skill 无权限直接执行文件写入或改设置，由宿主工具授予能力。

## 3. 统一扩展中心（不仅“插件商店”）

应用入口下分 **Themes / Plugins / Skills / Integrations**，每类明确兼容范围与安全等级：

| 类别 | 范围 | 第一期处理 |
|---|---|---|
| App themes | 颜色、控件、图标与字体 token | 允许声明式 ZIP 安装、切换、恢复 |
| Reviewer JS add-ons | 复习界面 WebView 的受限增强 | 仅经审计选定包与 API 版本校验 |
| Agent Skills | `SKILL.md`，加可选受限资产和执行配方 | 支持导入、启停、校验、版本回退 |
| External integrations | 自定义 LLM API、MCP 服务、伴侣 Android App | 按服务/tool 粒度授权 |
| Desktop Python add-ons | Anki PC 插件 | 标记“不兼容 Android”；可展示已有**明确**的 Android 替代品，绝不冒充可直接安装 |

信息架构：发现（官方/精选/社区/本地、可按兼容性过滤）、已安装（版本/启停/权限/占用）、更新（差异/可回滚）、诊断（日志/故障安全模式），统一版本钉扎、校验和/发布者信息、manifest 权限清单、安装确认、禁用/删除/一键恢复。不能把本 Fork 的精选来源叫“AnkiDroid 官方商店”；需独立 branding。默认无匿名第三方脚本自动执行。

权限验证原则：导入 Skill 的文本可能包含工具指令，但不授予额外权限；每个工具由 Android host 根据来源、当前会话、用户选择的能力和业务对象进行最终校验。需要禁止 archive path traversal、symlink、zip bomb、远端页面 JavascriptInterface 特权暴露和秘钥传递给任意网页。第三方 JS 可读到的只有最小数据投影；禁止任意私有文件路径、外部域名访问（除明确授权 allowlist）。

## 4. 内嵌 Agent：架构和成熟项目筛选

**推荐主线**：
- **Koog** `https://github.com/JetBrains/koog` — 与 AnkiDroid Kotlin 技术栈最契合，具备 type-safe tools、LLM providers、MCP、流式和错误处理，默认用已有自定义 OpenAI-compatible 模型端点；**必须先做独立 Android 兼容性/依赖体积 Spike**，因官方资料中目标平台描述存在口径差异、暂未在本 Fork 构建验证。必要时降级为轻量 Kotlin coroutine + JSON tool loop，但依然重用成熟 SDK 的解析与 client，而不是自行重写完整推理算法。
- **Google AI Edge Gallery** `https://github.com/google-ai-edge/gallery` — 原生 Android、开源 Apache 2.0；真实支持 SKILL.md、Featured/URL/local skill、WebView JS skill、Native intents、LLM on-device 和 MCP / 工具许可。强烈推荐**研究并移植合适的设计/有限模块**，而不是混入整个 App、其资产、模型管理 UI。尤其应重新审计其 Skill import 的潜在路径穿越/密钥泄漏（上游 issue #949 是公开风险报告，不能视为已确认漏洞或已经修复）。
- **PocketPal AI** `https://github.com/a-ghorbani/pocketpal-ai` — MIT、React Native、llama.rn/GGUF 本地推理和 agent tool loop，适合参考设备端加速、模型加载与工具交互，但嵌入原生 AnkiDroid 会增加 React Native/模型运行时体积与复杂度，不作主框架。

**模块布局**：

```text
In-app Agent (Compose chat + attachments / tool approvals)
  ↳ Skills Registry (SKILL.md, imports from files/URL, version pinning)
  ↳ ModelProvider (user OpenAI-compatible HTTPS endpoint, optional Edge on-device)
  ↳ Agent Orchestrator (Koog compatibility spike)
  ↳ Host ToolGateway (typed args -> scope/permission/preflight -> app service)
    ├── Notes/Decks/Browser/Tags/Templates/Media
    ├── Study/Review operations (controlled; FSRS weights/scheduling are not model-managed)
    ├── App Settings / Profile / Search / Navigation / Sync trigger
    ├── Theme Manager / Extension Manager / Skill manager
    └── Files SAF / Preview / Import & Export
  ↳ Journal & Recovery (action log, undo when supported, snapshot, safe mode)
```

目标是**覆盖应用中人能执行的有意义操作**，包括设置和扩展，而不是模拟自动点击所有 Android UI。首版不能保证所有人类操作已有 Agent 工具；建立 `UI-action-to-tool coverage matrix`，每功能对应实体 service/action、输入/错误、权限、undo/QA，并逐阶段补齐。

**工具操作安全分级**：read/search/preview 自动；创建或修改单个 note、调整普通偏好可设会话授权，默认展示 diff；批量改卡、删牌组、替换模板、插件安装和权限变更需显式确认与预备份；替换整个 collection / full download sync 单独更高等级确认；不提供原始 SQL、任意执行 shell、执行未知 DEX。限制步骤数、令牌、触发时长、网络域名、范围（当前牌组/允许文件夹）；模型返回工具参数必须再次验证。自定义 baseURL/key/model/headers 通过 HTTPS 与 Android Keystore/密文存储，不把 key 放进 prompt/日志/主题或 JS Skill；外部 MCP 的每个 tool 都是独立可授权对象，支持关闭/撤销/超时。

## 5. APKG、附件和手机 Skills：复用现有管道而非重新造格式

- **用户现有 canonical 输入**：`card-forge-v2` v2.5.1 采用 `prism-deck/v1` JSON；原版 Prism 卡面及层级 `章节::题型`，加 GUID/稳定模型 ID/源文保真和报告。
- 第一阶段 Agent 实现：通过 Android 文件选择器（SAF）取 PDF/图片/Markdown/JSON/现有 `.apkg`；先由可用模型理解内容、调用 Skill 输出 `prism-deck/v1`，用嵌入的 JSON Schema 和卡片质量约束 validate，生成预览并让用户确认。
- 导入路线 A（**优先**）：使用 AnkiDroid **已有的内部 note/model/deck/media service** 写入有效、去重且有 GUID 的内容，保留真实子牌组与 Anki 同步语义；谨慎处理模型相容、卡片重复和事务/回滚。不是在手机上启动 Windows Python 制卡脚本，也不直接写私有 SQLite。
- 导入路线 B（兼容外部分享）：由受信任的 PC/CloudBase 制卡服务运行现有 Python `prism_builder.py` 出 `.apkg`，手机通过已有 APK 导入器导入；未来可以单独验证纯 Android 打包，但不让它阻塞首版手机制卡。
- 已有 `.apkg`：直接从手机文件选择器导入原生 importer；应区别 `.apkg` 牌组合并与 `.colpkg` 全 collection 替换，后者必须单独确认，不允许 Agent 误触。
- 能力示例：`card.create_batch`, `deck.create_subdeck`, `card.find_duplicates`, `media.import`, `note.update`, `theme.propose`, `theme.restore_official`, `extension.search`, `extension.install`, `settings.list_editable`, `settings.set`, `import.preview_apkg`, `import.commit`, `sync.start`；每个工具有独立权限声明、回滚与能力检查。
- 内置 Skills 初始集：`prism-card-author`（现有制卡规则的轻量可移植版本）、`apkg-import-auditor`（来源/去重/套牌检查）、`theme-author`（主题生成/截图评估/恢复）、`note-maintenance`（去重/纠错/溯源）、`review-assistant`（不修改 FSRS 的学习整理）。
- 导入任意第三方 `SKILL.md` 是**导入指导**，不表示可运行它引用的 Python 或 Node 脚本。为 Skills 增加 `runtime: instruction-only | host-tools | sandboxed-js` 和 `capabilities` 白名单，禁止直接声称与 Windows Skill 完全等价；单独标注不可移植依赖。

## 6. 分期、难度和严格验收

| 阶段 | 工作与边界 | 复杂度 | 必要证据 |
|---|---|---|---|
| P0 | Android 构建、模拟器运行、现有 UI 与数据保护基线、3rd-party code 安全条款 | 3/5 | debug APK + screenshot/logcat + Git clean baseline |
| P1 | 运行时主题中间层、官方恢复、图标包、首页/设置等首批屏幕 | 3–4/5 | 亮/暗/系统及官方一键复原；字重/字体缩放、触摸、平板无回退 |
| P2 | Agent UI、Provider 自定义、Host ToolGateway、SAF 附件、Prism JSON→原生制卡/import | 4–5/5 | 卡包层级/媒体/重复/GUID/历史/sync/否定权限用例 |
| P3 | 统一商店第一版（声明式 themes、text skills、受控 JS）、管理/升级/回滚 | 4–5/5 | 包验证、恶意样本拒绝、版本回滚、安全模式 |
| P4 | MCP/custom API、Agent 全应用 action coverage、权限与审计 | 5/5 | tool 对照矩阵、错误/取消/确认/恢复负例，安卓实机 |
| P5 | 可选设备端模型推理与压力/耗电/内存评测 | 5/5 | 真实设备参数、连续运行、模型大小和速度实测 |

**先做不代表全部完工**：目前 P0 只有工作环境发现和源码审计，无构建/安装/截图，因此阶段验收仍为 NOT_RUN。建议只将上述原型限定在独立 Fork 和独立 Android `applicationId` 的 debug 变体（与正式已安装 AnkiDroid 不同）；不要直接覆盖用户手机上的既有数据。切勿为方便测试而默认把真实数据同步至第三方云端。

### GO / NO-GO

- GO：主题包原生 UI、官方还原、内置几套矢量图标、主题定制 Agent Skill、结构化制卡+导入、清晰的插件管理。
- CONDITIONAL GO：Koog 在原生 Android 上编译/工具调用与体积实测后集成，可信受限 JS 插件以及自定义 MCP/工具权限。
- NO-GO（第一阶段）：直接执行桌面 Python Anki addons、从网络下载任意 DEX/JAR/so、让第三方 JS 访问全部笔记/密钥/设备、让 LLM 自行直接写 SQLite/FSRS 参数。
- **先验证最小闭环**：官方恢复 → 下载并预览主题→激活/还原；上传 Markdown→Agent Skill→Prism JSON preview→导入 3 层牌组→sync 回归；搜选一个已审计插件→安装→禁用/回滚。

## 7. 参考与保守说明

- AnkiDroid 官方源码：`https://github.com/ankidroid/Anki-Android`；`docs/ankidroid`/源码路径见本文；JS add-on 仍开放 `https://github.com/ankidroid/Anki-Android/issues/7959`。
- AnkiDroid API：`api/src/main/java/com/ichi2/anki/api/AddContentApi.kt`；导入说明 `https://docs.ankidroid.org/manual.html`。
- Google AI Edge Gallery Agent Skills：`https://github.com/google-ai-edge/gallery/blob/main/skills/README.md`；MCP 宣布 `https://developers.googleblog.com/en/a-smarter-google-ai-edge-gallery-mcp-integration-notifications-and-session-continuity/`；公开潜在安全风险报告 `https://github.com/google-ai-edge/gallery/issues/949`。
- Koog 文档 `https://docs.koog.ai/`、仓库 `https://github.com/JetBrains/koog`；PocketPal `https://github.com/a-ghorbani/pocketpal-ai`。
- Google Play 动态代码政策：`https://support.google.com/googleplay/android-developer/answer/16559646`。
- 上游 `AI_POLICY.md` 限制 AI 生成的官方仓库贡献：本 Fork 自主研究不构成向官方提交 AI 产物的许可；有 upstream PR 意图需遵守官方准则。

## 8. 操作记录

- 查询 Fork/分支前续状态、源代码关键模块、Windows JDK/SDK、两台 AVD、AnkiDroid API、主题引擎和现有 card-forge-v2 Skills。
- 阅读上述第三方官方仓库/文档，比较 Agent/Skills/插件系统的**功能与可移植性**。未复制第三方项目源码，未下载完整模型，未执行应用安装、APK 构建或模拟器 UI 功能测试。
- 本文为审核后策略；尚未实施改造，不应将“可行”误记为“完成”。
