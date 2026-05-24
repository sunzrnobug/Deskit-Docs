# 桌面端提效工具箱 · 项目文档体系

> 课题：26 飞书工程训练营 - 前端组 - 实现一个桌面端提效工具箱
> 竞品：uTools / Alfred / Raycast
> 项目代号：**Deskit**（以下文档统一使用该代号）

本目录是 Deskit 的**完整研发文档体系**，对标一线大厂（字节 / 飞书）的产品-技术-工程-质量全流程。任何一名新成员可凭本目录从 0 启动、开发、测试、发布本项目。

---

## 1. 文档导航（按研发生命周期组织）

| 阶段 | 文档 | 说明 | 负责角色 |
| --- | --- | --- | --- |
| **产品** | [00-product/PRD.md](./00-product/PRD.md) | 产品需求文档（背景/目标/用户故事/功能需求/验收标准/指标） | PM |
| | [00-product/competitive-analysis.md](./00-product/competitive-analysis.md) | 竞品分析（uTools/Alfred/Raycast 拆解与定位） | PM |
| **选型** | [01-tech-selection/tech-selection.md](./01-tech-selection/tech-selection.md) | 技术选型论证（含决策矩阵与 ADR） | Tech Lead |
| **架构** | [02-architecture/architecture.md](./02-architecture/architecture.md) | 系统架构设计（进程模型/分层/模块） | Architect |
| | [02-architecture/plugin-system.md](./02-architecture/plugin-system.md) | 插件系统设计（沙箱/权限/生命周期/应用市场） | Architect |
| | [02-architecture/data-sync.md](./02-architecture/data-sync.md) | 数据同步设计（增量同步/冲突解决/端到端加密） | Architect |
| | [02-architecture/security.md](./02-architecture/security.md) | 安全设计（STRIDE 威胁建模/Electron 安全基线） | Security |
| **详设** | [03-design/api-ipc.md](./03-design/api-ipc.md) | IPC 与接口契约设计（类型安全 IPC / 服务端 OpenAPI） | Dev |
| | [03-design/data-model.md](./03-design/data-model.md) | 数据模型与存储设计（本地 + 云端 schema/迁移） | Dev |
| **实施** | [04-implementation/roadmap.md](./04-implementation/roadmap.md) | 实施规划（里程碑/Sprint/WBS/甘特图/DoD） | PM + Tech Lead |
| | [04-implementation/engineering-standards.md](./04-implementation/engineering-standards.md) | 研发规范（Git/分支/提交/CodeReview/代码风格） | Tech Lead |
| | [04-implementation/cicd-release.md](./04-implementation/cicd-release.md) | CI/CD 与发布流程（构建/签名/自动更新/灰度） | DevOps |
| **质量** | [05-quality/test-plan.md](./05-quality/test-plan.md) | 测试方案（测试金字塔/用例/覆盖率/性能） | QA |
| | [05-quality/risk-management.md](./05-quality/risk-management.md) | 风险登记册（技术/进度/安全/合规风险与缓解） | PM |

---

## 2. 技术栈速览（结论，论证见技术选型文档）

