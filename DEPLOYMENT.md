# Sparka 生产环境部署指南

## 📋 部署概览

**部署架构：**
- **应用服务**：Docker 容器运行 Next.js（生产模式）
- **数据库**：Docker 运行 PostgreSQL 16 + Redis 7
- **反向代理**：Nginx（HTTPS + 域名）
- **系统要求**：Ubuntu 22.04 / 4GB+ 内存 / 20GB+ 磁盘空间

**当前环境信息：**
- 系统：Ubuntu 22.04.5 LTS
- 内存：125GB
- 可用磁盘：539GB
- Docker：28.3.3
- Node.js：v20.19.5

---

## 🚀 完整部署流程

### 步骤 1：安装 Bun 运行时

Bun 是项目的包管理器和构建工具，版本要求 ≥1.1.34。

```bash
# 安装 Bun
curl -fsSL https://bun.sh/install | bash

# 重新加载 Shell 配置
source ~/.bashrc  # 如果使用 bash
# 或
source ~/.zshrc   # 如果使用 zsh

# 验证安装
bun --version  # 应该显示 1.1.34 或更高版本
```

---

### 步骤 2：准备项目目录

```bash
# 进入项目目录
cd /home/yan/codes/sparka

# 安装项目依赖
bun install

# 验证依赖安装
ls -la node_modules  # 应该能看到大量依赖包
```

---

### 步骤 3：配置环境变量

创建生产环境配置文件 `.env.production.local`：

```bash
# 创建环境变量文件
cat > .env.production.local << 'EOF'
# ==================== 数据库配置 ====================
# PostgreSQL 连接字符串（Docker 内部）
POSTGRES_URL="postgresql://sparka:your_secure_password_here@postgres:5432/sparka?sslmode=disable"

# Redis 连接字符串（Docker 内部，用于流式对话恢复）
REDIS_URL="redis://:your_redis_password_here@redis:6379/0"

# ==================== 认证配置 ====================
# Better Auth 密钥（生成命令见下方）
AUTH_SECRET="your_auth_secret_here"

# 应用访问地址（替换为你的域名）
BETTER_AUTH_URL="https://your-domain.com"

# Google OAuth（可选，如需 Google 登录）
AUTH_GOOGLE_ID="your_google_client_id"
AUTH_GOOGLE_SECRET="your_google_client_secret"

# GitHub OAuth（可选，如需 GitHub 登录）
AUTH_GITHUB_ID="your_github_client_id"
AUTH_GITHUB_SECRET="your_github_client_secret"

# ==================== AI 服务配置 ====================
# Vercel AI Gateway API Key（必需）
AI_GATEWAY_API_KEY="your_vercel_ai_gateway_key"

# OpenAI API Key（可选，如直接调用 OpenAI）
OPENAI_API_KEY="your_openai_api_key"

# ==================== 存储配置 ====================
# Vercel Blob 存储（用于图片上传）
BLOB_READ_WRITE_TOKEN="your_vercel_blob_token"

# ==================== 其他服务（可选）====================
# 定时任务验证密钥
CRON_SECRET="your_cron_secret_here"

# Tavily 搜索 API（Web 搜索功能）
TAVILY_API_KEY="your_tavily_api_key"

# Exa 搜索 API（Web 搜索功能）
EXA_API_KEY="your_exa_api_key"

# Firecrawl API（网页抓取功能）
FIRECRAWL_API_KEY="your_firecrawl_api_key"

# E2B Sandbox（代码执行功能）
SANDBOX_TEMPLATE_ID="your_e2b_template_id"

# ==================== 生产环境配置 ====================
NODE_ENV="production"
EOF

echo "环境变量模板已创建，请编辑 .env.production.local 填写实际值"
```

**必需的密钥生成：**

```bash
# 生成 AUTH_SECRET（64字符随机字符串）
openssl rand -base64 48

# 生成 Redis 密码
openssl rand -base64 32

# 生成 PostgreSQL 密码
openssl rand -base64 32

# 生成 CRON_SECRET
openssl rand -base64 32
```

**重要提示：**
1. 将生成的密钥填入 `.env.production.local` 对应位置
2. 替换 `your-domain.com` 为你的实际域名
3. 最少需要配置：`POSTGRES_URL`、`AUTH_SECRET`、`BETTER_AUTH_URL`、`AI_GATEWAY_API_KEY`、`BLOB_READ_WRITE_TOKEN`

**获取第三方 API Key：**
- **Vercel AI Gateway**：https://vercel.com/docs/ai-gateway
- **Vercel Blob**：https://vercel.com/docs/storage/vercel-blob
- **Google OAuth**：https://console.cloud.google.com/apis/credentials
- **GitHub OAuth**：https://github.com/settings/developers
- **Tavily**：https://tavily.com/
- **Exa**：https://exa.ai/
- **Firecrawl**：https://firecrawl.dev/

