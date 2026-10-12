# AnkiDroid 增强版 / Product Vision & Feasibility v1.0

日期：2026-10-12。分支：`eval/ui-plugin-market`。上游基线：`153ee7521b598049218c56ad8401948018fc18e5`。

状态：**产品需求整合 + 源码可行性评估**。不是已开发完成、已构建 APK 或真机验证。独立 Fork：`https://github.com/lite-milkfrog/Anki-Android`。前期资料：`ANKIDROID_UI_ADDONS_2026-10-12.md`、`ANKIDROID_THEMES_PLUGINS_AGENT_FEASIBILITY_2026-10-12.md`。

## 1. 产品定位和边界

AnkiDroid 是数据、FSRS/调度、APKG 和 AnkiWeb 同步的权威核心；新版是基于它的增强版，而不是新写一套记忆软件。手机端**本地 Harness**负责对话、上下文、Agent 运行循环、工具注册、Skills、审批、操作日志、本地记忆、文件预处理；**云端 LLM**负责推理与必要的文件理解。明确**不集成本地大模型**、不下载 GGUF 权重、不把云模型误称为端上推理。支持用户自定义 OpenAI-compatible 模型/API/baseURL/key、模型选择、请求限额、超时、工具级开关、MCP 连接与许可。

核心目标：“App 中用户能完成的有效操作，Agent 通过统一工具网关也应尽可能完成；App 内可见且授权的数据应通过只读数据适配层可被 Agent 查询”；**目标 ≠ 当前已有**；以覆盖矩阵、真实成功/失败/撤销样本逐项验收。不得通过无障碍模拟点击代替稳定内部 API，不直接给模型 SQL/密钥/私有任意文件访问。

## 2. 功能合同（用户设想汇总）

### A. 数据理解与个人记忆

- 只读数据清单：Profile/牌组及多层级、笔记与字段、卡片与卡片状态、标签、笔记类型/模板、媒体引用、浏览/搜索、牌组选项预设、新卡与每日复习数量、到期和逾期、复习日志、正确率、学习时长（仅当源数据支持）、FSRS 状态/参数、统计、已安装主题/插件/Skills、普通应用设置/运行状态、同步/备份状态（脱敏）。
- 本地 Agent memory：用户明确陈述的偏好、常用科目、导入的学习目标、配置习惯、既往操作决策、项目/Skills 使用偏好，存结构化 SQLite + 可选可检索 Markdown；支持查看/纠正/清空/导出，以及禁止某条记忆进入模型上下文。
- 从 App 行为统计推导“学习习惯”需要明示其为**统计推断**而非用户亲口告知，附数据范围和时间戳。历史复习记录、学习时长只能读已有可靠字段，不捏造历史。读取全部授权数据 ≠ 每次把全部个人卡片发送云端；本地筛选/聚合后仅将任务必要片段发往模型。
- “ChatGPT 自身记忆/跨对话历史”**并非 AnkiDroid 自动可读取的数据**；除非用户自行提供导出或未来存在授权连接机制，否则 App 不能直接抓取 ChatGPT 私有记忆。用户在 Obsidian/Vault 的额外资料同理，需要单独导入/授权，不能默认越过 Android 沙箱。

### B. 卡片、牌组与学习设置（读取 + 行为）

- 内容：创建、批量创建、查找、筛选、编辑、合并去重、删除、撤销、转移牌组、标签、暂停/恢复、埋藏、旗标、卡片类型、字段、显示模板/媒体，以及来源信息；批量操作须预览 diff。
- 对每卡可读：ID、所属 note/deck、front/back 对应字段、标签、状态、due、复习次数、lapses、interval、stability/difficulty/retrievability（后端可提供时）、日志摘要；敏感字段应使用权限与最小化输出。
- 牌组选项/预设：新卡数量、复习上限、学习/重新学习步骤、最大间隔、出题/埋卡规则、预设关联、FSRS 开关、期望保持率、优化参数操作、重排日程的后果模拟。**AI 不应任意编辑 FSRS 权重，也不应把单卡手动到期变化当作参数优化；“优化”应调用官方优化器，调度仍以 Anki 后端为唯一真源。**
- 日常操作：浏览、搜索、统计、创建筛选牌组、复习辅助、打开某页面、触发同步、备份与导入/导出。评分/改写学习历史必须有用户动作及审计确认，不应伪造复习记录。

