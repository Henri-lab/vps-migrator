# vps-migrator 🚚 VPS 迁移自动化构建器

> A **Codex skill** that turns "server migration / rebuild / disaster recovery" into a
> reviewed two-phase process — **Plan** (read-only scan) then **Apply** (backup → deploy →
> verify → cutover) — for GitHub + Docker based deployments. Docs are in Chinese.

<p align="center">
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-blue.svg"></a>
  <img alt="Version" src="https://img.shields.io/badge/version-v0.1.0-informational">
  <img alt="Validate" src="https://github.com/Henri-lab/vps-migrator/actions/workflows/validate.yml/badge.svg">
</p>

核心立场：**服务器是可随时重建的运行环境**。代码（GitHub）、部署配置（compose/脚本）、
持久化数据（数据库/上传/密钥）三者分开管理；迁移 = 新机重建 + 数据导入，而不是复制文件系统。

## 快速开始

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/Henri-lab/vps-migrator.git ~/.codex/skills/vps-migrator
```

之后在任意 Codex 会话中按 skill 名（`vps-migrator`）触发即可。

## 目录

```text
SKILL.md                   主入口：场景访谈 → Plan（只读）→ 用户确认 → Apply
agents/openai.yaml         agent 元数据
assets/                    迁移计划报告模板
references/
├── solution-tiers.md      迁移场景矩阵 A/B/C/D（推荐 B+C 架构）
├── migration-playbook.md  Plan/Apply 步骤、清单、健康检查、DNS 切换与回滚
├── data-migration.md      数据层：数据库/卷/上传/密钥/证书/DNS 的正确迁移与验证
├── infrastructure.md      新机初始化（幂等脚本）与 /srv 布局
├── lessons-log.md         经验账本（L-001 起）
└── maintenance.md         技能持续进化纪律
```

## 三个不寻常之处

1. **先扫描，再迁移**：Plan 阶段只读，未经用户确认绝不执行写操作；
2. **容器 Up ≠ 上线**：健康检查（HTTP/DB/关键 API/后台任务）全过才算迁移完成；
3. **默认 B+C 架构**：快速重建 + 灾难恢复分开设计，任何单机都可被替换。

## 起源与署名

设计来源：GitHub + Docker 四层迁移架构（应用/基础设施/数据/部署）与两阶段 Agent 流程的整理沉淀，
2026-10-15 独立为公开仓库。作者清单见 [AUTHORS.md](AUTHORS.md)；二次分发须保留 [LICENSE](LICENSE) 版权声明。

```text
vps-migrator（v0.1.0）
作者：henrifox
仓库：https://github.com/Henri-lab/vps-migrator
首发：2026-10-15｜许可：MIT（保留署名）
```

## License

[MIT](LICENSE) © 2026 henrifox
