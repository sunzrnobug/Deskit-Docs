# 03 详细设计

本章聚焦系统的**内部契约与数据结构**，是开发者实现功能时最直接的参考。包括主进程与渲染/插件进程间的类型安全 IPC 协议、对外 REST API 规范，以及所有持久化数据的 Schema 与存储策略。

---

## 文档目录

| 文档 | 说明 |
| --- | --- |
| [接口与 IPC 设计](./api-ipc.md) | 类型安全 IPC 封装（`ipc-contract`）、主进程处理器注册规范、插件侧 `window.deskit.*` API 映射、服务端 REST API 端点（市场 / 同步 / 鉴权）、错误码体系 |
| [数据模型与存储设计](./data-model.md) | SQLite 表结构（剪贴板 / 插件注册表 / 同步元数据）、Drizzle ORM Schema、electron-store 配置 KV 结构、FTS5 全文检索索引、数据迁移策略 |
