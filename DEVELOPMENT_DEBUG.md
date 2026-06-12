# BookStack 二次开发与调试指南

本文档面向二次开发，不是生产部署说明。BookStack 是 Laravel/PHP 后端加 Node/Sass/ESBuild 前端构建，不是 Spring Boot 那种单个 JVM 进程，但可以做到类似的断点调试、热构建和日志跟踪。

## 当前推荐方式

推荐使用项目自带的开发 Docker Compose 作为运行环境：

```bash
cd /app/BookStack
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src up -d
```

访问地址：

```text
BookStack: http://localhost:6876
MailHog:   http://localhost:8026
```

默认账号：

```text
admin@admin.com
password
```

这不是把项目“部署成容器”来开发。开发 compose 会把当前源码目录挂载到容器 `/app`，你改 `/app/BookStack` 下的 PHP、Blade、Sass、JS 文件，容器里立即使用这些源码。容器只负责提供 PHP、Apache、MySQL、Node 这些运行依赖。

## 容器说明

开发环境启动后主要有这些容器：

```text
bookstack-src-app-1      PHP 8.3 + Apache + Composer + Xdebug
bookstack-src-db-1       MySQL 8.4 开发库
bookstack-src-node-1     Node 22，监听并构建前端资源
bookstack-src-mailhog-1  邮件调试工具
```

查看状态：

```bash
docker ps --filter 'name=bookstack-src'
```

查看日志：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src logs -f app
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src logs -f node
```

## PHP 后端断点调试

开发镜像已经安装 Xdebug，配置文件在：

```text
dev/docker/php/conf.d/xdebug.ini
```

当前关键配置：

```ini
xdebug.mode=debug
xdebug.client_host=host.docker.internal
xdebug.start_with_request=yes
xdebug.client_port=9090
```

因此 IDE 需要监听 `9090` 端口。VS Code 可安装 `PHP Debug` 扩展，然后创建本地调试配置：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for BookStack Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9090,
      "pathMappings": {
        "/app": "${workspaceFolder}"
      }
    }
  ]
}
```

使用方式：

```text
1. 用 VS Code 打开 /app/BookStack。
2. 启动 Listen for BookStack Xdebug。
3. 在 PHP 控制器、服务、模型中打断点。
4. 浏览器访问 http://localhost:6876 触发请求。
```

如果断点不进，先看 `app` 日志里是否有 Xdebug 连接失败信息：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src logs -f app
```

## Artisan 与 Composer 调试

在 app 容器里运行 artisan：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src exec app php artisan list
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src exec app php artisan route:list
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src exec app php artisan migrate
```

运行 Composer：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src exec app composer install
```

进入容器 shell：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src exec app bash
```

## Node 与前端调试

前端源码主要在：

```text
resources/
dev/build/
public/dist/
```

常规开发不需要手动运行 Node，`bookstack-src-node-1` 会自动执行：

```bash
npm run watch
```

也就是并行执行：

```bash
npm run build:css:watch
npm run build:js:watch
```

如果改了 Sass 或 JS，没有看到效果，先看 node 日志：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src logs -f node
```

如果要调试构建脚本本身，例如 `dev/build/esbuild.mjs`，可以临时停掉默认 node watcher，然后用 inspect 模式启动：

```bash
cd /app/BookStack
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src stop node
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src run --rm -p 9229:9229 --entrypoint sh node
```

进入容器后执行：

```bash
node --inspect=0.0.0.0:9229 dev/build/esbuild.mjs watch
```

VS Code 可用 Node attach 配置：

```json
{
  "name": "Attach BookStack Node Build",
  "type": "node",
  "request": "attach",
  "address": "localhost",
  "port": 9229,
  "localRoot": "${workspaceFolder}",
  "remoteRoot": "/app"
}
```

调试完成后恢复默认 watcher：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src up -d node
```

## 数据库调试

开发数据库在 `bookstack-src-db-1` 容器内，数据保存在 Docker 命名卷：

```text
bookstack-src_db
```

连接 MySQL：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src exec db mysql -ubookstack-test -pbookstack-test bookstack-dev
```

重置开发环境数据库时要谨慎。确认不需要当前开发数据后再执行：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src down -v
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src up -d
```

## Git 工作区建议

WSL 和 Docker 可能会改变文件权限，导致 Git 显示大量没有内容变化的修改。建议本仓库保持：

```bash
git config core.filemode false
```

确认真实内容变化：

```bash
git diff --stat
git diff --name-status
```

提交前检查：

```bash
git status --short
git diff
```

## 常用启动与停止命令

启动：

```bash
cd /app/BookStack
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src up -d
```

停止但保留数据库：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src down
```

停止并删除开发数据库：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src down -v
```

重新构建开发镜像：

```bash
DEV_PORT=6876 DEV_MAIL_PORT=8026 docker compose -p bookstack-src up -d --build
```

## 什么时候不建议用开发 compose

以下情况不要用开发 compose：

```text
1. 正式给别人访问。
2. 保存长期生产数据。
3. 对外暴露公网。
4. 追求稳定升级与备份。
```

正式部署请看 `DEPLOYMENT_GUIDE.md`。
