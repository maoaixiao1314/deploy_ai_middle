# 云服务器部署指南

本文档提供两种部署方式：**Docker 部署**（推荐）和**传统部署**。

---

## 方式一：Docker 部署（推荐）

### 前置要求

- 云服务器（阿里云、腾讯云、AWS 等）
- Ubuntu 20.04+ / CentOS 7+ / Debian 10+
- 至少 2GB RAM，10GB 磁盘空间
- 开放端口：80（前端）、3000（后端）、5432（数据库，可选）

### 1. 安装 Docker 和 Docker Compose

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 启动 Docker
sudo systemctl start docker
sudo systemctl enable docker

# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 验证安装
docker --version
docker-compose --version
```

### 2. 上传代码到服务器

```bash
# 方式 A：使用 Git（推荐）
cd /opt
sudo git clone https://github.com/your-username/AI_Middle_Tier.git
cd AI_Middle_Tier

# 方式 B：使用 scp 上传
# 在本地执行：
scp -r /path/to/AI_Middle_Tier root@your-server-ip:/opt/
```

### 3. 配置环境变量

```bash
# 复制环境变量模板
cp .env.example .env

# 编辑环境变量
nano .env
```

**重要配置项：**

```bash
# 修改数据库密码（必须）
POSTGRES_PASSWORD=your_strong_password_here

# 修改 JWT 密钥（必须，至少 32 字符）
JWT_SECRET=your_very_long_and_random_jwt_secret_key_here

# 配置域名或 IP（必须）
CORS_ORIGIN=http://your-domain.com
VITE_API_URL=http://your-domain.com:3000

# 配置 AI API Key（可选，不配置则使用正则解析）
OPENAI_API_KEY=sk-...
# 或
ANTHROPIC_API_KEY=sk-ant-...
```

### 4. 构建并启动服务

```bash
# 构建镜像
sudo docker-compose build

# 启动所有服务
sudo docker-compose up -d

# 查看服务状态
sudo docker-compose ps

# 查看日志
sudo docker-compose logs -f
```

### 5. 验证部署

```bash
# 检查后端健康状态
curl http://localhost:3000/api/health

# 检查前端
curl http://localhost
```

访问 `http://your-server-ip` 即可看到前端页面。

### 6. 常用管理命令

```bash
# 停止服务
sudo docker-compose stop

# 重启服务
sudo docker-compose restart

# 查看日志
sudo docker-compose logs -f backend
sudo docker-compose logs -f frontend

# 进入容器
sudo docker-compose exec backend sh
sudo docker-compose exec postgres psql -U postgres -d ai_middle_tier

# 更新代码后重新部署
git pull
sudo docker-compose down
sudo docker-compose build
sudo docker-compose up -d

# 备份数据库
sudo docker-compose exec postgres pg_dump -U postgres ai_middle_tier > backup_$(date +%Y%m%d).sql

# 恢复数据库
sudo docker-compose exec -T postgres psql -U postgres ai_middle_tier < backup_20260309.sql
```

---

## 方式二：传统部署（不使用 Docker）

### 1. 安装依赖

```bash
# 安装 Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# 安装 PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# 安装 Nginx
sudo apt install -y nginx

# 安装 PM2（进程管理器）
sudo npm install -g pm2
```

### 2. 配置 PostgreSQL

```bash
# 切换到 postgres 用户
sudo -u postgres psql

# 在 psql 中执行：
CREATE DATABASE ai_middle_tier;
CREATE USER ai_user WITH PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE ai_middle_tier TO ai_user;
\q
```

### 3. 部署后端

```bash
cd /opt/AI_Middle_Tier/backend

# 安装依赖
npm install

# 配置环境变量
cp .env.example .env
nano .env

# 修改 DATABASE_URL
# DATABASE_URL="postgresql://ai_user:your_password@localhost:5432/ai_middle_tier?schema=public"

# 运行数据库迁移
npx prisma migrate deploy

# 构建项目
npm run build

# 使用 PM2 启动
pm2 start dist/server.js --name ai-middle-tier-backend
pm2 save
pm2 startup
```

### 4. 部署前端

```bash
cd /opt/AI_Middle_Tier/frontend

# 安装依赖
npm install

# 配置环境变量
cp .env.example .env
nano .env

# 修改 VITE_API_URL
# VITE_API_URL=http://your-domain.com:3000

# 构建生产版本
npm run build

# 复制构建文件到 Nginx 目录
sudo cp -r dist/* /var/www/html/
```

