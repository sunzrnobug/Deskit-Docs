# Deskit 技术选型论证

| 项 | 内容 |
| --- | --- |
| 文档状态 | ✅ Accepted |
| 版本 | v1.0 |
| 决策人 | Tech Lead + 全体研发评审 |
| 关联 | [架构设计](../02-architecture/architecture.md) · [竞品分析](../00-product/competitive-analysis.md) |

---

## 0. 选型方法论

每个关键决策遵循：**候选枚举 → 评估维度加权打分 → 决策矩阵 → ADR 记录**。
评分制：1（差）~ 5（优）；总分 = Σ(维度分 × 权重)。权重之和为 100%。
约束前提：**训练营周期约 2 周、前端组、推荐 Electron、需跨平台、需可扩展插件生态**。

---

## ADR-001：桌面应用外壳 → **Electron**

**候选**：Electron / Tauri / Flutter Desktop / 原生(Qt/Swift+WinUI)

| 维度 | 权重 | Electron | Tauri | Flutter | 原生 |
| --- | --- | :---: | :---: | :---: | :---: |
| 跨平台一致性 | 20% | 5 | 4 | 4 | 2 |
| 前端组上手成本 | 20% | 5 | 4 | 2 | 1 |
| 插件 Web 化契合度 | 20% | 5 | 4 | 2 | 2 |
| 生态/三方库 | 15% | 5 | 3 | 3 | 3 |
| 性能/内存 | 15% | 3 | 5 | 4 | 5 |
| 包体积 | 10% | 2 | 5 | 3 | 5 |
| **加权总分** | | **4.40** | **4.05** | **2.95** | **2.65** |

**决策**：选 **Electron 31+**。
**理由**：①课题明确推荐且竞品（uTools）同栈；②前端组零迁移成本；③插件以 Web 技术编写时，主程序与插件同构，`WebContentsView` 天然适配（见 [插件系统](../02-architecture/plugin-system.md)）；④生态最成熟（globalShortcut/Tray/desktopCapturer/autoUpdater 开箱即用）。
**代价与缓解**：性能/包体劣于 Tauri → 用懒加载、视图池、V8 快照、`asar` + 按需下载、关键路径原生插件（如截图）来兜底；包体目标 < 120MB（[NFR-09](../00-product/PRD.md)）。
**状态**：Accepted。Tauri 作为 **2.0 长期演进备选**（若性能成为核心矛盾）。

---

## ADR-002：UI 框架 → **React 19**

**候选**：React 19 / Vue 3 / Svelte / Solid

| 维度 | 权重 | React | Vue 3 | Svelte | Solid |
| --- | --- | :---: | :---: | :---: | :---: |
| 字节/飞书生态契合 | 25% | 5 | 4 | 2 | 2 |
| 人才池/招聘 | 15% | 5 | 4 | 2 | 1 |
| 插件生态对齐(Raycast 用 React) | 20% | 5 | 3 | 2 | 2 |
| 组件库/设计系统(Arco/Radix) | 15% | 5 | 4 | 2 | 2 |
| 学习曲线 | 10% | 3 | 5 | 4 | 4 |
| 性能 | 15% | 4 | 4 | 5 | 5 |
| **加权总分** | | **4.55** | **3.95** | **2.55** | **2.45** |

**决策**：选 **React 19 + TypeScript（strict）**。
**理由**：①字节/飞书内部 React 为主，与训练营目标生态一致；②Raycast 扩展即用 React，**插件作者复用同一心智**，利于生态；③Arco Design（字节自研）与 Radix UI 一线组件资源充沛；④并发特性（`useTransition`）利于"边输入边搜索"不卡顿。
**代价**：相比 Vue 3 学习曲线略陡（uTools 用 Vue）→ 通过脚手架模板、ESLint 规则、组件库屏蔽复杂度。
**状态**：Accepted。

> 说明：用户将框架决策授权给技术评审，此处以**字节/飞书生态对齐 + 插件作者心智统一**为决定性因素选定 React。若团队 Vue 背景更强，可在 ADR 中走 Superseded 流程切换，架构其余部分不受影响（UI 层隔离良好）。

---

## ADR-003：构建与打包 → **electron-vite + electron-builder**

**候选**：electron-vite / electron-forge / Webpack 手搓 / electron-builder 单用

| 维度 | 权重 | electron-vite(+builder) | electron-forge | Webpack 手搓 |
| --- | --- | :---: | :---: | :---: |
| 开发体验/HMR | 30% | 5 | 4 | 2 |
| 多平台打包+签名+公证 | 25% | 5 | 4 | 3 |
| 配置复杂度 | 20% | 5 | 4 | 1 |
| 自动更新集成 | 15% | 5 | 3 | 3 |
| 社区活跃 | 10% | 5 | 4 | 3 |
| **加权总分** | | **5.0** | **3.9** | **2.25** |

