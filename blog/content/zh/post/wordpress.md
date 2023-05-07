---
title: "Wordpress"
date: 2023-05-07T22:35:05+08:00
draft: false
---

## 阿里云函数镜像部署 WordPress

[使用 Docker 模板](https://github.com/VinerFiner/start-docker-project)

[![Deploy with Severless Devs](https://img.alicdn.com/imgextra/i1/O1CN01w5RFbX1v45s8TIXPz_!!6000000006118-55-tps-95-28.svg)](https://fcnext.console.aliyun.com/applications/create?template=start-docker-project)

- 修改应用名称

- 配置镜像

> 镜像仓库地址，需要在 https://cr.console.aliyun.com/ 中开通服务、创建仓库以及设置访问凭证

- 部署完成之后我们可以看到一个简单的 Nginx 页面

## 修改为 WordPress 应用

```shell
FROM wordpress:latest

## 复制文件
# COPY index.html /usr/share/nginx/html/index.html
# COPY index.html /mnt/index.html

# 映射文件
ENV PERSISTENCE_NAS /mnt/auto/wordpress
RUN set -eux; \
	if [ ! -d  $PERSISTENCE_NAS ]; then \
		mkdir -p $PERSISTENCE_NAS; \
		cp -r /var/www/html $PERSISTENCE_NAS; \
		rm -rf /var/www/html; \
		echo "create $PERSISTENCE_NAS"; \
	else \
		echo -e "directory already exists"; \
	fi; \
	\
	ln -sf $PERSISTENCE_NAS /var/www/html
# END
```

## 修改云函数配置

- 添加 `nasConfig`
- 添加环境变量

```yaml

edition: 1.0.0 #  命令行YAML规范版本，遵循语义化版本（Semantic Versioning）规范
name: website-wordpress
access: "default" #  秘钥别名

vars:
  region: "ap-southeast-1" # 这是地域
  service:
    name: "website-wordpress"
    description: 'project by Serverless Devs'
    role: 'acs:ram::${config(AccountID)}:role/AliyunFcDefaultRole'
    nasConfig: auto # 开启 NAS挂载
  functionName: "wordpress"
  environmentVariables: #  为函数配置环境变量
    App_Version: "0.0.1"
    WORDPRESS_DB_HOST: ${env(WORDPRESS_DB_HOST)} # 数据库 host
    WORDPRESS_DB_NAME: ${env(WORDPRESS_DB_NAME)} # 数据库 name
    WORDPRESS_DB_USER: ${env(WORDPRESS_DB_USER)} # 数据库 user
    WORDPRESS_DB_PASSWORD: ${env(WORDPRESS_DB_PASSWORD)}
    TZ: "Asia/Shanghai"
  image: ${env(imageurl)}
```
- 修改内存大小 字段 `memorySize`
- 修改 cpu大小 字段 `cpu`

```yaml
  # 项目
  project-function: #  服务名称
    component: fc # 组件名称，
    actions: # 自定义执行逻辑，关于actions 的使用，可以参考：https://www.serverless-devs.com/serverless-devs/yaml#行为描述
      pre-deploy: # 在deploy之前运行
        - component: fc build --use-docker --dockerfile ./code/project/Dockerfile
            --context ./code/project
      # post-deploy: # 初始化 nas
      #   - component: fc invoke --service-name ${fc-nas-init.output.service.name} --function-name nas-init
    props: # 组件的属性值
      region: ${vars.region}
      service: ${vars.service}
      function:
        name: ${vars.functionName} # 函数名
        description: 'project  Function'
        timeout: 60
        memorySize: 512 # 内存大小
        cpu: 0.35 # CPU大小
        diskSize: 512
        instanceConcurrency: 12
        caPort: 80
        environmentVariables: ${vars.environmentVariables} #  为函数配置环境变量
        runtime: custom-container
        customContainerConfig:
          image: ${vars.image}
        codeUri: './code/project'
      triggers:
        - name: httpTrigger
          type: http
          config:
            authType: anonymous
            methods:
              - GET
              - POST
              - PUT
              - DELETE
              - HEAD
              - OPTIONS
```

## 修改 nas-init应用

```python
# coding=utf-8
import os


def handler(event, context):
    if not os.path.exists("/mnt/auto/wordpress"):
        os.system(
            "mkdir -p /mnt/auto/wordpress && cd -")
    return "nas init"

```

## 重新部署