### C. 软件设置尽量全覆盖

建立 `SettingsRegistry`：可列举每个暴露的设置 ID、标题、当前值、范围、默认值、是否可修改、关联副作用、依赖与风险；读取由 `settings.get/list/search`，更新由 `settings.preview/set/reset` 进行验证和业务回调。设置包括 General / Reviewing / Reviewer / Sync / Notifications / Reminders / Appearance / Profiles / Controls / Accessibility / Backup / Advanced / Developer 等，但账号凭据/证书、私有令牌、部分系统权限不是可读取明文设置。高风险设置（替换数据库、批量重排、全量同步覆盖、删除备份、关闭关键保护）必须显式审批。

### D. App 全局主题，而不是卡面主题

官方原有 Follow System/Light/Plain/Dark/Black/E-ink 等真实选项为可恢复的官方主题，永远保留。自定义 `.ankitheme` 声明式 ZIP（JSON Manifest + Tokens + 内置/安全图标资产），支持预览、导入、复制、保存、编辑、安装、切换、删除、导出和一键**还原官方主题**。起步允许颜色/排版/间距/圆角/图标映射；组件式动态适配，而非用 JSON 假装能改变静态 `R.style`。少量精选内置图标预设，允许安全替换。Theme Author Agent Skill 生成/修订/审查主题包，但激活与危险资源权限由宿主授予。**卡片 HTML/CSS Prism 等独立存在**。

### E. 统一扩展中心

标签：发现、已安装、更新、权限/日志；内容分类 Themes、Reviewer JS 插件、Agent Skills、外部集成。支持精选、第三方、GitHub/npm 元数据（逐项兼容性校验）和本地 ZIP/Skills；下载、安装、启停、卸载、恢复、更新、版本钉扎及回滚。桌面 Anki Python 插件**不视为**手机兼容插件。插件安装不等于给模型或插件无限权限；可按应用域与工具粒度授予/撤销。应有异常隔离与安全模式。禁止下载执行任意 DEX/JAR/native 或运行不受控 shell。

### F. 手机内嵌通用 Agent

固定对话入口 + 全屏工作台 + 上下文侧面板，可上传 Android SAF 选择的 PDF/图片/Markdown/JSON/APKG，带文件预览、大小限制、权限与处理反馈。主模型默认用户配置的云端 API；Agent 主循环本地执行；支持会话/Skills/工具权限/操作计划/审计日志/可中断或重试。Koog（https://github.com/JetBrains/koog）优先作为 Kotlin/Android 依赖和能力兼容性试验；若体积/性能不能接受，则用成熟模型 SDK + Kotlin coroutines 的薄执行循环，**不依赖本地模型，也不捆绑完整 AI Edge Gallery**。借鉴 Google AI Edge Gallery（https://github.com/google-ai-edge/gallery）的 SKILL.md、MCP、文件/URL 导入、授权 UI 与适度插件机制，并重新审查其安全模式。

Skills 是含 `SKILL.md` 的**说明和声明式能力入口**：安装、启停、版本控制、权限清单、需运行时识别。Windows Python/Node Skill **不能原样在 Android 执行**；可把规则移植为结构化提示 + 主机工具，或受信服务进行 APKG 构建。可选 MCP 客户端需 HTTPS/受控链路，连接权限不能大于用户授权。

### G. 制卡工作流复用

用户现有 `card-forge-v2` v2.5.1：Prism 默认卡面，`prism-deck/v1` JSON，真正 `科目::章节::题型` 多级子牌组、稳定 GUID/模型及审计。Harness 支持 文件→分析与检索→生成结构化 JSON→schema/题目质量校验→预览/批量编辑→用户批准→调用 AnkiDroid 原生 note/model/deck/media/import API 写入→验证数量/结构/重复/同步。需要共享或原系统脚本时再经可信服务生成 `.apkg`；本地直接导入现成 APKG 是独立功能。绝不修改现有 FSRS 学习历史来“重新导入”。

## 3. 数据/动作统一技术架构