### 5. 配置 Nginx

```bash
sudo nano /etc/nginx/sites-available/ai-middle-tier
```

添加以下配置：

```nginx
server {
    listen 80;
    server_name your-domain.com;

    # 前端
    location / {
        root /var/www/html;
        try_files $uri $uri/ /index.html;
    }

    # 后端 API 代理
    location /api {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # WebSocket 支持
    location /socket.io {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

启用配置：

```bash
sudo ln -s /etc/nginx/sites-available/ai-middle-tier /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 配置 HTTPS（推荐）

### 使用 Let's Encrypt 免费证书

```bash
# 安装 Certbot
sudo apt install -y certbot python3-certbot-nginx

# 获取证书（Docker 部署需要先停止前端容器）
sudo docker-compose stop frontend  # Docker 部署时执行
sudo certbot --nginx -d your-domain.com

# 自动续期
sudo certbot renew --dry-run
```

修改 `.env` 文件：

```bash
CORS_ORIGIN=https://your-domain.com
VITE_API_URL=https://your-domain.com:3000
```

重新构建前端：

```bash
sudo docker-compose build frontend
sudo docker-compose up -d
```

---

## 防火墙配置

```bash
# Ubuntu/Debian (UFW)
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 3000/tcp  # Backend API
sudo ufw enable

# CentOS/RHEL (Firewalld)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

---

## 性能优化建议

### 1. 启用 Nginx 缓存

```nginx
# 在 nginx.conf 的 http 块中添加
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m max_size=1g inactive=60m;
```

### 2. 配置数据库连接池

在 `backend/src/utils/prisma.ts` 中：

```typescript
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  log: ['error'],
  // 连接池配置
  __internal: {
    engine: {
      connection_limit: 10,
    },
  },
});
```

### 3. 启用 PM2 集群模式（传统部署）

```bash
pm2 start dist/server.js -i max --name ai-middle-tier-backend
```

---

## 监控和日志

### Docker 部署

```bash
# 实时查看日志
sudo docker-compose logs -f

# 查看资源使用
sudo docker stats

# 导出日志
sudo docker-compose logs > logs_$(date +%Y%m%d).txt
```

### 传统部署

```bash
# PM2 监控
pm2 monit

# 查看日志
pm2 logs ai-middle-tier-backend

# Nginx 日志
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

---

## 故障排查

### 后端无法启动

```bash
# 检查数据库连接
sudo docker-compose exec backend npx prisma db pull

# 查看详细日志
sudo docker-compose logs backend
```

### 前端无法访问后端

1. 检查 CORS 配置是否正确
2. 确认防火墙已开放 3000 端口
3. 检查 `VITE_API_URL` 是否配置正确

### 数据库连接失败

```bash
# 检查 PostgreSQL 是否运行
sudo docker-compose ps postgres

# 测试数据库连接
sudo docker-compose exec postgres psql -U postgres -d ai_middle_tier -c "SELECT 1;"
```

---

## 备份策略

### 自动备份脚本

创建 `/opt/backup.sh`：

```bash
#!/bin/bash
BACKUP_DIR="/opt/backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# 备份数据库
docker-compose exec -T postgres pg_dump -U postgres ai_middle_tier | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# 删除 7 天前的备份
find $BACKUP_DIR -name "db_*.sql.gz" -mtime +7 -delete

echo "Backup completed: db_$DATE.sql.gz"
```

设置定时任务：

```bash
chmod +x /opt/backup.sh
crontab -e

# 每天凌晨 2 点备份
0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1
```

---

## 更新部署

```bash
# 拉取最新代码
cd /opt/AI_Middle_Tier
git pull

# Docker 部署
sudo docker-compose down
sudo docker-compose build
sudo docker-compose up -d

# 传统部署
cd backend
npm install
npm run build
pm2 restart ai-middle-tier-backend

cd ../frontend
npm install
npm run build
sudo cp -r dist/* /var/www/html/
```

---

## 推荐云服务商配置

| 服务商 | 最低配置 | 推荐配置 |
|--------|---------|---------|
| 阿里云 | 2核2G | 2核4G |
| 腾讯云 | 2核2G | 2核4G |
| AWS | t2.small | t2.medium |
| DigitalOcean | $12/月 | $24/月 |

---

**部署完成后，访问 `http://your-server-ip` 即可使用系统！**

