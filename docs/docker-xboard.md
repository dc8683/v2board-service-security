# docker xboard 编排快速部署使用

本文主要介绍在现有的 xboard docker compose 下，增加 security 服务的编排方式

可以和现有的 xboard 面板服务使用局域网无缝对接，关闭 xboard 面板对外端口暴露， 只保留 security 服务的端口暴露，避免 xboard 面板被外网攻击

## 1. 下载 security 镜像并导入到 docker

可以从 [release](https://github.com/dc8683/v2board-service-security/releases) 中下载 docker 产物，文件名称为 `airbuddy-security-docker-image-版本号.tar`，也可以在服务器终端使用以下命令直接下载

```bash
  cd /www/wwwroot/security # 进入你希望存放镜像的目录
```

如果不想使用命令，可以在宝塔中打开 docker - 本地镜像，点击导入镜像，选择刚才上传的 `airbuddy-security-docker-image-版本号.tar` 文件，此时镜像列表中会多出一个 `airbuddy-security:版本号` 镜像，如图所示:

![](https://github.com/dc8683/picx-images-hosting/raw/master/docs/Clipboard---2025-06-17-16.34.38.2obsamib0p.webp)

## 2. 现有 xboard docker compose 加入 security 的编排

再修改现有的 xboard docker compose 之前，进入到 xb 目录下的 `compose.yml` 文件所在目录，在终端中执行 `docker compose down` 命令来停止现有的 xboard 服务，然后打开 `compose.yml` 文件，添加以下内容到 `services` 下:

```yaml
### 原有内容 ###
services:
  web:
    image: ghcr.io/cedar2025/xboard:new
    volumes:
      - ./.docker/.data/redis/:/data/
      - ./.env:/www/.env
      - ./.docker/.data/:/www/.docker/.data
      - ./storage/logs:/www/storage/logs
      - ./storage/theme:/www/storage/theme
      - ./plugins:/www/plugins
    environment:
      - docker=true
    depends_on:
      - redis
    # network_mode: host
    command: php artisan octane:start --host=0.0.0.0 --port=7001
    restart: on-failure
    ports:
      - 7001:7001
  horizon:
    image: ghcr.io/cedar2025/xboard:new
    volumes:
      - ./.docker/.data/redis/:/data/
      - ./.env:/www/.env
      - ./.docker/.data/:/www/.docker/.data
      - ./storage/logs:/www/storage/logs
      - ./plugins:/www/plugins
    restart: on-failure
    # network_mode: host
    command: php artisan horizon
    depends_on:
      - redis
  redis:
    image: redis:7-alpine
    command: redis-server --unixsocket /data/redis.sock --unixsocketperm 777 --save 900 1 --save 300 10 --save 60 10000
    restart: unless-stopped
    volumes:
      - ./.docker/.data/redis:/data
  ################################### 新增 security 服务 ###################################
  airbuddy-security-service:
    # 替换为你实际的镜像名称和标签
    image: airbuddy-security:1.0.1
    container_name: airbuddy-security-service
    restart: always
    ports:
      # 将容器内部的 3000 端口映射到主机的 12020 端口
      - "12020:3000"
    environment:
      # 以下是环境变量配置，请根据实际情况修改
      - ADMIN_EMAIL=${ADMIN_EMAIL}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - BACKEND_API_PREFIX=${BACKEND_API_PREFIX}
      - DOMAIN=${DOMAIN}
      - PASSWORD=${PASSWORD}
      - MAIL_HOST=${MAIL_HOST}
      - MAIL_PORT=${MAIL_PORT}
      - MAIL_SECURE=${MAIL_SECURE}
      - MAIL_USER=${MAIL_USER}
      - MAIL_PASS=${MAIL_PASS}
      - MAIL_NEWUSER_SUBJECT=${MAIL_NEWUSER_SUBJECT}
      - MAIL_NEWUSER_URL=${MAIL_NEWUSER_URL}
```

## 3. 添加反向代理站点

服务启动后，在网站 - 反向代理，点击添加反代，域名设置你对外公开的域名，目标 url 填写本机地址 + compose 中的端口号，例如上面compose 对应的端口示例: http://127.0.0.1:12020 , 名称可以随意填写，例如 `airbuddy-security`，然后点击提交即可，如下所示:

![](https://github.com/dc8683/picx-images-hosting/raw/master/docs/fandai.4n7z15bffe.webp)

## 4.查看 security 服务状态

在浏览器中访问 `http://<your-domain>/status`，此页面会显示 security 服务各个模块的运行状态