```text
Android native surfaces (Home, Browser, Stats, Study, More, Agent)
          |
Agent Harness in-app: messages / Skills / memory / context / audit
    |                       |
    | HTTPS                 v
    +--> Cloud LLM      Capability Registry (typed tools)
                            |
                    Permission / approval / diff / undo
                            |
              Anki Services & Settings / Themes / Extensions
                            |
                Anki Collection / FSRS / revlog / Media / Sync
```

建议工具域及示例：

| 域 | 读 | 写 |
|---|---|---|
| `collection` | profile/info/counts/health | backup/restore(高风险) |
| `deck` | list/tree/options/stats | create/rename/move/preset/delete |
| `note/card` | find/get/history/schema | add/edit/flag/bury/suspend/delete |
| `scheduler` | FSRS stats/retention/projection | preset update/optimize(需确认)/reposition |
| `settings` | list/get/defaults | set/reset via validated registry |
| `media/import` | fileinfo/preview/duplicates | import with confirmation |
| `theme` | list/preview/current | generate/install/apply/restore |
| `extension` | search/list/inspect/compat | install/enable/update/uninstall |
| `skill` | search/list/inspect | import/enable/disable |
| `integration` | status/capabilities/redacted config | add/remove/authorize/revoke |
| `ui` | current screen/selection | navigate/open editor/focus context |
| `memory` | preferences/decisions/derived habits | save/edit/forget/export |

技术风险：`DeckOptions.kt` 是 WebView 与内部后端数据流；原生设置分布在众多 `preferences_*.xml`、Prefs/PreferencesFragment；公开 `AddContentApi` 不是全部内部操作接口。实现“近乎全覆盖”要逐项桥接业务 service、测试负例，不应直接赋予 LLM 一个可执行任意内部代码的万能工具。设置内部 key 不完全代表稳定公共 API。

## 4. 导航/界面信息架构

推荐底部四入口：**主页 | 浏览 | 统计 | Agent**。更多入口放顶部菜单/抽屉/设置，避免底部过多 Tab；任意页面可快速召唤 Agent，自动带当前牌组/卡片上下文，明确显示可访问范围。

1. **主页/牌组**：多级牌组、计数、到期、同步、新建、搜索及快速制卡。
2. **复习**：官方核心保持；可选 Agent 解释题目/标记待优化，不接管用户评分。
3. **浏览/编辑**：高级检索、字段编辑、标签、筛选/批量操作、模板预览。
4. **统计**：总卡/今日到期/完成/积压、保留率、各学科、习惯与 FSRS 信息的可解释数据源。
5. **Agent 对话**：云 API 模型、附件、上下文、调用计划、实时工具结果、确认/取消、近期历史。
6. **Agent 能力/记忆**：本地偏好与记忆、数据访问范围、Skills 列表、可见的工具覆盖矩阵、成本与模型上下文预算。
7. **牌组选项/参数**：官方选项可编辑，新增“用 Agent 解释/建议/预览变更”入口。
8. **应用设置**：完整分类/搜索、默认值/当前值/参数说明、Agent 可否读/写标签。
9. **主题工作室**：官方恢复、主题包市场/本地安装、编辑预览、内置图标、双主题效果。
10. **扩展中心**：发现/已安装/更新/权限与日志（Themes/JS/Skills/Integrations 分类）。
11. **模型/互联**：Provider/baseURL/key、本地加密存储、模型与权限、MCP 的 server/tool 列表。
12. **文件/导入工作台**：文件预览、APKG 内容检查、Prism JSON、写入验证与 QA 报告。
13. **操作记录/恢复**：变更日志、按对象和时间检索、undo/备份、版本回退与安全模式。

## 5. 已验证的工程基础