---

### 步骤 4：创建 Docker Compose 配置

创建 `docker-compose.yml` 用于数据库服务：

```bash
cat > docker-compose.db.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: sparka-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: sparka
      POSTGRES_USER: sparka
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  # 从环境变量读取
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - sparka-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sparka"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: sparka-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}  # 从环境变量读取
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - sparka-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  sparka-network:
    driver: bridge
EOF

echo "数据库 Docker Compose 配置已创建"
```

创建用于 Docker Compose 的环境变量文件：

```bash
cat > .env.docker << 'EOF'
# 从 .env.production.local 中提取密码填入此处
POSTGRES_PASSWORD=your_postgres_password_here
REDIS_PASSWORD=your_redis_password_here
EOF

echo "请编辑 .env.docker 填写数据库密码（与 .env.production.local 中的密码一致）"
```

---

### 步骤 5：启动数据库服务

```bash
# 启动 PostgreSQL 和 Redis
docker-compose -f docker-compose.db.yml --env-file .env.docker up -d

# 查看服务状态
docker-compose -f docker-compose.db.yml ps

# 查看日志（确认启动成功）
docker-compose -f docker-compose.db.yml logs -f

# 验证 PostgreSQL 连接
docker exec -it sparka-postgres psql -U sparka -d sparka -c "SELECT version();"

# 验证 Redis 连接（替换 your_redis_password）
docker exec -it sparka-redis redis-cli -a your_redis_password_here ping
# 应返回：PONG
```

---

### 步骤 6：运行数据库迁移

在启动应用之前，需要初始化数据库表结构：

```bash
# 确保 .env.production.local 中的 POSTGRES_URL 使用 localhost:5432
# 因为迁移脚本在宿主机运行，而不是在 Docker 容器内

# 临时修改数据库连接（仅用于迁移）
export POSTGRES_URL="postgresql://sparka:your_postgres_password_here@localhost:5432/sparka?sslmode=disable"

# 运行迁移
bun db:migrate

# 验证迁移成功
docker exec -it sparka-postgres psql -U sparka -d sparka -c "\dt"
# 应该能看到多个表：User, Session, Chat, Message, Vote 等
```

---

### 步骤 7：同步 AI 模型数据

项目需要从外部 API 同步 AI 模型列表到数据库：

```bash
# 同步所有模型（首次部署使用）
bun models:sync:all

# 或者仅同步新增模型（后续更新使用）
bun models:sync

# 验证模型数据
docker exec -it sparka-postgres psql -U sparka -d sparka -c "SELECT COUNT(*) FROM \"Model\";"
# 应该显示 120+ 条记录
```

---

### 步骤 8：构建生产版本

```bash
# 构建 Next.js 生产版本
bun run build

# 构建完成后检查输出目录
ls -lh .next/standalone  # 应该包含服务器文件
ls -lh .next/static      # 应该包含静态资源
ls -lh public            # 应该包含公共文件

# 测试本地运行（可选）
bun start
# 访问 http://localhost:3000 验证应用是否正常
# 验证后按 Ctrl+C 停止
```

---

### 步骤 9：创建应用 Dockerfile

```bash
cat > Dockerfile << 'EOF'
# ==================== 基础镜像 ====================
FROM imbios/bun-node:22-alpine AS base

# ==================== 依赖安装阶段 ====================
FROM base AS deps
WORKDIR /app

# 复制依赖配置文件
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile --production

# ==================== 构建阶段 ====================
FROM base AS builder
WORKDIR /app

# 复制依赖（包含 devDependencies）
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile

# 复制源代码
COPY . .

# 构建应用
ENV NEXT_TELEMETRY_DISABLED=1
RUN bun run build

# ==================== 运行阶段 ====================
FROM base AS runner
WORKDIR /app

# 创建非 root 用户
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# 设置生产环境
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# 复制构建产物
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# 切换到非 root 用户
USER nextjs

# 暴露端口
EXPOSE 3000

# 设置环境变量
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# 启动应用
CMD ["node", "server.js"]
EOF

echo "Dockerfile 已创建"
```

**创建 .dockerignore：**

```bash
cat > .dockerignore << 'EOF'
node_modules
.next
.git
.gitignore
README.md
DEPLOYMENT.md
.env*
!.env.production.local
docker-compose*.yml
Dockerfile
.dockerignore
*.md
.vscode
.idea
coverage
.turbo
dist
EOF

echo ".dockerignore 已创建"
```

---

