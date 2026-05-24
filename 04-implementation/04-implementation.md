# 04 实施规划

本章覆盖 Deskit 从开发到交付的**完整工程实施体系**。包括 2 周内按天推进的里程碑排期与 WBS 任务分解、保障代码质量的研发规范，以及多平台打包、签名、自动更新的 CI/CD 流程。

---

## 文档目录

| 文档 | 说明 |
| --- | --- |
| [实施规划（里程碑 / Sprint / WBS）](./roadmap.md) | 4 个里程碑（M0 脚手架 → M1 MVP → M2 Alpha → M3 Beta → M4 RC/1.0）、10 天甘特图、WBS 任务分解、Sprint 仪式（D1 Planning / D5 中期校准 / D10 Review）、DoD 与 P1 降级预案 |
| [研发规范](./engineering-standards.md) | Git 分支策略与 Commit 规范（Conventional Commits）、PR / Code Review 流程、代码风格（ESLint / Prettier）、目录结构约定、新成员快速上手指南 |
| [CI/CD 与发布流程](./cicd-release.md) | GitHub Actions 流水线（lint / typecheck / test / build 矩阵）、三平台打包与代码签名公证、electron-updater 自动更新灰度通道、版本号管理策略 |
