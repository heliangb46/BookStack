# BookStack 部署指南

本文档面向正式部署。正式部署和二次开发调试要分开处理：开发环境追求断点、热构建、快速改代码；正式部署追求稳定、可备份、可升级、少暴露调试能力。

## 推荐部署方式

如果你只是做界面配置、主题、少量自定义头部样式，推荐使用官方或 LinuxServer BookStack 镜像部署。

如果你已经修改了 BookStack 源码，例如 PHP、Blade、JS、Sass 逻辑，那么推荐先把代码提交到你自己的 GitHub 仓库，再选择源码部署或自定义镜像部署。

不要把项目根目录的开发 compose 直接当生产部署使用。它会启用开发依赖、Xdebug、watcher、MailHog，不适合生产。

## 方式一：使用 LinuxServer 镜像部署

适合场景：

```text
1. 不改 BookStack 核心 PHP 源码。
2. 主要通过管理后台、自定义 HTML Head、主题目录做定制。
3. 希望升级维护简单。
```

目录建议：

```bash
mkdir -p /app/docker/bookstack/data/bookstack
mkdir -p /app/docker/bookstack/data/mariadb
cd /app/docker/bookstack
```

示例 `docker-compose.yml`：

```yaml
services:
  bookstack-db:
    image: lscr.io/linuxserver/mariadb:latest
    container_name: bookstack-db
    restart: unless-stopped
    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: "change-this-root-password"
      MYSQL_DATABASE: "bookstackapp"
      MYSQL_USER: "bookstack"
      MYSQL_PASSWORD: "change-this-db-password"
    volumes:
      - ./data/mariadb:/config
    networks:
      - bookstack-net

  bookstack:
    image: lscr.io/linuxserver/bookstack:latest
    container_name: bookstack
    restart: unless-stopped
    depends_on:
      - bookstack-db
    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: Asia/Shanghai
      APP_URL: "http://localhost:6875"
      APP_KEY: "base64:replace-with-generated-app-key"
      APP_DEBUG: "false"
      DB_HOST: "bookstack-db"
      DB_PORT: "3306"
      DB_DATABASE: "bookstackapp"
      DB_USERNAME: "bookstack"
      DB_PASSWORD: "change-this-db-password"
    volumes:
      - ./data/bookstack:/config
    ports:
      - "6875:80"
    networks:
      - bookstack-net

networks:
  bookstack-net:
    driver: bridge
```

启动：

```bash
docker compose up -d
```

查看日志：

```bash
docker compose logs -f bookstack
docker compose logs -f bookstack-db
```

访问：

```text
http://localhost:6875
```

## APP_KEY 要求

生产环境的 `APP_KEY` 一旦投入使用，不要随便更换。更换会导致已加密的数据无法解密。

可以用临时容器生成：

```bash
docker run --rm lscr.io/linuxserver/bookstack:latest php artisan key:generate --show
```

如果镜像不支持直接执行该命令，也可以在开发环境中生成一个 Laravel key，再复制到生产环境：

```bash
cd /app/BookStack
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src exec app php artisan key:generate --show
```

## 方式二：源码部署

适合场景：

```text
1. 你要部署自己改过的 PHP、Blade、JS、Sass 源码。
2. 你希望服务器直接从自己的 GitHub 仓库拉代码。
3. 你能维护 PHP、Composer、Node、Web Server、MySQL。
```

服务器依赖：

```text
PHP 8.3+
Composer
Node.js 22+
MySQL 8.4 或 MariaDB
Apache 或 Nginx
```

部署流程示例：

```bash
cd /app
git clone git@github.com:heliangb46/BookStack.git bookstack-prod-src
cd /app/bookstack-prod-src
git checkout main
cp .env.example .env
```

编辑 `.env`，至少设置：

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=http://your-domain-or-ip
APP_KEY=base64:replace-with-generated-app-key

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=bookstackapp
DB_USERNAME=bookstack
DB_PASSWORD=change-this-db-password
```

安装依赖并构建资源：

```bash
composer install --no-dev --optimize-autoloader
npm ci
npm run production
php artisan migrate --force
php artisan cache:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

Web 根目录必须指向：

```text
/app/bookstack-prod-src/public
```

需要给 Web 用户写权限的目录：

```bash
chown -R www-data:www-data storage bootstrap/cache public/uploads
```

## 方式三：自定义镜像部署

适合场景：

```text
1. 你想把自己 GitHub main 分支构建成固定镜像。
2. 生产服务器只拉镜像，不直接保留源码构建工具。
3. 你准备做 CI/CD。
```

基本思路：

```text
1. 从自己的 main 分支构建镜像。
2. 镜像内执行 composer install --no-dev。
3. 镜像内执行 npm ci && npm run production。
4. 容器启动时连接外部 MySQL/MariaDB。
5. storage/uploads 和数据库必须持久化。
```

这种方式需要单独维护生产 Dockerfile。项目自带的 `dev/docker/Dockerfile` 是开发镜像，不建议直接作为生产镜像。

## 数据持久化

生产环境必须持久化：

```text
数据库数据
BookStack 上传文件
BookStack 配置目录或 .env
APP_KEY
```

如果使用 LinuxServer 镜像，主要持久化路径是：

```text
/app/docker/bookstack/data/mariadb
/app/docker/bookstack/data/bookstack
```

如果使用源码部署，至少要备份：

```text
MySQL/MariaDB 数据库
storage/
public/uploads/
.env
```

## 备份

数据库备份示例：

```bash
docker exec bookstack-db mariadb-dump \
  -ubookstack \
  -p'change-this-db-password' \
  bookstackapp > bookstack-db-$(date +%F).sql
```

文件备份示例：

```bash
tar -czf bookstack-files-$(date +%F).tar.gz /app/docker/bookstack/data/bookstack
```

恢复前先停服务：

```bash
docker compose down
```

恢复后再启动：

```bash
docker compose up -d
```

## 升级

LinuxServer 镜像升级：

```bash
cd /app/docker/bookstack
docker compose pull
docker compose up -d
docker compose logs -f bookstack
```

源码部署升级：

```bash
cd /app/bookstack-prod-src
git fetch origin
git checkout main
git pull --ff-only
composer install --no-dev --optimize-autoloader
npm ci
npm run production
php artisan migrate --force
php artisan cache:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

升级前必须先备份数据库和上传文件。

## 生产安全注意事项

生产环境建议：

```text
APP_DEBUG=false
使用稳定 APP_KEY，且不要泄漏
数据库密码使用强密码
不要暴露 MySQL 端口到公网
不要启用 Xdebug
不要运行 npm watch
不要使用开发 compose 的 MailHog
反向代理层配置 HTTPS
定期备份数据库和上传文件
```

## 当前本机环境关系

本机现在可以同时存在两套环境：

```text
http://localhost:6875  现有 BookStack 容器环境，偏部署/使用
http://localhost:6876  源码开发调试环境，偏二次开发
```

二次开发时改 `/app/BookStack`。确认效果后提交到自己的 GitHub `main` 分支，再按本文档选择生产部署方式。
