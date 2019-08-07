---
 title: docker教程
date: 2019-08-03 11:04:39
tags: docker
---

# docker基本命令

## 镜像相关

```bash
# 1. 查找镜像
docker search mysql
docker search -s 100 mysql # 搜索100star以上的mysql镜像

# 2. 获取镜像
docker pull centos7

# 3. 查找镜像
docker image ls

# 4. 删除镜像
docker rmi od16d0a97dd1 

# 5. 创建镜像
docker build -t nginx:1.14.0 .
```

## 容器相关

```bash
# 1. 启动容器
docker run -it --rm php:7 bash  # 启动一个容器，并分配一个为终端，退出容器就会被删除
docker run -d -p 9000:9000 php:7 # 运行我们常用的容器

# 2. 查看容器的信息
docker inspect a47342323232 # image id

# 3. 进入容器
docker exec -it es2dffg2f2gh bash #image id ,bash根据系统会有不同

# 4. 停止容器
docker stop 34fvd234234f

# 5. 启动容器
docker start 3fg23fsfsfs

# 6. 查看容器
docker ps # 查看启动的容器
docker ps -a # 查看全部容器

# 7. 删除容器
docker rm d989ds82kjk 
```

## volume

### 创建

```bash
# 使用匿名volume
docker run -v 容器路径 ...

docker run -v host路径:容器路径 ...

# 使用volumes-from
1. 创建一个容器
2. Docker run —volumes-from data-container ubuntu ,可以共享文件
```

### 删除

这个功能可能会更加重要，如果你已经使用

```
docker rm
```

来删除你的容器，那可能有很多的孤立的Volume仍在占用着空间。

Volume只有在下列情况下才能被删除：

- 该容器是用`docker rm －v`命令来删除的（`-v`是必不可少的）。
- `docker run`中使用了`--rm`参数

即使用以上两种命令，也只能删除没有容器连接的Volume。连接到用户指定主机目录的Volume永远不会被docker删除。

除非你已经很小心的，总是像这样来运行容器，否则你将会在

```
/var/lib/docker/vfs/dir
```

目录下得到一些僵尸文件和目录，并且还不容易说出它们到底代表什么。

## dockerfile

### 参考

