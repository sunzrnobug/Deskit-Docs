# 02 系统架构

本章从宏观到专题，完整描述 Deskit 的**技术架构**。包括整体分层与进程模型、插件生态的隔离与扩展机制、跨设备数据同步的离线优先策略，以及贯穿全系统的安全防护体系。

各文档相互关联：架构设计是总纲，插件系统、数据同步、安全设计是三大专题深化。

---

## 文档目录

| 文档 | 说明 |
| --- | --- |
| [系统架构设计](./architecture.md) | 整体分层（主进程 / 渲染进程 / 插件进程 / 服务端）、核心模块、技术栈概览、关键非功能约束（性能 / 安全 / 跨平台） |
| [插件系统设计](./plugin-system.md) | 插件生命周期、WebContentsView 沙箱隔离、manifest 能力清单与权限授权、plugin-sdk 与 `create-deskit-plugin` CLI、开发者 API `window.deskit.*` |
| [数据同步设计](./data-sync.md) | 离线优先架构、增量同步协议、LWW + CRDT 冲突解决、端到端加密（AES-256-GCM + Argon2id）、多 Provider 抽象 |
| [安全设计](./security.md) | 威胁模型、CSP / contextIsolation 基线、SecurityGateway 鉴权、插件签名与验签、隐私保护与审计日志 |
