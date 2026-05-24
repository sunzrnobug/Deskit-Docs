# Deskit 接口与 IPC 设计

| 项 | 内容 |
| --- | --- |
| 文档状态 | ✅ Reviewed |
| 版本 | v1.0 |
| 关联 | [架构设计](../02-architecture/architecture.md) · [插件系统](../02-architecture/plugin-system.md) · [数据同步](../02-architecture/data-sync.md) · [安全设计](../02-architecture/security.md) |

本文定义三类接口契约：**① 进程内 IPC（main↔renderer）、② 插件 API（plugin↔host）、③ 服务端 HTTP API（client↔server）**。所有契约以 `packages/ipc-contract` 与 `packages/shared` 的 TS 类型为**唯一事实源**。

---

## 1. 类型安全 IPC 设计

### 1.1 原则
- **契约先行**：channel 名、入参、返回值在 `ipc-contract` 中以类型定义，main 与 preload 双向引用，编译期保证一致。
- **invoke/handle 优先**：请求-响应用 `ipcRenderer.invoke` ↔ `ipcMain.handle`；单向事件用 `webContents.send` ↔ `ipcRenderer.on`。
- **校验在边界**：handler 入口用 zod 校验 payload，防止渲染层异常数据。
- **统一错误**：返回 `Result<T>`，错误带 `code`，渲染层据码处理与 i18n 提示。

### 1.2 契约定义形态
```ts
// packages/ipc-contract/src/index.ts
export interface IpcContract {
  'command:query':   { req: { text: string }; res: ResultItem[] };
  'command:run':     { req: { id: string; args?: unknown }; res: RunResult };
  'theme:set':       { req: { theme: ThemeMode; skin?: string }; res: void };
  'i18n:set':        { req: { locale: Locale }; res: void };
  'shortcut:set':    { req: { action: string; accelerator: string }; res: { ok: boolean } };
  'clipboard:list':  { req: { query?: string; group?: string; limit: number }; res: ClipItem[] };
  'clipboard:favorite': { req: { id: string; favorite: boolean }; res: void };
  'sync:status':     { req: void; res: SyncStatus };
  'sync:trigger':    { req: void; res: SyncStatus };
  'market:list':     { req: MarketQuery; res: Paginated<PluginMeta> };
  'plugin:install':  { req: { id: string; version?: string }; res: InstallResult };
  'plugin:uninstall':{ req: { id: string }; res: void };
  'plugin:setEnabled': { req: { id: string; enabled: boolean }; res: void };
  'lan:devices':     { req: void; res: LanDevice[] };
  'lan:send':        { req: { deviceId: string; payload: LanPayload }; res: TransferTicket };
  'capture:start':   { req: { mode: 'region' | 'fullscreen' }; res: CaptureResult };
}
```

```ts
// 类型安全封装（renderer 侧）
const ipc = createTypedIpc<IpcContract>();
const results = await ipc.invoke('command:query', { text: 'ts 1700000000' });
//    ^? ResultItem[]   入参与返回均被类型约束
```

### 1.3 命名与版本约定
- channel 命名：`<domain>:<action>`，全小写冒号分隔。
- 事件（主→渲染）：`<domain>:<event>`，如 `theme:changed`、`sync:progress`、`plugin:reloaded`。
- 破坏性变更：新增 `vN` 后缀 channel 并保留旧版一个版本周期。

### 1.4 统一返回与错误码
```ts
type Result<T> = { ok: true; data: T } | { ok: false; code: ErrorCode; message: string };

enum ErrorCode {
  OK = 0,
  INVALID_ARG = 1001,
  PERMISSION_DENIED = 1002,   // 插件能力未授权
  NOT_FOUND = 1003,
  PLUGIN_LOAD_FAILED = 2001,
  SIGNATURE_INVALID = 2002,
  SYNC_CONFLICT = 3001,
  NETWORK_ERROR = 4001,
  INTERNAL = 5000,
}
```

## 2. 主要 IPC channel 清单（按域）

| Channel | 方向 | 用途 | 关联需求 |
| --- | --- | --- | --- |
| `command:query` / `command:run` | R→M invoke | 命令面板搜索/执行 | FR-002 |
| `window:toggleLauncher` | M 内部/全局快捷键 | 唤起/隐藏主窗 | FR-001 |
| `floatingBall:setEnabled` | R→M | 悬浮球开关/位置 | FR-003 |
| `theme:set` / `theme:changed` | R→M / M→R(广播) | 深浅色/换肤切换 | FR-005/006 |
| `i18n:set` / `i18n:changed` | R→M / M→R | 语言切换 | FR-004 |
| `shortcut:set` / `shortcut:list` | R→M | 自定义快捷键 | FR-001 |
| `clipboard:*` | R→M | 历史列表/收藏/分组 | FR-040~042 |
| `clipboard:changed` | M→R | 新剪贴内容事件 | FR-040 |
| `sync:status/trigger` / `sync:progress` | R↔M | 同步状态/触发/进度 | FR-030/031/043 |
| `market:*` / `plugin:*` | R→M | 市场与插件管理 | FR-010~014 |
| `lan:devices/send` / `lan:incoming` | R↔M | 局域网设备/传输 | FR-050/051 |
| `capture:start` / `capture:done` | R↔M | 截图流程 | FR-060~063 |

