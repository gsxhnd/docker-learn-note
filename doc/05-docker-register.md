# 5. Docker仓库

## 6. Dcoker 仓库

一个容易混淆的概念是注册服务器（Registry）。 实际上注册服务器是管理仓库的具体服务器，每个服务器上可以有多个仓库，而每个仓库下面有多个镜像。从这方面来说， 仓库可以被认为是一个具体的项目或目录。例如对于仓库地址`registry.hub.docker.com/ubuntu` 来说，`registry.hub.docker.com`是注册服务器地址，`ubuntu`是仓库名。大部分时候，并不需要严格区分这两者的概念。

### 6.1. Docker Hub

目前 Docker 官方维护了一个公共仓库 Docker Hub：[https://hub.docker.com](https://hub.docker.com/)

我们可以在 Docker Hub 上完成注册。这样就可以使用 Docker Hub 来托管我们的镜像了。

通过`docker search`命令来查找官方仓库中的镜像，并利用`docker pull` 命令来将它下载到本地。

```text
$ docker search alpine
NAME                                   DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
alpine                                 A minimal Docker image based on Alpine Linux…   7869      [OK]
mhart/alpine-node                      Minimal Node.js built on Alpine Linux           483
anapsix/alpine-java                    Oracle Java 8 (and 7) with GLIBC 2.28 over A…   473                  [OK]
frolvlad/alpine-glibc                  Alpine Docker image with glibc (~12MB)          268                  [OK]
alpine/git                             A  simple git container running in alpine li…   191                  [OK]
```

### 6.2. 使用国内镜像

参考地址：[https://www.docker-cn.com/registry-mirror](https://www.docker-cn.com/registry-mirror)

临时性的使用：

```text
$ docker pull registry.docker-cn.com/library/alpine:3.13.6
3.13.6: Pulling from library/alpine
4e9f2cdf4387: Pull complete
Digest: sha256:2582893dec6f12fd499d3a709477f2c0c0c1dfcd28024c93f1f0626b9e3540c8
Status: Downloaded newer image for alpine:3.13.6
docker.io/library/alpine:3.13.6
```

永久性的使用：

修改 `/etc/docker/daemon.json` 文件（没有的话可以手动创建，需要通过`root`用户操作）并添加上 registry-mirrors 键值。

```javascript
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

修改保存后重启 Docker 服务以使配置生效。

```text
$ systemctl restart docker
```

### 5.2. Docker 私有仓库搭建

**环境准备**

环境：两个装有 Docker 的虚拟机

机器名

ip

功能

docker-registry

192.168.119.1

docker 私有仓库服务器

docker-app

192.168.119.2

运行 docker 服务的普通服务器

**安装私有仓库**

登录 docker-registry 机器\(推荐使用 SecureCRT\);

## 创建数据存储的文件夹 ，将容器的/tem/registry 目录映射到/docker/registry-

$ mkdir /docker/registry

执行以下命令，会启动一个 registry 容器，该容器用于提供私有仓库的服务：- $ sudo docker run -d -p 5000:5000 --name registry-ser --restart=always --privileged=true -v /docker/registry:/var/lib/registry registry

* --restart=always 表示自动启动容器
* -v &lt;宿主机目录&gt;:&lt;容器目录&gt; 将宿主机的目录映射到容器上
* --privileged=true 给容器加权限，这样上传就不会因为目录权限出错
* /var/lib/registry 这个目录是 docker 私有仓库，镜像的存储目录

--restart 的参数说明- \# always：无论容器的退出代码是什么，Docker 都会自动重启该容器。- \# on-failure：只有当容器的退出代码为非 0 值的时候才会自动重启。- \#另外，该参数还接受一个可选的重启次数参数，\`--restart=on-fialure:5\`表示当容器退出代码为非 0 时，Docker\#会尝试自动重启该容器，最多 5 次。

#### **测试**

接下来我们就要操作把一个本地镜像 push 到私有仓库中。首先在 153 机器下 pull 一个比较小的镜像来测试（此处使用的是 busybox）。

接下来修改一下该镜像的 tag。

docker tag busybox 192.168.119.1:5000/busybox

接下来把打了 tag 的镜像上传到私有仓库。

docker push 192.168.119.1:5000/busybox

可以看到 push 失败：

Error: Invalid registry endpoint [https://192.168.119.1:5000/v1/](https://192.168.119.1:5000/v1/): Get [https://192.168.119.1:5000/v1/\_ping](https://192.168.119.1:5000/v1/_ping): dial tcp 192.168.119.1:5000: connection refused. If this private registry supports only HTTP or HTTPS with an unknown CA certificate, please add \`--insecure-registry 192.168.119.1:5000\` to the daemon's arguments. In the case of HTTPS, if you have access to the registry's CA certificate, no need for the flag; simply place the CA certificate at /etc/docker/certs.d/192.168.119.1:5000/ca.crt

因为 Docker 从 1.3.X 之后，与 docker registry 交互默认使用的是 https，然而此处搭建的私有仓库只提供 http 服务，所以当与私有仓库交互时就会报上面的错误。为了解决这个问题需要在启动 docker server 时增加启动参数为默认使用 http 访问。修改 docker 启动配置文件：

第一种方法：

修改 DOCKER\_OPTS 参数

DOCKER\_OPTS="-H 0.0.0.0:2375 --insecure-registry 192.168.119.1:5000"

第二种方法：

vi /etc/docker/daemon.json

将下面的代码放进去保存并退出。

"insecure-registries":\["192.168.119.1:5000"\]

最终如下所示：

{ "registry-mirrors": \["[https://njrds9qc.mirror.aliyuncs.com"\](https://njrds9qc.mirror.aliyuncs.com"\)\], "insecure-registries":\["192.168.119.1:5000"\] }

重新启动 docker：

sudo service restart docker

重启完之后我们再次运行推送命令，把本地镜像推送到私有服务器上。

docker push 192.168.119.1:5000/busybox

接下来我们从私有仓库中 pull 下来该镜像。

sudo docker pull 192.168.119.1:5000/busybox

**查看镜像**

```text
$ curl -XGET http://registry:5000/v2/_catalog
$ curl -XGET http://registry:5000/v2/image_name/tags/list
```