**决策**：开发用 **electron-vite**（main/preload/renderer 三端 Vite 编译 + 渲染层 HMR），打包用 **electron-builder**（nsis/dmg/AppImage + 代码签名 + `electron-updater` 自动更新一体）。
**状态**：Accepted。详见 [CI/CD 与发布](../04-implementation/cicd-release.md)。

---

## ADR-004：状态管理 → **Zustand + TanStack Query**

**候选**：Redux Toolkit / Zustand / Jotai / MobX（本地态）；TanStack Query（服务端态）

**决策**：本地 UI 态用 **Zustand**（极简、无样板、易在 main/renderer 共享逻辑）；服务端态（市场列表、同步状态）用 **TanStack Query**（缓存/重试/失效一站式）。
**理由**：Redux 样板过重，训练营周期下 Zustand 性价比最高；服务端态与本地态分治避免"全塞 Redux"的反模式。
**状态**：Accepted。

---

## ADR-005：样式与主题 → **Tailwind CSS + CSS Variables + Radix UI**

**决策**：
- **Tailwind CSS** 做原子化样式，开发快、产物小。
- **CSS Variables（设计令牌）** 驱动**深浅色与换肤**：主题切换只换变量值，主程序与插件统一继承（满足 FR-005/006）。
- **Radix UI** 提供无障碍交互基元（Dialog/Popover/DropdownMenu），叠加 Tailwind 皮肤。
- 设置类复杂表单可选用 **Arco Design**（字节自研，与生态一致）。

**理由**：换肤需求要求"运行时主题切换且插件继承"，CSS 变量是唯一低成本满足者；Radix 解决可访问性这一大厂硬指标。
**状态**：Accepted。设计令牌规范见 [架构设计 §主题](../02-architecture/architecture.md)。

---

## ADR-006：本地存储 → **better-sqlite3 + Drizzle ORM（结构化） + electron-store（配置）**

**候选**：SQLite(better-sqlite3) / lowdb(JSON) / IndexedDB / LevelDB

| 维度 | 权重 | better-sqlite3 | lowdb | IndexedDB | LevelDB |
| --- | --- | :---: | :---: | :---: | :---: |
| 查询能力(剪贴板检索/分组) | 30% | 5 | 2 | 3 | 2 |
| 性能(同步API/万级数据) | 25% | 5 | 2 | 3 | 4 |
| 事务/迁移 | 20% | 5 | 1 | 3 | 2 |
| 类型安全(配合 Drizzle) | 15% | 5 | 3 | 3 | 2 |
| 易用性 | 10% | 4 | 5 | 3 | 3 |
| **加权总分** | | **4.85** | **2.25** | **3.0** | **2.6** |

**决策**：结构化数据（剪贴板历史、插件注册表、同步元数据）用 **better-sqlite3 + Drizzle ORM**（类型安全 + 迁移）；轻量配置（语言/主题/快捷键）用 **electron-store**（JSON KV，简单可靠）。
**状态**：Accepted。Schema 与迁移见 [数据模型](../03-design/data-model.md)。

---

## ADR-007：插件运行时与隔离 → **WebContentsView + preload 桥 + 能力清单（渐进式沙箱）**

**候选**：①`WebContentsView`（新版 BrowserView）隔离 + preload；②`<iframe sandbox>`；③Node `vm`/子进程沙箱；④`<webview>` 标签

**决策**：**主路线 = `WebContentsView` + `contextIsolation` + 受限 preload API + 能力清单（manifest 权限声明 + 用户授权）**；对**不可信第三方插件**演进到**子进程/`vm` 隔离 + IPC 代理**（二阶段）。
**理由**：
- `WebContentsView` 是官方推荐、替代废弃 `BrowserView` 的隔离单元，独立 `webContents` 进程级隔离，性能与隔离平衡最佳。
- `<webview>` 官方不推荐（不稳定）；纯 `iframe` 无法承载需要受控 Node 能力的插件。
- 安全要求（FR-015 ⭐⭐⭐⭐⭐）通过"**默认零权限 + 声明式能力清单 + 运行时授权 + 签名校验**"满足。
**状态**：Accepted。完整设计见 [插件系统设计](../02-architecture/plugin-system.md) 与 [安全设计](../02-architecture/security.md)。

---

## ADR-008：应用市场 / 同步后端 → **NestJS + PostgreSQL + Prisma + 对象存储 + Ed25519**

**候选语言/框架**：NestJS(Node) / Go(Gin) / Spring Boot
**决策**：**NestJS**（与前端同 TS 技术栈，团队复用类型与心智，模块化/DI 适合中型服务）。
- 数据库：**PostgreSQL + Prisma**（关系建模 + 类型安全迁移）。
- 插件包存储：**对象存储（S3 / 自建 MinIO）+ CDN**。
- 包完整性：**Ed25519 签名**，客户端验签后安装。
- 鉴权：**OAuth2/OIDC + JWT**（登录态），上传走 RBAC。
**理由**：训练营全程 TS，前后端共享 `packages/shared` 协议类型，减少认知切换；NestJS 工程规范度高，符合大厂分层。
**状态**：Accepted。接口见 [API/IPC 设计](../03-design/api-ipc.md)。

