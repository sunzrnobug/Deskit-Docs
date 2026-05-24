# Deskit 系统架构设计

| 项 | 内容 |
| --- | --- |
| 文档状态 | ✅ Reviewed |
| 版本 | v1.0 |
| 关联 | [技术选型](../01-tech-selection/tech-selection.md) · [插件系统](./plugin-system.md) · [安全设计](./security.md) · [API/IPC](../03-design/api-ipc.md) |

---

## 1. 架构总览（C4 - Context）

```mermaid
graph TB
  User([用户]) -->|快捷键/悬浮球| Desktop[Deskit 桌面端<br/>Electron]
  Dev([插件开发者]) -->|SDK/CLI 开发上传| Desktop
  Desktop -->|安装/更新/上传插件| Market[(应用市场服务<br/>NestJS)]
  Desktop -->|端到端加密同步| Sync[(同步服务<br/>NestJS)]
  Desktop <-->|mDNS + WebRTC| Peer[局域网内其他设备]
  Market --> ObjStore[(对象存储/CDN<br/>插件包)]
  Market --> PG[(PostgreSQL)]
  Sync --> PG
  Desktop -->|崩溃/日志| Sentry[(Sentry)]
```

## 2. 进程模型（Electron 多进程）

Deskit 遵循 Electron 安全最佳实践：**主进程掌权、渲染进程零 Node、插件进程隔离**。

```mermaid
graph TB
  subgraph Main["主进程 (Main Process · Node 全权限)"]
    Core[核心内核 Kernel]
    SVC[系统服务层<br/>快捷键/托盘/窗口/存储/同步]
    PM[插件管理器 PluginManager]
    SEC[安全/权限网关 SecurityGateway]
  end

  subgraph RenderHost["宿主渲染进程 (Renderer · 无 Node)"]
    Launcher[命令面板 / 设置 / 市场 UI<br/>React]
    PreloadH[[preload: 受限 deskit API]]
  end

  subgraph FloatWin["悬浮球窗口 (Renderer)"]
    Ball[FloatingBall]
  end

  subgraph PluginViews["插件视图 (WebContentsView × N · 隔离)"]
    P1[插件A]
    P2[插件B]
    PreloadP[[preload: 能力受限 Plugin API]]
  end

  Launcher <-->|contextBridge| PreloadH
  PreloadH <-->|ipcRenderer.invoke| Core
  P1 <-->|contextBridge| PreloadP
  PreloadP <-->|受控 IPC| SEC
  SEC --> Core
  Core --> SVC
  Core --> PM
  PM --> PluginViews
```

**关键原则**（详见 [安全设计](./security.md)）：
- 所有渲染进程 `nodeIntegration:false`、`contextIsolation:true`、`sandbox:true`。
- 渲染层只能通过 `contextBridge` 暴露的**白名单 API** 与主进程通信。
- 插件视图的 IPC 必须经 **SecurityGateway** 校验权限后才转发到核心服务。

## 3. 分层架构（逻辑分层）

```mermaid
graph TB
  subgraph L1[表现层 Presentation]
    UI1[命令面板] 
    UI2[设置中心]
    UI3[应用市场]
    UI4[悬浮球]
    UI5[插件 UI]
  end
  subgraph L2[应用层 Application · 主进程编排]
    CMD[命令调度 CommandRouter]
    LIFE[插件生命周期]
    SYNCO[同步编排]
  end
  subgraph L3[领域服务层 Domain Services]
    S1[搜索引擎 SearchEngine]
    S2[插件管理 PluginManager]
    S3[剪贴板服务 ClipboardSvc]
    S4[同步服务 SyncSvc]
    S5[主题/i18n 服务]
    S6[局域网服务 LanSvc]
    S7[截图服务 CaptureSvc]
  end
  subgraph L4[基础设施层 Infrastructure]
    I1[(SQLite/Drizzle)]
    I2[(electron-store)]
    I3[加密模块 Crypto]
    I4[网络客户端 HttpClient]
    I5[日志/遥测]
    I6[原生能力 globalShortcut/Tray/...]
  end
  L1 --> L2 --> L3 --> L4
```