```dockerfile
FROM debian:wheezy
MAINTAINER Steeve Morin "steeve.morin@gmail.com"

RUN apt-get update && apt-get -y install  unzip \
                        xz-utils \
                        curl \
                        bc \
                        git \
                        build-essential \
                        cpio \
                        gcc-multilib libc6-i386 libc6-dev-i386 \
                        kmod \
                        squashfs-tools \
                        genisoimage \
                        xorriso \
                        syslinux \
                        automake \
                        pkg-config

ENV KERNEL_VERSION  3.16.1
ENV AUFS_BRANCH     aufs3.16

# Fetch the kernel sources
RUN curl --retry 10 https://www.kernel.org/pub/linux/kernel/v3.x/linux-$KERNEL_VERSION.tar.xz | tar -C / -xJ && \
    mv /linux-$KERNEL_VERSION /linux-kernel

# Download AUFS and apply patches and files, then remove it
RUN git clone -b $AUFS_BRANCH --depth 1 git://git.code.sf.net/p/aufs/aufs3-standalone && \
    cd aufs3-standalone && \
    cd /linux-kernel && \
    cp -r /aufs3-standalone/Documentation /linux-kernel && \
    cp -r /aufs3-standalone/fs /linux-kernel && \
    cp -r /aufs3-standalone/include/uapi/linux/aufs_type.h /linux-kernel/include/uapi/linux/ &&\
    for patch in aufs3-kbuild aufs3-base aufs3-mmap aufs3-standalone aufs3-loopback; do \
        patch -p1 < /aufs3-standalone/$patch.patch; \
    done

COPY kernel_config /linux-kernel/.config

RUN jobs=$(nproc); \
    cd /linux-kernel && \
    make -j ${jobs} oldconfig && \
    make -j ${jobs} bzImage && \
    make -j ${jobs} modules

# The post kernel build process

ENV ROOTFS          /rootfs
ENV TCL_REPO_BASE   http://tinycorelinux.net/5.x/x86
ENV TCZ_DEPS        iptables \
                    iproute2 \
                    openssh openssl-1.0.0 \
                    tar \
                    gcc_libs \
                    acpid \
                    xz liblzma \
                    git expat2 libiconv libidn libgpg-error libgcrypt libssh2 \
                    nfs-utils tcp_wrappers portmap rpcbind libtirpc \
                    curl ntpclient

# Make the ROOTFS
RUN mkdir -p $ROOTFS

# Install the kernel modules in $ROOTFS
RUN cd /linux-kernel && \
    make INSTALL_MOD_PATH=$ROOTFS modules_install firmware_install

# Remove useless kernel modules, based on unclejack/debian2docker
RUN cd $ROOTFS/lib/modules && \
    rm -rf ./*/kernel/sound/* && \
    rm -rf ./*/kernel/drivers/gpu/* && \
    rm -rf ./*/kernel/drivers/infiniband/* && \
    rm -rf ./*/kernel/drivers/isdn/* && \
    rm -rf ./*/kernel/drivers/media/* && \
    rm -rf ./*/kernel/drivers/staging/lustre/* && \
    rm -rf ./*/kernel/drivers/staging/comedi/* && \
    rm -rf ./*/kernel/fs/ocfs2/* && \
    rm -rf ./*/kernel/net/bluetooth/* && \
    rm -rf ./*/kernel/net/mac80211/* && \
    rm -rf ./*/kernel/net/wireless/*

# Install libcap
RUN curl -L ftp://ftp.de.debian.org/debian/pool/main/libc/libcap2/libcap2_2.22.orig.tar.gz | tar -C / -xz && \
    cd /libcap-2.22 && \
    sed -i 's/LIBATTR := yes/LIBATTR := no/' Make.Rules && \
    sed -i 's/\(^CFLAGS := .*\)/\1 -m32/' Make.Rules && \
    make && \
    mkdir -p output && \
    make prefix=`pwd`/output install && \
    mkdir -p $ROOTFS/usr/local/lib && \
    cp -av `pwd`/output/lib64/* $ROOTFS/usr/local/lib

# Make sure the kernel headers are installed for aufs-util, and then build it
RUN cd /linux-kernel && \
    make INSTALL_HDR_PATH=/tmp/kheaders headers_install && \
    cd / && \
    git clone git://git.code.sf.net/p/aufs/aufs-util && \
    cd /aufs-util && \
    git checkout aufs3.9 && \
    CPPFLAGS="-m32 -I/tmp/kheaders/include" CLFAGS=$CPPFLAGS LDFLAGS=$CPPFLAGS make && \
    DESTDIR=$ROOTFS make install && \
    rm -rf /tmp/kheaders

# Download the rootfs, don't unpack it though:
RUN curl -L -o /tcl_rootfs.gz $TCL_REPO_BASE/release/distribution_files/rootfs.gz

# Install the TCZ dependencies
RUN for dep in $TCZ_DEPS; do \
    echo "Download $TCL_REPO_BASE/tcz/$dep.tcz" &&\
        curl -L -o /tmp/$dep.tcz $TCL_REPO_BASE/tcz/$dep.tcz && \
        unsquashfs -f -d $ROOTFS /tmp/$dep.tcz && \
        rm -f /tmp/$dep.tcz ;\
    done

COPY VERSION $ROOTFS/etc/version

# Get the Docker version that matches our boot2docker version
# Note: `docker version` returns non-true when there is no server to ask
RUN curl -L -o $ROOTFS/usr/local/bin/docker https://get.docker.io/builds/Linux/x86_64/docker-$(cat $ROOTFS/etc/version) && \
    chmod +x $ROOTFS/usr/local/bin/docker && \
    { $ROOTFS/usr/local/bin/docker version || true; }

# get generate_cert
RUN curl -L -o $ROOTFS/usr/local/bin/generate_cert https://github.com/SvenDowideit/generate_cert/releases/download/0.1/generate_cert-0.1-linux-386/ && \
    chmod +x $ROOTFS/usr/local/bin/generate_cert

# Get the git versioning info
COPY .git /git/.git
RUN cd /git && \
    GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD) && \
    GITSHA1=$(git rev-parse --short HEAD) && \
    DATE=$(date) && \
    echo "${GIT_BRANCH} : ${GITSHA1} - ${DATE}" > $ROOTFS/etc/boot2docker

COPY rootfs/isolinux /isolinux

# Copy our custom rootfs
COPY rootfs/rootfs $ROOTFS

# These steps can only be run once, so can't be in make_iso.sh (which can be run in chained Dockerfiles)
# see https://github.com/boot2docker/boot2docker/blob/master/doc/BUILD.md
RUN    \
# Make sure init scripts are executable && \
	find $ROOTFS/etc/rc.d/ $ROOTFS/usr/local/etc/init.d/ -exec chmod +x '{}' ';' && \
# Download Tiny Core Linux rootfs  && \
	( cd $ROOTFS && zcat /tcl_rootfs.gz | cpio -f -i -H newc -d --no-absolute-filenames ) && \
# Change MOTD && \
	mv $ROOTFS/usr/local/etc/motd $ROOTFS/etc/motd && \
# Make sure we have the correct bootsync && \
	mv $ROOTFS/boot*.sh $ROOTFS/opt/ && \
	chmod +x $ROOTFS/opt/*.sh && \
# Make sure we have the correct shutdown && \
	mv $ROOTFS/shutdown.sh $ROOTFS/opt/shutdown.sh && \
	chmod +x $ROOTFS/opt/shutdown.sh && \
# Add serial console && \
	echo "#!/bin/sh" > $ROOTFS/usr/local/bin/autologin && \
	echo "/bin/login -f docker" >> $ROOTFS/usr/local/bin/autologin && \
	chmod 755 $ROOTFS/usr/local/bin/autologin && \
	echo 'ttyS0:2345:respawn:/sbin/getty -l /usr/local/bin/autologin 9600 ttyS0 vt100' >> $ROOTFS/etc/inittab && \
# fix su - && \
	echo root > $ROOTFS/etc/sysconfig/superuser


COPY rootfs/make_iso.sh /
RUN /make_iso.sh
CMD ["cat", "boot2docker.iso"]
```