- 独立 Fork `lite-milkfrog/Anki-Android`、源码已克隆；评估分支 `eval/ui-plugin-market`。
- `AnkiDroid/src/main/java/com/ichi2/themes/Themes.kt` 采用 `PrefsRepository`/静态 `R.style`，`com/ichi2/compose/theme/Theme.kt` 提供 Material3 桥；部分界面为 XML View + Compose，牌组选项是 `pages/DeckOptions.kt` 的 WebView。
- 设置头部在 `res/xml/preference_headers.xml`，有 General / Reviewing / Sync / Notifications / Appearance / Profile / Controls / Accessibility / Backup / Advanced 等分类。存在 `settings/Prefs.kt` 供配置读取；`api/.../AddContentApi.kt` 已提供 `addNote`、批量 `addNotes`、`addMediaFromUri`、`findDuplicateNotes`、`updateNoteFields/Tags`、`addNewDeck` 等；非完整 Agent Tool API。
- 有 `jsaddons/AddonData.kt`、`TgzPackageExtract.kt` 和 Reviewer JS API v0.0.3；**不代表完整扩展市场可用**。
- Windows 有 JDK、Android SDK/Emulator，检测到 `CloudCLIRemote_API_36_1` 和 `Medium_Phone_API_36.1` 两个 AVD；ADB 当时没有已启动设备，尚无 APK build/设备截图/性能测试。**不得把环境可用性充作测试通过。**

## 6. 评估与分期

| 工作包 | 难度主观评分 | 结论 |
|---|---|---|
| 基线 Android Debug APK、模拟器、源码/数据回归 | 3/5 | 优先验证 |
| 可安装的主题包、可恢复官方主题和图标 | 3–4/5 | GO（需要动态样式适配） |
| 完整设置目录及基础读写工具 | 4/5 | GO（逐模块补齐） |
| 全集数据查询/习惯/本地记忆与上下文预算 | 4/5 | GO（需权限/隐私/性能设计） |
| 云端模型 + 手机上的轻量 Agent Harness | 4/5 | GO，Koog Android Spike 后择优 |
| Prism 自动制卡、文件、原生导入 | 4/5 | GO（复用现有 Skill/Anki 后端） |
| 插件商店 + 第三方安全隔离、版本回滚 | 5/5 | CONDITIONAL GO |
| 对所有用户操作进行 Agent 能力映射 | 5/5 | 长期里程碑，不能宣称一步到位 |

验收顺序：
P0：Debug 包 + 两个 AVD 启动/运行、官方 UI/数据快照、独立包名、备份/同步门禁。
P1：主题协议、官方恢复、主题切换、关键屏幕。
P2：云端 Harness、API 配置、只读数据目录、Agent 本地记忆、上下文/日志。
P3：Agent 可调用的卡片/牌组/FSRS/设置工具、逐域权限/取消/undo 负例。
P4：Skills 导入、Prism JSON、文件上传、APKG 导入和外部分享，全部经过真实设备验收。
P5：主题工作室和插件中心，审核目录、安装、启停、更新、异常恢复。
P6：扩大到剩余设置/工具、MCP、自定义 API、多 Profile、权限与资源压力测试。

**当前仅完成 Fork、源码审查、路线设计、Skills 与 AVD 发现。产品工程交付进度不可由研究比例推导；主要功能仍未实施。**

## 7. 风险和非功能目标

- **数据完整性第一**：独立测试应用 ID / profile，正式资料不被覆盖；可靠备份、事务和回滚。
- **云端隐私**：只传问题所需的卡片/摘要，不默认上传全部牌组/本地记忆；可配置不向云发送某一类数据；密钥用 Android 安全存储。
- **工具授权**：读/搜索与写/批量/全局重排/删除分级，安装插件和授权敏感设置需明确用户动作；插件和 Skills 无法提升自身权限。
- **操作可审计**：每个工具调用保存参数、结果、时间戳、对象、风险、取消/回滚，敏感信息脱敏。
- **真实“所有数据”**：软件**拥有且当前授权可读取**的内容应能检索，但平台沙箱、AnkiWeb 服务端独有数据、ChatGPT 私有记忆、第三方软件文件和加密密钥不等于可自动读取。
- **升级维护**：持续与官方 AnkiDroid 上游合并，不长期魔改核心调度；遵守 upstream `AI_POLICY.md`，独立 Fork 开发与向官方仓库提交贡献是两个不同程序。

外部参考：https://docs.koog.ai/ ; https://docs.ankiweb.net/deck-options ; https://github.com/google-ai-edge/gallery ; https://github.com/ankidroid/Anki-Android .
