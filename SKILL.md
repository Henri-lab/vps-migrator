---
name: vps-migrator
description: 为 GitHub + Docker 的服务器迁移/重建/灾难恢复设计并执行可复核的迁移方案：先扫描旧机生成 Plan（容器/数据卷/数据库/密钥/证书/DNS 清单与风险），用户确认后再 Apply（备份 → 初始化新机 → 部署 → 健康检查 → DNS 切换与回滚窗口）。支持四种场景：整机迁移、快速重建、灾难恢复、多机统一管理；不适用于无 Docker/GitHub 的环境或一次性的手工搬站。
---

# VPS 迁移自动化构建器

## 产出物

一次运行产出：

1. 一份**迁移计划报告**（Plan 阶段，按 [assets/migration-plan-template.md](assets/migration-plan-template.md)）；
2. 一份**执行记录**（Apply 阶段：每步命令、结果、健康检查输出、DNS 切换时间与回滚窗口）。

核心立场：**服务器是可随时重建的运行环境**——代码（GitHub）、部署配置（compose/脚本）、
持久化数据（数据库/上传/密钥）三者分开管理；服务器本身不做不可替代的资产。

## 第一步：选型访谈 → 迁移场景（必做，禁止静默默认）

按 [references/solution-tiers.md](references/solution-tiers.md) 一次问清三件事——
**目的**（换服务器 / 灾难恢复 / 日常可重建 / 多机管理）、**规模**（服务数、数据量、是否多机）、
**硬约束**（域名与 DNS API、密钥来源、预算、停机容忍度）——然后确定迁移场景并说明理由。
未回答的项按最保守假设处理，并在计划中显式标注"未确认"。

| 场景 | 含义 | 何时选 |
| --- | --- | --- |
| **A 整机迁移** | 旧机搬到新机，尽量不停机 | 换服务商、升级配置 |
| **B 快速重建** | 旧机不保留，新机从 GitHub 重建 + 数据导入 | 日常换机器 |
| **C 灾难恢复** | 旧机失联，从 GitHub + 备份恢复 | 故障后 |
| **D 多机统一管理** | 随时增删服务器，按清单部署任意项目 | 多项目/多机 |

**默认推荐 B+C 架构**：代码部署、数据备份、服务器恢复三件事分开，任何单机都可被替换——
这也是 A 和 D 的底层能力（A = B 的完整版；D = B 的批量化）。

## 第二步：强制两阶段（先扫描，再迁移）

**阶段 Plan（只读，绝不执行任何写操作）**

1. 扫描旧机：运行中容器、compose 项目、数据卷、端口、定时任务、磁盘与内存；
2. 识别持久化数据：数据库、上传目录、`.env`、证书、DNS 记录；
3. 检查 GitHub：仓库是否包含 Dockerfile/compose/部署脚本，缺什么列出来；
4. 生成迁移清单 + 风险清单（见 `references/migration-playbook.md`）；
5. 交给用户确认。**用户未确认前禁止进入 Apply。**

**阶段 Apply（用户确认后）**

1. 备份（数据库一致性备份 + 文件快照），备份**先验证可恢复**再继续；
2. 初始化新机（见 `references/infrastructure.md`，幂等脚本）；
3. 部署：拉取代码 → 安全注入环境变量 → 构建/拉镜像 → `compose up`；
4. 健康检查：HTTP 响应、数据库连接与数据量、关键 API、后台任务、日志无错误——
   **容器 Up ≠ 网站可用**；
5. 用户确认切换 DNS（预先调低 TTL）→ 切换 → 监测；
6. 保留旧机作回滚窗口（默认 72 小时），窗口过后再销毁。

## 第三步：按分层读取参考

- [references/solution-tiers.md](references/solution-tiers.md) —— 迁移场景矩阵与理由（第一步必读）
- [references/migration-playbook.md](references/migration-playbook.md) —— Plan/Apply 步骤、清单、健康检查与回滚
- [references/data-migration.md](references/data-migration.md) —— 数据层：数据库/卷/上传/密钥/证书/DNS 的正确迁移与验证
- [references/infrastructure.md](references/infrastructure.md) —— 新机初始化（幂等）与 `/srv` 布局
- [references/lessons-log.md](references/lessons-log.md) —— 经验账本（使用前扫最近条目）
- [references/maintenance.md](references/maintenance.md) —— 技能维护手册

## 不可妥协项

1. **先 Plan 后 Apply**：扫描与计划是只读的；未经用户确认不得执行任何写操作。
2. **不打活库目录**：禁止直接打包运行中的数据库容器目录；必须用数据库原生一致性备份并验证可恢复。
3. **密钥不进仓库**：`.env` 只传 `.env.example`；真实密钥经安全通道（服务器端 600 权限文件/密钥管理）注入。
4. **容器 Up ≠ 上线**：必须通过健康检查门槛（HTTP/DB/关键 API/后台任务）才算迁移完成。
5. **可回滚**：DNS 切换前降低 TTL；旧机保留回滚窗口；切换前有可恢复快照。
6. **证书优先重签**：HTTPS 证书在新机重新申请，不迁移私钥（特殊情况按 data-migration 处理）。
7. **每步留证**：命令、输出、时间戳写入执行记录；失败即停并报告，不静默跳过。
8. **初始化幂等**：新机初始化脚本可重复执行（重复跑无副作用）。

## 必须显式禁止的反面模式

- 把旧机文件系统整体打包复制到新机（数据不一致 + 配置漂移）；
- 用 `scp/rsync` 直接搬运行中数据库的 data 目录；
- 把 SSH 私钥、数据库密码提交进仓库或塞进镜像；
- compose up 成功就宣布迁移完成（没做健康检查）；
- 未降 TTL 就切 DNS，且没有回滚窗口；
- 跳过备份验证直接部署（备份打不开 = 没有备份）；
- Plan 阶段就顺手"修"旧机配置（只读原则）。

## 完成后

- 把本轮新经验按 [references/lessons-log.md](references/lessons-log.md) 格式追加（唯一进化入口）；
- 与服务器安全相关时，另用 `server-hardening` 技能处理新机加固，不在本技能内展开。

## 持续进化（每次使用都必须执行）

1. **使用前**：读 lessons-log 最近条目；
2. **使用中/后**：新经验按"日期｜来源｜场景 / 结论 / 为什么＋落地"追加，注明真实项目或推理；
3. **提升与归档**：同主题 ≥2 条或触及不可妥协项 → 提升进对应 reference，保留溯源。