## 3. 插件 API 契约（plugin↔host）

插件侧通过 preload 注入的 `window.deskit.*`（见 [插件系统 §6.2](../02-architecture/plugin-system.md)）。底层每个调用映射为带身份的受控 IPC：

```ts
// 插件调用 → preload → 受控 IPC（携带不可伪造 pluginId）
'plugin-api:clipboard:read'   { req: {}; res: ClipItem }            // 需 clipboard:read
'plugin-api:storage:get'      { req: { key: string }; res: unknown } // storage:plugin
'plugin-api:storage:set'      { req: { key: string; value: unknown }; res: void }
'plugin-api:fetch'            { req: { url: string; init?: FetchInit }; res: FetchResponse } // network:fetch=origin
'plugin-api:fs:readText'      { req: { path: string }; res: string }  // fs:read=scope
'plugin-api:results:set'      { req: { items: ResultItem[] }; res: void }
'plugin-api:toast'            { req: { type; message }; res: void }
```

- 主进程 `SecurityGateway` 拦截所有 `plugin-api:*`，按 [安全设计 §5](../02-architecture/security.md) 校验权限+scope，再转发领域服务。
- 插件 API 版本随 SDK 语义化版本演进，`minHostVersion` 保证兼容。

## 4. 服务端 HTTP API（client↔server）

RESTful + OpenAPI 3.1（NestJS Swagger 自动生成）。鉴权 `Authorization: Bearer <JWT>`。

### 4.1 应用市场
| 方法 | 路径 | 说明 | 鉴权 |
| --- | --- | --- | --- |
| GET | `/api/v1/plugins` | 列表（分页/分类/关键字/排序） | 否 |
| GET | `/api/v1/plugins/:id` | 详情（含版本历史/权限/评分） | 否 |
| GET | `/api/v1/plugins/:id/versions/:v/download` | 获取临时签名下载 URL | 否 |
| POST | `/api/v1/plugins` | 上传新插件（multipart，含包+manifest） | 是(开发者) |
| POST | `/api/v1/plugins/:id/versions` | 发布新版本 | 是(作者) |
| GET | `/api/v1/plugins/updates?installed=...` | 批量检查更新 | 否 |

### 4.2 账号与鉴权
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/v1/auth/login` | OAuth2/OIDC 登录 |
| POST | `/api/v1/auth/refresh` | 刷新令牌 |
| POST | `/api/v1/auth/logout` | 注销 |
| GET | `/api/v1/devices` / DELETE `/api/v1/devices/:id` | 设备管理/吊销 |

### 4.3 数据同步
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/v1/sync/pull` | 拉取 `> lastSeq` 的密文 op |
| POST | `/api/v1/sync/push` | 推送本地密文 op（乐观并发，冲突返回 latest） |
| GET | `/api/v1/sync/meta` | 同步元信息（serverSeq、配额） |
| DELETE | `/api/v1/sync/data` | 删除云端数据（隐私权利） |

> 同步请求体为**密文 + 路由元数据**，服务端无解密能力（见 [数据同步 §5](../02-architecture/data-sync.md)）。

### 4.4 请求/响应规范
- 统一响应包：`{ code, message, data, traceId }`；错误码与端侧 `ErrorCode` 对齐。
- 分页：`{ items, page, pageSize, total }`。
- 幂等：写接口支持 `Idempotency-Key`（同步 push 防重放）。
- 限流：`429` + `Retry-After`；DTO 用 `class-validator` 校验。

### 4.5 同步 DTO 示例
```ts
// POST /api/v1/sync/push
interface PushRequest {
  deviceId: string;
  baseSeq: number;
  ops: Array<{
    opId: string;          // 幂等去重
    entity: 'setting' | 'plugin-setting' | 'clipboard';
    ciphertext: string;    // base64(AES-256-GCM)
    nonce: string;
    version: VectorClock;  // 冲突解决用
  }>;
}
interface PushResponse {
  accepted: string[];      // opId
  conflict?: { latestOps: EncryptedOp[]; serverSeq: number };
  serverSeq: number;
}
```

## 5. 契约治理
- `packages/shared` 定义跨端领域类型（`ClipItem`/`PluginMeta`/`SyncStatus`...），前后端、插件 SDK 共享，**改类型即全链路类型报错**。
- 服务端用同一份类型生成 OpenAPI，客户端可生成请求 SDK，避免手写漂移。
- 契约变更走 PR 评审 + 版本化；E2E 契约测试守护（见 [测试方案](../05-quality/test-plan.md)）。