| 层 | 职责 | 不可做 |
| --- | --- | --- |
| 表现层 | 渲染、交互、状态展示 | 直接访问文件/网络/数据库 |
| 应用层 | 编排用例、路由命令、协调服务 | 写具体业务算法 |
| 领域服务层 | 业务逻辑（搜索/插件/同步/剪贴板） | 关心 UI/具体存储实现 |
| 基础设施层 | 存储/加密/网络/原生 API 适配 | 含业务规则 |

## 4. 核心模块设计

### 4.1 命令调度内核（CommandRouter）
命令面板的"大脑"，将用户输入路由到不同处理器：

```mermaid
flowchart LR
  Input[用户输入] --> Tokenize[解析/分词]
  Tokenize --> Match{多源并行匹配}
  Match --> A[内置命令 Provider]
  Match --> B[插件命令 Provider]
  Match --> C[应用搜索 Provider]
  Match --> D[即时计算/时间戳 Provider]
  A & B & C & D --> Rank[统一排序<br/>Fuse.js+拼音+使用频率]
  Rank --> Render[结果列表渲染]
  Render --> Exec[回车执行→对应 Provider.run]
```

- **Provider 模式**：每类结果源实现统一 `CommandProvider` 接口（`query()` / `run()`），内置功能与插件平权接入。
- **排序**：模糊匹配分 + 拼音匹配 + 历史使用频率（LRU/频次加权），保证常用命令置顶。
- **性能**：`useTransition` + 防抖 + Web Worker 内匹配，保证输入 < 50ms 响应（NFR-01）。

### 4.2 窗口管理（WindowManager）
| 窗口 | 类型 | 特性 |
| --- | --- | --- |
| 主窗（命令面板） | 无边框、居中、失焦自动隐藏 | 快捷键唤起，常驻后台 |
| 悬浮球 | 无边框、透明、置顶、可拖拽吸边 | 多屏适配，单击唤主窗 |
| 设置/市场 | 标准窗口 | 独立打开 |
| 插件视图 | `WebContentsView` 挂载于主窗 | 池化复用，隔离 |
| 截图遮罩 | 全屏透明置顶 | 多屏拼接坐标系 |
| 贴图窗 | 无边框置顶小窗 | 每张图一个实例 |

### 4.3 主题与设计令牌系统（满足 FR-005/006）
- 设计令牌（Design Tokens）以 CSS 变量承载，分三层：**基础调色板 → 语义令牌 → 组件令牌**。

```css
:root {                 /* 语义令牌（浅色） */
  --fx-color-bg: #ffffff;
  --fx-color-fg: #1f2329;
  --fx-color-primary: #3370ff;   /* 飞书蓝，可被换肤覆盖 */
  --fx-radius-md: 8px;
}
[data-theme="dark"] {   /* 深色覆盖 */
  --fx-color-bg: #1f1f1f;
  --fx-color-fg: #e8e8e8;
}
[data-skin="forest"] {  /* 换肤：仅改主色族 */
  --fx-color-primary: #2bab6b;
}
```

- **运行时切换**：JS 改 `<html data-theme data-skin>` 属性即可，**无重渲染成本**。
- **插件继承**：插件 `WebContentsView` 注入同一份令牌 CSS，主程序广播 `theme:changed` 事件，插件实时跟随（满足"切换实时生效含已开插件"）。
- 用户自定义主色 → 写入 `electron-store` → 同步服务下发到其他设备。

### 4.4 国际化（i18n）
- `i18next` + `react-i18next`，命名空间按模块拆分，懒加载。
- 语言资源：`packages/shared/locales/{zh-CN,en-US}/*.json`。
- 插件可注册自身语言包，主程序合并命名空间。
- 默认跟随系统语言（`app.getLocale()`），用户可覆盖并同步。

### 4.5 存储拓扑
```mermaid
graph LR
  subgraph 本地
    A[electron-store<br/>设置/快捷键/主题] 
    B[(SQLite<br/>剪贴板/插件注册/同步元数据)]
    C[文件系统<br/>插件包/截图/缓存]
    D[OS 安全凭据库<br/>同步密钥 keytar]
  end
  A & B -.增量加密.-> S[(同步服务)]
  D -.密钥不出端.-> X[端到端加密]
```