| 维度 | 选型 | 一句话理由 |
| --- | --- | --- |
| 桌面外壳 | **Electron 31+** | 生态成熟、竞品同栈、插件 Web 化最自然 |
| 语言 | **TypeScript 5（strict）** | 大型工程类型安全基线 |
| UI 框架 | **React 19** | 字节/飞书主流、人才池大、Raycast 同路线 |
| 构建 | **electron-vite + electron-builder** | 极速 HMR + 多平台打包/签名/更新一体 |
| 状态管理 | **Zustand + TanStack Query** | 轻量本地态 + 服务端态分治 |
| 样式/主题 | **Tailwind CSS + CSS Variables + Radix UI** | 原子化 + 运行时换肤 + 无障碍基元 |
| 国际化 | **i18next + react-i18next** | 成熟、支持懒加载与命名空间 |
| 本地存储 | **better-sqlite3 + Drizzle ORM / electron-store** | 结构化高性能 + 配置 KV |
| 模糊搜索 | **Fuse.js + 拼音匹配（pinyin-pro）** | 命令面板中英文+拼音首字母检索 |
| 插件运行时 | **WebContentsView + preload 桥 + 能力清单** | uTools 式隔离，渐进演进到进程/VM 沙箱 |
| 应用市场后端 | **NestJS + PostgreSQL + Prisma + 对象存储 + Ed25519 签名** | 大厂标准服务端栈 |
| 数据同步 | **增量同步 + 版本向量 + LWW/CRDT + AES-256-GCM 端到端加密** | 离线优先、冲突可解、隐私安全 |
| 局域网传输 | **mDNS 发现 + WebSocket/WebRTC 传输** | 零配置发现 + 大文件直传 |
| 截图 | **desktopCapturer + 选区遮罩窗口 + Canvas 标注** | 纯前端可控，必要时上原生插件 |
| 工程结构 | **pnpm workspaces + Turborepo（Monorepo）** | 桌面端/服务端/SDK/共享包一体管理 |
| 测试 | **Vitest + Testing Library + Playwright(Electron)** | 单元/组件/E2E 全覆盖 |
| 可观测性 | **electron-log + Sentry + 自研可关闭遥测** | 崩溃/日志/埋点闭环且尊重隐私 |
| CI/CD | **GitHub Actions（三平台矩阵）+ electron-updater** | 构建-签名-公证-发布-更新全自动 |

---

## 3. 仓库结构（Monorepo 落地形态）

```text
deskit/
├─ apps/
│  ├─ desktop/                # Electron 桌面端（main / preload / renderer）
│  └─ server/                 # 应用市场 + 同步服务（NestJS）
├─ packages/
│  ├─ plugin-sdk/             # 插件开发 SDK + 类型定义 + CLI
│  ├─ ui/                     # 共享 UI 组件库（设计系统落地）
│  ├─ shared/                 # 通用类型 / 协议 / 工具（前后端共享）
│  └─ ipc-contract/           # 类型安全 IPC 契约定义
├─ plugins/                   # 内置官方插件（时间戳/剪贴板/截图/局域网...）
│  ├─ timestamp/
│  ├─ clipboard/
│  ├─ screenshot/
│  └─ lan-share/
├─ docs/                      # ← 本文档体系
├─ e2e/                       # 端到端测试
├─ scripts/                   # 构建/发版/工具脚本
├─ .github/workflows/         # CI/CD
├─ turbo.json
├─ pnpm-workspace.yaml
└─ package.json
```

---

## 4. 文档规范

- **格式**：Markdown（CommonMark），图用 [Mermaid](https://mermaid.js.org/)，可直接在 GitHub/飞书渲染。
- **决策记录**：架构/选型重大决策以 **ADR（Architecture Decision Record）** 形式内嵌，编号 `ADR-00X`，状态 ∈ {Proposed, Accepted, Superseded}。
- **需求编号**：`FR-xxx`（功能需求）/ `NFR-xxx`（非功能需求），全程可追溯到测试用例 `TC-xxx`。
- **优先级**：沿用课题 `P0 / P1`，挑战项以 `⭐` 数标注难度。
- **更新约定**：文档变更走 PR 评审，与代码同仓同生命周期（Docs as Code）。

## 5. 阅读路径建议

- **我是评委 / 新同学**：`README → PRD → architecture → roadmap`。
- **我要写插件**：`plugin-system → api-ipc → packages/plugin-sdk`。
- **我负责安全**：`security → data-sync → plugin-system`。
- **我负责发布**：`cicd-release → engineering-standards → test-plan`。

---

_本文档体系版本：v1.1 ｜ 维护：Deskit 研发组 ｜ 最近更新：2026-05-22（按训练营最新定位调整内置应用优先级：剪贴板/截图 → P0，数据同步 → P1）_
