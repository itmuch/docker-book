# Docker安装

## 2.1 CentOS

### 2.1.1  系统要求  

* **CentOS 7**或更高版本
* `centos-extras` 仓库必须处于启用状态，该仓库默认启用，但如果您禁用了该仓库，请按照<https://wiki.centos.org/AdditionalResources/Repositories> 中的描述重新启用。
* 建议使用`overlay2` 存储驱动

### 2.1.2 yum安装

#### 2.1.2.1 卸载老版本的Docker

在CentOS中，老版本Docker名称是`docker` 或`docker-engine` ，而Docker CE的软件包名称是`docker-ce` 。因此，如已安装过老版本的Docker，需使用如下命令卸载。

```Shell
sudo yum remove docker \
                  docker-common \
                  docker-selinux \
                  docker-engine
```

需要注意的是，执行该命令只会卸载Docker本身，而不会删除Docker存储的文件，例如镜像、容器、卷以及网络文件等。这些文件保存在`/var/lib/docker` 目录中，需要手动删除。

#### 2.1.2.2 安装仓库

1. 执行以下命令，安装Docker所需的包。其中，`yum-utils` 提供了`yum-config-manager` 工具；`device-mapper-persistent-data` 及 `lvm2` 则是`devicemapper` 存储驱动所需的包。

   ```Shell
   sudo yum install -y yum-utils device-mapper-persistent-data lvm2
   ```

2. 执行如下命令，安装`stable` 仓库。必须安装`stable` 仓库，即使你想安装`edge` 或`test` 仓库中的Docker构建版本。

   ```Shell
   sudo yum-config-manager \
       --add-repo \
       https://download.docker.com/linux/centos/docker-ce.repo
   ```

3. [**可选**] 执行如下命令，启用`edge` 及`test` 仓库。edge/test仓库其实也包含在了`docker.repo` 文件中，但默认是禁用的，可使用以下命令来启用。

   ```Shell
   sudo yum-config-manager --enable docker-ce-edge    # 启用edge仓库
   sudo yum-config-manager --enable docker-ce-test    # 启用test仓库
   ```

   如需再次禁用，可加上`--disable` 标签。例如，执行如下命令即可禁用edge仓库。

   ```shell
   sudo yum-config-manager --disable docker-ce-edge
   ```

   **TIPS**：从Docker 17.06起，stable版本也会发布到edge以及test仓库中。


#### 2.1.2.3 安装Docker CE

1. 执行以下命令，更新`yum`的包索引

   ```Shell
   sudo yum makecache fast
   ```

2. 执行如下命令即可安装最新版本的Docker CE

   ```Shell
   sudo yum install docker-ce
   ```

3. 在生产环境中，可能需要指定想要安装的版本，此时可使用如下命令列出当前可用的Docker版本。

   ```shell
   yum list docker-ce.x86_64  --showduplicates | sort -r
   ```

   这样，列出版本后，可使用如下命令，安装想要安装的Docker CE版本。

   ```shell
   sudo yum install docker-ce-<VERSION>
   ```

4. 启动Docker

   ```Shell
   sudo systemctl start docker
   ```

5. 验证安装是否正确。

   ```shell
   sudo docker run hello-world
   ```

   这样，Docker将会下载测试镜像，并使用该镜像启动一个容器。如能够看到类似如下的输出，则说明安装成功。

   ```
   Unable to find image 'hello-world:latest' locally
   latest: Pulling from library/hello-world
   b04784fba78d: Pull complete
   Digest: sha256:f3b3b28a45160805bb16542c9531888519430e9e6d6ffc09d72261b0d26ff74f
   Status: Downloaded newer image for hello-world:latest

   Hello from Docker!
   This message shows that your installation appears to be working correctly.

   To generate this message, Docker took the following steps:
    1. The Docker client contacted the Docker daemon.
    2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    3. The Docker daemon created a new container from that image which runs the
       executable that produces the output you are currently reading.
    4. The Docker daemon streamed that output to the Docker client, which sent it
       to your terminal.

   To try something more ambitious, you can run an Ubuntu container with:
    $ docker run -it ubuntu bash

   Share images, automate workflows, and more with a free Docker ID:
    https://cloud.docker.com/

   For more examples and ideas, visit:
    https://docs.docker.com/engine/userguide/
   ```

#### 2.1.2.4 升级Docker CE

如需升级Docker CE，只需执行如下命令：

```shell
sudo yum makecache fast
```

然后按照安装Docker的步骤，即可升级Docker。



#### 2.1.2.5 参考文档

CentOS 7安装Docker官方文档：<https://docs.docker.com/engine/installation/linux/docker-ce/centos/> ，文档中还讲解了在CentOS 7中安装Docker CE的其他方式，本文不作赘述。

### 2.1.3 shell一键安装

