# 独立部署目录方案

## 目录结构

在服务器上创建如下结构：

```
~/
├── AI_Middle_Tier/              # 项目代码（git clone）
└── ai-deploy/                   # 部署配置（独立目录）
    ├── docker-compose.yml
    ├── Dockerfile.backend
    ├── Dockerfile.frontend
    ├── nginx.conf
    ├── .env
    ├── .dockerignore
    └── backups/                 # 数据库备份目录
```

## 部署步骤

### 1. 在服务器上创建部署目录

```bash
# SSH 登录服务器后执行
cd ~

# 克隆项目代码
git clone https://github.com/your-username/AI_Middle_Tier.git

# 创建独立的部署目录
mkdir -p ai-deploy/backups
cd ai-deploy
```

### 2. 创建部署文件

将以下文件内容复制到服务器的 `~/ai-deploy/` 目录中：

#### 2.1 创建 `docker-compose.yml`

```bash
nano docker-compose.yml
```

复制以下内容：

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: ai-middle-tier-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-your_secure_password}
      POSTGRES_DB: ${POSTGRES_DB:-ai_middle_tier}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Backend API
  backend:
    build:
      context: ~/AI_Middle_Tier/backend
      dockerfile: ~/ai-deploy/Dockerfile.backend
    container_name: ai-middle-tier-backend
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD:-your_secure_password}@postgres:5432/${POSTGRES_DB:-ai_middle_tier}?schema=public
      JWT_SECRET: ${JWT_SECRET:-your_jwt_secret_change_this}
      JWT_EXPIRES_IN: ${JWT_EXPIRES_IN:-7d}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      CORS_ORIGIN: ${CORS_ORIGIN:-http://localhost}
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-network
    volumes:
      - ~/AI_Middle_Tier/backend:/app:ro
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Frontend
  frontend:
    build:
      context: ~/AI_Middle_Tier/frontend
      dockerfile: ~/ai-deploy/Dockerfile.frontend
      args:
        VITE_API_URL: ${VITE_API_URL:-http://localhost:3000}
    container_name: ai-middle-tier-frontend
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - app-network

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

#### 2.2 创建 `Dockerfile.backend`

```bash
nano Dockerfile.backend
```

```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Build TypeScript
RUN npm run build

# Production stage
FROM node:18-alpine

WORKDIR /app

# Copy package files and install production dependencies only
COPY package*.json ./
RUN npm ci --only=production

# Copy Prisma schema
COPY prisma ./prisma

# Copy built files from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

# Expose port
EXPOSE 3000

# Run migrations and start server
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/server.js"]
```

#### 2.3 创建 `Dockerfile.frontend`

```bash
nano Dockerfile.frontend
```

```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# Build for production
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

# Production stage with nginx
FROM nginx:alpine

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**注意：** 这个 Dockerfile 需要 nginx.conf 文件，我们需要修改一下构建方式。

#### 2.4 创建 `nginx.conf`

```bash
nano nginx.conf
```

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json application/javascript;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Frontend routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Disable cache for index.html
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }
}
```

#### 2.5 创建 `.env`

```bash
nano .env
```

```bash
# Database Configuration
POSTGRES_USER=postgres
POSTGRES_PASSWORD=你的强密码_必须修改
POSTGRES_DB=ai_middle_tier

# Backend Configuration
NODE_ENV=production
PORT=3000

# JWT Configuration (至少32字符)
JWT_SECRET=你的超长随机JWT密钥_必须修改_至少32字符
JWT_EXPIRES_IN=7d

# AI API Keys (可选)
OPENAI_API_KEY=
ANTHROPIC_API_KEY=

# CORS Configuration (改成你的域名或IP)
CORS_ORIGIN=http://你的服务器IP或域名

# Frontend Configuration (改成你的域名或IP)
VITE_API_URL=http://你的服务器IP或域名:3000
```

#### 2.6 创建 `.dockerignore`

```bash
nano .dockerignore
```

```
node_modules
npm-debug.log
.env
.env.local
.git
.gitignore
*.md
dist
build
.DS_Store
coverage
.vscode
.idea
*.log
```

### 3. 修改 docker-compose.yml（重要）

由于 Dockerfile 在独立目录，需要调整构建配置：

```bash
nano docker-compose.yml
```

修改为：

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: ai-middle-tier-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB:-ai_middle_tier}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ../AI_Middle_Tier/backend
      dockerfile: ../../ai-deploy/Dockerfile.backend
    container_name: ai-middle-tier-backend
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB:-ai_middle_tier}?schema=public
      JWT_SECRET: ${JWT_SECRET}
      JWT_EXPIRES_IN: ${JWT_EXPIRES_IN:-7d}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      CORS_ORIGIN: ${CORS_ORIGIN}
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-network

  frontend:
    build:
      context: ../AI_Middle_Tier/frontend
      dockerfile: ../../ai-deploy/Dockerfile.frontend
      args:
        VITE_API_URL: ${VITE_API_URL}
    container_name: ai-middle-tier-frontend
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - app-network
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

### 4. 启动部署

```bash
cd ~/ai-deploy

# 检查配置
cat .env

# 构建并启动
docker-compose up -d

# 查看日志
docker-compose logs -f
```

### 5. 更新代码流程

当项目代码更新时：

```bash
# 1. 更新代码
cd ~/AI_Middle_Tier
git pull

# 2. 重新构建并部署
cd ~/ai-deploy
docker-compose down
docker-compose build --no-cache
docker-compose up -d

# 3. 查看状态
docker-compose ps
```

### 6. 常用管理命令

```bash
# 在 ~/ai-deploy 目录下执行

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f
docker-compose logs -f backend
docker-compose logs -f frontend

# 重启服务
docker-compose restart

# 停止服务
docker-compose stop

# 完全停止并删除容器
docker-compose down

# 备份数据库
docker-compose exec postgres pg_dump -U postgres ai_middle_tier > backups/backup_$(date +%Y%m%d_%H%M%S).sql

# 恢复数据库
docker-compose exec -T postgres psql -U postgres ai_middle_tier < backups/backup_20260309_120000.sql

# 进入容器调试
docker-compose exec backend sh
docker-compose exec postgres psql -U postgres -d ai_middle_tier
```

### 7. 自动备份脚本

创建备份脚本：

```bash
nano ~/ai-deploy/backup.sh
```

```bash
#!/bin/bash
cd ~/ai-deploy
BACKUP_DIR="./backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# 备份数据库
docker-compose exec -T postgres pg_dump -U postgres ai_middle_tier | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# 删除 7 天前的备份
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: backup_$DATE.sql.gz"
```

设置权限和定时任务：

```bash
chmod +x ~/ai-deploy/backup.sh

# 添加定时任务（每天凌晨2点备份）
crontab -e
# 添加这行：
0 2 * * * ~/ai-deploy/backup.sh >> ~/ai-deploy/backup.log 2>&1
```

## 优势

✅ **代码库干净**：部署文件不会污染项目代码  
✅ **无冲突更新**：`git pull` 不会有冲突  
✅ **独立管理**：部署配置和代码分离，更清晰  
✅ **灵活调整**：可以随时修改部署配置而不影响代码  
✅ **多环境支持**：可以创建多个部署目录（dev/staging/prod）

## 目录说明

- `~/AI_Middle_Tier/` - 项目源代码，只用 git 管理
- `~/ai-deploy/` - 部署配置，包含所有 Docker 和环境配置
- `~/ai-deploy/backups/` - 数据库备份存储

这样项目代码更新时，只需要 `git pull`，然后在 `ai-deploy` 目录重新构建即可！

