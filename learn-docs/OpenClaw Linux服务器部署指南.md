# OpenClaw Linux服务器部署指南

本文档提供OpenClaw项目在Linux服务器（Ubuntu/Debian）上的完整部署流程，采用Docker容器化部署。

> **注意**: 本文档基于实际部署经验编写，将常见问题融入各步骤中，帮助您一次性成功部署。

## 目录

- [前置要求](#前置要求)
- [Docker部署](#docker部署)
  - [方案A：使用预构建镜像（推荐，无沙箱）](#方案a使用预构建镜像推荐)
  - [方案B：本地构建镜像（支持沙箱）](#方案b本地构建镜像支持沙箱)
- [HTTPS安全配置](#https安全配置)
- [设备配对](#设备配对)
- [部署后配置](#部署后配置)
  - [配置AI模型提供商](#配置ai模型提供商)
  - [配置消息通道](#配置消息通道)
  - [接入微信（企业微信机器人）](#接入微信企业微信机器人)
- [运维管理](#运维管理)
- [常见问题快速索引](#常见问题快速索引)

---

## 前置要求

### 系统要求

- **操作系统**: Ubuntu 20.04 LTS / Debian 12 或更高版本
- **内存**: 运行时 1GB+，本地构建需要 4GB+（或配置 Swap）
- **磁盘空间**: 预构建镜像约 500MB，本地构建需要 20GB+
- **网络**: 能够访问外网（用于下载依赖和模型API）

### 步骤1：安装基础工具

```bash
# 更新系统包
sudo apt update && sudo apt upgrade -y

# 安装基础工具（含 C/C++、Python 和 Node.js 环境）
sudo apt install -y curl git build-essential python3 python3-pip python3-venv wget vim procps

# 安装最新的 Node.js (v20.x 或 v22.x)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### 步骤2：安装Docker

```bash
# 安装Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 将当前用户添加到docker组（可选，避免每次使用sudo）
sudo usermod -aG docker $USER
newgrp docker

# 验证安装
docker --version
docker compose version
```

### 步骤3：配置Swap空间（本地构建必须）

> ⚠️ **如果选择本地构建且服务器内存 < 4GB，必须执行此步骤！** 否则构建会因为OOM失败（exit code 137）。

```bash
# 检查当前内存和Swap
free -h

# 创建4GB Swap文件
sudo fallocate -l 4G /swapfile
# 如果fallocate失败，使用dd命令
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096

# 设置权限并启用
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 开机自动挂载
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 验证
free -h  # 应该看到 Swap: 4.0Gi
```

### 步骤4：配置Docker镜像加速器（国内服务器必须）

> ⚠️ **国内服务器必须配置！** 否则无法拉取Docker Hub镜像。

```bash
# 创建Docker配置目录
sudo mkdir -p /etc/docker

# 配置镜像加速器
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ]
}
EOF

# 重启Docker服务
sudo systemctl restart docker

# 验证配置
docker info | grep -A 3 "Registry Mirrors"
```

---

## Docker部署

### 方案A：使用预构建镜像（推荐）

> ⏱️ **部署时间**: 约 5-10 分钟（无需构建，直接拉取镜像）
>
> ⚠️ **限制**: 预构建镜像（`slim` 版本）不包含 Docker CLI，**不支持沙箱隔离功能**。如需沙箱功能，请使用方案B。

#### 1. 拉取预构建镜像

```bash
# 查看可用版本：https://github.com/openclaw/openclaw/pkgs/container/openclaw
# 推荐：使用 slim 版本（体积更小，约 500MB）

# 方式1：拉取最新版本
docker pull ghcr.io/openclaw/openclaw:latest-slim

# 方式2：拉取特定版本（推荐生产环境）
docker pull ghcr.io/openclaw/openclaw:2026.4.1-slim
```

> 💡 **提示**: 如果拉取超时，请确认已配置 Docker 镜像加速器（见前置要求步骤4）。

#### 2. 创建部署目录和配置文件

```bash
# 创建部署目录
mkdir -p ~/openclaw && cd ~/openclaw

# 创建 docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  openclaw-gateway:
    image: ghcr.io/openclaw/openclaw:latest-slim
    container_name: openclaw-gateway
    restart: unless-stopped
    user: "root"  # 推荐：以 root 用户运行以获得完整权限
    ports:
      - "18789:18789"
      - "18790:18790"
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - OPENCLAW_CONFIG_DIR=/home/node/.openclaw
      - OPENCLAW_WORKSPACE_DIR=/home/node/.openclaw/workspace
      # AI 模型 API Keys（可选，也可在 Control UI 中配置）
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY:-}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY:-}
      - OPENAI_API_KEY=${OPENAI_API_KEY:-}
    volumes:
      - openclaw-data:/home/node/.openclaw
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:18789/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  openclaw-data:
EOF
```

#### 3. 配置环境变量

```bash
# 创建 .env 文件
cat > .env << EOF
# Gateway 访问令牌（必须修改！）
OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

# AI 模型 API Keys（可选，根据需要填写）
# DEEPSEEK_API_KEY=sk-...
# ANTHROPIC_API_KEY=sk-ant-...
# OPENAI_API_KEY=sk-...
EOF

# 保存 Token（重要！登录时需要）
echo "=========================================="
echo "Gateway Token: $(grep OPENCLAW_GATEWAY_TOKEN .env | cut -d'=' -f2)"
echo "=========================================="
echo "请妥善保存此 Token！"
```

#### 4. 启动服务

```bash
# 启动容器
docker compose up -d

# 等待服务就绪
sleep 10

# 检查状态
docker compose ps
docker compose logs --tail 20 openclaw-gateway

# 测试健康检查
curl http://127.0.0.1:18789/healthz
curl http://127.0.0.1:18789/readyz
```

> ⚠️ **如果启动失败**：检查端口是否被占用 `sudo lsof -i :18789`

#### 5. 访问 Web 界面（HTTP）

```
http://<服务器IP>:18789
```

首次访问需要输入 Gateway Token（从 `.env` 文件获取）。

> ⚠️ **远程访问注意**: 如果通过远程 IP 访问，会遇到 `control ui requires device identity` 错误。这是因为浏览器要求 HTTPS 才能使用设备身份 API。请继续完成 [HTTPS安全配置](#https安全配置)。

---

### 方案B：本地构建镜像（支持沙箱）

> ⏱️ **构建时间**: 首次构建需要下载大量依赖，耗时可能较长
>
> 💡 **优势**: 支持沙箱隔离功能（Agent 在独立 Docker 容器中执行命令）

#### 0. 选择部署模式

在开始之前，请确定是否需要沙箱功能：

| 模式 | 镜像 | 沙箱功能 | 适用场景 |
|------|------|----------|----------|
| 基础模式 | `openclaw:local` | ❌ 不支持 | 简单部署，无需隔离 |
| 沙箱模式 | `openclaw:sandbox` | ✅ 支持 | 生产环境，需要安全隔离 |

> ⚠️ **推荐**: 生产环境建议使用沙箱模式，避免 Agent 执行危险命令影响宿主机。

#### 1. 克隆项目代码

```bash
# 克隆OpenClaw仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# （可选）切换到特定版本
# git checkout v2026.4.1
```

#### 2. 配置环境变量

```bash
# 创建配置目录并设置权限（重要！提前设置避免权限问题）
mkdir -p ~/.openclaw/workspace
sudo chown -R 1000:1000 ~/.openclaw/
sudo chmod -R 755 ~/.openclaw/workspace

# 创建 .env 文件
GATEWAY_TOKEN=$(openssl rand -hex 32)
cat > .env << EOF
# Gateway 访问令牌
OPENCLAW_GATEWAY_TOKEN=$GATEWAY_TOKEN

# 配置路径（直接设置具体值，不要使用变量引用）
OPENCLAW_CONFIG_DIR=/root/.openclaw
OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790

# Docker 组 ID（沙箱模式需要，查看方法：getent group docker | cut -d: -f3）
DOCKER_GID=$(getent group docker | cut -d: -f3)
EOF

# 保存 Token（重要！登录时需要）
echo "=========================================="
echo "Gateway Token: $GATEWAY_TOKEN"
echo "=========================================="
echo "请妥善保存此 Token！"
```

> ⚠️ **重要**: `.env` 文件中不要使用变量引用（如 `${VAR}`），直接设置具体值。否则会出现 `invalid spec: :/home/node/.openclaw: empty section between colons` 错误。

#### 3. 构建基础镜像

```bash
# 拉取基础镜像（加速构建）
docker pull docker/dockerfile:1.7
docker pull node:24-bookworm

# 后台构建镜像（推荐，避免SSH断开中断）
nohup docker build -t openclaw:local -f Dockerfile . > /tmp/docker-build.log 2>&1 &

# 监控构建进度
tail -f /tmp/docker-build.log

# 按 Ctrl+C 退出监控，构建会继续在后台运行

# 检查构建状态
ps aux | grep 'docker build' | grep -v grep  # 有输出表示还在构建
docker images openclaw:local                  # 有输出表示构建完成
```

> ⏱️ **构建时间参考**: 在2核4G服务器上约30-60分钟。主要耗时：
> - 下载Bun运行时（~5分钟）
> - 安装pnpm依赖（~15分钟）
> - 下载GitHub资源如matrix-sdk-crypto（~10分钟）
> - TypeScript编译（~5分钟）

> ⚠️ **构建失败（Exit Code 137）**: 这是内存不足导致的OOM。解决方案：
> 1. 确保已配置 Swap（见前置要求步骤3）
> 2. 或使用预构建镜像（方案A）

#### 4. 构建沙箱镜像（可选，沙箱模式需要）

> ⚠️ **如果选择基础模式，跳过此步骤，直接进入步骤5**

```bash
# 创建沙箱 Dockerfile
cat > Dockerfile.sandbox << 'EOF'
FROM node:24-bookworm-slim

# 安装 Docker CLI（用于沙箱隔离）
RUN apt-get update && \
    apt-get install -y docker.io && \
    rm -rf /var/lib/apt/lists/*

# 从本地构建的镜像复制应用
COPY --from=openclaw:local /app /app

WORKDIR /app
ENV NODE_ENV=production
EOF

# 后台构建沙箱镜像
nohup docker build -f Dockerfile.sandbox -t openclaw:sandbox . > /tmp/docker-build-sandbox.log 2>&1 &

# 监控构建进度
tail -f /tmp/docker-build-sandbox.log

# 检查构建状态
ps aux | grep 'docker build' | grep -v grep
docker images | grep openclaw
```

> ⏱️ **构建时间**: 约5-10分钟，主要是安装 docker.io 包（~40MB）

#### 5. 创建 docker-compose.yml

**基础模式**（无沙箱）：

```bash
cat > docker-compose.yml << 'EOF'
services:
  openclaw-gateway:
    image: openclaw:local
    container_name: openclaw-gateway
    restart: unless-stopped
    ports:
      - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"
      - "${OPENCLAW_BRIDGE_PORT:-18790}:18790"
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - HOME=/home/node
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:18789/healthz').then((r)=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 20s
EOF
```

**沙箱模式**（推荐生产环境）：

```bash
cat > docker-compose.yml << 'EOF'
services:
  openclaw-gateway:
    image: openclaw:sandbox
    container_name: openclaw-gateway
    restart: unless-stopped
    user: "root"  # 推荐：以 root 用户运行以获得完整权限
    ports:
      - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"
      - "${OPENCLAW_BRIDGE_PORT:-18790}:18790"
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - HOME=/home/node
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
      # 沙箱隔离需要挂载 Docker Socket
      - /var/run/docker.sock:/var/run/docker.sock
    group_add:
      - "${DOCKER_GID:-999}"
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:18789/healthz').then((r)=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 20s
EOF
```

> ⚠️ **安全提示**: 沙箱模式挂载 Docker Socket 会赋予容器完全的 Docker 访问权限，请确保在受信任的环境中使用。

#### 6. 启动服务

```bash
# 启动容器
docker compose up -d

# 等待服务就绪
sleep 10

# 检查状态
docker compose ps
docker compose logs --tail 30 openclaw-gateway

# 测试健康检查
curl http://127.0.0.1:18789/healthz
```

预期输出：
```json
{"ok":true,"status":"live"}
```

#### 7. 验证沙箱功能（沙箱模式）

```bash
# 进入容器
docker exec -it openclaw-gateway bash

# 验证 Docker CLI 可用
docker --version
# 输出: Docker version 24.x.x, build xxx

# 退出容器
exit
```

---

## HTTPS安全配置

> ⚠️ **重要**: 浏览器要求 HTTPS 才能提供"安全上下文"（Secure Context），否则设备身份 API 无法使用，会导致 `control ui requires device identity` 错误。

### 为什么需要 HTTPS？

OpenClaw Control UI 需要使用浏览器的设备身份 API，这些 API 仅在安全上下文中可用：

- ✅ HTTPS 连接
- ✅ localhost / 127.0.0.1
- ❌ HTTP + 远程 IP（不安全上下文）

### 方案一：使用 Caddy 反向代理（推荐）

#### 1. 安装 Caddy

```bash
# Debian/Ubuntu
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

#### 2. 生成自签名证书（IP 地址场景）

由于 IP 地址无法获取 Let's Encrypt 证书，需要生成自签名证书：

```bash
# 创建证书目录
sudo mkdir -p /etc/caddy/ssl

# 生成自签名证书（替换为你的服务器IP）
SERVER_IP="175.178.39.180"
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/caddy/ssl/server.key \
  -out /etc/caddy/ssl/server.crt \
  -subj "/CN=$SERVER_IP" \
  -addext "subjectAltName=IP:$SERVER_IP"

# 设置权限
sudo chown caddy:caddy /etc/caddy/ssl/*
```

#### 3. 配置 Caddyfile

```bash
sudo tee /etc/caddy/Caddyfile << 'EOF'
{
  admin off
}

# HTTP 自动重定向到 HTTPS
http:// {
  redir https://{host}{uri}
}

# HTTPS 服务
:443 {
  tls /etc/caddy/ssl/server.crt /etc/caddy/ssl/server.key
  
  reverse_proxy localhost:18789 {
    header_up Host {host}
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-Proto https
  }
}
EOF
```

#### 4. 启动 Caddy

```bash
# 重启 Caddy
sudo systemctl restart caddy
sudo systemctl enable caddy

# 检查状态
sudo systemctl status caddy
```

#### 5. 配置 Gateway 信任代理

```bash
# 获取 Docker 网络地址（通常是 172.18.0.0/16）
docker network inspect $(docker network ls -q | head -1) --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'

# 配置 Gateway 信任代理（替换为实际网络地址）
docker compose exec openclaw-gateway node dist/index.js config set gateway.trustedProxies '["172.18.0.0/16","127.0.0.1"]'

# 配置允许的来源（替换 YOUR_SERVER_IP）
docker compose exec openclaw-gateway node dist/index.js config set gateway.controlUi.allowedOrigins '["https://YOUR_SERVER_IP","http://YOUR_SERVER_IP:18789","http://localhost:18789"]'

# 重启 Gateway
docker compose restart openclaw-gateway
```

> ⚠️ **错误 "Proxy headers detected from untrusted address"**: 说明 `trustedProxies` 配置不正确，请确认 Docker 网络地址。

> ⚠️ **错误 "origin not allowed"**: 说明 `allowedOrigins` 未包含当前访问地址，请添加 HTTPS URL。

#### 6. 开放防火墙端口

```bash
# 云服务器需要在安全组开放 443 端口
# 腾讯云轻量应用服务器：控制台 -> 防火墙 -> 添加规则
# 协议: TCP, 端口: 443, 策略: 允许
```

#### 7. 访问 HTTPS

```
https://YOUR_SERVER_IP/#token=YOUR_GATEWAY_TOKEN
```

> ⚠️ **首次访问**: 由于使用自签名证书，浏览器会显示安全警告。点击"高级" → "继续访问"即可。

### 方案二：使用域名 + Let's Encrypt（生产推荐）

如果有域名，可以使用 Let's Encrypt 获取免费的可信证书：

```bash
# 配置 Caddyfile（使用域名）
sudo tee /etc/caddy/Caddyfile << 'EOF'
your-domain.com {
  reverse_proxy localhost:18789 {
    header_up Host {host}
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-Proto https
  }
}
EOF

# Caddy 会自动获取 Let's Encrypt 证书
sudo systemctl restart caddy
```

### 方案三：SSH 隧道（临时测试）

如果只是临时测试，可以使用 SSH 隧道映射到本地：

```bash
# 在本地电脑执行
ssh -L 18789:localhost:18789 user@server-ip

# 然后访问
# http://localhost:18789
```

---

## 设备配对

> ⚠️ **重要**: OpenClaw 的安全机制要求新设备首次访问时必须配对，即使拥有 Gateway Token。

### 什么是设备配对？

设备配对是 OpenClaw 的安全机制：
- 首次从新设备/浏览器访问时，需要管理员批准
- 防止 Token 泄露后被人滥用
- 每个设备有唯一的 Device ID

### 配对流程

#### 1. 用户访问

用户访问 Control UI 时，如果设备未配对，会收到 `pairing required` 错误。

#### 2. 管理员批准配对

```bash
# 列出待配对设备
docker compose exec openclaw-gateway node dist/index.js devices list --json

# 输出示例：
# {
#   "pending": [{
#     "requestId": "a08aef8c-xxx",
#     "deviceId": "6ad158e8xxx",
#     "platform": "Win32",
#     "remoteIp": "119.147.23.208",
#     "clientId": "openclaw-control-ui"
#   }],
#   "paired": [...]
# }

# 批准特定请求
docker compose exec openclaw-gateway node dist/index.js devices approve <requestId>

# 示例
docker compose exec openclaw-gateway node dist/index.js devices approve a08aef8c-027c-43f0-90cc-7e6a6b9e9f6d
```

#### 3. 用户刷新页面

批准后，用户刷新浏览器页面即可正常访问。

### 管理已配对设备

```bash
# 列出所有已配对设备
docker compose exec openclaw-gateway node dist/index.js devices list --json | jq '.paired'

# 移除已配对设备（如需撤销访问权限）
docker compose exec openclaw-gateway node dist/index.js devices remove <deviceId>
```

---

## 部署后配置

### 配置AI模型提供商

OpenClaw 支持多种 AI 模型提供商，推荐使用环境变量方式配置。

#### 方式一：环境变量配置（推荐）

**1. 编辑 `.env` 文件：**

```bash
cd ~/openclaw  # 或你的部署目录

# 添加 API Keys
cat >> .env << 'EOF'

# -----------------------------------------------------------------------------
# AI 模型提供商 API Keys
# -----------------------------------------------------------------------------
# DeepSeek（推荐，性价比高）
DEEPSEEK_API_KEY=sk-...

# Anthropic (Claude)
ANTHROPIC_API_KEY=sk-ant-...

# OpenAI (GPT)
OPENAI_API_KEY=sk-...

# Google Gemini
GEMINI_API_KEY=...

# OpenRouter（多模型聚合）
OPENROUTER_API_KEY=sk-or-...
EOF
```

**2. 更新 `docker-compose.yml` 添加环境变量：**

```yaml
services:
  openclaw-gateway:
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY:-}    # 添加此行
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY:-}  # 可选
      - OPENAI_API_KEY=${OPENAI_API_KEY:-}        # 可选
```

**3. 重启服务：**

```bash
docker compose down && docker compose up -d
```

#### 方式二：在 Control UI 中配置

1. 打开 Control UI：`https://YOUR_SERVER_IP`
2. 点击左下角 ⚙️ **Settings**
3. 找到 **Providers** 部分
4. 点击对应提供商（如 DeepSeek）
5. 输入 API Key 并保存

#### 支持的模型提供商

| 提供商     | 环境变量             | 模型示例                             |
| ---------- | -------------------- | ------------------------------------ |
| DeepSeek   | `DEEPSEEK_API_KEY`   | `deepseek-chat`, `deepseek-reasoner` |
| Anthropic  | `ANTHROPIC_API_KEY`  | `claude-sonnet-4-20250514`           |
| OpenAI     | `OPENAI_API_KEY`     | `gpt-4.1`, `gpt-5.4`                 |
| Google     | `GEMINI_API_KEY`     | `gemini-2.5-pro`                     |
| OpenRouter | `OPENROUTER_API_KEY` | 聚合多提供商                         |

#### 设置默认模型

```bash
# 命令行设置默认模型
docker compose exec openclaw-gateway node dist/index.js config set agents.defaultModel "deepseek:deepseek-chat"

# 或在 Control UI 中：Settings → Agents → Default Model
```

#### 选择模型发送消息

在 Control UI 聊天界面：
1. 点击输入框上方的模型选择器
2. 从下拉列表选择模型（如 `deepseek:deepseek-chat`）
3. 输入消息并发送

> ⚠️ **权限错误**: 如果发送消息时报错 `EACCES: permission denied`，请执行：
> ```bash
> sudo chown -R 1000:1000 ~/.openclaw/
> docker compose restart openclaw-gateway
> ```

### 配置消息通道

OpenClaw支持多种消息平台：

```bash
# WhatsApp
docker compose run --rm openclaw-cli channels login

# Telegram
docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

# Discord
docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
```

### 接入微信（企业微信机器人）

> 💡 **推荐**: 微信通道使用企业微信机器人（iLinkAI）实现，支持通过微信扫码登录后与 AI 对话。

#### 1. 安装微信插件

**方式一：通过 CLI 安装（推荐）**

```bash
# 在容器内安装微信插件
docker exec openclaw-gateway node openclaw.mjs plugins install "@tencent-weixin/openclaw-weixin@latest"
```

安装过程可能需要 1-2 分钟，请等待完成。

> 💡 **提示**: OpenClaw 支持**多账户**接入。您可以多次执行 `channels login --channel openclaw-weixin` 来扫描不同的二维码并添加新账户。

**方式二：手动安装（解决 API 限流问题）**

---

### 启用浏览器工具 (Playwright)

OpenClaw 支持基于 `Playwright` 的浏览器工具（如搜索网页、网页截图、UI 分析等）。

#### 1. 确保容器以 `root` 用户运行

在 `docker-compose.yml` 中配置 `user: "root"`，并重启容器：

```yaml
services:
  openclaw-gateway:
    user: "root"
    # ... 其他配置
```

#### 2. 安装 Chromium 浏览器及其依赖

在服务器上执行以下命令：

```bash
# 自动安装 Chromium 引擎及其 Debian 系统依赖（约 300MB）
docker exec openclaw-gateway npx playwright install chromium --with-deps
```

> ⚠️ **超时处理**: 如果安装过程中出现 `TIMEOUT` 错误，说明安装耗时超过了某些系统的限制。请再次执行上述命令以继续安装。

#### 3. 验证浏览器功能

```bash
# 测试浏览器搜索命令（需要配置 AI 代理）
docker exec openclaw-gateway node openclaw.mjs browser search "OpenClaw 架构"
```

---

> ⚠️ **ClawHub API 限流**: 如果遇到 `Rate limit exceeded` 错误，使用手动安装方式：

```bash
# 1. 创建插件目录
docker exec openclaw-gateway mkdir -p /home/node/.openclaw/extensions

# 2. 下载并解压插件包
docker exec openclaw-gateway sh -c 'cd /home/node/.openclaw/extensions && \
  npm pack @tencent-weixin/openclaw-weixin@2.1.3 && \
  tar -xzf *.tgz && rm *.tgz && \
  mv package openclaw-weixin'

# 3. 安装插件依赖
docker exec openclaw-gateway sh -c 'cd /home/node/.openclaw/extensions/openclaw-weixin && \
  npm install --omit=dev'

# 4. 修复权限（重要！）
docker exec openclaw-gateway chown -R root:root /home/node/.openclaw/extensions/openclaw-weixin

# 5. 重启 Gateway 加载插件
docker restart openclaw-gateway
```

> ⚠️ **插件依赖问题**: 微信插件依赖 `zod` 和 `qrcode-terminal`，必须执行 `npm install --omit=dev` 安装依赖，否则会报 `Cannot find module 'zod'` 错误。

#### 2. 获取登录二维码

由于服务器环境没有图形界面，需要通过 API 获取登录二维码：

```bash
# 获取登录二维码
curl -s 'https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3'
```

返回示例：
```json
{
  "qrcode": "fbe79d5b66fa1cb6eeb4bedc2691f216",
  "qrcodeUrl": "https://liteapp.weixin.qq.com/q/7GiQu1?qrcode=fbe79d5b66fa1cb6eeb4bedc2691f216&bot_type=3"
}
```

#### 3. 扫码登录

用微信扫描返回的 `qrcodeUrl` 链接，然后监控登录状态：

```bash
# 替换 YOUR_QRCODE 为上一步返回的 qrcode 值
QRCODE="fbe79d5b66fa1cb6eeb4bedc2691f216"

# 循环检查登录状态
for i in {1..60}; do
  result=$(curl -s "https://ilinkai.weixin.qq.com/ilink/bot/get_qrcode_status?qrcode=$QRCODE")
  echo "$result"
  status=$(echo "$result" | grep -o '"status":"[^"]*"' | cut -d'"' -f4)
  
  if [ "$status" = "connected" ]; then
    echo "✅ 登录成功！"
    echo "$result"
    break
  elif [ "$status" != "wait" ]; then
    echo "状态: $status"
    break
  fi
  
  sleep 3
done
```

登录成功后，记录返回的信息：
- `bot_token`: 机器人令牌
- `bot_id`: 机器人 ID
- `user_id`: 用户微信 ID

#### 4. 保存账户配置

登录成功后，需要手动创建账户配置文件：

```bash
# 进入容器
docker compose exec openclaw-gateway bash

# 创建配置目录
mkdir -p /home/node/.openclaw/openclaw-weixin/accounts

# 创建账户索引文件（替换 YOUR_ACCOUNT_ID 为 bot_id 转换后的值）
# 注意：bot_id 格式如 "b2a1d3dfb3ec@im.bot"，需要转换为 "b2a1d3dfb3ec-im-bot"
ACCOUNT_ID="b2a1d3dfb3ec-im-bot"
echo "[\"$ACCOUNT_ID\"]" > /home/node/.openclaw/openclaw-weixin/accounts.json

# 创建账户凭证文件
cat > /home/node/.openclaw/openclaw-weixin/accounts/$ACCOUNT_ID.json << 'EOF'
{
  "token": "YOUR_BOT_TOKEN",
  "baseUrl": "https://ilinkai.weixin.qq.com",
  "userId": "YOUR_USER_ID",
  "savedAt": "2026-04-02T12:00:00Z"
}
EOF

# 退出容器
exit
```

> ⚠️ **注意**: `ACCOUNT_ID` 是将 bot_id 中的 `@` 替换为 `-` 后的值。例如 `b2a1d3dfb3ec@im.bot` → `b2a1d3dfb3ec-im-bot`。

#### 5. 配置绑定规则

```bash
# 配置微信通道绑定到 main agent（替换 YOUR_ACCOUNT_ID）
docker compose exec openclaw-gateway node dist/index.js config set 'bindings' '[{"agentId":"main","match":{"channel":"openclaw-weixin","accountId":"YOUR_ACCOUNT_ID"}}]'
```

#### 6. 重启并验证

```bash
# 重启 Gateway
docker compose restart openclaw-gateway

# 等待服务启动
sleep 15

# 验证微信通道状态
docker compose exec openclaw-gateway node dist/index.js channels status --probe
```

成功输出示例：
```
- openclaw-weixin b2a1d3dfb3ec-im-bot: enabled, configured, running
```

#### 7. 开始对话

现在你可以通过微信与 OpenClaw 对话了：
1. 在微信中找到你的企业微信机器人
2. 发送消息，OpenClaw 会自动回复

#### 常见问题

| 问题                            | 解决方案                                                     |
| ------------------------------- | ------------------------------------------------------------ |
| 二维码过期                      | 重新执行步骤2获取新二维码                                    |
| 账户未显示                      | 检查 `accounts.json` 格式是否正确                            |
| 权限错误                        | 执行 `sudo chown -R 1000:1000 ~/.openclaw/`                  |
| 无法发送消息                    | 确认 bindings 配置正确，accountId 匹配                       |
| ClawHub Rate limit exceeded     | 使用手动安装方式（方式二）                                   |
| Cannot find module 'zod'        | 执行 `npm install --omit=dev` 安装插件依赖                   |
| 插件 suspicious ownership       | 执行 `chown -R root:root` 修复权限                           |
| unknown command 'dist/index.js' | 检查 docker-compose.yml 中 command 配置，使用 `openclaw.mjs` |

#### 8. 配置插件信任列表（可选）

如果日志显示 `blocked plugin candidate: suspicious ownership`，需要在配置中添加插件信任：

```bash
# 方法一：通过 CLI 设置
docker exec openclaw-gateway node openclaw.mjs config set plugins.allow '["openclaw-weixin"]'

# 方法二：直接编辑配置文件
# 编辑 ~/.openclaw/openclaw.json，添加：
# "plugins": {
#   "allow": ["openclaw-weixin"],
#   ...
# }

# 重启 Gateway
docker restart openclaw-gateway
```

---

## 运维管理

### 日志管理

```bash
# 实时日志
docker compose logs -f openclaw-gateway

# 最近100行
docker compose logs --tail 100 openclaw-gateway
```

### 更新升级

```bash
# 使用预构建镜像
docker compose down
docker pull ghcr.io/openclaw/openclaw:latest-slim
docker compose up -d

# 本地构建
git pull origin main
docker compose down
docker compose build
docker compose up -d
```

### 备份与恢复

```bash
# 备份配置和数据
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz \
  ~/.openclaw \
  .env \
  docker-compose.yml

# 恢复
tar -xzf openclaw-backup-YYYYMMDD.tar.gz
```

### 性能监控

```bash
# 容器资源使用
docker stats openclaw-gateway

# 磁盘使用
du -sh ~/.openclaw/*
```

---

## 常见问题快速索引

| 问题                                | 解决方案章节                                                      |
| ----------------------------------- | ----------------------------------------------------------------- |
| 构建时 OOM (Exit 137)               | [前置要求 - Swap配置](#步骤3配置swap空间本地构建必须)             |
| Docker Hub 访问超时                 | [前置要求 - 镜像加速器](#步骤4配置docker镜像加速器国内服务器必须) |
| 环境变量未设置警告                  | [方案B - 步骤2](#2-配置环境变量)                                  |
| 权限错误 EACCES                     | [方案B - 步骤5](#5-修复权限问题重要)                              |
| control ui requires device identity | [HTTPS安全配置](#https安全配置)                                   |
| pairing required                    | [设备配对](#设备配对)                                             |
| origin not allowed                  | [HTTPS安全配置 - 步骤5](#5-配置-gateway-信任代理)                 |
| Proxy headers detected              | [HTTPS安全配置 - 步骤5](#5-配置-gateway-信任代理)                 |
| HTTPS 证书警告                      | [HTTPS安全配置 - 步骤7](#7-访问-https)                            |
| 微信二维码过期                      | [接入微信 - 步骤2](#2-获取登录二维码)                             |
| 微信账户未显示                      | [接入微信 - 常见问题](#常见问题)                                  |
| 微信无法发送消息                    | [接入微信 - 步骤5](#5-配置绑定规则)                               |
| ClawHub API 限流                    | [接入微信 - 步骤1](#1-安装微信插件)                               |
| 插件缺少 zod 依赖                   | [接入微信 - 步骤1](#1-安装微信插件)                               |
| unknown command 'dist/index.js'     | [接入微信 - 常见问题](#常见问题)                                  |
| 沙箱构建网络慢                      | [方案B - 步骤6](#6-构建沙箱镜像可选)                              |

---

## 部署验证清单

部署完成后，请逐项检查：

- [ ] Docker 已正确安装
- [ ] Swap 已配置（本地构建需要）
- [ ] Docker 镜像加速器已配置（国内服务器）
- [ ] Gateway 服务正常启动 (`docker compose ps`)
- [ ] HTTPS 已配置（远程访问必须）
- [ ] 防火墙已开放 18789/18790/443 端口
- [ ] Web 界面可访问 (`https://服务器IP`)
- [ ] Gateway Token 已保存
- [ ] 设备已配对批准
- [ ] API 密钥已配置
- [ ] 模型选择器中显示可用模型
- [ ] 发送测试消息成功

### 微信通道验证清单（可选）

如果配置了微信通道，额外检查：

- [ ] 微信插件已安装 (`@tencent-weixin/openclaw-weixin`)
- [ ] 插件依赖已安装 (`node_modules` 包含 `zod`, `qrcode-terminal`)
- [ ] 插件目录权限正确 (`root:root` 或 `1000:1000`)
- [ ] 扫码登录成功，获得 bot_token
- [ ] 账户配置文件已创建 (`accounts.json` + `accounts/<id>.json`)
- [ ] bindings 规则已配置
- [ ] `channels status --probe` 显示微信账户 running
- [ ] 微信发送测试消息成功

---

## 附录：快速部署命令汇总

### 预构建镜像一键部署

```bash
# 1. 创建目录
mkdir -p ~/openclaw && cd ~/openclaw

# 2. 创建配置文件
cat > docker-compose.yml << 'EOF'
services:
  openclaw-gateway:
    image: ghcr.io/openclaw/openclaw:latest-slim
    container_name: openclaw-gateway
    restart: unless-stopped
    user: "root"
    ports:
      - "18789:18789"
      - "18790:18790"
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY:-}
    volumes:
      - openclaw-data:/home/node/.openclaw
volumes:
  openclaw-data:
EOF

# 3. 生成 Token 并启动
echo "OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)" > .env
docker compose up -d

# 4. 验证
docker compose ps
curl http://127.0.0.1:18789/healthz
```

### 完整部署（含 HTTPS）

```bash
# === 前置配置 ===
# Swap
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && \
sudo mkswap /swapfile && sudo swapon /swapfile && \
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Docker 镜像加速器
sudo mkdir -p /etc/docker && \
sudo tee /etc/docker/daemon.json << 'EOF'
{"registry-mirrors": ["https://docker.1ms.run", "https://docker.xuanyuan.me"]}
EOF
sudo systemctl restart docker

# === 部署 OpenClaw ===
mkdir -p ~/openclaw && cd ~/openclaw
# ... (按上述步骤创建 docker-compose.yml 和 .env)
docker compose up -d

# === HTTPS 配置 ===
SERVER_IP="YOUR_SERVER_IP"
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/caddy/ssl/server.key \
  -out /etc/caddy/ssl/server.crt \
  -subj "/CN=$SERVER_IP" -addext "subjectAltName=IP:$SERVER_IP"
# ... (按上述步骤配置 Caddy)

# === Gateway 配置 ===
docker compose exec openclaw-gateway node dist/index.js config set gateway.trustedProxies '["172.18.0.0/16","127.0.0.1"]'
docker compose exec openclaw-gateway node dist/index.js config set gateway.controlUi.allowedOrigins '["https://'$SERVER_IP'"]'
docker compose restart openclaw-gateway
```

### 本地构建沙箱模式一键部署

```bash
# === 1. 克隆代码 ===
git clone https://github.com/openclaw/openclaw.git && cd openclaw

# === 2. 配置环境 ===
mkdir -p ~/.openclaw/workspace && sudo chown -R 1000:1000 ~/.openclaw/
GATEWAY_TOKEN=$(openssl rand -hex 32)
cat > .env << EOF
OPENCLAW_GATEWAY_TOKEN=$GATEWAY_TOKEN
OPENCLAW_CONFIG_DIR=/root/.openclaw
OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace
DOCKER_GID=$(getent group docker | cut -d: -f3)
EOF
echo "Gateway Token: $GATEWAY_TOKEN"

# === 3. 构建基础镜像 ===
nohup docker build -t openclaw:local -f Dockerfile . > /tmp/docker-build.log 2>&1 &
# 等待构建完成，监控：tail -f /tmp/docker-build.log

# === 4. 构建沙箱镜像 ===
cat > Dockerfile.sandbox << 'EOF'
FROM node:24-bookworm-slim
RUN apt-get update && apt-get install -y docker.io && rm -rf /var/lib/apt/lists/*
COPY --from=openclaw:local /app /app
WORKDIR /app
ENV NODE_ENV=production
EOF
nohup docker build -f Dockerfile.sandbox -t openclaw:sandbox . > /tmp/docker-build-sandbox.log 2>&1 &
# 等待构建完成，监控：tail -f /tmp/docker-build-sandbox.log

# === 5. 创建 docker-compose.yml（沙箱模式） ===
cat > docker-compose.yml << 'EOF'
services:
  openclaw-gateway:
    image: openclaw:sandbox
    container_name: openclaw-gateway
    restart: unless-stopped
    user: "root"
    ports:
      - "18789:18789"
      - "18790:18790"
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - HOME=/home/node
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
      - /var/run/docker.sock:/var/run/docker.sock
    group_add:
      - "${DOCKER_GID:-999}"
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:18789/healthz').then((r)=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 20s
EOF

# === 6. 启动服务 ===
docker compose up -d && sleep 10 && docker compose ps && curl http://127.0.0.1:18789/healthz
```

### 微信插件快速安装

```bash
# === 手动安装（解决 API 限流） ===
docker exec openclaw-gateway mkdir -p /home/node/.openclaw/extensions
docker exec openclaw-gateway sh -c 'cd /home/node/.openclaw/extensions && \
  npm pack @tencent-weixin/openclaw-weixin@2.1.3 && \
  tar -xzf *.tgz && rm *.tgz && \
  mv package openclaw-weixin && \
  npm install --omit=dev && \
  chown -R root:root openclaw-weixin'

# 重启加载插件
docker restart openclaw-gateway

# 验证
sleep 15 && docker logs openclaw-gateway 2>&1 | grep -i weixin
```
