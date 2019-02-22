# Docker

## 制作docker初始镜像
https://github.com/moby/moby

## docker安装
1. 稳定版本下载地址：https://download.docker.com/linux/centos/7/x86_64/stable/Packages/
2. 安装：
```
yum -y install yum-utils device-mapper-persistent-data lvm2 container-selinux libcgroup pigz libseccomp
rpm -ivh libltdl7-2.4.2-alt7.x86_64.rpm
rpm -ivh docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm
sudo curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://04be47cf.m.daocloud.io
```
3. 启动：`systemctl start docker`
4. 更新：`yum -y upgrade  /path/to/package.rpm`

## docker常用命令

### 镜像命令

docker images
```
docker images [OPTIONS] [REPOSITORY[:TAG]]

OPTIONS说明：

-a :列出本地所有的镜像（含中间映像层，默认情况下，过滤掉中间映像层）；
--digests :显示镜像的摘要信息；
-f :显示满足条件的镜像；
--format :指定返回值的模板文件；
--no-trunc :显示完整的镜像信息；
-q :只显示镜像ID。
```
```
docker images # 查看当前服务器上的所有镜像
```

docker search
```
docker search [OPTIONS] TERM

OPTIONS说明：

--automated :只列出 automated build类型的镜像；
--no-trunc :显示完整的镜像描述；
-s :列出收藏数不小于指定值的镜像。
```
```
docker search mysql #搜索名称为mysql的镜像
```

docker pull
```
docker pull [OPTIONS] NAME[:TAG|@DIGEST]

OPTIONS说明：

-a :拉取所有 tagged 镜像
--disable-content-trust :忽略镜像的校验,默认开启
```
```
docker pull centos:latest # 拉取最新版本centos
```

docker commit
```
docker commit [OPTIONS] CONTAINER
[REPOSITORY[:TAG]]

OPTIONS说明：

-a :提交的镜像作者；
-c :使用Dockerfile指令来创建镜像；
-m :提交时的说明文字；
-p :在commit时，将容器暂停。
```
```
a404c6c174a2 # 老容器ID
nginx:v1     # 新镜像名称
docker commit -a "zhubo" -m "message"  a404c6c174a2 nginx:v1 # 提交镜像到本地仓库
```

docker push
```
docker push [OPTIONS] NAME[:TAG]

OPTIONS说明：

--disable-content-trust :忽略镜像的校验,默认开启
```
```
docker push myapache:v1 # 上传本地镜像到myapache:v1到镜像仓库中
```

docker import
```

```

docker export
```

```

docker save
```

```

docker load
```

```

docker rmi
```
docker rmi 镜像ID # 删除镜像
```

### 容器命令

docker volume
```
docker volume create my-vol # 创建my-vol数据卷

docker volume ls # 列出当前所有的数据卷

docker volume rm my-vol # 删除my-vol数据卷
```

docker create
```

```

docker start/stop/restart
```

```

docker run
```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

OPTIONS说明：

-a stdin: 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；
-d: 后台运行容器，并返回容器ID；
-i: 以交互模式运行容器，通常与 -t 同时使用；
-p: 端口映射，格式为：主机(宿主)端口:容器端口
-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
--name="nginx-lb": 为容器指定一个名称；
--dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；
--dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；
-h "mars": 指定容器的hostname；
-e username="ritchie": 设置环境变量；
--env-file=[]: 从指定文件读入环境变量；
--cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；
-m :设置容器使用内存最大值；
--net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；
--link=[]: 添加链接到另一个容器；
--expose=[]: 开放一个端口或一组端口；
```
```
docker run -dti --name web --mount source=my-vol,target=/data/server Ubuntu
# docker 启动一个容器
# --mount source:宿主机目录，target:容器中目录
```

docker exec
```
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

OPTIONS说明：

-d :分离模式: 在后台运行
-i :即使没有附加也保持STDIN 打开
-t :分配一个伪终端
```
```
docker exec -i -t  mynginx /bin/bash # 在容器mynginx中开启一个交互模式的终端
```

docker attach
```
docker attach [OPTIONS] CONTAINER 
```
```
docker attach 容器ID或容器名称 # 进入容器名称或容器ID
```

docker inspect
```
docker inspect [OPTIONS] NAME|ID [NAME|ID...]

OPTIONS说明：
-f :指定返回值的模板文件。
-s :显示总的文件大小。
--type :为指定类型返回JSON。
```
```
docker inspect nginx:v1 # 查看nginx:v1中的信息
```

docker rm
```
docker rm [OPTIONS] CONTAINER [CONTAINER...]

OPTIONS说明：

-f :通过SIGKILL信号强制删除一个运行中的容器
-l :移除容器间的网络连接，而非容器本身
-v :-v 删除与容器关联的卷
```
```
docker rm 容器ID # 删除创建的容器
```

docker logs
```
docker logs [OPTIONS] CONTAINER

OPTIONS说明：

-f : 跟踪日志输出
--since :显示某个开始时间的所有日志
-t : 显示时间戳
--tail :仅列出最新N条容器日志
```
```
docker logs 容器ID或容器名称
```

公共镜像库：https://hub.docker.com

修改镜像仓库为国内镜像仓库:  
`curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s https://04be47cf.m.daocloud.io`