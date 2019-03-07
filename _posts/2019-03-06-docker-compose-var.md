---
layout: post
title: Docker Compose 引用环境变量
categories: docker
description: 在项目中，往往需要在 docker-compose.yml 文件中使用环境变量来控制不同的条件和使用场景。本文集中介绍 docker compose 引用环境变量的方式。
keywords: docker
---

> 说明：本文的演示环境为 ubuntu 16.04。

# Compose CLI 与环境变量

Compose CLI(compose command-line 即 docker-compose 程序)能够识别名称为 COMPOSE_PROJECT_NAME 和 COMPOSE_FILE 等环境变量(具体支持的环境变量请参考这里)。比如我们可以通过这两个环境变量为 docker-compose 指定 project 的名称和配置文件：

```bash
$ export COMPOSE_PROJECT_NAME=TestVar
$ export COMPOSE_FILE=~/projects/composecounter/docker-compose.yml
```

![ ](/images/20190306-docker-compose-var-01.png)

然后启动应用，显示的 project 名称都是我们在环境变量中指定的：

![ ](/images/20190306-docker-compose-var-02.png)

如果设置了环境变量的同时又指定了命令行选项，那么会应用命令行选项的设置：

```$ docker-compose -p nickproject up -d```

![ ](/images/20190306-docker-compose-var-03.png)

# 在 compose file 中引用环境变量

我们还可以在 compose file 中直接引用环境变量，比如下面的 demo：

```YAML
version: '3'
  services:
    web:
      image: ${IMAGETAG}
      ports:
       - "5000:5000"
    redis:
      image: "redis:alpine"
```

我们通过环境变量 ${IMAGETAG} 指定了 web 的镜像，下面通过 export 的方式来为 compose 配置文件中的环境变量传值：

![ ](/images/20190306-docker-compose-var-04.png)

注意，如果对应的环境变量没有被设置，那么 compose 就会把它替换为一个空字符串：

![ ](/images/20190306-docker-compose-var-05.png)

碰到这种情况，我们可以在 compose 的配置文件中为该变量设置一个默认值：

```YAML
version: '3'
services:
  web:
    image: ${IMAGETAG:-defaultwebimage}
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
```

这样，如果没有设置 IMAGETAG 变量，就会应用 defaultwebimage：

![ ](/images/20190306-docker-compose-var-06.png)

除了这种方式，我们还可以通过后面将介绍的 .env 文件来为环境变量设置默认值。

# 把环境变量传递给容器

先来看一下在 compose file 中如何为容器设置环境变量：

```YAML
web:
  environment:
    DEBUG: 1
```

compose file 中的 environment 节点用来为容器设置环境变量，上面的写法等同于：

```$ docker run -e DEBUG=1```

要把当前 shell 环境变量的值传递给容器的环境变量也很简单，去掉上面代码中的赋值部分就可以了：

```YAML
web:
  environment:
    DEBUG:
```

这种情况下，如果没有在当前的 shell 中导出环境变量 DEBUG，compose file 中会把它解释为 null：

![ ](/images/20190306-docker-compose-var-07.png)

在试试导出环境变量 DEBUG 的情况：

```$ export DEBUG=1```

![ ](/images/20190306-docker-compose-var-08.png)

这才是我们设计的正确的使用场景！

#使用文件为容器设置多个环境变量

如果觉得通过 environment 为容器设置环境变量不够过瘾，我们还可以像 docker -run 的 --env-file 参数一样通过文件为容器设置环境变量：

```YAML
web:
  env_file:
    - web-variables.env
```

注意，web-variables.env 文件的路径是相对于 docker-compose.yml 文件的相对路径。上面的代码效果与下面的代码相同：

```$ docker run --env-file=web-variables.env```

web-variables.env 文件中可以定义一个或多个环境变量：

```YAML
# define web container env
APPNAME=helloworld
AUTHOR=Nick Li
VERSION=1.0
```

检查下结果：

![ ](/images/20190306-docker-compose-var-09.png)

原来 compose 把 env_file 的设置翻译成了 environment！

# .env 文件

当我们在 docker-compose.yml 文件中引用了大量的环境变量时，对每个环境变量都设置默认值将是繁琐的，并且也会影响 docker-compose.yml 简洁程度。此时我们可以通过 .env 文件来为 docker-compose.yml 文件引用的所有环境变量设置默认值！

修改 docker-compose.yml 文件的内容如下：

```YAML
version: '3'
services:
  web:
    image: ${IMAGETAG}
    environment:
      APPNAME:
      AUTHOR:
      VERSION:
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
```

然后在相同的目录下创建 .env 文件，编辑其内容如下：

```YAML
# define env var default value.
IMAGETAG=defaultwebimage
APPNAME=default app name
AUTHOR=default author name
VERSION=default version is 1.0
```

检查下结果，此时所有的环境变量都显示为 .env 文件中定义的默认值：

![ ](/images/20190306-docker-compose-var-10.png)

# 配置不同场景下的环境变量

从前面的部分中我们可以看到，docker compose 提供了足够的灵活性来让我们设置 docker-compose.yml 文件中引用的环境变量，它们的优先级如下：

1. Compose file
2. Shell environment variables
3. Environment file
4. Dockerfile
5. Variable is not defined

首先，在 docker-compose.yml 文件中直接设置的值优先级是最高的。
然后是在当前 shell 中 export 的环境变量值。
接下来是在环境变量文件中定义的值。
再接下来是在 Dockerfile 中定义的值。
最后还没有找到相关的环境变量就认为该环境变量没有被定义。

根据上面的优先级定义，我们可以把不同场景下的环境变量定义在不同的 shell 脚本中并导出，然后在执行 docker-compose 命令前先执行 source 命令把 shell 脚本中定义的环境变量导出到当前的 shell 中。通过这样的方式可以减少维护环境变量的地方，下面的例子中我们分别在 docker-compose.yml 文件所在的目录创建 test.sh 和 prod.sh，test.sh 的内容如下：

```bash
#!/bin/bash
# define env var default value.
export IMAGETAG=web:v1
export APPNAME=HelloWorld
export AUTHOR=Nick Li
export VERSION=1.0
```

prod.sh 的内容如下：

```bash
#!/bin/bash
# define env var default value.
export IMAGETAG=webpord:v1
export APPNAME=HelloWorldProd
export AUTHOR=Nick Li
export VERSION=1.0LTS
```

在测试环境下，执行下面的命令：

```bash
$ source test.sh
$ docker-compose config
```

![ ](/images/20190306-docker-compose-var-11.png)

此时 docker-compose.yml 中的环境变量应用的都是测试环境相关的设置。

而在生产环境下，执行下面的命令：

```bash
$ source prod.sh
$ docker-compose config
```

![ ](/images/20190306-docker-compose-var-12.png)

此时 docker-compose.yml 中的环境变量应用的都是生产环境相关的设置。

# 总结

docker compose 对环境变量的使用提供了非常丰富支持和灵活的使用方式。希望通过本文的总结可以帮助大家理清相关的用法，并能够以简洁的方式为不同的使用场景提供支持。

> 原文链接：<https://www.cnblogs.com/sparkdev/p/9826520.html>