---
title: Docker基础教程
date: 2025-04-02 14:00
tags:
  - Docker
categories:
  - [Tools, Docker]
index_img: /img/categories/docker.png
---

Docker是一种虚拟容器应用这使我们可以在性能损失很小的情况下，方便地创建并管理虚拟环境。 本教程涵盖Docker的基础操作，帮助初学者快速上手。

<!-- more -->

## 1 安装Docker

要安装docker需要添加Docker的源，推荐使用清华的[镜像源](https://mirrors4.tuna.tsinghua.edu.cn/help/docker-ce/)来安装。

根据清华镜像站中的使用教程一般顺利完成docker的安装。

## 2 配置nvidia gpu支持

要在docker容器中调用nvidia gpu，无需在容器内安装nvidia显卡驱动，而可以令容器调用宿主机的显卡驱动，秩序宿主机正确安装显卡驱动即可。要完成这一点需要安装nvidia提供的nvidia-container-toolkit和nvidia-container-runtime，并正确配置daemon。

### 2.1 安装nvidia-container-toolkit和runtime

首先需要添加相关的仓库源。根据nvidia官方教程，指令如下：

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

之后使用apt安装即可。

```bash
sudo apt update
sudo apt install -y nvidia-container-toolkit nvidia-container-runtime
```

### 2.2 配置docker daemon

要使用nvidia显卡还需要配置docker daemon的runtime，修改`/etc/docker/daemon.json`文件，添加如下内容：

```json
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "/usr/bin/nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```

之后重启docker服务即可。

```bash
sudo systemctl restart docker
```

## 3 容器与镜像基础操作

Docker容器是由Docker镜像创建的运行实例，镜像是一个只读的模板，容器是镜像的运行实例。容器可以被启动、开始、停止、删除、暂停等。

一个容器相当于一个独立于宿主机的系统环境，我们可以像在宿主机上一样在容器内实现各种操作，比如安装软件、运行程序等等。

我们可以保存容器的状态成一个镜像，也可以运行镜像创建一个与该镜像创建时的容器一模一样的新容器。

### 3.1 获取镜像

要获取镜像文件，有三种方式，第一种是直接拉取云端的镜像或加载被保存到本地的离线镜像文件，第二种是从已存在的容器创建镜像，第三种是通过编写Dockerfile创建镜像。

拉取镜像的命令为，若运行容器时使用了本地没有的镜像，会先尝试拉取该镜像。

```bash
sudo docker pull image_name
```

### 3.2 查看镜像

docker镜像名称通常由两部分组成，一个是仓库名，一个是tag，写成`repo:tag`的形式。比如`ubuntu:22.04`, `ubuntu:latest`。

要查看本地的镜像可以使用下面命令。

```bash
sudo docker images
```

输出结果类似下面。

```text
REPOSITORY                      TAG                   IMAGE ID       CREATED        SIZE
osrf/ros                        noetic-desktop-full   f19749f1e3da   5 months ago   3.44GB
ubuntu                          20.04                 6013ae1a63c2   5 months ago   72.8MB
```

### 3.3 运行容器

运行容器需要使用命令`run`，格式为

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

以下列举常用的选项：

- `-d` 表示后台运行
- `-e ENVVAR=VAL` 添加环境变量的值，ENVVAR为环境变量名，VAL为值
- `--name container_name` 为容器指定一个名字
- `-it` 表示交互式运行终端，默认为bash，在[COMMAND]位置可以指定具体命令，如zsh。
- `-p host_port:container_port` 映射端口，host_port为宿主机端口，container_port为容器端口
- `--device DEVICE_PATH` 挂载宿主机设备，是容器可以访问该设备，一般为`/dev`目录下的
- `--privileged` 给予容器高权限(不太安全，使容器拥有破坏宿主机环境的能力)

例如，要运行一个ubuntu容器，可以使用下面命令。

```bash
sudo docker run --name ubuntu20 -it ubuntu:20.04 bash
```

### 3.4 查看容器信息

查看信息可以使用`ps`命令，格式为

```bash
sudo docker ps -a
```

输出类似下方

```text
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS                     PORTS     NAMES
3c02dd5ef563   mysql:latest   "docker-entrypoint.s…"   12 days ago   Exited (0) 6 days ago                rdnb_mysql
```

### 3.5 从容器运行程序

对于一个正在运行的容器，我们可以在宿主机中通过`exec`命令运行容器内的程序。

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

`-e`, `-d`, `-it`依旧可以使用，`CONTAINER`为容器名称，`COMMAND`为要运行的命令，`ARG`为命令的参数。

例如我们想打开一个容器中的终端

```bash
sudo docker exec -it ubuntu20 bash
```

### 3.6 其他常用命令

- `docker start CONTAINER` 启动容器
- `docker stop CONTAINER` 停止容器
- `docker restart CONTAINER` 重启容器
- `docker rm CONTAINER` 删除容器
- `docker image rm IMAGE` 删除镜像

### 3.7 保存容器到镜像

我们可以使用`commit`指令保存容器当前状态到镜像

```bash
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```

例如将名称为ubuntu20的容器保存为名为myenv:ubuntu20

```bash
sudo docker commit ubuntu20 myenv:ubuntu20
```

### 3.8 镜像迁移

Docker镜像可以保存到.tar文件，并且加载被保存在文件中的镜像。这使得docker容器便于部署和分发。

保存到文件

```bash
docker save -o image.tar image_name
```

从文件加载

```bash
docker load -i image.tar
```

## 4 运行GUI程序

要运行docker容器内的GUI程序，一般使用xorg进行。这需要容器内安装了x server，将GUI界面通过宿主机桌面上显示。一般ubuntu系统的容器已默认安装。

### 4.1 配置宿主机权限

要在宿主机上显示容器内的GUI界面，需要配置宿主机的权限，使得容器可以访问宿主机的桌面。

下面这行命令直接允许所有的本地访问。

```bash
xhost +local:
```

### 4.2 配置容器参数

要运行GUI程序，需要在运行容器时添加相应参数，使得容器可以正常访问宿主机的桌面显示图形界面。

- `-e "DISPLAY=$DISPLAY"` 将环境变量DISPLAY设置为主机显示器;
- `--mount type=bind,src=/tmp/.X11-unix,dst=/tmp/.X11-unix` 将主机X服务器套接字挂载到相同路径下的容器内;
- `--device=/dev/dri:/dev/dri` 允许容器直接访问主机上的直接渲染(DRI) 设备。

例如，要运行一个ubuntu容器并运行终端，可以使用下面命令。

```bash
sudo docker run -e "DISPLAY=$DISPLAY" --mount type=bind,src=/tmp/.X11-unix,dst=/tmp/.X11-unix --device=/dev/dri:/dev/dri -it ubuntu:20.04
```

之后在容器内运行GUI程序一般可以直接在宿主机上打开。

## 5 调用nvidia gpu加速容器

### 5.1 使容器可以访问宿主机显卡

要调用nvidia gpu，需要在运行容器时添加相应参数，使得容器可以正常访问宿主机的显卡。

一般需要添加以下参数：

- `--gpus all` 使得容器可以访问宿主机的显卡
- `--runtime=nvidia` 使用nvidia runtime
- `-e NVIDIA_DRIVER_CAPABILITIES=all` 设置环境变量，开启所有nvidia驱动功能

例如打开一个需要调用gpu并且要打开图形界面的容器可以使用如下命令。

```bash
sudo docker run -e "DISPLAY=$DISPLAY" --mount type=bind,src=/tmp/.X11-unix,dst=/tmp/.X11-unix --device=/dev/dri:/dev/dri --gpus all --runtime=nvidia -e NVIDIA_DRIVER_CAPABILITIES=all -it ubuntu:20.04
```

无需在容器内安装nvidia显卡驱动，在容器内可以通过调用`nvidia-smi`来确认是否可以正常调用宿主机显卡驱动来访问显卡。

### 5.2 容器中配置cuda、cudnn等

要在容器中使用cuda等工具，推荐做法是直接使用已经配置好cuda、cudnn等工具的镜像，nvidia官方提供了相关镜像。[官网](https://catalog.ngc.nvidia.com/containers)

但如果确实需要自己安装cuda等工具，就像在宿主机上安装一样直接在容器中安装即可。

**注意**：不建议使用包管理器安装，这往往会同时安装nvidia显卡驱动，导致驱动版本不兼容问题。推荐做法是使用nvidia的安装程序安装，并选择不安装驱动。

## 6 数据可持久化

容器是一个独立的环境，容器内的数据在容器删除后会丢失。为了保存数据，docker提供了数据可持久化的方法。

### 6.1 复制文件

Docker提供了`cp`命令，可以在宿主机和容器之间复制文件或文件夹。格式如下：

```bash
docker cp SRC DST
```

会将SRC文件或文件夹复制到DST文件或文件夹，宿主机上的文件直接写路径即可，容器内的文件需要在容器内路径前加上`CONTAINER:`。

### 6.2 卷

Docker可以通过创建卷(volume)将数据可持久化，卷可以被映射为容器内的某个文件夹，该文件夹下的文件都会被保存到宿主机上，不会随着容器被删除而消失，可以重新挂载。

多个容器挂载相同的卷可以用来共享数据。

#### 6.2.1 运行时挂载卷

在运行容器时可以使用`-v`参数来创建卷，格式如下：

`-v volume_name:container_path`

如果volume_name不存在，docker会自动创建一个新的卷。

如果要将宿主机的某个文件夹映射到容器内的某个文件夹，可以使用下面格式，这并不会创建卷。

`-v host_path:container_path`

#### 卷其他操作

Docker中卷相关的命令以`docker volume`开头，

- `docker volume create` 创建卷
- `docker volume ls` 列出所有卷
- `docker volume rm` 删除卷
- `docker volume inspect` 查看卷信息
- `docker volume prune` 删除所有未被使用的卷

## 7 网络network

Docker可以为容器配置network，使得容器间通过网络访问更加方便。挂载同一个network的docker容器处于同一个网络环境内，可以通过network的名字作为域名来互相访问。

例如，容器A和B挂载了名叫abnet的network，那么在A中访问`http://abnet:3000`可以访问B中localhost:3000上的服务。在B中也同理。

我们使用`-p`暴露端口将容器内端口映射到到宿主机的一个端口上。

### 7.1 运行时挂载network

要在运行容器时为其挂载network只需要加上参数即可，如果该名称网络不存在则会创建一个新的。

```bash
--network NETWORK_NAME
```

Docker默认创建三个network，`host`，`bridge`，`none`。使用`host`网络，则容器使用宿主机的网络环境，没有自己独立的网络环境。`none`则表示不配置网络。

默认的网络为`bridge`，每个容器有自己独立的网络环境，包括ip地址等。我们可以创建新的使用`bridge`模式的网络，实现更好的隔离以避免端口冲突。

### 7.2 管理network

要管理所有的network，需使用`docker network`相关命令

- `docker network ls` 列出网络
- `docker network inspect NETWORK` 查看网络详细信息
- `docker network connect NETWORK CONTAINER` 将容器连接到某个网络
- `docker network disconnect NETWORK CONTAINER` 解除连接
- `docker network create NETWORK` 创建网络，默认使用`bridge`模式

## 8 Dockerfile构建镜像

上面提到，我们使用可以使用Dockerfile构建脚本来构建镜像。

### 8.1 Dockerfile语法

Dockerfile脚本实际上是基于某个镜像，再通过在容器中运行shell命令来完成容器配置。

我们创建一个文件夹，该文件夹下创建名为`Dockerfile`的文件

Dockerfile语法十分简单，常用命令为

- `RUN COMMAND` 在容器内终端中运行COMMAND命令，
- `WORKDIR PATH` 将当前目录设置为PATH，推荐使用，不推荐用cd
- `COPY SRC DST` 复制文件或文件夹，前者为宿主机上的路径，后者为容器内路径，无需加前缀
- `FROM REPO:TAG` 选择基于哪个镜像，若本地没有该镜像则尝试拉取
- `USER USERNAME` 指定之后RUN等命令的执行用户
- `SHELL [command, args...]` command和args均为字符串，指定后面RUN使用的shell，如使用zsh(确认已安装)则为`SHELL ['/usr/bin/zsh', '-c']`。

### 8.2 构建镜像

将当前目录设置为上面创建的文件夹，运行以下命令即开始创建名为`repo:tag`的镜像。

```bash
sudo docker build . -t repo:tag
```

会一步一步运行Dockerfile中的命令，并且默认每一步都会保存一个缓存。如果你修改了一行命令，那么build时会加载该行命令前的缓存，从这一行开始构建。

## 9 其它配置

### 9.1 配置用户权限

Docker命令仅能由root用户和docker用户组中的用户运行。要使普通用户也可运行docker命令，除了使用sudo命令外，还可以将该用户加入到docker权限组。

使用如下命令可将当前用户添加到docker用户组，之后可以正常运行docker命令。

```bash
sudo usermod -aG docker $USER
```

### 9.2 Docker Desktop卸载残留

若安装过Docker Desktop但又卸载，在用户目录下可能残留配置和数据文件。当我们将当前用户加入到docker用户组之后，尝试直接运行docker命令时，可能会错误地尝试加载这些配置文件而产生错误。

例如提示无法连接至`/home/username/.docker/desktop/docker.sock`。

要解决该问题，只需删除用户目录下的`.docker`文件夹。
