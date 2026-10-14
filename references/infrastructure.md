# 新机初始化（幂等）

> 目标：脚本可重复执行，重复跑无副作用（idempotent）。示例为 Debian/Ubuntu。

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. 系统更新与基础包
apt-get update && apt-get -y upgrade
apt-get -y install ca-certificates curl gnupg rsync ufw jq

# 2. SSH 加固（允许公钥、禁 root 密码；先确认已有 sudo 用户再执行）
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh

# 3. 防火墙（默认拒绝入站，放行 22/80/443）
ufw default deny incoming && ufw default allow outgoing
ufw allow 22/tcp && ufw allow 80/tcp && ufw allow 443/tcp && ufw --force enable

# 4. Docker Engine + Compose 插件（官方源，幂等）
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list
apt-get update && apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. 部署目录与 deploy key
install -d -m 750 /srv
install -d -m 700 /srv/secrets
# 部署用只读 deploy key（只允许拉取，不授予写权限），密钥放 /root/.ssh/ 或专用账户

# 6. 监控兜底：磁盘/内存告警（最小实现用 cron + df/free 阈值）
cat > /etc/cron.d/disk-check <<'EOF'
*/10 * * * * root df -P / | awk 'NR==2 && $5+0>90 {print "disk>90%"}' | logger -t disk-check
EOF
```

目录规划（示例，按项目调整）：

```text
/srv/
├── app-a/            # 每个 compose 项目一个目录
├── app-b/
├── infrastructure/   # proxy / backup / monitoring
└── secrets/          # 700 权限，绝不在 git 里
```

验收：脚本重跑一遍无变化（幂等）；`docker version`、`docker compose version` 正常；22/80/443 放行、其余拒绝。
进一步加固（SSH 爆破、0.0.0.0 误绑、TLS/安全头）用 `server-hardening` 技能，不在本技能内重复。
