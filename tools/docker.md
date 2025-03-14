Docker速查表
===



## Docker安装、配置

### 安装

- Rocky下安装

  ```bash
  sudo dnf install -y docker-ce docker-ce-cli containerd.io
  ```

### 配置

建议做的一些配置：

#### 代理配置

目前镜像站全挂了，必须配置代理服务器。

TODO

#### 用户组配置（Linux）

Linux下将个人用户加入到docker组中，这样无需切到root用户也可正常使用docker命令。

#### 自动联想（zsh）

强烈推荐使用zsh下的docker插件，补全能力大大提升

修改~/.zshrc文件，启动docker插件(zsh默认启用了git插件）

```
plugins=(git docker)
```

**效果：**

```bash
➜  ~ docker con[按下tab触发联想]
config     -- Manage Swarm configs
container  -- Manage containers
context    -- Manage contexts
```



## 基本概念

### Docker 镜像

**Docker 镜像** 是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像 **不包含** 任何动态数据，其内容在构建之后也不会被改变。

####  分层存储

因为镜像包含操作系统完整的 `root` 文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 [Union FS](https://en.wikipedia.org/wiki/Union_mount)的技术，将其设计为分层存储的架构。所以严格来说，镜像并非是像一个 `ISO` 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。

### Docker 容器

每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为 **容器存储层**。

容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。

按照 Docker 最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 数据卷（Volume）、或者 绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。

### Docker Registry

镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，**Docker Registry**就是这样的服务。

一个 **Docker Registry** 中可以包含多个 **仓库**（`Repository`）；每个仓库可以包含多个 **标签**（`Tag`）；每个标签对应一个镜像。

以 [Ubuntu 镜像](https://hub.docker.com/_/ubuntu)为例，`ubuntu` 是仓库的名字，其内包含有不同的版本标签，如，`16.04`, `18.04`。我们可以通过 `ubuntu:16.04`，或者 `ubuntu:18.04` 来具体指定所需哪个版本的镜像。如果忽略了标签，比如 `ubuntu`，那将视为 `ubuntu:latest`。

```bash
$ docker images
REPOSITORY              TAG                  IMAGE ID       CREATED         SIZE
hello_docker            1.0                  6e928b6135b5   3 hours ago     15.6MB
frolvlad/alpine-glibc   latest               a40ab1a88ba2   3 weeks ago     15.6MB
node                    20.10.0-alpine3.19   0dc964c4b6e5   15 months ago   136MB
```

`frolvlad/alpine-glibc` 这样的`REPOSITORY`前面的frolvlad为用户名，没有用户名的`REPOSITORY`为官方镜像

#### Docker Registry 公开服务

最常使用的 Registry 公开服务是官方的 [Docker Hub](https://hub.docker.com/)

#### 私有 Docker Registry

除了使用公开服务外，用户还可以在本地搭建私有 Docker Registry。Docker 官方提供了 [Docker Registry](https://hub.docker.com/_/registry/)镜像。



## 使用镜像

#### 获取镜像

从 Docker 镜像仓库获取镜像的命令是 `docker pull`

```bash
$ docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

- Docker 镜像仓库地址：地址的格式一般是 `<域名/IP>[:端口号]`。默认地址是 Docker Hub(`docker.io`)。
- 仓库名：如之前所说，这里的仓库名是两段式名称，即 `<用户名>/<软件名>`。对于 Docker Hub，如果不给出用户名，则默认为 `library`，也就是官方镜像。

比如：

```bash
$ docker pull ubuntu:18.04
```

#### 运行

有了镜像后，我们就能够以这个镜像为基础启动并运行一个容器。以上面的 `ubuntu:18.04` 为例，如果我们打算启动里面的 `bash` 并且进行交互式操作的话，可以执行下面的命令。

```bash
$ docker run -it ubuntu:18.04 bash

root@e7009c6ce357:/# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.1 LTS (Bionic Beaver)"
...
```

- `-it`：这是两个参数，一个是 `-i`：交互式操作，一个是 `-t` 终端。我们这里打算进入 `bash` 执行一些命令并查看返回结果，因此我们需要交互式终端。
- `ubuntu:18.04`：这是指用 `ubuntu:18.04` 镜像为基础来启动容器。
- `bash`：放在镜像名后的是 **命令**，这里我们希望有个交互式 Shell，因此用的是 `bash`。

### 列出镜像

要想列出已经下载下来的镜像，可以使用 `docker image ls` 或`docker images`命令。

```bash
$ docker image ls
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
redis                latest              5f515359c7f8        5 days ago          183 MB
nginx                latest              05a60462f8ba        5 days ago          181 MB
```

#### 虚悬镜像

镜像列表中，有时看到一个特殊的镜像，这个镜像既没有镜像名，也没有标签，均为 `<none>`。：

```bash
<none>               <none>              00285df0df87        5 days ago          342 MB
```

这个镜像原本是有镜像名和标签的，比如原来为 `mongo:3.2`，随着官方镜像维护，发布了新版本后，重新 `docker pull mongo:3.2` 时，`mongo:3.2` 这个镜像名被转移到了新下载的镜像身上，而旧的镜像上的这个名称则被取消，从而成为了 `<none>`。

除了 `docker pull` 可能导致这种情况，`docker build` 也同样可以导致这种现象。由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 `<none>` 的镜像。这类无标签镜像也被称为 **虚悬镜像(dangling image)** ，可以用下面的命令专门显示这类镜像：

一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的。可以用下面的命令批量删除。

```bash
$ docker image prune
```

### 删除本地镜像

如果要删除本地的镜像，可以使用 `docker image rm` 命令(或docker rmi ...)，其格式为：

```bash
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```



## 操作容器

### 新建、启动容器

#### 常规启动

所需要的命令主要为 `docker run`。每次`docker run`都会创建一个新的容器并运行。

**示例:** 下面的命令输出一个 “Hello World”，之后终止容器。

```bash
$ docker run ubuntu:18.04 /bin/echo 'Hello world'
Hello world
```

**示例**:下面的命令则启动一个 bash 终端，允许用户进行交互。

```bash
$ docker run -t -i ubuntu:18.04 /bin/bash
root@af8bae53bdd3:/#
```

其中，`-t` 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， `-i` 则让容器的标准输入保持打开。

#### 启动后台运行的容器

**示例**:下面的命令则启动一个 Nginx容器，并在后台运行。

```bash
docker run --name webserver -d -p 80:80 nginx
```

- `-d` 参数表示以**后台运行（detached 模式）**启动容器
- `-p 80:80` 表示端口映射
- nginx为镜像名称，webserver为容器名称
- 容器是否会长久运行，是和 `docker run` 指定的命令有关，和 `-d` 参数无关。

#### 启动已终止容器

可以利用 `docker ps -a` 命令，查看本地所有的容器

可以利用 `docker container start 容器ID（或容器名称）` 命令（也可以简化为docker start ...），直接将一个已经终止（`exited`）的容器启动运行。

#### 流程说明

容器的核心为所执行的应用程序，所需要的资源都是应用程序运行所必需的。除此之外，并没有其它的资源。可以在伪终端中利用 `ps` 或 `top` 来查看进程信息。

当利用 `docker run` 来创建容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从 [registry](https://vuepress.mirror.docker-practice.com/repository/) 下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止

### 终止容器

可以使用 `docker container stop` 或简化为`docker stop`来终止一个运行中的容器。

此外，当 Docker 容器中指定的应用终结时，容器也自动终止。

例如对于上一章节中只启动了一个终端的容器，用户通过 `exit` 命令或 `Ctrl+d` 来退出终端时，所创建的容器立刻终止。

此外，`docker container restart` 命令会将一个运行态的容器终止，然后再重新启动它。



### 进入容器

在使用 `-d` 参数时，容器启动后会进入后台。

某些时候需要进入容器进行操作，包括使用 `docker attach` 命令或 `docker exec` 命令，==推荐==使用 `docker exec` 命令

#### exec 命令

`docker exec` 后边可以跟多个参数，这里主要说明 `-i` `-t` 参数。

只用 `-i` 参数时，由于没有分配伪终端，界面没有我们熟悉的 Linux 命令提示符，但命令执行结果仍然可以返回。

当 `-i` `-t` 参数一起使用时，则可以看到我们熟悉的 Linux 命令提示符。

```bash
$ docker run -dit ubuntu
$ docker exec -it [容器名称/ID] bash
root@69d137adef7a:/#
```

如果从这个 终端中 exit，不会导致容器的停止。这就是为什么推荐大家使用 `docker exec` 的原因。

#### attach 命令

下面示例如何使用 `docker attach` 命令。

```bash
$ docker run -dit ubuntu
$ docker attach [容器名称/ID]
root@243c32535da7:/#
```

*注意：* 如果从这个 终端中 exit，会导致容器的停止。



### 删除容器

可以使用 `docker container rm` (可简化为docker rm)来删除一个处于终止状态的容器。例如

```bash
$ docker container rm [容器名称/ID]
```

如果要删除一个运行中的容器，可以添加 `-f` 参数。

#### 清理所有处于终止状态的容器

用 `docker container ls -a` 命令可以查看所有已经创建的包括终止状态的容器，如果数量太多要一个个删除可能会很麻烦，用下面的命令可以清理掉所有处于终止状态的容器。

```bash
$ docker container prune
```



## 一些高级功能

### 镜像（容器）的导出和导入

#### 镜像导入导出(推荐)

##### 镜像导出：

docker save -o <导出的文件名> <镜像名称>,如：

```bash
$ docker save -o cm_api_image.tar cm-api:1.0
```

##### 镜像导入

```bash
$ docker load -i cm_api_image.tar 
Loaded image: cm-api:1.0
```

#### 容器导入导出

##### 容器导出

如果要导出本地某个容器，可以使用 `docker export` 命令

```bash
$ docker export [容器名称/容器ID] -o ubuntu.tar
```

##### 容器导入

可以使用 `docker import` 从容器快照文件中再导入为镜像，例如

```bash
$ docker import ubuntu.tar ubuntu:v1.0
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
test/ubuntu         v1.0                9d37a6082e97        About a minute ago   171.3 MB
```

此外，也可以通过指定 URL 或者某个目录来导入，例如:

```bash
$ docker import http://example.com/exampleimage.tgz example/imagerepo
```

### 通过docker commit生成新的镜像

#### 作用

把在容器中的一些文件改动，保存为一个新的容器

#### 示例

1. 创建一个标准Nginx容器

   ```bash
   $ docker run --name webserver -d -p 80:80 nginx
   ```

2. 修改容器中的内容

   ```bash
   $ docker exec -it webserver bash
   root@3729b97e8226:/# echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
   root@3729b97e8226:/# exit
   exit
   ```

   容器里的改动可用diff命令查看
   
    ```bash
    $ docker diff webserver
    ```

3. 执行docker commit

   `docker commit` 的语法格式为：

   ```bash
   docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]
   ```

   我们可以用下面的命令将容器保存为镜像：

   ```bash
   $ docker commit \
       --author "zwd" \
       --message "修改了默认网页" \
       webserver \
       nginx:v2
   ```

4. 查看变更

   `docker history` 可查看镜像内的历史记录，如：

    ```bash
    $ docker history nginx:v2 
    IMAGE          CREATED          CREATED BY                                       SIZE      COMMENT
    c02d40cdf684   29 seconds ago   nginx -g daemon off;                             1.23kB    修改了默认网页
    b52e0b094bc0   5 weeks ago      CMD ["nginx" "-g" "daemon off;"]                 0B        buildkit.dockerfile.v0
    <missing>      5 weeks ago      STOPSIGNAL SIGQUIT                               0B        buildkit.dockerfile.v0
    <missing>      5 weeks ago      EXPOSE map[80/tcp:{}]                            0B        buildkit.dockerfile.v0
    ```





## Dockerfile

### 初见dockerfile

Dockerfile 是一个文本文件，其内包含了一条条的 **指令(Instruction)**，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

#### 示例

以之前定制 `nginx` 镜像为例，这次我们使用 Dockerfile 来定制。

1. 在一个空白目录中，建立一个文本文件，并命名为 `Dockerfile`，内容为

   ```dockerfile
   FROM nginx
   RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
   ```

   - FROM 指定基础镜像

     如果你以 `scratch` 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

   - RUN 执行命令，其格式有两种：

     - *shell* 格式：`RUN <命令>`
     - *exec* 格式：`RUN ["可执行文件", "参数1", "参数2"]`

     **说明：**无论使用哪种格式，每条 `RUN` 指令都会生成一个新的镜像层。因此，**推荐合并多个命令**以减少层数：

     ```dockerfile
     # 不推荐（生成 3 层）
     RUN apt-get update
     RUN apt-get install -y nginx
     RUN apt-get clean
     
     # 推荐（生成 1 层）
     RUN apt-get update && apt-get install -y nginx && apt-get clean
     ```

2. 构建镜像

   ```bash
   docker build -t nginx:my .
   ```

3. 运行

   ```bash
   docker run --name mynginx -d -p 8080:80 nginx:my
   ```

#### 镜像构建上下文（Context）

怎么理解`docker build -t nginx:my .`这条命令里最后的'.' ？

‘.’ 表示当前目录，而 `Dockerfile` 就在当前目录，因此不少初学者以为这个路径是在指定 `Dockerfile` 所在路径，这么理解其实是不准确的。如果对应上面的命令格式，你可能会发现，这是在指定 **上下文路径**。

构建的时候，用户会指定构建镜像上下文的路径，`docker build` 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

如果在 `Dockerfile` 中这么写：

```docker
COPY ./package.json /app/
```

这并不是要复制执行 `docker build` 命令所在的目录下的 `package.json`，也不是复制 `Dockerfile` 所在目录下的 `package.json`，而是复制 **上下文（context）** 目录下的 `package.json`。

#### 构建hello docker示例镜像

1. 新建c源码

   ```bash
   vim hello_docker.c 
   ```

   ```c
   #include <stdio.h>
   int main() {
      printf("Hello, Docker!\n");
      return 0;
   }
   ```

2. 编译

   ```bash
   gcc -o hello_docker hello_docker.c
   ```

3. 新建Dockerfie

   ```dockerfile
   FROM frolvlad/alpine-glibc
   
   # 复制编译好的二进制文件到容器
   COPY hello_docker /hello_docker
   
   # 运行程序
   CMD ["/hello_docker"]
   ```

   - 没有使用scratch，是因为编译生成的hello_docker还依赖glic的库。想完全的静态编译，还需要额外安装相关的静态库软件。

   - 没有使用alpine作为基础镜像，是因为alpine提供的不是glibc，而是musl。我仍希望继续使用gcc编译，所以使用alpine-glibc。

4. 构建、运行 Docker 镜像

   ```bash
   $ docker build -t hello-docker:1.0 .
   $ docker run --rm hello-docker:1.0
   Hello, Docker!
   ```

   

## Compose



## 推荐资源

- [Docker — 从入门到实践](https://vuepress.mirror.docker-practice.com/)