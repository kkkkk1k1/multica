# 在 K3（RISC-V64）上部署 Multica

本文面向运行 Linux `riscv64` 的 K3 主机。这里的 K3 指 RISC-V 主机；如果你说的是 K3s Kubernetes 集群，请使用 [Self-hosting guide](../SELF_HOSTING.md) 中的 Helm 部署章节。

## 先了解限制

Multica 当前发布的 `multica-backend` 和 `multica-web` 镜像只有 `linux/amd64`、`linux/arm64` 两种架构。K3 上直接执行下面的命令会失败：

```text
docker pull ghcr.io/multica-ai/multica-web:<tag>
no matching manifest for linux/riscv64
```

因此不要在 K3 上直接执行 `make selfhost`，也不要把 `linux/riscv64` 直接加入发布矩阵。QEMU 可以模拟 amd64 镜像，但 Next.js 服务端渲染会明显变慢，而且这不是原生 RISC-V 支持。

本指南使用下面的拓扑：

```text
支持架构的构建机（amd64/arm64）
  ├─ 构建 Go backend 的 riscv64 二进制
  └─ 构建 Next.js standalone 前端产物
                │ rsync / scp
                ▼
K3（riscv64）
  ├─ 原生 PostgreSQL 17+
  ├─ 原生 Go backend
  └─ 原生 Node.js 执行已构建的 Next.js JavaScript
                │
                ▼
局域网浏览器和 Multica daemon
```

K3 上运行的是已经生成的前端 JavaScript，不是在 K3 上执行 `next build`。当前 Next.js 构建会尝试加载 `linux/riscv64` 的 SWC 原生包，因此在 K3 上重新构建通常会报 `Failed to load SWC binary for linux/riscv64`。

## 1. 准备 K3

在 K3 上确认架构和运行时：

```bash
uname -m                         # 应为 riscv64
node --version                   # 建议 Node.js 22.x
go version                       # 用于运行或构建 CLI/daemon
psql --version                   # PostgreSQL 17 或更高版本
```