详细 schema 见 [数据模型](../03-design/data-model.md)。

## 5. 关键时序

### 5.1 快捷键唤起命令面板（FR-001/002）
```mermaid
sequenceDiagram
  participant U as 用户
  participant OS as 操作系统
  participant M as 主进程
  participant W as 主窗(Renderer)
  U->>OS: 按下 Alt+Space
  OS->>M: globalShortcut 回调
  M->>M: WindowManager.toggleLauncher()
  M->>W: 显示并聚焦(预热窗口,不销毁)
  U->>W: 输入关键字
  W->>M: ipc invoke command:query(text)
  M->>M: CommandRouter 多 Provider 并行匹配+排序
  M-->>W: 返回结果列表
  U->>W: 回车
  W->>M: ipc invoke command:run(id)
  M->>M: 路由到对应 Provider.run
```
> 性能关键：主窗**预创建并隐藏**（非每次新建），唤起即 show，达成 < 300ms（NFR-01）。

### 5.2 插件调用受控能力（与安全联动）
```mermaid
sequenceDiagram
  participant P as 插件(WebContentsView)
  participant PB as preload(Plugin API)
  participant SG as SecurityGateway(主进程)
  participant SVC as 领域服务
  P->>PB: deskit.clipboard.read()
  PB->>SG: ipc invoke (channel, pluginId, payload)
  SG->>SG: 校验 pluginId 是否声明 clipboard 权限 & 用户已授权
  alt 已授权
    SG->>SVC: 转发调用
    SVC-->>P: 返回结果
  else 未授权
    SG-->>P: 抛出 PermissionDenied
  end
```

## 6. 目录结构（apps/desktop）

```text
apps/desktop/src/
├─ main/                      # 主进程
│  ├─ kernel/                 # CommandRouter / 事件总线
│  ├─ windows/                # WindowManager + 各窗口工厂
│  ├─ services/               # search/clipboard/sync/lan/capture/theme/i18n
│  ├─ plugins/                # PluginManager + 加载/沙箱/生命周期
│  ├─ security/               # SecurityGateway + 权限模型
│  ├─ ipc/                    # IPC handler 注册（按契约）
│  └─ infra/                  # db(drizzle)/store/crypto/http/log
├─ preload/
│  ├─ host.ts                 # 宿主渲染层 API 桥
│  └─ plugin.ts               # 插件受限 API 桥
├─ renderer/                  # React 应用
│  ├─ launcher/               # 命令面板
│  ├─ settings/               # 设置中心
│  ├─ market/                 # 应用市场 UI
│  ├─ floating-ball/          # 悬浮球
│  └─ shared-ui/              # 复用 packages/ui
└─ shared/                    # 主/渲染共享类型（来自 packages/shared）
```

## 7. 跨切面关注点（Cross-cutting）

| 关注点  | 方案                                                         |
| ---- | ---------------------------------------------------------- |
| 错误处理 | 统一 `Result<T,E>`/错误码；主进程兜底 `uncaughtException`；UI 错误边界     |
| 日志   | electron-log 分级滚动；插件日志单独通道，开发者模式可见                         |
| 配置   | electron-store + schema 校验；环境区分 dev/prod                   |
| 性能   | 窗口预热、视图池、懒加载路由、Worker 计算、虚拟列表                              |
| 可访问性 | Radix 基元 + 键盘全可达 + 焦点管理                                    |
| 可测试性 | 服务层依赖注入，IPC 契约可 mock（见 [测试方案](../05-quality/test-plan.md)） |

## 8. 架构演进路线
- **v1.0**：WebContentsView 插件隔离 + 自建同步/市场。
- **v1.x**：不可信插件子进程/`vm` 沙箱（[安全设计 §演进](./security.md)）。
- **v2.0（备选）**：评估 Tauri 重写性能敏感外壳，插件层协议保持稳定以平滑迁移。

> 架构稳定性策略：**协议（IPC 契约 + 插件 manifest + 同步协议）优先稳定**，实现可替换。所有跨边界交互都以 `packages/ipc-contract` 与 `packages/shared` 的类型为唯一事实源。
