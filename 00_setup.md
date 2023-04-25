# Setup

使用docker构建sel4教程的步骤文档

## Docker Setup

### 参考

[Ubuntu - Docker — 从入门到实践 (gitbook.io)](https://yeasy.gitbook.io/docker_practice/install/ubuntu)

### 卸载旧版本

```apt
sudo apt-get remove docker \
               docker-engine \
               docker.io
```

### 使用 APT 安装

由于 `apt` 源使用 HTTPS 以确保软件下载过程中不被篡改。因此，我们首先需要添加使用 HTTPS 传输的软件包以及 CA 证书。

```bash
sudo apt-get update
sudo apt-get install \
 apt-transport-https \
 ca-certificates \
 curl \
 gnupg \
 lsb-release
```

为了确认所下载软件包的合法性，需要添加软件源的 `GPG` 密钥。

```bash
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

然后，我们需要向 `sources.list` 中添加 Docker 软件源

```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 安装docker

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### 启动docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 建立docker用户组

```bash
#建立 docker 组
sudo groupadd docker
#将当前用户加入 docker 组
sudo usermod -aG docker $USER
```

### 测试docker是否安装正确

```bash
docker run --rm hello-world
```

> 期望输出： 
>
> Unable to find image 'hello-world:latest' locally
>
> latest: Pulling from library/hello-world
>
> b8dfde127a29: Pull complete
>
> Digest: sha256:308866a43596e83578c7dfa15e27a73011bdd402185a84c5cd7f32a88b501a24
>
> Status: Downloaded newer image for hello-world:latest
>
> 
>
> Hello from Docker!
>
> This message shows that your installation appears to be working correctly.
>
> 
>
> To generate this message, Docker took the following steps:
>
> 1. The Docker client contacted the Docker daemon.
> 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
>
> ​    (amd64)
>
> 3. The Docker daemon created a new container from that image which runs the
>
> ​    executable that produces the output you are currently reading.
>
> 4. The Docker daemon streamed that output to the Docker client, which sent it
>
> ​    to your terminal.
>
> 
>
> To try something more ambitious, you can run an Ubuntu container with:
>
>  $ docker run -it ubuntu bash
>
> 
>
> Share images, automate workflows, and more with a free Docker ID:
>
>  https://hub.docker.com/
>
> For more examples and ideas, visit:
>
>  https://docs.docker.com/get-started/



### docker换源

```bash
systemctl cat docker | grep '\-\-registry\-mirror'
```

若该命令有输出，执行 `$ systemctl cat docker` 查看 `ExecStart=` 出现的位置，修改对应的文件内容去掉 `--registry-mirror` 参数及其值，并按接下来的步骤进行配置。

如果以上命令没有任何输出，那么就可以在 `/etc/docker/daemon.json` 中写入如下内容（如果文件不存在请新建该文件）：

```json
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://ustc-edu-cn.mirror.aliyuncs.com",
    "https://ghcr.io",
    "https://mirror.baidubce.com"
  ]
}
```



## seL4 Tutorial Setup

### 初始化docker环境

```bash
git clone https://github.com/seL4/seL4-CAmkES-L4v-dockerfiles.git
cd seL4-CAmkES-L4v-dockerfiles
make user
```

### [optional] 在docker中使用宿主机代理

上一步中执行`make user`时，若在执行 `./seL4-CAmkES-L4v-dockerfiles/dockerfiles/extras.Dockerfile`中的

```dockerfile
RUN apt-get update -q \
    && apt-get install -y --no-install-recommends \
        # Add more dependencies here
        cowsay \
        sudo
```

apt-get update命令时过于缓慢，需配置docker容器中可用的代理。这是由于sel4镜像的apt使用debian源，国内下载速度太慢。解决方案：修改该dockerfile，以使用docker宿主机一侧的代理。在run命令前增加：

```dockerfile
ENV http_proxy "http://172.17.0.1:7890"
ENV HTTP_PROXY "http://172.17.0.1:7890"
ENV https_proxy "http://172.17.0.1:7890"
ENV HTTPS_PROXY "http://172.17.0.1:7890"
```

其中172.17.0.1是docker容器中的docker宿主机的ip地址，端口号7890需要配置为宿主机上代理端口。具体参考[使用代理进行 docker build 问题的解决思路 - simpleapples](http://simpleapples.com/2019/04/18/building-docker-image-behind-proxy/)。该文中提到还可以通过设置docker使用宿主网络代理模式。



### [optional] 将当前目录开启容器操作绑定为shell快捷指令

```bash
echo $'alias container=\'make -C /<path>/<to>/seL4-CAmkES-L4v-dockerfiles user HOST_DIR=$(pwd)\'' >> ~/.bashrc
source ~/.bashrc
```

1. ~/.bashrc可替换为 ~/.zshrc等终端配置文件
2. Replace `/<path>/<to>/` to match where you cloned the git repo of the docker files. 

配置完成，可使用

```
container
```

将当前目录挂载到容器中的\host并构建运行sel4tutorial docker image



## 安装Google Repo

```bash
sudo apt-get update
sudo apt-get install repo
```

## A example workflow

https://docs.sel4.systems/projects/dockerfiles/#:~:text=An%20example%20workflow%3A