### 步骤 10：创建应用 Docker Compose 配置

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  sparka-app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: sparka-app
    restart: unless-stopped
    ports:
      - "3000:3000"
    env_file:
      - .env.production.local
    networks:
      - sparka-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  postgres:
    image: postgres:16-alpine
    container_name: sparka-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: sparka
      POSTGRES_USER: sparka
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - sparka-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sparka"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: sparka-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - sparka-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  sparka-network:
    driver: bridge
EOF

echo "应用 Docker Compose 配置已创建"
```

---

### 步骤 11：构建并启动应用

```bash
# 构建 Docker 镜像
docker-compose build

# 启动所有服务
docker-compose --env-file .env.docker up -d

# 查看服务状态
docker-compose ps

# 查看应用日志
docker-compose logs -f sparka-app

# 验证应用运行
curl http://localhost:3000
# 应该返回 HTML 内容
```

---

### 步骤 12：安装和配置 Nginx

```bash
# 安装 Nginx
sudo apt update
sudo apt install -y nginx

# 验证安装
nginx -v

# 安装 Certbot（用于 SSL 证书）
sudo apt install -y certbot python3-certbot-nginx
```

**创建 Nginx 配置文件：**

```bash
# 替换 your-domain.com 为你的实际域名
DOMAIN="your-domain.com"

sudo tee /etc/nginx/sites-available/sparka << EOF
# 重定向 HTTP 到 HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name ${DOMAIN};

    # Let's Encrypt 验证路径
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # 重定向到 HTTPS
    location / {
        return 301 https://\$host\$request_uri;
    }
}

# HTTPS 配置
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name ${DOMAIN};

    # SSL 证书路径（Certbot 会自动配置）
    ssl_certificate /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # 安全头
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # 日志
    access_log /var/log/nginx/sparka_access.log;
    error_log /var/log/nginx/sparka_error.log;

    # 客户端最大上传大小（用于图片上传）
    client_max_body_size 50M;

    # 反向代理到 Next.js
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;

        # WebSocket 支持（用于流式对话）
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";

        # 代理头
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_set_header X-Forwarded-Host \$host;
        proxy_set_header X-Forwarded-Port \$server_port;

        # 超时设置（用于长时间 AI 响应）
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;

        # 缓冲设置
        proxy_buffering off;
        proxy_request_buffering off;
    }

    # 静态资源缓存
    location /_next/static {
        proxy_pass http://localhost:3000;
        proxy_cache_valid 200 365d;
        add_header Cache-Control "public, immutable";
    }

    # 图片缓存
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
        proxy_pass http://localhost:3000;
        proxy_cache_valid 200 30d;
        add_header Cache-Control "public, max-age=2592000";
    }
}
EOF

echo "Nginx 配置已创建"
```

**启用配置并获取 SSL 证书：**

```bash
# 启用站点配置
sudo ln -s /etc/nginx/sites-available/sparka /etc/nginx/sites-enabled/

# 删除默认配置
sudo rm -f /etc/nginx/sites-enabled/default

# 测试配置
sudo nginx -t

# 重启 Nginx
sudo systemctl restart nginx

# 获取 SSL 证书（替换邮箱和域名）
sudo certbot --nginx -d your-domain.com --email your-email@example.com --agree-tos --no-eff-email

# 设置证书自动续期
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer

# 验证自动续期任务
sudo certbot renew --dry-run
```

---

### 步骤 13：配置防火墙

```bash
# 检查防火墙状态
sudo ufw status

# 如果未启用，配置规则
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS

# 启用防火墙
sudo ufw enable

# 验证规则
sudo ufw status verbose
```

---

### 步骤 14：验证部署

```bash
# 1. 检查所有 Docker 容器状态
docker-compose ps
# 应该显示 3 个容器都是 "Up (healthy)"

# 2. 检查应用日志
docker-compose logs --tail=50 sparka-app
# 应该没有错误信息

# 3. 检查数据库连接
docker exec -it sparka-postgres psql -U sparka -d sparka -c "SELECT COUNT(*) FROM \"User\";"

# 4. 检查 Nginx 状态
sudo systemctl status nginx

# 5. 测试 HTTPS 访问
curl -I https://your-domain.com
# 应该返回 200 OK

# 6. 在浏览器访问
# https://your-domain.com
```

---

## 🔧 日常运维命令

### 查看日志

```bash
# 查看应用日志
docker-compose logs -f sparka-app

# 查看数据库日志
docker-compose logs -f postgres

# 查看 Redis 日志
docker-compose logs -f redis

# 查看 Nginx 日志
sudo tail -f /var/log/nginx/sparka_access.log
sudo tail -f /var/log/nginx/sparka_error.log
```

### 重启服务

```bash
# 重启应用
docker-compose restart sparka-app

