
Docker自2013年开源到现在已经是一项成熟的“老技术”了。其实早些时候Docker在国内很火的时候就经常看到他的身影，但是作为一个见识过前端娱乐圈波澜壮阔的小前端，对这类新技术是持观望态度的。一是觉得这类新技术很多都只是火一时，不久就会悄无声息的死掉；二是当时我接触到的业务没有使用到他的场景，也没有过多精力去深入看他。但是时至今日，Docker虽然不如刚开始那时如日中天但也依然坚挺，同时我个人也报着积极折腾的心态，来一探Docker在我日常开发中的用武之地。  

于我看来，Docker最大的好处就是不用再折腾环境，方便应用迁移。越来越多的时候搭建一个应用需要很多基础环境，常用的比如`node.js/MySQL/Redis/nginx`等等web常用的环境，这使得我本地开发环境需要把它们都装一遍，云服务器上还要再来一遍，而且可能在不同系统装这些环境会遇到不同的坑。光这么装上两遍也还能接受，但是哪天换了家云服务器，迁移还得再来一遍；最近入手的树莓派上那就更麻烦，arm架构的环境坑多，树莓派系统也经常出问题迫使偶尔需要重装系统...那么，Docker就成了解救我于水火之中的神器了。我只要在各个环境下装一个Docker，再也不用管其他的工具链了。

## Docker相关的基本概念
Docker有几个重要概念得记一记：
- 镜像（Image）: Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。(都是抄的)  
关于镜像，它采用分层存储: 镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。带来的好处是方便复用，省磁盘空间。
- 容器（Container）: 镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。  
每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为 容器存储层。
- volume 数据卷（Volume）: 数据卷 是一个可供一个或多个容器使用的特殊目录,数据卷 的使用，类似于 Linux 下对目录或文件进行 mount。数据卷 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除 数据卷，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的 数据卷。

其他构建Image，启动Container相关的命令和参数，常用的就那么多，需要的时候可随时查，没必要都记住。

## 安装使用
安装指南什么的可以看官网。  
Win10有可执行文件安装，但是得先开启Win10 的 Hyper-V  
一般的linux可以直接apt-get安装，但是需要建立 docker 用户组: `sudo groupadd docker`然后将当前用户加入 docker 组：`sudo usermod -aG docker $USER`.  
树莓派上得用官方的脚本安装: 
```bash
$ curl -fsSL get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh --mirror Aliyun
```
然后建用户组，加用户, systemctl start docker启动就可以了。
再就是得设置下国内的仓库镜像，不然pull镜像的时候贼慢。用 systemd 的linux系统都可以通过编辑`/etc/docker/daemon.json`来设置：
```json
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://reg-mirror.qiniu.com"
  ]
}
```
可以执行`docker info`命令查看加速镜像是否设置成功.

然后就是pull镜像，运行镜像了。我们可以直接用官方或社区的镜像，然后运行：
```bash
# 拉取镜像
docker pull node
# 运行镜像
docker run -idt -p 8080:8888 node /bin/bash
```

### Dockerfile
但是这样就没有完全发挥Docker的优势，这样跑起来的容器不便于后期维护，常规做法是编写 `Dockerfile`来定制化自己的Image。`docker run`命令的众多参数可以卸载这里面。  
下面是一个简单的Node.js应用的`Dockerfile`:
```bash
# 指定基础镜像
FROM node:latest

# 指定工作目录
WORKDIR /usr/src/app

# 复制应用文件 COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY package*.json ./

# 执行命令, 就像 Shell 脚本，不同的是，每一次执行RUN都是在上一层镜像的基础上新建一层
RUN npm install

# Bundle app source
COPY . .

# 对外暴露的接口
EXPOSE 5200

# 启动应用的命令
CMD [ "node", "index.js"]
```
这样就可以直接运行镜像，把我们的应用跑起来了：`docker run -p 8080:5200 -d <your username>/node-web-app`  

### docker-compose
但是，如果我们的服务有依赖服务，比如用了 Redis 做缓存，这时候直接这样是没法运行我们的服务的。这时候就需要docker生态的另一大杀器：`docker-compose`，专门用于需要多个容器相互配合来完成某项任务的情况。  
`docker-compose` 通过 `docker-compose.yml` 模板文件来定义一组相关联的应用容器为一个项目（project）。  

Compose 中有两个重要的概念：

- 服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
- 项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义。

Compose 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。  
以下是一个简单的node.js与redis组合的例子：
```yaml
version: "3"

services:
  web:
    build: ./
    ports:
      - "5200:5200"
    environment:
      - NODE_ENV=development
    # 解决容器的依赖、启动先后的问题
    depends_on:
      - redis
    networks: 
      - webnet
  redis:
    image: redis:latest
    networks: 
      - webnet
  
# 配置容器连接的网络, node与redis在同一网络才能通信
networks: 
  webnet:
```
这样再在项目目录执行`docker-compose up`，我们使用了redis的nodejs应用才真正运行起来，如果想让容器后台运行则加上`-d`参数。  

### 遇到的坑
当我写好`docker-compose.yml`后，运行`docker-compose up`在启动web service 的时候，一直报错：`connect ECONNREFUSED 127.0.0.1:6379`，几经查资料无果，后来先执行`docker-compose build`后再`docker-compose up`才正常启动。看文档里说的`up`命令是会先`build`的，这个问题还是没太弄明白。为避免钻牛角尖，打算后续用熟练了看到文档多了再探究这个问题。


## 参考资料
- [Docker从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/compose/commands.htm)
- [把一个 Node.js web 应用程序给 Docker 化](https://nodejs.org/zh-cn/docs/guides/nodejs-docker-webapp/)
- [Docker and Node.js Best Practices](https://github.com/nodejs/docker-node/blob/master/docs/BestPractices.md)
- [GitHub issue: Fails to connect to redis, when running inside of docker container](https://github.com/luin/ioredis/issues/763)