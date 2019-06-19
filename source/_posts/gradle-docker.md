---
title: docker 结合 spring boot 项目运行
categories:
  - 技术
tags:
  - docker
abbrlink: '26e25788'
date: 2019-06-19 12:04:15
---

这两天，想把自己的项目放到 docker 容器中运行，按照下面的步骤，你也能成功运行起来。这里是基于 gradle 配置的，maven 配置类似，如果是 maven 配置的也可以参考网上。首先看下 docker 常用的命令
# 命令
## 镜像相关
### 镜像仓库相关
1. 查找镜像

> docker search 【条件】

```
## 查询 3 颗星及以上且名字包含 mysql 的镜像
docker search -f=stars=3 mysql
```

2. 获取镜像

> docker pull 【仓库】:【tag】

仓库格式为 [仓库url]/[用户名]/[应用名] , 除了官方仓库外的第三方仓库要指定 url, 用户名就是在对应仓库下建立的账户, 一般只有应用名的仓库代表 官方镜像, 如 ubuntu、tomcat 等, 而 tag 表示镜像的版本号, 不指定时默认为 latest。

```
# 获取alpine Linux 的镜像
docker pull alpine
```

3. 推送镜像到仓库

> docker push [镜像名]:[tag]

### 本地镜像
1. 查看本地镜像

> docker images

2. 删除本地镜像

> docker rmi 【镜像名 or 镜像 id】

3. 查看镜像详情

> docker inspect 【镜像名 or 镜像 id】

4. 打包本地镜像，使用压缩包来完成迁移

> docker save 【镜像名】>【文件路径】

```
# 默认为文件流输出
docker save alpine > /usr/anyesu/docker/alpine.img

# 或者使用 '-o' 选项指定输出文件路径
docker save -o /usr/anyesu/docker/alpine.img alpine
```

5. 导入镜像压缩包

> docker load < 【文件路径】

```
# 默认从标准输入读取
ubuntu@VM-84-201-ubuntu:~$ docker load < /usr/anyesu/docker/alpine.img
3fb66f713c9f: Loading layer [==================================================>]  4.221MB/4.221MB
Loaded image: alpine:latest

# 用 '-i' 选项指定输入文件路径
ubuntu@VM-84-201-ubuntu:~$ docker load -i /usr/anyesu/docker/alpine.img
Loaded image: alpine:latest
Loaded image ID: sha256:665ffb03bfaea7d8b7472edc0a741b429267db249b1fcead457886e861eae25f
Loaded image ID: sha256:a41a7446062d197dd4b21b38122dcc7b2399deb0750c4110925a7dd37c80f118
```

6. 修改镜像 tag

> docker tag [ 镜像名 or 镜像 id ] [ 新镜像名 ]:[ 新 tag ]

```
docker tag a41 anyesu/alpine:1.0
```

## 容器相关
1. 创建、启动容器并执行相应的命令

> docker run [ 参数 ] [ 镜像名 or 镜像 id ] [ 命令 ]

run 命令常用选项：