```shell
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

搞定一切。

## 2.2 Ubuntu

### 2.2.1 系统要求

* Docker支持以下版本的Ubuntu，要求64位。
  * Zesty 17.04
  * Xenial 16.04 (LTS)
  * Trusty 14.04 (LTS)
* 支持运行的平台：`x86_64` 、`armhf` 、`s390x(IBM Z)` 。其中，如选择IBM Z，那么只支持Ubuntu Xenial以及Zesty。
* 本文使用**Ubuntu 16.04 LTS**，下载地址：<http://cn.ubuntu.com/download/>

### 2.2.2 安装步骤

#### 2.2.2.1 卸载老版本Docker

在Ubuntu中，老版本的软件包名称是`docker` 或者`docker-engine` ，而Docker CE的软件包名称是`docker-ce` 。因此，如已安装过老版本的Docker，需要先卸载掉。执行以下命令，即可卸载老版本的Docker及其依赖。

```shell
sudo apt-get remove docker docker-engine docker.io
```

需要注意的是，执行该命令只会卸载Docker本身，而不会删除Docker内容，例如镜像、容器、卷以及网络。这些文件保存在`/var/lib/docker` 目录中，需要手动删除。

#### 2.2.2.2 Ubuntu Trusty 14.04 额外建议安装的包

除非你有不得已的苦衷，否则强烈建议安装`linux-image-extra-*` 软件包，以便于Docker使用`aufs` 存储驱动。执行如下命令，即可安装`linux-image-extra-*` 。

```shell
sudo apt-get update

sudo apt-get install \
    linux-image-extra-$(uname -r) \
    linux-image-extra-virtual
```

对于Ubuntu 16.04或更高版本，Linux内核包含了对OverlayFS的支持，Docker CE默认会使用`overlay2` 存储驱动。

#### 2.2.2.3 安装仓库

1. 执行如下命令，更新`apt` 的包索引。

   ```shell
   sudo apt-get update
   ```

2. 执行如下命令，从而允许`apt` 使用HTTPS仓库。

   ```shell
   sudo apt-get install \
       apt-transport-https \
       ca-certificates \
       curl \
       software-properties-common
   ```

3. 添加Docker官方的GPG key

   ```shell
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   ```

   确认指纹是`9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88` 。

   ```shell
   sudo apt-key fingerprint 0EBFCD88
   ```

4. 执行如下命令，安装`stable` 仓库。无论如何都必须安装`stable` 仓库，即使你想安装`edge` 或`test` 仓库中的Docker构建。如需添加`edge` 或`test` 仓库，可在如下命令中的“stable" 后，添加`edge` 或`test` 或两者。请视自己Ubuntu所运行的平台来执行如下命令。
   **NOTE**：如下命令中的`lsb_release -cs` 子命令用于返回您Ubuntu的发行版名称，例如`xenial` 。有时，在例如Linux Mint这样的发行版中，您可能需要将如下命令中的`$(lsb_release -cs)` 更改为系统的父级Ubuntu发行版。例如，如果您使用的是Linux Mint Rafaela，则可以使用`trusty` 。
   **amd64**:

   ```shell
   $ sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
   ```

   **armhf**:

   ```shell
   $ sudo add-apt-repository \
      "deb [arch=armhf] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
   ```

   **s390x**:

   ```Shell
   $ sudo add-apt-repository \
      "deb [arch=s390x] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
   ```
   **NOTE**：从Docker 17.06起，stable版本也会发布到edge以及test仓库中。


#### 2.2.2.4 安装Docker CE

1. 执行如下命令，更新`apt` 包索引。

   ```shell
   sudo apt-get update
   ```

2. 执行如下命令，即可安装最新版本的Docker CE。任何已存在的Docker将会被覆盖安装。

   ```shell
   sudo apt-get install docker-ce
   ```
   **WARNING**：如启用了多个Docker仓库，使用命令apt-get install 或apt-get update 命令安装或升级时，如未指定版本，那么将会安装最新的版本。这可能不适合您的稳定性要求。

3. 在生产环境中，我们可能需要指定想要安装的版本，此时可使用如下命令列出当前可用的Docker版本。

   ```shell
   apt-cache madison docker-ce
   ```
   这样，列出版本后，可使用如下命令，安装想要安装的Docker CE版本。
   ```shell
   sudo apt-get install docker-ce=<VERSION>
   ```
   Docker daemon会自动启动。

4. 验证安装是否正确。

   ```shell
   sudo docker run hello-world
   ```


#### 2.2.2.5 升级Docker CE

如需升级Docker CE，只需执行如下命令：

```shell
sudo apt-get update
```

然后按照安装Docker的步骤，即可升级Docker。

#### 2.2.2.6 参考文档

Ubuntu安装Docker官方文档：<https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/> ，文档还讲解了在Ubuntu中安装Docker CE的其他方式，本文不作赘述。



## 2.3 macOS

### 2.3.1 系统要求

macOS Yosemite 10.10.3或更高版本

### 2.3.2 安装步骤

* 前往<https://store.docker.com/editions/community/docker-ce-desktop-mac> ，点击页面右侧的“Get Docker”按钮，下载安装包；
* 双击即可安装。




## 2.4 Windows(docker for windows)

### 2.4.1 系统要求

Windows 10 Professional 或 Windows 10 Enterprise X64

对于Win 7，可使用Docker Toolbox（不建议使用）

### 2.4.2 安装步骤

* 前往<https://store.docker.com/editions/community/docker-ce-desktop-windows> ，点击页面右侧的“Get Docker”按钮，下载安装包；
* 双击即可安装。





## 2.5 其他系统

详见官方文档：<https://docs.docker.com/engine/installation/>



## 2.6 加速安装

注册阿里云，参考该页面的内容安装即可：<https://cr.console.aliyun.com/#/accelerator>