PostgreSQL 需要启用 `pgcrypto` 和 `pg_trgm`。Multica 不要求 `pgvector`；官方容器使用 `pgvector/pgvector` 镜像主要是历史命名。数据库要求和扩展说明见 [SELF_HOSTING_ADVANCED.md](../SELF_HOSTING_ADVANCED.md#postgresql)。

创建数据库和用户（按你的发行版调整服务管理命令）：

```bash
sudo -u postgres psql <<'SQL'
CREATE USER multica WITH PASSWORD 'replace-with-a-long-password';
CREATE DATABASE multica OWNER multica;
SQL

sudo -u postgres psql -d multica <<'SQL'
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SQL
```

不要把示例密码用于实际部署。生产环境应将 PostgreSQL 只绑定到本机或受限的内网地址。

## 2. 在支持架构的机器上构建产物

在 amd64 或 arm64 构建机上检出与部署版本一致的 release tag。构建机需要 Git、Go 1.26+、Node.js 22.x、Corepack 和 pnpm。

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
git fetch --tags --depth 1
git checkout <release-tag>

corepack enable
pnpm install --frozen-lockfile
```

构建 riscv64 backend。`CGO_ENABLED=0` 让产物尽量不依赖 K3 上的 C 运行库：

```bash
cd server
CGO_ENABLED=0 GOOS=linux GOARCH=riscv64 \
  go build -ldflags "-s -w" -o bin/server ./cmd/server
CGO_ENABLED=0 GOOS=linux GOARCH=riscv64 \
  go build -ldflags "-s -w" -o bin/multica ./cmd/multica
CGO_ENABLED=0 GOOS=linux GOARCH=riscv64 \
  go build -ldflags "-s -w" -o bin/migrate ./cmd/migrate
cd ..
```

构建 Next.js standalone 前端。这个步骤必须在 amd64 或 arm64 构建机上完成：

```bash
STANDALONE=true pnpm --filter @multica/web build
```

准备要复制到 K3 的目录。`standalone`、`.next/static` 和 `public` 三部分都必须保留：

```bash
mkdir -p dist/web/apps/web/.next
cp -a apps/web/.next/standalone/. dist/web/
cp -a apps/web/.next/static dist/web/apps/web/.next/
cp -a apps/web/public dist/web/apps/web/

mkdir -p dist/server
cp server/bin/{server,multica,migrate} dist/server/
cp -a server/migrations dist/server/
cp LICENSE NOTICE dist/server/
```

如果构建产物包含平台相关的原生 addon（例如图像处理模块），它可能仍然带有 amd64 二进制。先在 K3 上按第 5 节验证；遇到 `ELFCLASS`、`Exec format error` 或 native binding 错误时，说明该 addon 也需要单独准备 RISC-V 构建，不能只替换 Node.js。

## 3. 上传到 K3

```bash
rsync -a dist/server/ k3:/opt/multica/server/
rsync -a dist/web/ k3:/opt/multica/web/
ssh k3 'chmod +x /opt/multica/server/{server,multica,migrate}'
```

最终目录应至少包含：

```text
/opt/multica/server/server
/opt/multica/server/multica
/opt/multica/server/migrate
/opt/multica/server/migrations/...
/opt/multica/web/apps/web/server.js
/opt/multica/web/apps/web/.next/static/...
/opt/multica/web/apps/web/public/...
```

## 4. 迁移数据库并配置 backend

在 K3 上创建配置目录和运行用户，再创建 `/etc/multica/backend.env`：

```bash
sudo install -d -m 750 /etc/multica
sudo useradd --system --home /var/lib/multica --create-home multica || true
```

```dotenv
DATABASE_URL=postgres://multica:replace-with-a-long-password@127.0.0.1:5432/multica?sslmode=disable
JWT_SECRET=replace-with-output-of-openssl-rand-hex-32
PORT=8080
APP_ENV=production
FRONTEND_ORIGIN=http://<k3-lan-ip>:3000
MULTICA_APP_URL=http://<k3-lan-ip>:3000
MULTICA_PUBLIC_URL=http://<k3-lan-ip>:3000
ALLOW_SIGNUP=true
RESEND_FROM_EMAIL=noreply@multica.ai
LOCAL_UPLOAD_DIR=/var/lib/multica/uploads
# 没有邮件服务时可留空，验证码会写入 backend 日志
RESEND_API_KEY=
```

生成密钥并收紧权限：

```bash
openssl rand -hex 32
sudo chown root:multica /etc/multica/backend.env
sudo chmod 640 /etc/multica/backend.env
```

运行一次 migration：

```bash
cd /opt/multica/server
set -a
. /etc/multica/backend.env
set +a
sudo -u multica env DATABASE_URL="$DATABASE_URL" ./migrate up
```

migration 必须使用和 backend 完全相同的数据库。

创建 `/etc/systemd/system/multica-backend.service`：

```ini
[Unit]
Description=Multica backend
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=multica
Group=multica
WorkingDirectory=/opt/multica/server
EnvironmentFile=/etc/multica/backend.env
ExecStart=/opt/multica/server/server
Restart=on-failure
RestartSec=3
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

创建上传目录后启动：

```bash
sudo mkdir -p /var/lib/multica/uploads
sudo chown -R multica:multica /var/lib/multica /opt/multica
sudo systemctl daemon-reload
sudo systemctl enable --now multica-backend
curl -fsS http://127.0.0.1:8080/readyz
```

backend 如果使用本地附件存储，还需要将上传目录映射到应用实际配置的路径。若你需要长期保存附件，建议配置 S3；否则确认当前版本的上传目录配置后再将 `/var/lib/multica/uploads` 挂载到对应路径。

## 5. 启动原生 Node 前端

前端使用 standalone server，不要在 K3 上运行 `next build` 或 `next start`。创建 `/etc/multica/frontend.env`：

```dotenv
NODE_ENV=production
HOSTNAME=127.0.0.1
PORT=3001
REMOTE_API_URL=http://127.0.0.1:8080
NEXT_TELEMETRY_DISABLED=1
```

创建 `/etc/systemd/system/multica-frontend.service`：

```ini
[Unit]
Description=Multica frontend
After=network-online.target multica-backend.service
Wants=network-online.target

[Service]
Type=simple
User=multica
Group=multica
WorkingDirectory=/opt/multica/web/apps/web
EnvironmentFile=/etc/multica/frontend.env
ExecStart=/usr/bin/node /opt/multica/web/apps/web/server.js
Restart=on-failure
RestartSec=3
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

启动并确认前端只监听本机回环地址：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now multica-frontend
curl -fsS http://127.0.0.1:3001/ >/dev/null
```

## 6. 用 Caddy 提供局域网访问

建议让 backend 和 Node 前端只监听 `127.0.0.1`，由 Caddy 统一监听 K3 的局域网端口。这样 `/health`、浏览器 WebSocket 和 daemon WebSocket 都能明确转发到 backend。

安装 Caddy 后创建 `/etc/caddy/Caddyfile`：

```caddyfile
:3000 {
    # CLI 可达性探测
    handle /health {
        reverse_proxy 127.0.0.1:8080
    }

    handle /readyz {
        reverse_proxy 127.0.0.1:8080
    }

    # 浏览器实时连接
    handle /ws* {
        reverse_proxy 127.0.0.1:8080
    }

    # daemon 长连接
    handle /api/daemon/ws* {
        reverse_proxy 127.0.0.1:8080
    }

    # 其余页面和 API 代理由前端处理
    handle {
        reverse_proxy 127.0.0.1:3001
    }
}
```

```bash
sudo systemctl enable --now caddy
curl -i -H 'Host: <k3-lan-ip>:3000' http://127.0.0.1:3000/health
```

在浏览器和运行 daemon 的电脑上访问 `http://<k3-lan-ip>:3000`。只在可信局域网开放该端口；公网部署必须使用 HTTPS，并同步修改 `FRONTEND_ORIGIN`、`MULTICA_APP_URL` 和 `MULTICA_PUBLIC_URL`。

## 7. 绑定本机运行时

在运行 AI 编程工具的电脑上安装 Multica CLI，然后把服务地址指向 Caddy 的同源地址：

```bash
multica setup self-host \
  --server-url http://<k3-lan-ip>:3000 \
  --app-url http://<k3-lan-ip>:3000
multica daemon status --output json
```

如果 daemon 就运行在 K3 上，需要准备 RISC-V 的 `multica` CLI：

```bash
cd /path/to/multica/server
CGO_ENABLED=0 GOOS=linux GOARCH=riscv64 \
  go build -ldflags "-s -w" -o bin/multica ./cmd/multica
./bin/multica setup self-host \
  --server-url http://<k3-lan-ip>:3000 \
  --app-url http://<k3-lan-ip>:3000
```

daemon 运行用户必须能直接执行你配置的 AI 编程工具，并且工作目录可写。前端、backend 和 daemon 三者的地址不要混用 `localhost`：浏览器或另一台执行电脑上的 `localhost` 指的是它自己。

## 8. 验证和常见故障

```bash
systemctl --no-pager --full status multica-backend multica-frontend caddy
curl -fsS http://<k3-lan-ip>:3000/health
curl -fsS http://<k3-lan-ip>:3000/readyz
ss -ltnp | grep -E ':3000|:3001|:8080|:5432'
```

| 现象 | 原因和处理 |
|---|---|
| `no matching manifest for linux/riscv64` | 官方镜像没有 RISC-V 变体；按本文从支持架构构建机复制产物。 |
| `Failed to load SWC binary for linux/riscv64` | 在 K3 上执行了 Next.js 构建；把 `next build` 移到 amd64/arm64 构建机。 |
| 页面 200，但实时状态不更新 | 检查 Caddy 的 `/ws*` 和 `/api/daemon/ws*` 是否直通 backend。 |
| `multica setup` 报服务器不可达 | 从运行 daemon 的电脑执行 `curl <server-url>/health`；不要只在 K3 上测试。 |
| `exec format error` 或 native binding 错误 | standalone 产物中包含其他架构的原生 addon，需要为 RISC-V 准备该依赖或暂时关闭对应功能。 |
| 登录验证码收不到 | 未配置邮件时查看 `journalctl -u multica-backend -f`；生产环境建议配置 Resend 或 SMTP。 |

备份至少包括 PostgreSQL 数据库和附件目录。升级时先在支持架构的构建机生成同一版本的 backend/web 产物，停止对应 systemd 服务，替换文件，运行 migration，再按顺序启动 backend 和 frontend。
