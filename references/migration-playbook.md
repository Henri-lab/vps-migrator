# 迁移执行手册（Plan / Apply）

> 两阶段不可跳过：Plan 只读、Apply 需确认。所有命令示例默认 Debian/Ubuntu + Docker Compose v2。

## 1. Plan（只读扫描）

```bash
# 容器与 compose 项目
ssh old docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
ssh old docker compose ls -a
ssh old 'for c in $(docker ps -aq); do docker inspect -f "{{.Name}} | {{index .Config.Labels \"com.docker.compose.project.config_files\"}}" $c; done'
# 数据卷与挂载
ssh old 'docker volume ls && docker ps -aq | xargs -r -I{} docker inspect -f "{{.Name}}: {{range .Mounts}}{{.Type}}={{.Source}}{{end}}" {}'
# 端口、定时任务、磁盘
ssh old 'ss -lntp; crontab -l; df -h; free -h'
```

产出四份清单：

1. **服务清单**：compose 项目 → 容器 → 镜像 → 端口 → 依赖顺序；
2. **数据清单**：数据库（类型/库名/体积）、上传目录、其它卷；
3. **配置清单**：`.env`（只列 key 名，不列值）、Caddy/Nginx 配置、DNS 记录、定时任务；
4. **缺失清单**：GitHub 仓库里没有的 Dockerfile/compose/部署脚本/健康检查——**这决定能不能重建**。

风险清单（任一命中必须先解决再 Apply）：

- 仓库缺 compose 或 Dockerfile（重建无源）；
- 数据库在容器内但无备份策略；
- `.env` 只存在于旧机、无备份渠道；
- 证书与域名绑定、DNS 无 API 或 TTL 很高；
- 定时任务只写在旧机 crontab、未进仓库。

## 2. Apply（用户确认后）

1. **备份并验证**：数据库一致性备份（见 data-migration.md）→ 在临时实例恢复验证 → 文件快照；
2. **初始化新机**：`infrastructure.md` 的幂等脚本；记录每步输出；
3. **部署**：拉代码 → 注入环境变量（服务器端文件，600 权限）→ 构建/拉镜像 → `compose up -d`；
4. **健康检查门槛**（全部通过才算迁移完成）：
   - HTTP：关键域名返回 2xx/3xx（`curl -sS -o /dev/null -w '%{http_code}'`）；
   - 数据库：能连接、关键表行数与旧机一致（抽样 + `COUNT(*)`）；
   - 关键 API：登录/下单/回调各打一发；
   - 后台任务：worker/cron 在跑且无错误日志（`docker compose logs --since 10m` 无 ERROR）；
5. **DNS 切换**：提前把 TTL 降到 300 → 用户确认 → 改记录 → 验证解析逐步收敛到新 IP；
6. **回滚窗口**：旧机保留 72 小时可回切；窗口过后再销毁，销毁前再备份一次。

## 3. 执行记录格式（每步必留）

```text
时间（UTC） | 步骤 | 命令摘要 | 结果 | 证据（输出文件/截图）
```

失败即停：任何一步非预期结果 → 停止并报告，禁止"继续看后面能不能跑通"。
