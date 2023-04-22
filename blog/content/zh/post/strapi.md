---
title: "Strapi"
date: 2023-04-20T19:10:08+08:00
draft: false
---

# Strapi Docker 镜像部署 到 阿里云 Serverless

[使用 Docker 模板](https://github.com/VinerFiner/start-docker-project)

[![Deploy with Severless Devs](https://img.alicdn.com/imgextra/i1/O1CN01w5RFbX1v45s8TIXPz_!!6000000006118-55-tps-95-28.svg)](https://fcnext.console.aliyun.com/applications/create?template=start-docker-project)

- 修改应用名称
- 配置镜像
- 提交部署

## 使用 官方 cli 脚手架创建项目

- 使用 sqlite

```
cd code/project

npx create-strapi-app@latest strapi-api --quickstart
```

## 修改 Dockerfile 文件

### 开发者模式

- 添加阿里源

> `sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories`

- 创建 `Dockerfile`， `vim Dockerfile`

```shell
# fc 云函数流水线
FROM gcr.io/kaniko-project/executor:debug AS kaniko

FROM node:16-alpine as build
# Installing libvips-dev for sharp Compatibility
# 配置国内源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
RUN apk update && apk add --no-cache build-base gcc autoconf automake zlib-dev libpng-dev vips-dev && rm -rf /var/cache/apk/* > /dev/null 2>&1
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}
WORKDIR /opt/
COPY ./package.json ./package-lock.json ./
ENV PATH /opt/node_modules/.bin:$PATH
RUN npm install --production
WORKDIR /opt/app
COPY ./ .
RUN npm run build

FROM node:16-alpine
# Installing libvips-dev for sharp Compatibility
RUN apk add vips-dev
RUN rm -rf /var/cache/apk/*
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}
# 工作目录
# WORKDIR /opt/app
WORKDIR /data/strapi
COPY --from=build /opt/node_modules ./node_modules
ENV PATH /opt/node_modules/.bin:$PATH
COPY --from=build /opt/app ./
EXPOSE 1337
CMD ["npm", "run","start"]
```

### 修改 s 文件

`vim s.yaml`

- 添加环境变量

```shell
  functionName: "strapi"
  environmentVariables: #  为函数配置环境变量
    ADMIN_JWT_SECRET: ${env(ADMIN_JWT_SECRET)}
    API_TOKEN_SALT: ${env(API_TOKEN_SALT)}
    APP_KEYS: ${env(APP_KEYS)}
    TRANSFER_TOKEN_SALT: ${env(TRANSFER_TOKEN_SALT)}
    JWT_SECRET: ${env(JWT_SECRET)}
    DATABASE_CLIENT: ${env(DATABASE_CLIENT)}
    DATABASE_FILENAME: ${env(DATABASE_FILENAME)}
```

- 修改镜像构建上下文

```shell
    actions: # 自定义执行逻辑，关于actions 的使用，可以参考：https://www.serverless-devs.com/serverless-devs/yaml#行为描述
      pre-deploy: # 在deploy之前运行
        - component: fc build --use-docker --dockerfile ./code/project/Dockerfile --context ./code/project/strapi-api
```

- 修改配置以及端口号

```shell
      function:
        name: ${vars.functionName} # 函数名
        description: 'project  Function'
        timeout: 60
        memorySize: 256
        cpu: 0.2
        diskSize: 512
        instanceConcurrency: 12
        caPort: 1337
```

## 提交到代码仓库，配置环境变量就可以了

### END