1. FROM指令和MAINTAINER指令

   脚本的第1行是`FROM`指令。通过`FROM`指令，`docker`编译程序能够知道在哪个基础镜像执行来进行编译。所有的Dockerfile都必须以`FROM`指令开始。第二条指令`MAINTAINER`，用来标明这个镜像的维护者信息。

2. RUN指令

   接下来是`RUN`指令。这条指令用来在`docker`的编译环境中运行指定命令。上面这条指令会在编译环境运行`/bin/sh -c "apt-get update && apt-get -y install ..."`。`RUN`指令还有另外一种格式：

   ```
   RUN ["程序名", "参数1", "参数2"]
   ```

   这种格式运行程序，可以免除运行`/bin/sh`的消耗。这种格式使用Json格式将程序名与所需参数组成一个字符串数组，所以如果参数中有引号等特殊字符，需要进行转义。

3. ENV指令

   `ENV`指令用来指定在执行`docker run`命令运行镜像时，自动设置的环境变量。这些环境变量可以通过`docker run`命令的`--evn`参数来进行修改。

4. COPY指令和ADD指令

   `COPY`指令用来将本地（Dockerfile所在位置）的文件或文件夹复制到编译环境的指定路径下。上面的例子里，boot2docker的Dockerfile希望将与Dockerfile同一目录下的`kernel_config`文件复制到编译环境的`/linux-kernal/.config`。Dockerfile还提供了另外一个类似的指令：`ADD`。在复制文件方面`ADD`指令和`COPY`指令的格式和效果是完全一样的。这两个指令的区别主要由两点：

   1. `ADD`指令可以从一个URL地址下载内容复制到容器的文件系统中;
   2. `ADD`指令会将压缩打包格式的文件解开后复制到指定位置，而`COPY`指令只做复制操作。