---

## ADR-009：数据同步策略 → **离线优先 + 增量 + 版本向量 + LWW/CRDT + 端到端加密**

**决策**：
- **离线优先**：本地为真源，联网做增量同步（满足 NFR 离线可用）。
- **冲突解决**：普通设置用 **LWW（Last-Write-Wins）+ 版本向量**；剪贴板等列表型用 **CRDT（OR-Set/LWW-Element-Set）** 保证收敛无丢失。
- **隐私**：**端到端加密（AES-256-GCM，密钥由用户口令经 Argon2id 派生）**，服务端只存密文（满足 FR-015/NFR-08）。
**理由**：兼顾"跨设备一致"与"隐私安全"两大硬约束；CRDT 用于易冲突的列表数据，LWW 用于标量配置，复杂度可控。
**状态**：Accepted。详见 [数据同步设计](../02-architecture/data-sync.md)。

---

## ADR-010：局域网传输 → **mDNS 发现 + WebSocket/WebRTC 传输**

**决策**：用 **mDNS（bonjour-service）** 做零配置设备发现；小文本走 **WebSocket**，大文件走 **WebRTC DataChannel**（P2P 直传，避免中转），降级到本地 HTTP 直连。
**理由**：满足"同 WiFi 自动发现 + 大文件直传"（FR-050/051），无需公网与服务器中转，隐私友好。
**状态**：Accepted。

---

## ADR-011：截图实现 → **desktopCapturer + 选区遮罩窗 + Canvas 标注（必要时原生插件）**

**决策**：默认纯前端方案——`desktopCapturer` 抓屏 → 全屏透明置顶窗渲染遮罩选区 → Canvas 做标注/马赛克 → 贴图为独立置顶小窗。多屏 DPI 缩放需专门处理。若性能/取色不足，引入 **原生 Node 插件**（如 windows 端 GDI / mac ScreenCaptureKit）。
**理由**：先用 Electron 原生 API 快速达成 P0（选区截图/贴图为 P0），复杂度可控；为高难挑战（标注/马赛克）预留原生升级路径。
**状态**：Accepted。

---

## ADR-012：工程结构 → **pnpm workspaces + Turborepo（Monorepo）**

**决策**：单仓多包，`apps/*`（desktop、server）+ `packages/*`（plugin-sdk、ui、shared、ipc-contract）+ `plugins/*`（内置插件）。**Turborepo** 做任务编排与增量构建缓存。
**理由**：前端、后端、SDK、插件强关联，共享类型与协议；Monorepo 保证"改协议→全链路类型报错"的一致性，是大厂标准实践。
**状态**：Accepted。

---

## ADR-013：测试体系 → **Vitest + Testing Library + Playwright(Electron)**

**决策**：单元/逻辑用 **Vitest**；React 组件用 **Testing Library**；E2E 用 **Playwright 的 Electron 支持**（替代已停更的 Spectron）。
**状态**：Accepted。详见 [测试方案](../05-quality/test-plan.md)。

---

## ADR-014：可观测性 → **electron-log + Sentry + 可关闭遥测**

**决策**：本地日志 **electron-log**（分级、滚动、可导出）；崩溃与异常 **Sentry**（带 sourcemap）；业务埋点自研轻量 SDK，**默认可关闭、匿名化**（满足隐私 NFR-08）。
**状态**：Accepted。

---

## 选型总览（决策一览表）

| ADR | 领域 | 结论 |
| --- | --- | --- |
| 001 | 外壳 | Electron 31+ |
| 002 | UI | React 19 + TS strict |
| 003 | 构建 | electron-vite + electron-builder |
| 004 | 状态 | Zustand + TanStack Query |
| 005 | 样式/主题 | Tailwind + CSS Variables + Radix（+Arco 可选） |
| 006 | 本地存储 | better-sqlite3 + Drizzle + electron-store |
| 007 | 插件运行时 | WebContentsView + preload + 能力清单（渐进沙箱） |
| 008 | 后端 | NestJS + PostgreSQL + Prisma + 对象存储 + Ed25519 |
| 009 | 同步 | 离线优先 + 增量 + 版本向量 + LWW/CRDT + E2E 加密 |
| 010 | 局域网 | mDNS + WebSocket/WebRTC |
| 011 | 截图 | desktopCapturer + 遮罩窗 + Canvas（原生兜底） |
| 012 | 工程 | pnpm workspaces + Turborepo |
| 013 | 测试 | Vitest + Testing Library + Playwright |
| 014 | 可观测 | electron-log + Sentry + 可关闭遥测 |

> 所有 ADR 与代码同仓维护；后续若推翻某决策，新增 ADR 并将旧条目标记 `Superseded by ADR-0XX`，保留演进可追溯性。