选项 | 说明
---|---
-d | 后台运行容器，并返回容器 ID；不指定时，启动后开始打印日志，ctrl + c 退出命令同时会关闭容器
-i | 以交互模式运行容器，通常与 -t 同时使用
-t | 为容器重新分配一个伪输入终端，通常与 -i 同时使用
--name 【name】 | 为容器指定一个别名，不指定时随机生成
-h 【name】 | 设置容器的主机名，默认随机生成
-dns 【dns】 | 指定容器使用的 DNS 服务器，默认和宿主机一致
-e 【environment】 | 设置环境变量
-cpuset="0-2" or -cpu="0,1,2" | 绑定容器到指定 CPU 运行
-m 100M | 指定容器使用内存最大值
--net bridge | 指定容器的网络连接类型，支持 bridge / host / none / container 四种类型
-ip 【ip】 | 为容器分配固定 ip，需要使用自定义网络
--expose 8081 --expose 8082 | 开放一个或一组端口，会覆盖镜像中开放的端口
-p [宿主机端口]:[容器内端口] | 宿主机到容器的端口映射, 可指定宿主机的要监听的 ip, 默认为 0.0.0.0
-P | 注意是大写的, 宿主机随机指定一组可用的端口映射容器 expose 的所有端口
-v [宿主机目录路径]:[容器内目录路径] | 挂载宿主机的指定目录 ( 或文件 ) 到容器内的指定目录 ( 或文件 )
--add-host [主机名]:[ip] | 为容器 hosts 文件追加 host , 默认会在 hosts 文件最后追加内容：[主机名]:[容器ip]
--volumes-from [其他容器名] | 将其他容器的数据卷添加到此容器
--link [其他容器名]:[在该容器中的别名] | 添加链接到另一个容器, 在本容器 hosts 文件中加入关联容器的记录, 效果类似于 --add-host

> 单字符选项可以合并, 如 -i -t 可以合并为 -it

2. 查看运行中的容器

> docker ps

加 -a 选项可以查看所有的容器

3. 开启/停止/重启容器

> # 关闭容器(发送SIGTERM信号,做一些'退出前工作',再发送SIGKILL信号)
docker stop anyesu-container
# 强制关闭容器(默认发送SIGKILL信号, 加-s参数可以发送其他信号)
docker kill anyesu-container
# 启动容器
docker start anyesu-container
# 重启容器
docker restart anyesu-container

4. 删除容器

> docker rm [ 容器名 or 容器 id ]

5. 查看容器详情

> docker inspect [ 容器名 or 容器 id ]

6. 查看容器中正在运行的进程
 
> docker top [ 容器名 or 容器 id ]

7. 将容器保存为镜像

> docker commit [ 容器名 or 容器 id ] [ 镜像名 ]:[ tag ]

8. 使用 Dockerfile 构建镜像

> docker build -t [ 镜像名 ]:[ tag ] -f [ DockerFile 名 ] [ DockerFile 所在目录 ]

# 配置
下面是我项目中的配置，可以作为参考

gradle 配置：