5. CMD指令

   这是整个Dockerfile脚本的最后一条指令。当Dockerfile已经完成了所有环境的安装与配置，通过`CMD`指令来指示`docker run`命令运行镜像时要执行的命令。上面的例子里，在完成所有工作后，boot2docker的编译脚本将编译结果输出到本地环境下。

6. 其他指令

   上面我们通过boot2docker的Dockerfile脚本学习了几个最常用的指令。接下来我们再学习剩下的几个指令。

   ### `EXPOSE`指令

   `EXPOSE <端口> [<端口>...]`指令用于标明，这个镜像中的应用将会侦听某个端口，并且希望能将这个端口映射到主机的网络界面上。但是，为了安全，`docker run`命令如果没有带上响应的端口映射参数，`docker`并不会将端口映射出了。

   ### `ENTRYPOINT`指令

   `ENTRYPOINT`指令和前面介绍过的`CMD`一样，用于标明一个镜像作为容器运行时，最后要执行的程序或命令。这两个指令有相同之处，也有区别。通过两个指令的配合使用可以配置出不同的效果。

   `ENTRYPOINT`指令有两种格式，`CMD`指令有三种格式：

   ```
   ENTRYPOINT ["程序名", "参数1", "参数2"]
   ENTRYPOINT 命令 参数1 参数2
   
   CMD ["程序名", "参数1", "参数2"]
   CMD 命令 参数1 参数2
   CMD 参数1 参数2
   ```

   `ENTRYPOINT`是容器运行程序的入口。也就是说，在`docker run`命令中指定的命令都将作为参数提供给`ENTRYPOINT`指定的程序。同样，上面列举的`CMD`指令格式的后面两种格式也将作为参数提供给`ENTRYPOINT`指定的程序。

   默认的`ENTRYPOINT`是`/bin/sh -c`。你可以根据实际需要任意设置。但是如果在一个Dockerfile中出现了多个`ENTRYPOINT`指令，那么，只有最后一个`ENTRYPOINT`指令是起效的。

   一种常用的设置是将命令与必要参数设置到`ENTRYPOINT`中，而运行时只提供其他选项。例如：你有一个MySQL的客户端程序运行在容器中，而客户端所需要的主机地址、用户名和密码你不希望每次都输入，你就可以将`ENTRYPOINT`设置成：`ENTRYPOINT mysql -u <用户名> -p <密码> -h <主机名>`。而你运行时，只需要指定数据库名。

   ### `VOLUME`指令

   ```
   VOLUME ["路径"]
   ```

   `VOLUME`指令用于在容器内创建一个或多个卷。而更多的时候，是在执行`docker run`时指定要创建的卷以及本地路径来进行映射。关于这个用法将在后面的章节学习到。

   ### `USER`指令

   ```
   USER 用户名或用户ID
   ```

   `USER`指令用于容器内运行`RUN`指令或`CMD`指令的用户。例如，在构建一个nginx镜像时，你希望最后运行nginx的用户为nginx，就可以在`CMD ["nginx"]`之前将用户设置为`nginx`。

   如果在运行`docker run`命令时设置了`-u 用户名`参数，那么将覆盖`USER`指令设置的用户。

   ### `WORKDIR`指令

   ```
   WORKDIR 路径
   ```

   `WORKDIR`指令用于设置执行`RUN`指令、`CMD`指令和`ENTRYPOINT`指令执行时的工作目录。在Dockerfile中可以多次设置`WORKDIR`，在每次设置之后的命令将使用新的路径。

   ### `ONBUILD`指令

   ```
   ONBUILD 指令
   ```

   `ONBUILD`指令用于设置一些指令，当本镜像作为基础镜像被其他Dockerfile用`FROM`指令引用时，在所有其他指令执行之前先执行这些指令。

# Compose

## 基本命令