# 重启所有服务
docker-compose restart

# 重启 Nginx
sudo systemctl restart nginx
```

### 更新应用

```bash
# 1. 拉取最新代码
git pull origin main

# 2. 重新构建镜像
docker-compose build sparka-app

# 3. 重启应用（无缝更新）
docker-compose up -d sparka-app

# 4. 查看日志确认启动成功
docker-compose logs -f sparka-app
```

### 数据库备份

```bash
# 备份数据库
docker exec sparka-postgres pg_dump -U sparka sparka > backup_$(date +%Y%m%d_%H%M%S).sql

# 恢复数据库
docker exec -i sparka-postgres psql -U sparka sparka < backup_20250118_120000.sql
```

### 停止服务

```bash
# 停止所有服务
docker-compose down

# 停止并删除数据（谨慎使用！）
docker-compose down -v
```

---

## 🔍 故障排查

### 应用无法启动

```bash
# 1. 检查环境变量
docker-compose config

# 2. 查看详细日志
docker-compose logs sparka-app | grep -i error

# 3. 检查端口占用
sudo netstat -tlnp | grep 3000

# 4. 进入容器调试
docker exec -it sparka-app sh
```

### 数据库连接失败

```bash
# 1. 检查数据库是否运行
docker-compose ps postgres

# 2. 测试连接
docker exec -it sparka-postgres psql -U sparka -d sparka

# 3. 检查网络
docker network inspect sparka_sparka-network
```

### SSL 证书问题

```bash
# 检查证书有效期
sudo certbot certificates

# 手动续期
sudo certbot renew

# 重新申请证书
sudo certbot --nginx -d your-domain.com --force-renewal
```

### 性能问题

```bash
# 查看容器资源使用
docker stats

# 查看系统资源
htop  # 或 top

# 检查磁盘空间
df -h
```

---

## 📊 监控建议

### 推荐工具

1. **应用监控**：
   - Sentry（错误跟踪）
   - Vercel Analytics（性能分析）

2. **服务器监控**：
   - Prometheus + Grafana
   - Netdata

3. **日志聚合**：
   - Loki + Grafana
   - ELK Stack

### 健康检查端点

应用提供以下健康检查端点：

```bash
# 基础健康检查
curl http://localhost:3000/api/health

# 数据库连接检查（需要添加）
curl http://localhost:3000/api/health/db
```

---

## 🔐 安全建议

1. **定期更新系统**：
```bash
sudo apt update && sudo apt upgrade -y
```

2. **启用自动安全更新**：
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

3. **限制 SSH 访问**：
```bash
# 编辑 /etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config
# 设置：
# PermitRootLogin no
# PasswordAuthentication no
```

4. **定期备份数据**：
```bash
# 创建 cron 任务每天备份
0 2 * * * /path/to/backup_script.sh
```

5. **监控异常登录**：
```bash
# 查看失败登录
sudo grep "Failed password" /var/log/auth.log
```

---

## 📝 环境变量清单

### 必需配置

- ✅ `POSTGRES_URL` - 数据库连接
- ✅ `AUTH_SECRET` - 认证密钥
- ✅ `BETTER_AUTH_URL` - 应用地址
- ✅ `AI_GATEWAY_API_KEY` - AI Gateway
- ✅ `BLOB_READ_WRITE_TOKEN` - 文件存储

### 可选配置

- ⭕ `REDIS_URL` - 流式对话恢复
- ⭕ `AUTH_GOOGLE_ID/SECRET` - Google 登录
- ⭕ `AUTH_GITHUB_ID/SECRET` - GitHub 登录
- ⭕ `OPENAI_API_KEY` - OpenAI 直连
- ⭕ `TAVILY_API_KEY` - Web 搜索
- ⭕ `EXA_API_KEY` - Web 搜索
- ⭕ `FIRECRAWL_API_KEY` - 网页抓取
- ⭕ `SANDBOX_TEMPLATE_ID` - 代码执行

---

## 🎯 下一步

部署完成后，建议：

1. **配置监控告警**：设置 Sentry 或其他监控服务
2. **设置备份策略**：定期备份数据库和用户数据
3. **性能优化**：根据实际流量调整 Docker 资源限制
4. **CDN 配置**：使用 Cloudflare 加速静态资源
5. **负载均衡**：如需高可用，配置多实例 + Nginx 负载均衡

---

## 📞 支持

- **项目文档**：查看 README.md
- **技术问题**：检查 GitHub Issues
- **安全问题**：参考 SECURITY.md

---

**部署完成日期：** _待填写_
**部署人员：** _待填写_
**域名：** _待填写_
**版本：** _待填写_