```
// 放在最上面，优先级比较高
buildscript {
    repositories {
        maven {
            url "https://plugins.gradle.org/m2/"
        }
    }
    dependencies {
        classpath "gradle.plugin.com.arenagod.gradle:mybatis-generator-plugin:1.4"
        classpath "org.springframework.boot:spring-boot-gradle-plugin:2.1.3.RELEASE"
    }
}

plugins {
    id 'java'
    id "com.arenagod.gradle.MybatisGenerator" version "1.4"
    id "org.sglahn.gradle-dockerfile-plugin" version "0.4"
}

apply plugin: 'com.arenagod.gradle.MybatisGenerator'
apply plugin: 'jacoco'
apply plugin: 'java'
apply plugin: 'groovy'
apply plugin: 'org.springframework.boot'

group 'com.dxy'
version '1.0'

jar {
    baseName = 'turbo'
    version = '1.0'
}

sourceCompatibility = 1.8

repositories {
    maven {
        url "http://maven.aliyun.com/nexus/content/groups/public/"
    }
    dependencies {

    }
    mavenCentral()
}

sourceSets {
    main {
        java {
            srcDirs = []
        }
        groovy {
            srcDirs = ['src/main/groovy', 'src/main/java']
        }
        test {
            java {
                srcDirs = []
            }
            groovy {
                srcDirs = ['src/test/groovy', 'src/test/java']
            }
        }
    }
}

sourceCompatibility = 1.8
targetCompatibility = 1.8

dependencies {
    compile group: 'org.springframework.boot', name: 'spring-boot-starter-web', version: '2.1.3.RELEASE'
    compile group: 'org.mybatis.spring.boot', name: 'mybatis-spring-boot-starter', version: '2.0.0'
    compile group: 'org.springframework.boot', name: 'spring-boot-starter-data-jpa', version: '2.1.3.RELEASE'
    testCompile group: 'org.springframework.boot', name: 'spring-boot-starter-test', version: '2.1.3.RELEASE'
    compile group: 'com.alibaba', name: 'druid-spring-boot-starter', version: '1.1.14'
    compile group: 'org.projectlombok', name: 'lombok', version: '1.18.4'
    compile group: 'mysql', name: 'mysql-connector-java', version: '5.1.47'
    compile group: 'io.springfox', name: 'springfox-swagger2', version: '2.9.2'
    compile group: 'io.springfox', name: 'springfox-swagger-ui', version: '2.9.2'
    // mybatis-generator core 包
    compile group: 'org.mybatis.generator', name: 'mybatis-generator-core', version: '1.3.7'
    compile group: 'com.alibaba', name: 'fastjson', version: '1.2.56'
    compile group: 'com.google.code.gson', name: 'gson', version: '2.8.5'
    compile group: 'org.apache.shiro', name: 'shiro-core', version: '1.4.0'
    compile group: 'org.apache.shiro', name: 'shiro-spring', version: '1.4.0'
    compile group: 'org.codehaus.groovy', name: 'groovy-all', version: '2.5.6'
    // compile group: 'org.redisson', name: 'redisson-spring-boot-starter', version: '3.10.5'
}

configurations {
    mybatisGenerator
}
// mybatis-generator.xml 配置路径
mybatisGenerator {
    verbose = true
    configFile = './src/main/resources/generatorConfig.xml'
}

//configuration
docker {
    // Image version. Optional, default = project.version
    //imageVersion = version
    // Image name. Optional, default = project.name
    imageName = 'turbo_docker'
    // Docker repository. Optional, default == no repository
    // dockerRepository = 'sglahn'
    // Path or URL referring to the build context. Optional, default = ${project.projectDir.getAbsolutePath()}
    // buildContext = 'build-context'
    // Path to the Dockerfile to use (relative to ${project.projectDir}). Optional, default = ${buildContext}/Dockerfile
    dockerFile = 'Dockerfile'
    // Add a list of tags for an image. Optional, default = 'latest'
    //tags = [version, 'latest', 'Hello']
    // Set metadata for an image. Optional, default = no label applied
    //labels = ['branch=master', 'mylabel=test']
    // name and value of a buildarg. Optional, default = no build arguments
    //buildArgs = ['http_proxy="http://some.proxy.url"']
    // Always remove intermediate containers, even after unsuccessful builds. Optional, default = false
    removeIntermediateContainers = true
    // Isolation specifies the type of isolation technology used by containers. Optional, default = default
    //isolation = 'default'
    // Do not use cache when building the image. Optional, default = false
    //noCache = true
    // Always attempt to pull a newer version of the image. Optional, default false
    //pull = true
    // Suppress the build output and print image ID on success. Optional, default = true
    quiet = false
    // Remove image in local repository after push to a remote repository, useful for builds on CI agents. Optional, default = false
    //removeImagesAfterPush = true
}
```

执行命令：
```
brew info gradle;
brew upgrade gradle;
// 如果报错，可以使用 gradle build dockerBuild 等命令
gradle build docker
// 运行命令，将容器内的  8080 端口映射到 127.0.0.1 的 8087 端口
docker run -p 127.0.0.1:8087:8080 -t turbo_docker
```

> 注意：需要主要项目中的端口号的映射，容器里的端口号和你实际运行的端口号不是一个概念，需要手动设置一下。

# 参考
* [gradle 集成 dockerFile](https://www.jianshu.com/p/3cd9dbc165e9)
* [通过Gradle使用Docker部署 Spring Boot项目](https://www.jianshu.com/p/7571fa3b394c)
* [docker 常用命令](https://www.jianshu.com/p/7c9e2247cfbd)