```bash
# 批量启动
docker-compose up -d # 改名了实现了构建镜像、(重新)创建服务、启动服务，并关联服务相关的容器操作

# 启动
docker-compose start # -f指定配置文件

# 停止
docker-compose stop
```

## docker-compose.yaml

### 参考

```yaml
version: '2'
services:
  web:
    image: dockercloud/hello-world
    ports:
      - 8080
    networks:
      - front-tier
      - back-tier

  redis:
    image: redis
    links:
      - web
    networks:
      - back-tier

  lb:
    image: dockercloud/haproxy
    ports:
      - 80:80
    links:
      - web
    networks:
      - front-tier
      - back-tier
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock 

networks:
  front-tier:
    driver: bridge
  back-tier:
driver: bridge
```

### version

version是compose的版本，下表是compose版本与docker版本对照表：

### services

services是用来配置定义每个容器启动参数，每个service就是一个容器，services下一级配置即是服务名称，例如上面示例中的redis, db等。

###  image

image是指定服务的镜像名称或镜像 ID。如果镜像在本地不存在，Compose 将会尝试拉取这个镜像。

###  build

服务除了可以基于指定的镜像，还可以基于一份 Dockerfile，在使用 up 启动之时执行构建任务，这个构建标签就是 build，它可以指定 Dockerfile 所在文件夹的路径。Compose 将会利用它自动构建这个镜像，然后使用这个镜像启动服务容器。

```
build: /path/to/build/dir
```

也可以是相对路径，只要上下文确定就可以读取到 Dockerfile。

```
build: ./dir
```

设定上下文根目录，然后以该目录为准指定 Dockerfile。

```
build:
  context: ../
  dockerfile: path/of/Dockerfile
```

> 注意 build 都是一个目录，如果你要指定 Dockerfile 文件需要在 build 标签的子级标签中使用 dockerfile 标签指定，如上面的例子。
>  如果你同时指定了 image 和 build 两个标签，那么 Compose 会构建镜像并且把镜像命名为 image 后面的那个名字。

既然可以在 docker-compose.yml 中定义构建任务，那么一定少不了 arg 这个标签，就像 Dockerfile 中的 ARG 指令，它可以在构建过程中指定环境变量，但是在构建成功后取消，在 docker-compose.yml 文件中也支持这样的写法：

```
build:
  context: .
  args:
    buildno: 1
    password: secret
```

下面这种写法也是支持的，一般来说下面的写法更适合阅读。

```
build:
  context: .
  args:
    - buildno=1
    - password=secret
```

与 ENV 不同的是，ARG 是允许空值的。例如：

```
args:
  - buildno
  - password
```

这样构建过程可以向它们赋值。

> 注意：YAML 的布尔值（true, false, yes, no, on, off）必须要使用引号引起来（单引号、双引号均可），否则会当成字符串解析。

### command

使用 command 可以覆盖容器启动后默认执行的命令。

```
command: bundle exec thin -p 3000
```

也可以写成类似 Dockerfile 中的格式：

```
command: [bundle, exec, thin, -p, 3000]
```

### container_name

Compose 的容器名称格式是：<项目名称><服务名称><序号>
虽然可以自定义项目名称、服务名称，但是如果你想完全控制容器的命名，可以使用这个标签指定：

```
container_name: app
```

这样容器的名字就指定为 app 了。

### depends_on

在使用 Compose 时，最大的好处就是少打启动命令，但是一般项目容器启动的顺序是有要求的，如果直接从上到下启动容器，必然会因为容器依赖问题而启动失败。
 例如在没启动数据库容器的时候启动了应用容器，这时候应用容器会因为找不到数据库而退出，为了避免这种情况我们需要加入一个标签，就是 depends_on，这个标签解决了容器的依赖、启动先后的问题。
 例如下面容器会先启动 redis 和 db 两个服务，最后才启动 web 服务：

```
version: '2'
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```

注意的是，默认情况下使用 docker-compose up web 这样的方式启动 web 服务时，也会启动 redis 和 db 两个服务，因为在配置文件中定义了依赖关系。

### dns

和 --dns 参数一样用途，格式如下：

```
dns: 8.8.8.8
```

也可以是一个列表：

```
dns:
  - 8.8.8.8
  - 9.9.9.9
```

此外 dns_search 的配置也类似：

```
dns_search: example.com
dns_search:
  - dc1.example.com
  - dc2.example.com
```

### environment

和 arg 有几分类似，这个标签的作用是设置镜像变量，它可以保存变量到镜像里面，也就是说启动的容器也会包含这些变量设置，这是与 arg 最大的不同。
 一般 arg 标签的变量仅用在构建过程中。而 environment 和 Dockerfile 中的 ENV 指令一样会把变量一直保存在镜像、容器中，类似 docker run -e 的效果。

```
environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:
 
environment:
  - RACK_ENV=development
  - SHOW=true
  - SESSION_SECRET
```

### extra_hosts

添加主机名的标签，就是往/etc/hosts文件中添加一些记录，与Docker client的--add-host类似：

```
extra_hosts:
 - "somehost:162.242.195.82"
 - "otherhost:50.31.209.229"
```

启动之后查看容器内部hosts：

```
162.242.195.82  somehost
50.31.209.229   otherhost
```

### labels

向容器添加元数据，和Dockerfile的LABEL指令一个意思，格式如下：

```
labels:
  com.example.description: "Accounting webapp"
  com.example.department: "Finance"
  com.example.label-with-empty-value: ""
labels:
  - "com.example.description=Accounting webapp"
  - "com.example.department=Finance"
  - "com.example.label-with-empty-value"
```

映射端口的标签。
 使用HOST:CONTAINER格式或者只是指定容器的端口，宿主机会随机映射端口。

```
ports:
 - "3000"
 - "8000:8000"
 - "49100:22"
 - "127.0.0.1:8001:8001"
```

> 注意：当使用HOST:CONTAINER格式来映射端口时，如果你使用的容器端口小于60你可能会得到错误得结果，因为YAML将会解析xx:yy这种数字格式为60进制。所以建议采用字符串格式。

### volumes

挂载一个目录或者一个已存在的数据卷容器，可以直接使用 [HOST:CONTAINER] 这样的格式，或者使用 [HOST:CONTAINER:ro] 这样的格式，后者对于容器来说，数据卷是只读的，这样可以有效保护宿主机的文件系统。
 Compose的数据卷指定路径可以是相对路径，使用 . 或者 .. 来指定相对目录。
 数据卷的格式可以是下面多种形式：

```
volumes:
  // 只是指定一个路径，Docker 会自动在创建一个数据卷（这个路径是容器内部的）。
  - /var/lib/mysql
 
  // 使用绝对路径挂载数据卷
  - /opt/data:/var/lib/mysql
 
  // 以 Compose 配置文件为中心的相对路径作为数据卷挂载到容器。
  - ./cache:/tmp/cache
 
  // 使用用户的相对路径（~/ 表示的目录是 /home/<用户目录>/ 或者 /root/）。
  - ~/configs:/etc/configs/:ro
 
  // 已经存在的命名的数据卷。
  - datavolume:/var/lib/mysql
```

如果你不使用宿主机的路径，你可以指定一个volume_driver。

```
volume_driver: mydriver
```

### networks

加入指定网络，格式如下：

```
services:
  some-service:
    networks:
     - some-network
     - other-network
```

关于这个标签还有一个特别的子标签aliases，这是一个用来设置服务别名的标签，例如：

```
services:
  some-service:
    networks:
      some-network:
        aliases:
         - alias1
         - alias3
      other-network:
        aliases:
         - alias2
```

相同的服务可以在不同的网络有不同的别名。

### network_mode

网络模式，与Docker client的--net参数类似，只是相对多了一个service:[service name] 的格式。
 例如：

```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

# 参考

<https://juejin.im/post/5b319c3cf265da597d0aa79d>

<https://github.com/zhangpeihao/LearningDocker/blob/master/manuscript/04-WriteDockerfile.md>

<https://www.jianshu.com/p/4f14637f4b35>

<http://dockone.io/article/128>