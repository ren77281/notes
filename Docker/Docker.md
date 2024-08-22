```toc
```
## docker的本质
Docker本质是LXC的增加版，它本身不是容器，而是容器的易用工具。容器是Linux内核中的技术，docker只是把这种技术在使用上简单普及

### docker与传统虚拟机的区别
1. 虚拟化级别
 - docker采用容器虚拟化技术，在宿主机的内核上创建隔离的用户空间，容器之间**共享**宿主机的**内核**
 - 传统虚拟机对硬件层进行抽象，从而虚拟化一个完整的操作系统，每个虚拟机都有自己的操作系统内核
2. 启动时间
  - 显然，docker启动很快，因为docker共享内核。而传统虚拟机启动时需要加载自己内核
3. 资源利用率
  - docker共享内核，在资源利用方面更加高效。而传统虚拟机需要维护自己的内核，占用的资源显然更多
4. 管理与部署
  - docker的安装、管理方便，有着丰富的生态支持，体系化部署，实现了自动化。而传统虚拟机的管理需要更专业复杂的运维知识，同时需要手动部署，速度慢
5. 隔离性
  - docker的隔离为进程级别，传统虚拟机的隔离为系统级别
6. 封装程度
  - docker需要打包项目代码与依赖，粒度细。而传统虚拟机需要打包整个操作系统，粒度粗

### 为何docker比虚拟机资源利用率高，启动快？
docker有着比虚拟机更少的抽象层，程序不需要借助Hypervisor（虚拟机管理程序，用来实现硬件资源虚拟化）就能直接调用内核。这一点体现了docker的运行效率高
因为docker直接调用内核，所以不需要Guest OS（运行在虚拟环境中的操作系统），进而节省了Guest OS占用的资源。启动docker时也不需要引导、加载Guest OS，所以启动快

### docker与JVM的区别
1. 性能：docker几乎没有性能损失，而JVM需要占用一定的cpu和内存
2. 虚拟层面：docker基于操作系统层，而JVM基于函数库层，更加上层
3. 隔离性：docker通过命名空间实现了一定程度的资源隔离，JVM实现了完全的应用程序级别隔离，不同Java程序运行在同一主机上不会相互影响
4. 语言支持：docker可以容纳多种编程语言，而JVM只能容纳Java语言


## 安装
查看当前机器的cpu架构
```bash
uname -a 
```
docker的默认安装目录为`/var/lib/docker`，检查是否存在数据
如果有则删除两个目录
```bash
rm -rf /var/lib/docker
rm -rf /var/lib/containerd
```

创建目录以存放公钥文件
```bash
mkdir -m 0755 -p /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
```

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signedby=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7EA0A9C3F273FCD8
```

```bash
apt update
apt-get update
```

```bash
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

```bash
#配置加载
sudo systemctl daemon-reload
#启动服务
sudo systemctl start docker
#开启启动
sudo systemctl enable docker
#查看服务状态
sudo systemctl status docker
```

若遇到错误可以通过查看系统日志排查错误：
journalctl 是操作系统日志查看命令
-e 表示从末尾看
-u 表示看哪个系统组件的，我们的组件是 docker
journalctl -eu docker

```bash
docker run hello-world
```
![image.png](https://s2.loli.net/2023/10/24/8CdgJWz4EtpTP3Y.png)

可以在/etc/docker目录下创建一个daemon.json文件，添加docker镜像源到里面
执行，加载配置并重启docker服务
```bash
systemctl daemon-reload
systemctl restart docker
```
由于docker拉取的镜像一般存储在/var/lib/docker目录下，为防止目录被打满，我们需要将其挂载到一个很大的磁盘中
在daemon.json添加
```
"data-root": "/data/var/lib/docker"
```
执行
```bash
systemctl daemon-reload
systemctl restart docker
```
docker info出现以下字段表示修改成功
![image.png](https://s2.loli.net/2023/10/24/zswDvFaVuRdJixM.png)
docker的默认目录就被修改到我们指定的目录下

## 镜像仓库

### docker login
`docker login` 命令用于登录到 Docker Hub 或其他 Docker 镜像仓库，以便进行身份验证并执行需要登录的操作。以下是 `docker login` 命令的语法和可选参数：

```
docker login [OPTIONS] [SERVER]
```

可选参数：

- `OPTIONS`：用于配置登录的选项，包括 `--username` 和 `--password-stdin`。

- `SERVER`：可选参数，用于指定要登录的 Docker 镜像仓库的 URL。如果未提供此参数，默认将登录到 Docker Hub。

以下是 `docker login` 命令的可选参数的详细信息：

- `--username` 或 `-u`：用于指定登录时的用户名。

- `--password` 或 `-p`：用于指定登录时的密码。请注意，直接在命令行中提供密码不是安全的做法，因为密码会在命令历史记录中可见。建议使用 `--password-stdin` 选项结合输入重定向的方式来提供密码，以增加安全性。

- `--password-stdin`：使用此选项，您可以将密码通过标准输入输入，而不是在命令行中明文显示密码。例如：

  ```bash
  echo your_password | docker login --username your_username --password-stdin
  ```

- `--email`：用于指定登录时的电子邮件地址。虽然通常不再需要提供电子邮件地址，但某些镜像仓库可能要求它。

- `--server`：用于指定要登录的 Docker 镜像仓库的 URL。如果不提供此选项，默认将登录到 Docker Hub。

示例：

```bash
docker login --username your_username --password-stdin
```

这是一个示例，演示如何使用 `--username` 和 `--password-stdin` 来登录到 Docker 镜像仓库，并通过标准输入方式提供密码以增加安全性。确保替换 `your_username` 和 `your_password` 为您的实际用户名和密码。

### docker pull
`docker pull` 命令用于从 Docker 镜像仓库中拉取镜像到本地。以下是 `docker pull` 命令的语法和可选参数：

```
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

可选参数：

- `OPTIONS`：用于配置拉取操作的选项，如 `--all-tags`、`--disable-content-trust` 等。

- `NAME[:TAG|@DIGEST]`：指定要拉取的镜像的名称、标签或摘要。`TAG` 通常是镜像的版本标签，例如 `ubuntu:18.04`。`DIGEST` 是镜像的唯一摘要，用于确保拉取的镜像是指定版本的。

以下是 `docker pull` 命令的可选参数的详细信息：

- `--all-tags`：如果指定了镜像名称（不包括标签或摘要），则该选项允许拉取指定镜像的所有可用标签。

- `--disable-content-trust`：此选项用于禁用 Docker Content Trust，它是一种镜像内容的签名和验证机制。默认情况下，Docker Content Trust 启用。如果您想禁用内容信任，可以使用此选项。

- `--platform`：用于指定要拉取的镜像所针对的平台。在多平台构建和部署的情况下，这可以用于选择特定的平台。

示例：

```bash
docker pull ubuntu:20.04
```

在此示例中，我们使用 `docker pull` 命令拉取名为 `ubuntu` 的镜像，并指定标签 `20.04`，以拉取 Ubuntu 20.04 版本的镜像。可以在 `NAME[:TAG]` 部分指定要拉取的镜像名称和标签。

```bash
docker pull --all-tags ubuntu
```

在此示例中，我们使用 `--all-tags` 选项来拉取名为 `ubuntu` 的镜像的所有可用标签。

请注意，可选参数的可用性可能会因 Docker 版本的不同而有所变化，因此建议查看您使用的 Docker 版本的文档以获取最准确的信息。

### docker search
`docker search` 是 Docker 命令行工具用于在 Docker Hub 或其他 Docker 镜像仓库中搜索镜像的命令。它允许查找特定名称或关键字的镜像，以便找到感兴趣的镜像。以下是如何使用 `docker search` 命令的步骤：

1. 打开终端或命令行窗口，并确保已安装 Docker 并启动 Docker 客户端。

2. 在终端中输入以下命令，以搜索镜像。将 `<term>` 替换为要搜索的关键字或镜像名称。

   ```bash
   docker search <term>
   ```

   示例：如果要搜索所有与 "nginx" 相关的镜像，可以运行以下命令：

   ```bash
   docker search nginx
   ```

3. 执行命令后，Docker 将从 Docker Hub 或其他镜像仓库检索与搜索条件匹配的镜像，并将它们列出在终端中。

   输出通常包括以下列信息：
   - NAME：镜像的名称。
   - DESCRIPTION：关于镜像的简短描述。
   - STARS：用户对该镜像的星级评价。
   - OFFICIAL：指示是否官方维护的标志（如果为 "OK"，则表示官方维护）。
   - AUTOMATED：指示是否由自动化构建维护的标志。

以下是一个示例：

```bash
$ docker search nginx
NAME                             DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
nginx                            Official build of Nginx.                        11583               OK
jwilder/nginx-proxy              Automated Nginx reverse proxy for docker co…   1254
richarvey/nginx-php-fpm          Container running Nginx + PHP-FPM capable of…   290                                     [OK]
...
```

4. 根据搜索结果，也可以选择合适的镜像名称，并使用 `docker pull` 命令来拉取该镜像到本地，如之前所述。

`docker search` 是一个有用的工具，用于浏览 Docker Hub 上的可用镜像并找到需要的镜像。如果正在使用私有镜像仓库，也可以使用类似的方式进行搜索。

### docker logout
`docker logout` 命令用于注销 Docker 客户端的登录凭证，使用户从 Docker 镜像仓库中注销。它有以下语法和可选参数：

```
docker logout [OPTIONS] [SERVER]
```

可选参数：

- `OPTIONS`：暂时没有任何可选选项。

- `SERVER`：可选参数，用于指定要注销的 Docker 镜像仓库的 URL。如果未提供此参数，默认将注销 Docker Hub 帐户。如果您需要注销不同的镜像仓库，可以指定该仓库的 URL 作为参数。

以下是一些示例：

1. 注销当前 Docker Hub 帐户：

   ```bash
   docker logout
   ```

2. 注销指定镜像仓库（示例中为私有仓库的URL）：

   ```bash
   docker logout my-registry.example.com
   ```

请注意，目前的 Docker 版本（截止到我所知的 2021 年 9 月）可能不提供任何额外的选项或标志来扩展 `docker logout` 命令的功能。这个命令主要用于简单的注销操作，以使用户需要重新登录以执行需要身份验证的操作。

### docker images
`docker image` 命令已在较新的 Docker 版本中更改为 `docker images` 命令，用于列出本地 Docker 镜像。以下是 `docker images` 命令的语法和可选参数：

```
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

可选参数：

- `OPTIONS`：用于配置列出镜像的选项，如 `--all`, `--filter`, `--quiet` 等。

- `REPOSITORY[:TAG]`：用于过滤列出的镜像。您可以指定镜像的仓库名称和标签，以仅显示符合条件的镜像。

以下是 `docker images` 命令的可选参数的详细信息：

- `--all` 或 `-a`：此选项用于显示所有镜像，包括中间层镜像。默认情况下，`docker images` 仅显示顶层镜像。

- `--filter`：使用此选项可以根据指定的条件来筛选镜像。条件可以是 `dangling=true`（仅显示悬空镜像）或自定义过滤条件。

- `--quiet` 或 `-q`：此选项用于仅显示镜像的 ID，而不显示其他详细信息。

- `--digests`：使用 `--digests` 选项，`docker images` 命令将显示每个镜像的摘要（digest），而不仅仅是标签和ID。摘要是镜像内容的唯一标识，通常用于确保拉取的镜像是完全一致的。
  这将显示每个镜像的摘要信息。

- `--no-trunc`：使用 `--no-trunc` 选项，`docker images` 命令将不会截断（truncates）输出，而是显示完整的镜像信息，包括较长的仓库和标签名称。

示例：

```bash
docker images
```

这将列出本地所有镜像，包括镜像的仓库名称、标签、ID、创建时间等详细信息。

```bash
docker images -a
```

这将列出本地所有镜像，包括中间层镜像。

```bash
docker images --filter "dangling=true"
```

这将列出所有悬空（未与任何标签关联）的镜像。

```bash
docker images -q
```

这将仅显示本地镜像的 ID，而不包括其他详细信息。

请注意，可选参数的可用性可能会因 Docker 版本的不同而有所变化，因此建议查看您使用的 Docker 版本的文档以获取最准确的信息。

### docker image inspect
查看有关单个镜像的详细信息。以下是 `docker image inspect` 命令的详细解释：

语法：

```
docker image inspect [OPTIONS] IMAGE [IMAGE...]
```

- `OPTIONS`：用于配置检查操作的选项，例如 `--format` 等。

- `IMAGE`：要检查的一个或多个镜像的名称、ID 或摘要。

可选参数：

- `--format`：使用此选项可以指定输出的格式，以便更好地满足您的需求。您可以使用 Go 模板来定义自定义输出格式。例如，您可以使用 `--format '{{json .}}'` 来获取 JSON 格式的输出。这使您可以提取有关镜像的各种详细信息。

示例：

```bash
docker image inspect --format '{{.Config.Image}}' ubuntu:20.04
```

这将使用 `--format` 选项获取指定镜像的 `Image` 属性，以显示 Ubuntu 20.04 镜像的摘要信息。

`docker image inspect` 命令允许您深入了解特定镜像的详细信息，包括镜像的配置、层、标签、摘要等。通过使用 `--format` 选项，您可以根据需要选择提取镜像信息的特定部分，以满足您的要求。这对于了解镜像的构建过程、依赖关系和其他有用信息非常有帮助。

### docker tag
`docker tag` 命令用于为 Docker 镜像创建新的标签，以便更方便地识别和管理镜像。通过为镜像添加标签，您可以为同一镜像创建多个引用，方便了版本管理、组织和共享镜像。以下是 `docker tag` 命令的详细解释：

语法：

```
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

- `SOURCE_IMAGE[:TAG]`：要添加标签的源镜像名称和标签。如果不提供标签，将默认使用 "latest"。

- `TARGET_IMAGE[:TAG]`：为源镜像指定的新标签。这将为源镜像创建一个别名，以便您可以使用不同的名称来引用相同的镜像。

示例：

```bash
docker tag myapp:1.0 myapp:latest
```

这将为名为 `myapp` 的镜像创建一个新标签 `latest`，使 `myapp:latest` 引用与 `myapp:1.0` 相同的镜像。这对于创建别名、版本控制和标记特定镜像非常有用。

另一个常见用例是为镜像添加 registry 地址，以便将其推送到不同的 Docker 镜像仓库。例如：

```bash
docker tag myapp:1.0 myregistry.com/myapp:1.0
```

这将为 `myapp:1.0` 镜像添加一个新标签 `myregistry.com/myapp:1.0`，以便将其推送到 `myregistry.com` 镜像仓库。

`docker tag` 命令允许您更好地组织和标记您的镜像，使其更易于管理和共享。这在团队协作和自动化部署中特别有用。

## nginx安装
查看是否使用apt安装过nginx
```bash
dpkg -l | grep nginx
```

查看当前系统是否只在运行nginx
```bash
ps -ef | grep nginx
```

彻底卸载nginx，删除依赖和配置
```bash
apt --purge autoremove nginx -y
```
## docker run
`docker run` 命令用于创建并运行一个新的 Docker 容器。以下是 `docker run` 命令的语法和可选参数：

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

- `OPTIONS`：用于配置容器的选项，例如 `--name`、`--detach`、`--port` 等。
- `IMAGE`：要基于的镜像名称或 ID。
- `COMMAND`：在容器内执行的命令。这是可选的，如果不提供，则使用镜像中的默认命令。
- `ARG...`：传递给命令的参数。也是可选的。

以下是 `docker run` 命令的一些常用选项和详细信息：

- `--name`：用于指定容器的名称，使容器易于识别。例如：`--name my-container`。
- `--detach` 或 `-d`：将容器以后台模式运行，即不会附着到容器的标准输入（stdin）上，允许您在终端中继续使用。
- `--interactive` 或 `-i`：保持容器的标准输入（stdin）打开，以便与容器进行交互。
- `-h`：指定容器的hostname，例如：`-h mycontainer`，使用该容器交互时，其主机名为mycontainer
- `--tty` 或 `-t`：分配一个伪终端（TTY）以允许交互。通常与 `-i` 一起使用以模拟终端会话。
- `--rm`：容器退出后立即删除容器。非常适用于一次性任务。
- `--port` 或 `-p`：用于将容器的端口映射到主机的端口。例如，`-p 80:8080` 将容器的端口 8080 映射到主机的端口 80。
- `-P`：容器内部端口映射主机的随机端口
- `--volume` 或 `-v`：用于挂载主机目录或文件到容器内，以实现数据持久性和共享。例如，`-v /host/path:/container/path`。
- `--env`：用于设置环境变量。例如，`-e "ENV_VARIABLE=value"`。
- `--network`：用于将容器连接到指定的 Docker 网络。例如，`--network my-network`。
- `--link`：用于链接一个容器到另一个容器，以便它们可以互相通信。这是一种过时的选项，不建议使用。
- `--cpu-shares`：用于设置容器的 CPU 分配份额，以控制容器的 CPU 使用。例如，`--cpu-shares 512`。
- `--cpuset-cpus="0-2"`or`--cpuset-cpus="0,1,2"`：绑定容器到指定cpu运行
- `--memory`，`-m`：用于设置容器的内存限制。例如，`--memory 512m` 表示容器最多可以使用 512MB 内存。

这些选项可以根据您的需求来配置容器的运行方式。 `docker run` 命令非常强大，可以用于创建各种不同类型的容器，从简单的后台服务到交互式终端会话。根据您的具体用例，您可以选择适当的选项来配置容器。

拉取一个镜像
```bash
docker pull centos:7
```

查看所有镜像，不带选项表示查看正在运行的镜像
```bash
docker ps -a
```

```bash
# 交互式运行容器，只有-t的话，没有命令行提示符
docker run -it centos:7 bash 
# 退出容器
exit
```

```bash
# 后台运行容器
docker run -d nginx:1.24.0
```

### docker ps
`docker ps` 命令用于列出正在运行的 Docker 容器。它提供了有关容器的基本信息，如容器 ID、镜像名称、状态、端口映射等。以下是 `docker ps` 命令的详细解释以及可选参数：

语法：

```
docker ps [OPTIONS]
```

可选参数：

- `-a` 或 `--all`：默认情况下，`docker ps` 仅显示运行中的容器。使用 `-a` 选项将显示所有容器，包括已停止的容器。

- `-l` 或 `--last`：使用 `-l` 选项，可以显示最后一次运行的容器，包括已经停止的容器。

- `-n` 或 `--latest`：使用 `-n` 选项，可以限制显示最新创建的 `n` 个容器。例如，`docker ps -n 5` 将显示最新创建的 5 个容器。

- `--no-trunc`：使用 `--no-trunc` 选项可以显示完整的容器信息，而不截断显示。这对于查看较长的容器名称或命令行参数非常有用。

- `-q`：使用 `-q` 选项，可以仅显示容器的 ID，而不包括其他详细信息。

- `--filter`：使用 `--filter` 选项，可以根据自定义条件筛选要显示的容器。例如，`docker ps --filter "status=running"` 将只显示运行中的容器。

- `--format`：使用 `--format` 选项，可以自定义输出的格式，使用 Go 模板来定义自己的输出格式。

示例：

1. 列出所有运行中的容器：

   ```bash
   docker ps
   ```

2. 列出所有容器，包括已停止的容器：

   ```bash
   docker ps -a
   ```

3. 列出最后一个运行的容器：

   ```bash
   docker ps -l
   ```

4. 仅显示容器的 ID：

   ```bash
   docker ps -q
   ```

`docker ps` 命令提供了多种选项，以便根据您的需求查看和筛选容器的信息。这对于管理和监控 Docker 容器非常有用，特别是在具有大量容器的环境中。根据您的具体用例，可以选择适当的选项来列出所需的容器信息。

### 创建nginx容器并修改首页内容
交互式创建容器，并启动bash
```bash
docker run -p 8091:80 --name mynginx6 -h web.com -it nginx:1.24.0 bash
```
运行nginx，因为这个容器只运行了bash出现，未运行nginx程序
```bash
nginx
```
进入默认网页所在的目录
```bash
cd /usr/share/nginx/html
```
修改默认网页的html资源
```bash
echo "hello docker" > index.html
```
### 在docker hub上创建私有仓库
修改原有标签
![image.png](https://s2.loli.net/2023/10/24/JejQL9I4mGrxtOR.png)
使用docker push推送，推送之前需要docker login登录主机的账号
![image.png](https://s2.loli.net/2023/10/24/lhoWZnTwag7cQGO.png)

## 镜像
- 可以理解为模板或者类，而容器就是实例化的对象
- 包括了用于运行应用程序的所有必需的代码、运行时、系统工具和库，本质是一个只读文件
- 镜像中是一层层的文件系统Union FS，联合文件系统，将几层目录挂载到一起形成了一个虚拟文件系统
  - 可以将联合文件系统看成一层层楼。比如一个C++程序的镜像，最底层的楼是操作系统，其次是该程序需要的运行环境，最后是程序的可执行文件

为什么需要镜像？
- 开发环境与生产环境不同，docker定制了镜像制作的标准，遵循该标准的docker镜像实例化成容器后，能够在不同的环境中运行
- docker的分层技术：通常，镜像的底层包含基本操作系统文件，后续分层包含程序的特定文件与配置。这使得我们在构建和更新镜像时，只需要修改或添加必要的分层，而不需要添加整个镜像结构。总之，分层结构实现了可复用，大大节省了服务端资源
  - 并且在拉取镜像时，docker只会拉取本地不存在的分层，加速了软件分发

缺点：拉取一个很小的可执行文件的镜像，却需要拉取比它大得多的依赖环境
## 镜像的命令
### docker rmi
若镜像（的某一层）正在被其他容器使用，先删除容器`docker rm`再删除镜像`docker rmi`

`docker rmi`是Docker命令行工具中用于删除Docker镜像的命令。它允许您删除本地系统上的一个或多个Docker镜像。下面是`docker rmi`的详细解释以及一些常见的可选参数：

**语法:**
```
docker rmi [OPTIONS] IMAGE [IMAGE...]
```

**参数:**
- `IMAGE`：要删除的Docker镜像的名称或ID。您可以指定一个或多个镜像。

**可选参数:**
1. `-f, --force`：强制删除。即使镜像正在被容器使用，也可以强制删除。请小心使用此选项，因为它可能导致正在运行的容器出现问题。

2. `--no-prune`：不清理未使用的镜像。默认情况下，`docker rmi`会删除不再被容器或其他镜像引用的镜像。使用此选项可以防止清理操作。

3. `-q, --quiet`：静默模式。只输出被删除的镜像的ID，不显示其他信息。

**示例:**

- 删除单个镜像:
  ```
  docker rmi my-image
  ```

- 删除多个镜像:
  ```
  docker rmi image1 image2 image3
  ```

- 强制删除一个镜像，即使它正在被容器使用:
  ```
  docker rmi -f my-image
  ```

- 静默模式，只显示被删除的镜像ID:
  ```
  docker rmi -q my-image
  ```

- 阻止清理未使用的镜像:
  ```
  docker rmi --no-prune my-image
  ```

请谨慎使用`docker rmi`命令，特别是使用`-f`选项，因为它可以导致数据丢失或对正在运行的容器产生不良影响。确保您知道要删除的镜像，并检查是否存在其他容器正在使用它们。

### docker save
`docker save`命令用于将Docker镜像保存到tar归档文件中，以便在不同的Docker环境中复制、分享或备份镜像。这个命令将一个或多个镜像打包成一个tar文件，可以随后使用`docker load`命令来导入这些镜像。以下是`docker save`命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker save [OPTIONS] IMAGE [IMAGE...]
```

**参数:**
- `IMAGE`：要保存的Docker镜像的名称或ID。您可以指定一个或多个镜像。

**可选参数:**
1. `-o, --output`：指定输出文件的路径。默认情况下，`docker save`将输出到标准输出。使用此选项可以将归档保存到指定的文件中，例如：
   ```
   docker save -o my-images.tar my-image
   ```

2. `-q, --quiet`：静默模式。只输出生成的tar文件的名称，不显示其他信息。

**示例:**

- 保存单个镜像到tar文件:
  ```
  docker save -o my-image.tar my-image
  ```

- 保存多个镜像到单个tar文件:
  ```
  docker save -o my-images.tar image1 image2 image3
  ```

- 静默模式，只显示生成的tar文件的名称:
  ```
  docker save -q -o my-images.tar image1 image2 image3
  ```

使用`docker save`可以方便地将一个或多个Docker镜像打包成一个可传输的tar文件，然后在其他Docker环境中使用`docker load`导入这些镜像。这在跨主机或离线环境中复制和分享镜像时非常有用。

### docker load
`docker load` 命令用于从 tar 归档文件中导入 Docker 镜像或存储库。这个命令允许您将先前使用 `docker save` 命令保存的 Docker 镜像导入到本地 Docker 主机中。以下是 `docker load` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker load [OPTIONS]
```

**可选参数:**
- `-i, --input`：指定要导入的 tar 归档文件的路径。默认情况下，`docker load` 会从标准输入读取归档数据，但如果您提供了 `-i` 选项，它将从指定的文件中读取数据，例如：
   ```
   docker load -i my-images.tar
   ```

- `--input` 和 `-i` 选项是等效的，您可以选择其中一个来指定输入文件。

- `--quiet`：静默模式。只输出导入的镜像的名称和标签，不显示其他信息。

**示例:**

- 从 tar 文件导入 Docker 镜像：
  ```
  docker load -i my-images.tar
  ```

- 静默模式，只显示导入的镜像的名称和标签：
  ```
  docker load --quiet -i my-images.tar
  ```

`docker load` 常用于将以前保存的 Docker 镜像从 tar 归档文件中还原到本地 Docker 主机，以便在本地环境中使用这些镜像。这对于在离线环境中部署或在不同的 Docker 主机之间传输镜像非常有用。

### docker history
`docker history` 命令用于查看 Docker 镜像的历史记录。它显示了构建该镜像所采取的每一步操作，包括每个镜像层的创建者、命令、创建时间等信息。这可以帮助您了解镜像的构建过程，包括每个镜像层的来源。以下是 `docker history` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker history [OPTIONS] IMAGE
```

**参数:**
- `IMAGE`：要查看历史记录的 Docker 镜像的名称或 ID。

**可选参数:**
1. `--format`：自定义输出格式。您可以使用 Go 模板来定义要显示的信息，例如：
   ```
   docker history --format "{{.ID}}: {{.CreatedBy}}" my-image
   ```

2. `-q, --quiet`：静默模式。只输出镜像层的 ID，不显示其他信息。

3. `--no-trunc`：不截断输出。默认情况下，输出可能会被截断以适应终端窗口的宽度，使用此选项可以防止截断。

**示例:**

- 查看镜像的历史记录：
  ```
  docker history my-image
  ```

- 静默模式，只显示镜像层的 ID：
  ```
  docker history --quiet my-image
  ```

- 自定义输出格式，显示层的 ID 和创建者：
  ```
  docker history --format "{{.ID}}: {{.CreatedBy}}" my-image
  ```

- 不截断输出，以便查看完整的历史记录信息：
  ```
  docker history --no-trunc my-image
  ```

`docker history` 可以帮助您了解 Docker 镜像的构建历史，包括每个镜像层的创建步骤和来源，这对于调试、分析和了解镜像的组成非常有用。您可以使用可选参数来自定义输出格式以及显示更详细的信息。

### docker image prune
`docker image prune` 命令用于清理本地 Docker 主机上的不再使用的镜像。这有助于释放磁盘空间并删除不再需要的镜像，从而保持 Docker 主机的整洁。以下是 `docker image prune` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker image prune [OPTIONS]
```

**可选参数:**
1. `-a, --all`：清理所有镜像，包括被标记为 `<none>` 的镜像。这将删除未被任何容器使用的所有镜像，而不仅仅是未被标签引用的镜像。

2. `--filter`：允许您使用标签筛选要清理的镜像。例如，`--filter "until=24h"` 将清理所有在最后 24 小时内未被使用的镜像。

3. `-f, --force`：强制清理。默认情况下，`docker image prune` 命令会提示用户进行确认，使用 `-f` 选项可以避免确认提示。

4. `--volumes`：同时清理与已删除镜像关联的未使用的卷。

5. `--help`：显示命令的帮助信息。

**示例:**

- 清理未被标签引用的镜像（悬空镜像）：
  ```
  docker image prune
  ```

- 清理所有镜像，包括未被标签引用的镜像和被标记为 `<none>` 的镜像：
  ```
  docker image prune -a
  ```

- 根据筛选条件清理镜像，例如清理最后 24 小时内未被使用的镜像：
  ```
  docker image prune --filter "until=24h"
  ```

- 强制清理而不进行确认提示：
  ```
  docker image prune -f
  ```

- 同时清理未使用的卷：
  ```
  docker image prune --volumes
  ```

请小心使用 `docker image prune` 命令，特别是使用 `-a` 选项，因为它将删除所有未被标签引用的镜像，这可能导致数据丢失。确保您知道哪些镜像可以安全删除，并在进行清理之前仔细检查。
### 离线迁移镜像
不将镜像上传到云，而是直接与目标机进行传输

先将镜像归档成`docker save`.tar文件，再通过scp命令将其传输到另一台服务器上，接收到.tar文件后用`docker load`加载镜像就能得到该镜像了
### scp
`scp`（Secure Copy Protocol）是一个用于在本地计算机和远程计算机之间安全地复制文件和目录的命令行工具。它通过 SSH（Secure Shell）协议进行通信，提供了加密的数据传输。以下是 `scp` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
scp [OPTIONS] [SOURCE] [DESTINATION]
```

- `SOURCE`：要复制的文件或目录的路径。
- `DESTINATION`：目标位置，可以是本地路径或远程计算机的用户名和路径（通常以用户名@主机:路径的形式表示）。

**常见的可选参数:**
1. `-P`：指定远程 SSH 端口号。默认端口号是 22。例如，`-P 2222` 用于指定远程服务器的 SSH 端口号为 2222。

2. `-r`：递归复制目录及其内容。如果 `SOURCE` 是一个目录，使用 `-r` 参数可以复制目录及其下的所有文件和子目录。

3. `-v`：详细模式（verbose mode）。显示详细的进度和调试信息，有助于调试问题。

4. `-i`：指定密钥文件，用于身份验证。例如，`-i /path/to/your/key.pem` 指定了用于 SSH 身份验证的密钥文件。

5. `-q`：静默模式（quiet mode）。减少输出信息，只显示错误信息。

6. `-l`：限制带宽。可以使用 `-l` 参数来限制传输速度，例如，`-l 100` 限制传输速度为 100 KB/s。

7. `-C`：开启压缩传输。使用 `-C` 参数可以启用数据压缩，以减少传输的数据量，适用于带宽有限的情况。

8. `-4` 或 `-6`：指定使用 IPv4 或 IPv6 地址。例如，`-4` 表示使用 IPv4 地址。

9. `-h`：显示帮助信息，列出可用的选项和参数。

**示例:**

- 从本地计算机复制文件到远程计算机：
  ```
  scp localfile.txt username@remotehost:/path/to/destination
  ```

- 从远程计算机复制文件到本地计算机：
  ```
  scp username@remotehost:/path/to/remotefile.txt /local/destination
  ```

- 递归复制整个目录：
  ```
  scp -r sourcedir username@remotehost:/path/to/destination
  ```

- 指定非默认端口号：
  ```
  scp -P 2222 localfile.txt username@remotehost:/path/to/destination
  ```

`scp` 命令非常有用，因为它允许您在本地和远程计算机之间安全地传输文件和目录。可选参数允许您对传输进行自定义和优化，以满足不同的需求和情境。

### push和pull的压缩解压
docker push会将镜像进行压缩后再push，而docker会先pull镜像在本地完成解压

## 容器
容器作为镜像的实例，同一镜像可以实例化出多个容器。镜像的分层文件系统是只读的，容器在这些只读层上创建了自己的读写层，不会影响原有镜像
容器之间相互隔离，但是共享只读的文件系统。资源利用率高
### 生命周期
容器的生命周期是容器可能处于的状态
1. created：初建状态
2. running：运行状态
3. stopped：停止状态
4. paused：暂停状态
5. deleted：删除状态

### docker create
`docker create` 命令用于创建一个新的容器，但不会立即启动它。这个命令允许您在容器中进行一些初始化配置，然后稍后使用 `docker start` 命令启动该容器。以下是 `docker create` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
```

**参数:**
- `IMAGE`：要创建容器的镜像的名称或 ID。
- `COMMAND`：在容器内部要运行的命令（覆盖镜像中的默认命令）。
- `ARG...`：可选的参数，用于传递给容器内部命令。

**常见的可选参数:**
1. `-a, --attach`：连接到容器的标准输入、输出和错误流。通常与 `-i` 一起使用，以启用交互式模式。

2. `-i, --interactive`：以交互式模式运行容器，允许用户与容器进行交互。

3. `--name`：为容器指定一个名称。使用 `--name` 可以为容器分配一个易于识别的名称，而不是随机生成的名称。

4. `--env, -e`：设置环境变量。可以使用 `-e` 或 `--env` 来传递环境变量给容器。

5. `--link`：将容器连接到其他容器。通过 `--link` 可以创建连接，使一个容器能够访问另一个容器的服务。

6. `--expose`：指定容器要监听的端口。这可以将容器内的端口映射到主机上的端口。

7. `--volume, -v`：绑定挂载卷。可以使用 `-v` 或 `--volume` 来将主机上的目录或文件挂载到容器内部。

8. `--workdir, -w`：设置容器的工作目录。可以使用 `-w` 或 `--workdir` 来指定容器内部的工作目录。

9. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 创建一个容器，以交互式模式运行，并连接到其标准输入、输出和错误流：
  ```
  docker create -i -a --name my-container ubuntu:latest
  ```

- 创建一个容器并指定环境变量：
  ```
  docker create -e MYSQL_ROOT_PASSWORD=my-secret-pw --name mysql-container mysql:latest
  ```

- 创建一个容器并绑定挂载卷：
  ```
  docker create -v /host/data:/container/data --name data-container my-image
  ```

- 创建一个容器并设置容器的工作目录：
  ```
  docker create -w /app --name app-container my-app-image
  ```

`docker create` 命令通常用于准备容器的初始化配置，然后使用 `docker start` 命令来启动容器。可选参数允许您自定义容器的行为和环境。
### docker logs
`docker logs` 命令用于查看 Docker 容器的日志输出。这个命令允许您查看容器内部应用程序的标准输出和标准错误日志，以进行故障排除和监控。以下是 `docker logs` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker logs [OPTIONS] CONTAINER
```

**参数:**
- `CONTAINER`：要查看日志的容器的名称或 ID。

**常见的可选参数:**
1. `--details`：显示详细的日志信息，包括时间戳、标签和日志驱动程序信息。

2. `--follow`, `-f`：实时跟踪日志输出，类似于 `tail -f` 命令。这将连续显示日志的新增内容。

3. `--since`：仅显示自指定日期或时间之后的日志。您可以使用日期时间格式，例如 "2013-01-02T13:23:37"。

4. `--until`：仅显示自指定日期或时间之前的日志。同样，您可以使用日期时间格式。

5. `--timestamps`, `-t`：显示时间戳。将时间戳添加到每行日志的前面。

6. `--tail`：仅显示最后指定行数的日志。例如，`--tail 50` 将显示最后 50 行日志。

7. `--no-color`：禁用彩色输出，将日志以纯文本格式显示。

8. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 查看容器的日志（默认显示所有日志内容）：
  ```
  docker logs my-container
  ```

- 实时跟踪容器的日志输出：
  ```
  docker logs -f my-container
  ```

- 查看容器的最后 50 行日志：
  ```
  docker logs --tail 50 my-container
  ```

- 查看容器从指定日期时间之后的日志：
  ```
  docker logs --since "2023-01-02T13:00:00" my-container
  ```

- 查看容器的详细日志信息，包括时间戳和标签：
  ```
  docker logs --details my-container
  ```

`docker logs` 命令是一个有用的工具，可用于查看容器内部应用程序的输出和日志，以帮助您诊断问题、监控应用程序的状态以及查看容器的活动。可选参数使您能够自定义输出以满足您的需求。

### docker attach
`docker attach` 命令用于连接到正在运行的 Docker 容器，并附加到容器的标准输入（stdin）、输出（stdout）和错误（stderr）流。这允许您与容器内的应用程序进行交互，类似于在终端中直接运行容器内的命令。以下是 `docker attach` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker attach [OPTIONS] CONTAINER
```

**参数:**
- `CONTAINER`：要连接的正在运行的容器的名称或 ID。

**可选参数:**
1. `--detach-keys`：指定分离键序列。分离键允许您从已附加到的容器中分离出来，类似于按 Ctrl-p Ctrl-q 分离。您可以使用此选项来自定义分离键序列，例如，`--detach-keys="ctrl-]"`。

2. `--no-stdin`：不要附加到容器的标准输入。使用此选项将不会将容器的 stdin 与您的终端连接，只会连接 stdout 和 stderr。

3. `--sig-proxy`：控制信号代理。如果未指定此选项，容器中的信号将传播到容器进程。使用 `--sig-proxy=false` 可以禁用信号代理，以便在容器中运行的程序能够接收信号。

4. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 附加到正在运行的容器并与其交互：
  ```
  docker attach my-container
  ```

- 附加到容器但不与其标准输入相连：
  ```
  docker attach --no-stdin my-container
  ```

- 指定自定义分离键序列：
  ```
  docker attach --detach-keys="ctrl-]" my-container
  ```

- 禁用信号代理，容器中的信号不会传播到容器进程：
  ```
  docker attach --sig-proxy=false my-container
  ```

请注意，`docker attach` 主要用于与正在运行的容器进行交互，但它不会创建一个新的 shell 会话，因此您可能需要在容器中运行一个交互式 shell 或其他命令来与容器内的应用程序进行交互。

### docker exec
`docker exec` 命令用于在运行的 Docker 容器内部执行命令。与 `docker attach` 不同，`docker exec` 允许您在容器内部执行单个命令，而无需附加到容器并运行交互式 shell。以下是 `docker exec` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

**参数:**
- `CONTAINER`：要在其中执行命令的容器的名称或 ID。
- `COMMAND`：要在容器内执行的命令。
- `ARG...`：可选的参数，用于传递给命令。

**常见的可选参数:**
1. `-d, --detach`：在后台运行命令，不会将命令的输出附加到终端。使用此选项将在容器内部执行命令，但不会阻止您的终端。

2. `--env, -e`：设置环境变量。可以使用 `-e` 或 `--env` 来传递环境变量给执行的命令。

3. `--user`：指定要执行命令的用户名或 UID。例如，`--user 1000` 将以 UID 1000 的用户身份运行命令。

4. `--workdir, -w`：设置命令执行的工作目录。可以使用 `-w` 或 `--workdir` 来指定命令执行时所在的工作目录。

5. `--userns`：指定用户命名空间的模式。允许用户执行命令时以不同的用户命名空间运行。

6. `--privileged`：以特权模式运行命令。此选项会赋予命令访问容器内核的特权。

7. `--tty, -t`：分配一个伪终端（TTY）。通常用于交互式命令执行。

8. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 在容器内执行一个简单的命令（不分配 TTY）：
  ```
  docker exec my-container ls
  ```

- 在容器内执行交互式 shell（分配 TTY）：
  ```
  docker exec -it my-container /bin/bash
  ```

- 在容器内以后台模式运行命令：
  ```
  docker exec -d my-container ./my-background-process
  ```

- 为执行的命令设置环境变量：
  ```
  docker exec -e MY_ENV_VAR=my-value my-container my-command
  ```

`docker exec` 命令非常有用，因为它允许您在运行的容器内部执行命令，而无需启动新的容器实例。可选参数允许您自定义命令执行的方式和环境。

### docker start
`docker start` 命令用于启动已经创建但停止的 Docker 容器。以下是 `docker start` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker start [OPTIONS] CONTAINER
```

**参数:**
- `CONTAINER`：要启动的容器的名称或 ID。

**常见的可选参数:**
1. `--attach`, `-a`：连接到容器的标准输入、输出和错误流。这使您可以查看容器的输出，并在需要时与容器进行交互。

2. `--detach-keys`：指定分离键序列。分离键允许您从附加到容器的终端中分离出来，类似于按 Ctrl-p Ctrl-q 分离。您可以使用此选项来自定义分离键序列，例如，`--detach-keys="ctrl-]"`。

3. `--interactive`, `-i`：以交互式模式启动容器，允许用户与容器进行交互。通常与 `-a` 一起使用，以启用交互式模式。

4. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 启动一个已创建但停止的容器：
  ```
  docker start my-container
  ```

- 启动一个容器并连接到其标准输入、输出和错误流以进行交互：
  ```
  docker start -ai my-container
  ```

- 自定义分离键序列并启动容器：
  ```
  docker start --detach-keys="ctrl-]" my-container
  ```

`docker start` 命令用于启动已创建的容器，可选参数允许您自定义容器的启动方式和交互模式。这对于在容器停止后重新启动它们以及与容器进行交互非常有用。
### docker stop
`docker stop` 命令用于停止运行中的 Docker 容器。这个命令发送一个指定的信号（默认是SIGTERM，可被配置为其他信号），通知容器停止运行，并允许容器执行清理操作。以下是 `docker stop` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

**参数:**
- `CONTAINER`：要停止的容器的名称或 ID。您可以指定一个或多个容器。

**常见的可选参数:**
1. `--time`, `-t`：指定等待容器停止的时间，以秒为单位。如果容器在指定的时间内没有停止，Docker 将强制终止容器。默认情况下，等待时间是 10 秒。

2. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 停止一个运行中的容器：
  ```
  docker stop my-container
  ```

- 使用自定义等待时间停止容器（等待 30 秒）：
  ```
  docker stop -t 30 my-container
  ```

- 停止多个容器：
  ```
  docker stop container1 container2
  ```

`docker stop` 命令用于优雅地停止容器，允许容器内的应用程序执行清理操作。通过可选参数 `--time`，您可以指定等待容器停止的时间，以确保容器能够在一定时间内完成清理。如果容器在指定的等待时间内没有停止，Docker 将强制终止容器。这有助于确保容器的资源得到释放。

### docker restart
`docker restart` 命令用于重新启动正在运行的 Docker 容器。重新启动容器将会停止它，然后再次启动它。以下是 `docker restart` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

**参数:**
- `CONTAINER`：要重新启动的容器的名称或 ID。您可以指定一个或多个容器。

**常见的可选参数:**
1. `--time`, `-t`：指定等待容器停止的时间，以秒为单位。如果容器在指定的时间内没有停止，Docker 将强制终止容器。默认情况下，等待时间是 10 秒。

2. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 重新启动一个运行中的容器：
  ```
  docker restart my-container
  ```

- 使用自定义等待时间重新启动容器（等待 30 秒）：
  ```
  docker restart -t 30 my-container
  ```

- 重新启动多个容器：
  ```
  docker restart container1 container2
  ```

`docker restart` 命令用于重新启动容器，以重新加载其配置或应用某些更改。通过可选参数 `--time`，您可以指定等待容器停止的时间，以确保容器能够在一定时间内完成重启。如果容器在指定的等待时间内没有停止，Docker 将强制终止容器。这有助于确保容器的资源得到释放。

### docker kill
`docker kill` 命令用于强制终止运行中的 Docker 容器。与 `docker stop` 不同，它不会发送信号通知容器进行优雅停止，而是立即终止容器的运行，可能导致数据损失或不可控制的关闭。以下是 `docker kill` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker kill [OPTIONS] CONTAINER [CONTAINER...]
```

**参数:**
- `CONTAINER`：要终止的容器的名称或 ID。您可以指定一个或多个容器。

**常见的可选参数:**
1. `--signal`, `-s`：指定要发送的信号。默认信号是 SIGKILL，通常用于强制终止容器。您可以使用此选项来指定其他信号，例如，`--signal=SIGTERM`。

2. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 强制终止一个运行中的容器（使用默认的 SIGKILL 信号）：
  ```
  docker kill my-container
  ```

- 使用自定义信号终止容器（使用 SIGTERM 信号）：
  ```
  docker kill --signal=SIGTERM my-container
  ```

- 强制终止多个容器：
  ```
  docker kill container1 container2
  ```

`docker kill` 命令通常用于在容器不响应正常停止请求时，或在紧急情况下，立即停止容器的运行。通过可选参数 `--signal`，您可以指定要发送的信号，以适应不同的终止需求。请小心使用此命令，因为它可能导致数据损失或应用程序状态不一致。
### docker top
`docker top` 命令用于查看正在运行的 Docker 容器中的进程信息。它会列出容器内部的各个进程以及有关这些进程的信息，如进程 ID（PID）、用户、CPU 使用情况、内存使用情况等。以下是 `docker top` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker top CONTAINER [ps OPTIONS]
```

**参数:**
- `CONTAINER`：要查看进程信息的容器的名称或 ID。

**可选参数:**
`docker top` 命令的可选参数是 `ps` 命令的选项，用于进一步过滤和排序进程信息。这些选项与 Linux 中的 `ps` 命令选项相似，用于定制要显示的进程信息。常见的 `ps` 选项包括：

- `a`：显示所有进程，包括其他用户的进程。
- `u`：以用户为基础显示进程信息，包括用户名和其他用户相关信息。
- `x`：显示不与终端关联的进程。
- `r`：只显示正在运行的进程。
- `o`：自定义输出格式。您可以使用 `--format` 选项来指定输出格式。

这些选项可以与 `docker top` 命令一起使用，以满足您查看容器内部进程信息的需求。例如，`docker top my-container u` 将显示容器内所有用户的进程信息。

**示例:**

- 查看容器内所有进程的基本信息：
  ```
  docker top my-container
  ```

- 查看容器内正在运行的进程：
  ```
  docker top my-container r
  ```

- 查看容器内所有用户的进程信息：
  ```
  docker top my-container a
  ```

- 查看容器内进程的详细信息，包括用户信息：
  ```
  docker top my-container u
  ```

`docker top` 命令是一个有用的工具，用于查看容器内部运行的进程，有助于监控和调试容器中的应用程序。您可以使用 `ps` 选项来定制输出，以满足您的需求。
### docker stats
`docker stats` 命令用于实时显示正在运行的 Docker 容器的资源利用情况，包括 CPU 使用率、内存使用量、网络 I/O 和磁盘 I/O 等信息。这个命令允许您监视容器的性能并进行故障排除。以下是 `docker stats` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker stats [OPTIONS] [CONTAINER...]
```

**可选参数:**
- `CONTAINER`：要监视的容器的名称或 ID。可以指定一个或多个容器，如果不指定任何容器，`docker stats` 将显示所有正在运行的容器的统计信息。

**常见的可选参数:**
1. `--all`，`-a`：显示所有容器的统计信息，包括停止的容器。默认情况下，`docker stats` 仅显示正在运行的容器。

2. `--format`：指定输出格式。您可以使用此选项来自定义输出的格式，例如，`--format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"`。

3. `--no-stream`：只显示一次容器的统计信息，而不是实时更新。默认情况下，`docker stats` 会持续显示容器的统计信息，类似于监视器。

4. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 显示所有正在运行的容器的实时统计信息：
  ```
  docker stats
  ```

- 显示所有容器（包括停止的容器）的实时统计信息：
  ```
  docker stats -a
  ```

- 显示特定容器的实时统计信息：
  ```
  docker stats my-container
  ```

- 显示特定容器的实时统计信息，使用自定义输出格式：
  ```
  docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}" my-container
  ```

- 显示容器的统计信息一次，而不是实时更新：
  ```
  docker stats --no-stream my-container
  ```

`docker stats` 命令是一个有用的工具，用于监视容器的资源利用情况，帮助您了解容器的性能表现。您可以使用可选参数来自定义输出格式以及选择要监视的容器。

### docker container inspect
`docker container inspect` 命令用于获取关于 Docker 容器的详细信息，包括容器的配置、状态、网络设置、挂载卷等。这个命令对于诊断和了解容器的配置和状态非常有用。以下是 `docker container inspect` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker container inspect [OPTIONS] CONTAINER [CONTAINER...]
```

**参数:**
- `CONTAINER`：要获取详细信息的容器的名称或 ID。您可以指定一个或多个容器。

**常见的可选参数:**
1. `--format`：指定输出格式。您可以使用此选项来自定义输出的格式，例如，`--format '{{.NetworkSettings.IPAddress}}'`。

2. `--size`：显示容器的存储资源使用情况，包括总的数据卷大小和数据卷中已使用的空间。

3. `--type`：指定要获取的信息类型。您可以使用此选项来选择要检查的特定信息类型，如 `--type=config` 用于获取容器的配置信息，或 `--type=status` 用于获取容器的状态信息。

4. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 获取容器的详细信息：
  ```
  docker container inspect my-container
  ```

- 获取容器的 IP 地址信息（自定义输出格式）：
  ```
  docker container inspect --format '{{.NetworkSettings.IPAddress}}' my-container
  ```

- 获取容器的存储资源使用情况：
  ```
  docker container inspect --size my-container
  ```

- 获取容器的配置信息：
  ```
  docker container inspect --type=config my-container
  ```

- 获取容器的状态信息：
  ```
  docker container inspect --type=status my-container
  ```

`docker container inspect` 命令允许您深入了解容器的各个方面，包括配置、网络、存储、状态等。您可以使用可选参数来自定义输出格式和选择要获取的信息类型，以满足您的需求。这对于诊断容器问题、监视容器状态以及了解容器配置非常有用。
### docker port
`docker port` 命令用于查看正在运行的 Docker 容器内部的端口映射情况。它显示容器的网络端口和宿主机上相应的映射端口，以帮助您了解容器与主机之间的网络连接。以下是 `docker port` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker port [OPTIONS] CONTAINER [PRIVATE_PORT[/PROTO]]
```

**参数:**
- `CONTAINER`：要查看端口映射的容器的名称或 ID。
- `PRIVATE_PORT`：容器内部的私有端口，用于监听网络连接。
- `PROTO`：可选参数，指定网络协议，通常是 "tcp" 或 "udp"。如果不指定协议，默认为 "tcp"。

**常见的可选参数:**
1. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 查看容器内部的端口映射情况（默认显示所有端口映射）：
  ```
  docker port my-container
  ```

- 查看容器内部特定私有端口的映射情况：
  ```
  docker port my-container 80/tcp
  ```

- 查看容器内部特定私有端口和协议的映射情况：
  ```
  docker port my-container 443/udp
  ```

`docker port` 命令是一个有用的工具，用于了解容器的端口映射情况，尤其是在容器与主机之间需要进行网络通信时。默认情况下，它将显示容器内部的所有端口映射。您可以使用可选参数来查看特定端口或指定协议的映射情况。
### docker cp
`docker cp` 命令用于在本地文件系统和正在运行的 Docker 容器之间拷贝文件或目录。这个命令非常有用，可以用于将文件从容器中提取到本地主机，或将本地文件复制到容器内部。以下是 `docker cp` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker cp [OPTIONS] SRC_PATH CONTAINER:DEST_PATH
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH
```

**参数:**
- `SRC_PATH`：本地文件系统上的源文件或目录的路径。
- `CONTAINER`：正在运行的容器的名称或 ID。
- `DEST_PATH`：容器内部的目标路径，或者容器中的源路径，具体取决于命令的形式。

**常见的可选参数:**
1. `--archive`, `-a`：将文件或目录以归档格式拷贝，保留文件元数据（权限、所有者等）。这通常是推荐的选项，以确保文件完整性和元数据保持不变。

2. `--follow-link`, `-L`：如果源路径是符号链接，会将符号链接指向的文件拷贝到容器内。默认情况下，它会拷贝符号链接本身。

3. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 从容器内拷贝文件到本地主机：
  ```
  docker cp my-container:/path/to/file.txt /local/path/
  ```

- 从本地主机拷贝文件到容器内：
  ```
  docker cp /local/path/file.txt my-container:/path/in/container/
  ```

- 拷贝目录并保留元数据：
  ```
  docker cp -a /local/dir my-container:/path/in/container/
  ```

- 拷贝符号链接指向的文件到容器内：
  ```
  docker cp -L /local/link my-container:/path/in/container/
  ```

`docker cp` 命令是一个非常有用的工具，可用于在本地文件系统和容器之间传输文件和目录。您可以使用可选参数来自定义拷贝的方式，以满足您的需求。请注意，容器的名称或 ID 必须在命令中指定，以指示源和目标。
### docker commit
`docker commit` 命令用于将容器的当前状态保存为一个新的镜像。这个命令对于在容器中进行修改，然后将修改后的容器保存为新镜像非常有用。以下是 `docker commit` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```

**参数:**
- `CONTAINER`：要保存为新镜像的容器的名称或 ID。
- `REPOSITORY[:TAG]`：新镜像的名称和可选标签。如果不提供标签，默认为 "latest"。

**常见的可选参数:**
1. `--change`：设置容器的元数据。可以使用 `--change` 选项来指定容器的元数据，例如，`--change "LABEL com.example.vendor=ACME"`。

2. `--message`, `-m`：设置提交的说明信息。您可以使用此选项来提供有关提交的说明信息，类似于 Git 提交消息。

3. `--author`：设置提交的作者信息。可以用于指定提交的作者，例如，`--author "John Doe <john@example.com>"`。

4. `--pause`：在容器提交之前暂停容器的运行。默认情况下，容器在提交之前会被暂停，以确保文件一致性。

5. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 保存一个运行中的容器为新镜像（默认标签 "latest"）：
  ```
  docker commit my-container my-new-image
  ```

- 保存容器为新镜像并指定标签：
  ```
  docker commit my-container my-new-image:1.0
  ```

- 保存容器为新镜像并设置说明信息：
  ```
  docker commit -m "Added custom configuration" my-container my-new-image
  ```

- 保存容器为新镜像并设置作者信息：
  ```
  docker commit --author "John Doe <john@example.com>" my-container my-new-image
  ```

`docker commit` 命令允许您创建自定义的容器镜像，以便您可以将容器中的修改保存为新的镜像。您可以使用可选参数来设置元数据、说明信息、作者信息等，以自定义镜像的属性。这对于开发、测试和分发自定义镜像非常有用。
### docker pause
`docker pause` 命令用于暂停正在运行的 Docker 容器，使容器中的所有进程停止运行。这可以帮助您在不终止容器的情况下停止容器中的所有活动进程。以下是 `docker pause` 命令的详细解释，以及它的可选参数：

**语法:**
```
docker pause CONTAINER [CONTAINER...]
```

**参数:**
- `CONTAINER`：要暂停的容器的名称或 ID。您可以指定一个或多个容器，以一次性暂停多个容器。

**常见的可选参数:**
- `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 暂停一个运行中的容器：
  ```
  docker pause my-container
  ```

- 暂停多个容器：
  ```
  docker pause container1 container2
  ```

`docker pause` 命令用于暂停容器中的所有进程，使其进入挂起状态。被暂停的容器将不再响应新的网络请求或进程运行。这对于需要临时停止容器内的活动进程而不终止容器本身的情况非常有用。如果需要恢复容器的运行，可以使用 `docker unpause` 命令来解除暂停状态。但请注意，不是所有容器都可以被暂停，取决于容器内部的进程和配置。
### docker unpause
`docker unpause` 命令用于取消暂停状态的 Docker 容器，使容器内部的进程继续运行。这个命令通常与 `docker pause` 命令一起使用，以在暂停后重新激活容器。以下是 `docker unpause` 命令的详细解释以及它的可选参数：

**语法:**
```
docker unpause CONTAINER [CONTAINER...]
```

**参数:**
- `CONTAINER`：要取消暂停状态的容器的名称或 ID。您可以指定一个或多个容器，以一次性取消暂停多个容器。

**常见的可选参数:**
- `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 取消暂停一个暂停状态的容器：
  ```
  docker unpause my-container
  ```

- 取消暂停多个容器：
  ```
  docker unpause container1 container2
  ```

`docker unpause` 命令用于取消容器的暂停状态，允许容器内的进程继续运行。这对于需要在暂停后恢复容器内部进程的情况非常有用。请注意，只有暂停状态的容器才能被取消暂停，如果容器没有被暂停，使用该命令不会产生任何效果。
### docker rm
`docker rm` 命令用于删除一个或多个停止的 Docker 容器。这可以帮助您释放存储资源并清理不再需要的容器。以下是 `docker rm` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

**参数:**
- `CONTAINER`：要删除的容器的名称或 ID。您可以指定一个或多个容器，以一次性删除多个容器。

**常见的可选参数:**
1. `--force`, `-f`：强制删除正在运行的容器。默认情况下，`docker rm` 不会删除正在运行的容器，除非您使用此选项。

2. `--link`：删除容器的链接。如果容器被其他容器链接到，使用此选项可以删除链接。

3. `--volumes`, `-v`：同时删除容器相关的卷（volumes）。默认情况下，容器被删除后，与容器相关的卷不会被自动删除。

4. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 删除一个已停止的容器：
  ```
  docker rm my-container
  ```

- 删除多个已停止的容器：
  ```
  docker rm container1 container2
  ```

- 强制删除一个正在运行的容器：
  ```
  docker rm -f my-running-container
  ```

- 删除容器并同时删除与容器关联的卷：
  ```
  docker rm -v my-container
  ```

`docker rm` 命令是一个用于清理不再需要的 Docker 容器的重要工具。您可以使用可选参数来自定义删除的方式，例如强制删除正在运行的容器或同时删除容器相关的卷。请小心使用该命令，因为容器一旦被删除，其数据和状态将无法恢复。
### docker export
`docker export` 命令用于将 Docker 容器的文件系统导出为一个压缩的 tar 归档文件。这个命令不包括容器的元数据或配置信息，只导出文件系统内容。以下是 `docker export` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker export [OPTIONS] CONTAINER > output.tar
```

**参数:**
- `CONTAINER`：要导出文件系统的容器的名称或 ID。
- `output.tar`：要导出的 tar 归档文件的输出路径。通常使用 `>` 符号将导出的内容重定向到一个文件。

**常见的可选参数:**
1. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 导出容器的文件系统内容到一个 tar 归档文件：
  ```
  docker export my-container > container-export.tar
  ```

`docker export` 命令用于将容器的文件系统内容导出到一个 tar 归档文件，这对于备份容器内部的文件或将容器的文件系统内容移动到其他地方非常有用。请注意，这个命令不包括容器的配置或元数据，仅导出文件系统。如果需要导出完整的容器，包括配置信息和元数据，可以考虑使用 `docker save` 命令导出整个容器镜像。
### docker import
`docker import` 命令用于从 tar 归档文件中创建一个新的 Docker 镜像。这个命令通常用于将文件系统内容导入为 Docker 镜像，而不包括容器的元数据或配置信息。以下是 `docker import` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]
```

**参数:**
- `file|URL|-`：要导入的 tar 归档文件的路径，可以是本地文件的路径、URL 或 `-`（用于从标准输入导入）。
- `REPOSITORY[:TAG]`：要创建的新镜像的名称和可选标签。如果未提供标签，默认为 "latest"。

**常见的可选参数:**
1. `--change`：设置镜像的元数据。可以使用 `--change` 选项来指定镜像的元数据，例如，`--change "LABEL com.example.vendor=ACME"`。

2. `--message`, `-m`：设置镜像的描述信息。您可以使用此选项来提供有关镜像的说明信息，类似于 Git 提交消息。

3. `--help`：显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 从 tar 归档文件创建新的 Docker 镜像：
  ```
  docker import my-archive.tar my-image:1.0
  ```

- 从标准输入导入 tar 归档文件并创建新的镜像：
  ```
  cat my-archive.tar | docker import - my-image:1.0
  ```

- 创建镜像并设置元数据和描述信息：
  ```
  docker import --change "LABEL com.example.vendor=ACME" -m "Custom image" my-archive.tar my-image:1.0
  ```

`docker import` 命令用于创建新的 Docker 镜像，这个镜像的内容是从 tar 归档文件导入的。请注意，这个命令不包括容器的配置或元数据，仅导入文件系统内容。如果需要创建包括配置信息和元数据的完整镜像，可以使用 `docker build` 命令。
### docker build
`docker build` 命令用于根据 Dockerfile 构建 Docker 镜像。以下是 `docker build` 命令的基本用法及选项说明：

基本用法：

```sh
docker build [OPTIONS] PATH | URL | -
```

- `PATH`：指定 Dockerfile 所在的路径。可以是相对路径或绝对路径。
- `URL`：指定一个远程 URL，Docker 将从该 URL 下载 Dockerfile 和构建上下文。
- `-`：使用标准输入流作为 Dockerfile。可以与 `docker build` 命令结合使用，通过管道从标准输入流传递 Dockerfile 内容。

常用选项：

- `-f, --file string`：指定要使用的 Dockerfile 文件，默认为 `PATH/Dockerfile`。
- `--tag string`：为生成的镜像指定标签，格式为 `name:tag`。
- `-t, --tag list`：为生成的镜像指定一个或多个标签。
- `--build-arg list`：设置构建时的参数，格式为 `key=value`。
- `--no-cache`：禁用构建缓存，强制重新构建镜像。
- `--network string`：设置构建时的网络模式。
- `--progress string`：设置构建进度输出模式（auto、plain、tty）。
- `--pull`：在构建前尝试拉取最新的基础镜像。
- `--squash`：压缩构建时的中间层，减少生成的镜像大小。

示例用法：

1. 使用当前目录下的 Dockerfile 构建镜像，并打上标签 `myimage:v1`：
   ```sh
   docker build --tag myimage:v1 .
   ```

2. 使用指定的 Dockerfile 文件构建镜像，并设置构建参数：
   ```sh
   docker build --file Dockerfile.dev --tag myimage:dev --build-arg ENVIRONMENT=development .
   ```

3. 从远程 Git 仓库下载 Dockerfile 并构建镜像：
   ```sh
   docker build --tag myimage:v1 https://github.com/user/repo.git#branch:path/to/Dockerfile
   ```

4. 从标准输入流中读取 Dockerfile 并构建镜像：
   ```sh
   cat Dockerfile | docker build --tag myimage:latest -
   ```

### docker wait
`docker wait` 命令用于等待容器的运行进程退出，并返回该进程的退出状态码。这个命令通常在脚本中用于等待容器的某个特定进程完成执行，以便进一步处理。以下是 `docker wait` 命令的详细解释以及可选参数：

**语法:**
```
docker wait CONTAINER
```

**参数:**
- `CONTAINER`：要等待的容器的名称或 ID。

**可选参数:**
`docker wait` 命令没有常见的可选参数，它仅接受容器的名称或 ID 作为参数，以等待容器内的进程退出。退出状态码将成为命令的返回值，供后续处理使用。

**示例:**

- 等待容器的运行进程退出并获取退出状态码：
  ```bash
  docker wait my-container
  ```

这个命令通常在自动化脚本或编排工具中用于等待容器内的特定任务完成，以便根据任务的退出状态码执行后续操作。当容器内的任务完成后，`docker wait` 命令会返回退出状态码，您可以据此来决定下一步的操作。
### docker container prune
`docker container prune` 命令用于删除未运行的 Docker 容器。它可以帮助您清理系统中处于停止状态且不再需要的容器，从而释放存储空间和资源。以下是 `docker container prune` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker container prune [OPTIONS]
```

**可选参数:**
- `--filter`: 使用此选项来指定筛选条件，以决定哪些容器应该被删除。筛选条件可以是容器的状态、标签等等。例如，`--filter "status=exited"` 可以删除所有已经退出的容器。

- `--force`, `-f`: 强制删除容器而无需确认。默认情况下，命令会列出要删除的容器，并需要确认才能执行删除操作。使用此选项可以跳过确认步骤。

- `--volumes`: 同时删除容器相关的卷（volumes）。默认情况下，容器被删除后，与容器相关的卷不会被自动删除。使用此选项将删除容器及其关联的卷。

- `--help`: 显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 删除所有已停止的容器（需要确认）：
  ```
  docker container prune
  ```

- 强制删除所有已停止的容器，无需确认：
  ```
  docker container prune -f
  ```

- 删除所有已停止的容器及其关联的卷：
  ```
  docker container prune --volumes
  ```

- 使用筛选条件删除容器，例如，只删除状态为 "exited" 的容器：
  ```
  docker container prune --filter "status=exited"
  ```

`docker container prune` 命令是一个有用的工具，用于清理未运行的容器，特别是在开发和测试环境中容易积累大量未使用的容器时。您可以使用可选参数来自定义删除的方式，包括强制删除、删除关联卷、使用筛选条件等。
### docker update
`docker update` 命令用于更新正在运行的容器的配置。这包括可以更改容器的 CPU 分配、内存限制和其他资源限制，以及设置容器的重新启动策略等。以下是 `docker update` 命令的详细解释以及一些常见的可选参数：

**语法:**
```
docker update [OPTIONS] CONTAINER
```

**参数:**
- `CONTAINER`：要更新配置的容器的名称或 ID。

**常见的可选参数:**
1. `--blkio-weight`: 设置容器的块 I/O 权重。这可以用于控制容器对磁盘 I/O 的访问优先级。值范围为 10 到 1000。

2. `--cpu-shares`: 设置容器的 CPU 资源分配。这可以用于控制容器在竞争 CPU 资源时的份额。默认值为 1024。

3. `--cpu-period`: 设置容器的 CPU 周期。这指定了容器能够在 CPU 上运行的时间周期。默认为 100000（100毫秒）。

4. `--cpu-quota`: 设置容器的 CPU 配额。这限制了容器在 CPU 周期内的使用时间。默认值为 -1，表示没有限制。

5. `--cpuset-cpus`: 为容器指定可以使用的 CPU 内核。这可以用于将容器绑定到特定的 CPU 内核。

6. `--memory`, `-m`: 设置容器的内存限制。您可以指定一个整数值，单位可以是字节、千字节、兆字节或吉字节，例如，`-m 1g` 表示限制容器的内存为1GB。

7. `--memory-reservation`: 设置容器的内存保留值。这是容器最低内存使用量的保证，即使其他容器需要更多内存。

8. `--restart`: 设置容器的重新启动策略。可以是 "no"、"on-failure"、"always" 或 "unless-stopped"，用于确定容器何时自动重新启动。

9. `--help`: 显示命令的帮助信息，列出可用的选项和参数。

**示例:**

- 更新容器的 CPU 资源分配：
  ```
  docker update --cpu-shares 512 my-container
  ```

- 更新容器的内存限制：
  ```
  docker update --memory 512m my-container
  ```

- 设置容器的重新启动策略：
  ```
  docker update --restart unless-stopped my-container
  ```

`docker update` 命令允许您在容器正在运行时更改其资源限制和重新启动策略。这对于在容器需要进行调整以适应不同的工作负载时非常有用。您可以使用不同的可选参数来调整容器的 CPU、内存和重新启动行为。请注意，不是所有参数对所有容器类型都有效，具体效果取决于容器和主机的配置。

### 容器的批量处理
根据名称过滤得到容器编号
```bash
docker ps -af name=xxx
```
过滤指定状态的容器(created, paused,exited)
```bash
docker ps -af status=running
```
过滤指定镜像创建的容器（可以是镜像的tag也能是镜像的编号）
```bash
docker ps -af ancestor=xxx
```
`-q`：只返回ID信息

停止所有正在运行的容器
```bash
docker stop `docker ps -q`
```
启动所有容器
```bash
docker start `docker ps -aq`
```
### 容器的交互模式
#### attached模式
不使用-d，-i，-t选项启动容器
```bash
docker run --name mynginx5 -p 8090:80 nginx:1.22.0
```
前台运行，此时Ctrl+c将停止正在运行的attached容器
所以该模式不能用于生产环境
#### detached模式
使用-d选项后台运行容器
```bash
docker run -d nginx:1.22.0
```
使用attach命令，以前台方式控制后台容器，此时进入前台模式
```bash
attach xxx(容器名或者ID)
```
#### interactive模式
以-it选项启动容器，最后加上bash，以bash的方式与该容器交互
```bash
docker run -it --name my7 nginx:1.22.0 bash
```
此时只是创建了容器并进入bash，输入nginx才能启动该容器
**输入exit将退出容器并停止容器**

对于在后台运行的容器，通过exec + bash与其交互，xxx为容器名/ID
```bash
docker exec -it xxx bash
```
此时输入exit也不会stop该容器
### 容器与宿主机之间的内容拷贝
使用cp命令进行拷贝
```bash
docker cp mynginx4:usr/share/nginx/html/index.html .
```
修改内容后将文件拷贝回去
```bash
 docker cp ./index.html mynginx4:usr/share/nginx/html/
```
注意docker cp不支持容器之间的拷贝

**容器的自动删除**：
若启动容器时带上了--rm选项，那么无论该容器是前台运行还是后台运行，Ctrl+C/stop后都将删除容器

### 容器的自动重启

`--restart`: 设置容器的重新启动策略。可以是 "no"、"on-failure"、"always" 或 "unless-stopped"，用于确定容器何时自动重新启动。

1. "no"：这意味着不重新启动容器。如果容器终止，它将保持终止状态，不会自动重新启动。

2. "on-failure"：容器会在非正常退出（退出状态不为零）时自动重新启动。这是一种适合处理容器失败的策略，因为只有在容器终止状态为非零时才会触发重新启动。

3. "always"：无论容器如何终止，它都会自动重新启动。这意味着无论是正常退出还是非正常退出，都会触发容器的自动重新启动。用stop停止不会重新启动。

4. "unless-stopped"：这是一种持久的策略，容器会自动重新启动，除非您明确停止它。即使您使用 `docker stop` 命令停止容器，它仍会重新启动。
### 容器的环境变量
`docker run` 命令的 `--env-file` 选项用于从文件中加载环境变量并将它们传递给正在运行的 Docker 容器。这是一个很有用的方式，可以使您轻松管理容器的环境变量，特别是在需要设置多个环境变量时。

使用 `--env-file` 选项的基本语法如下：

```bash
docker run --env-file=/path/to/env-file my_container
```

在这个命令中：

- `--env-file` 用于指定包含环境变量的文件的路径。
- `/path/to/env-file` 是包含环境变量的文件的实际路径。
- `my_container` 是要运行的 Docker 容器的名称或镜像。

环境变量文件通常是一个文本文件，每一行定义一个环境变量，格式为 `VAR_NAME=value`，例如：

```plaintext
DB_HOST=mydbhost
DB_USER=mydbuser
DB_PASSWORD=secret
```

在运行容器时，Docker 将从指定的文件中读取这些环境变量，并将它们传递给容器。这使您可以更轻松地管理和维护大量环境变量，而无需在 `docker run` 命令中多次使用 `-e` 选项。

请注意，使用 `--env-file` 选项加载的环境变量会覆盖 Dockerfile 中使用 `ENV` 指令设置的同名环境变量。这允许您在不更改 Dockerfile 的情况下，根据需要覆盖容器的环境配置。

除此之外：
- `--env`：用于设置环境变量。例如，`-e "ENV_VARIABLE=value"`。
可以连续使用多次-e选项以设置多个环境变量

### 借助容器完成特殊操作

```bash
docker run --rm busybox:1.36.0 ping www.google.com
```
![image.png](https://s2.loli.net/2023/10/25/6CmWLYOhyS2IRg3.png)
创建一个停止后即删除的busybox容器，在run命令的最后带上要执行的特殊操作即可

使用--net host可以打断容器与宿主机之间的网络隔离，从而使用busybox执行网络相关的命令

`docker run` 命令的 `--net host` 选项是一种网络模式，用于将容器与主机的网络命名空间共享，从而允许容器使用主机的网络堆栈。这种模式使容器能够访问主机上的网络服务，端口，以及其他网络资源，就好像它们运行在主机上一样。

使用 `--net host` 选项的基本语法如下：

```bash
docker run --net host my_container
```

在这个命令中：

- `--net` 用于指定容器的网络模式。
- `host` 是一种特殊的网络模式，它表示容器将共享主机的网络命名空间。

一旦容器以 `--net host` 模式运行，它将具有与主机相同的 IP 地址，可以访问主机上的所有网络服务，无需进行端口映射。这对于需要与主机上的网络服务进行交互的容器非常有用，例如，如果容器需要访问主机上的数据库或 Web 服务，或者如果容器需要监听主机上的特定端口。

需要注意的是，`--net host` 模式会完全覆盖容器的网络设置，因此容器将不再拥有自己的网络命名空间，而是与主机共享。这可能会导致一些安全和隔离方面的问题，因此在使用时需要谨慎考虑。

通常情况下，只有在特定的使用场景下，才需要使用 `--net host` 模式。大多数容器可以使用默认的网络模式，而不需要共享主机网络命名空间。
### 容器镜像迁移的导入导出
用export将容器归档成tar文件，制作成镜像
```bash
docker export -o nginx.tar mynginx1
```
将tar文件传输后，用import命令将镜像导入即可
导入的镜像可能存在元数据的丢失，此时无法通过该镜像启动容器
`docker inspect`查看该镜像的信息，找到Cmd字段，在docker run启动容器的命令最后加上Cmd的命令即可
### 容器的日志查看
将日志重定向到xxx.log文件中，将标准错误流重定向到err.log文件中
```bash
docker logs xxx > xxx.log 2>err.log
```
容器的日志默认保存在/var/lib/docker/文件ID/文件ID-json.log文件下
### docker安装mysql
将容器的3306端口映射到宿主机的8888端口，并且设置环境变量（密码）
```bash
docker run -d --name some-mysql -p 8888:3306 -e MYSQL_ROOT_PASSWORD=99887766 mysql:5.7 
```
创建完成后启动该容器的bash
```bash
docker exec -it some-mysql bash
```
### docker安装redis
```bash
docker run --name redis -d -p 8400:6379 redis:7
```
### docker安装c++环境
随便下载一个Linux发行版
```bash
docker pull centos:7
```
启动容器，与该容器的bash进行交互
```bash
docker run -it --name mycpp centos:7 bash
```
根据具体发行版安装gcc
```bash
yum install gcc -y
```
编译一个c语言源文件进行编译并运行
![image.png](https://s2.loli.net/2023/10/27/OMAEny1bY7Jth4d.png)
### docker制作SpringBoot容器

### docker容器资源更新
学了Java再来补
## 存储卷
类似于硬连接，将宿主机上的文件系统与容器中的文件系统进行绑定，容器在该目录下写入数据时，意味着其直接向宿主机的目录写入数据，即**数据读写同步**

与容器形成绑定关系的目录叫做存储卷，其本质是文件或者目录，写入存储卷时可以直接**绕过联合文件系统**直接以文件或者目录的形式写入宿主机

并且销毁容器时，存储卷的数据**不会被删除**

**为什么需要存储卷？**
解决误删数据时带来的数据丢失问题，存储卷可以做到数据持久化（**核心问题**）
由于联合文件系统的读写效率低对于IO频繁的应用，读写速度慢
宿主机和容器之间的**共享**不方便，只能通过docker cp完成
而容器与容器之间的**共享**更不方便
由此，存储卷诞生了

### 管理卷
volume docker，存储卷的绑定关系中，宿主机的挂载目录由docker进行创建
优点是该卷由docker进行管理，缺点是用户无法做到精确管理该目录，该类型的卷通常用作临时存储
默认映射到`/var/lib/docker/volumes`目录下

命令了解即可：
#### docker volume create
`docker volume create` 命令用于在 Docker 中创建一个新的卷（volume）。卷是 Docker 的一种功能，用于在容器之间共享和持久化数据。创建卷允许你将数据存储在容器外部，以确保数据在容器销毁和重新创建时保持持久性。

以下是 `docker volume create` 命令的详细解释：

```bash
docker volume create [OPTIONS] [VOLUME]
```

- `OPTIONS`：可以包括一些选项，用于自定义卷的行为。
- `VOLUME`：是要创建的卷的名称，可以自定义，如果不指定名称，Docker 将自动分配一个唯一的名称。

下面是一些常用的选项和示例：

1. **指定卷的名称**：

   ```bash
   docker volume create my_volume
   ```

   这将创建一个名为 `my_volume` 的卷。

2. **指定驱动程序**：

   Docker 使用默认的本地驱动程序来创建卷。但你也可以使用 `--driver` 选项来指定其他存储后端的驱动程序。例如，你可以使用 `--driver` 选项创建一个使用 NFS 存储后端的卷：

   ```bash
   docker volume create --driver local --opt type=nfs --opt o=addr=192.168.1.100,vers=4 my_nfs_volume
   ```

3. **查看卷详细信息**：

   你可以使用 `docker volume inspect` 命令来查看新创建的卷的详细信息：

   ```bash
   docker volume inspect my_volume
   ```

   这将返回一个包含卷属性的 JSON 对象，包括名称、驱动程序、挂载点等。

4. **自定义卷的标签和注释**：

   你可以使用 `--label` 和 `--annotation` 选项来为卷添加标签和注释，以便更好地组织和描述卷。

   ```bash
   docker volume create --label my_label=my_value --annotation my_annotation=my_description my_custom_volume
   ```
#### docker volume inspect
`docker volume inspect` 命令用于获取有关 Docker 卷（volume）的详细信息。它提供了关于卷的元数据和配置信息的 JSON 格式输出。这些信息包括卷的名称、驱动程序、挂载点以及可选的标签和注释。

以下是 `docker volume inspect` 命令的详细解释：

```bash
docker volume inspect VOLUME [VOLUME...]
```

- `VOLUME`：要检查的一个或多个卷的名称或 ID。

你可以指定一个或多个卷的名称或 ID，然后 `docker volume inspect` 命令将返回有关这些卷的详细信息。

以下是一些示例和说明：

1. **检查单个卷的详细信息**：

   ```bash
   docker volume inspect my_volume
   ```

   这将返回有关名为 `my_volume` 的卷的详细信息，包括卷的名称、驱动程序、挂载点等。

2. **检查多个卷的详细信息**：

   你可以同时检查多个卷的详细信息，只需在命令中列出它们的名称或 ID：

   ```bash
   docker volume inspect volume1 volume2 volume3
   ```

   这将返回列出的多个卷的详细信息。
#### docker volume ls
`docker volume ls` 是用于列出 Docker 中所有卷（volumes）的命令。它提供了有关系统中可用卷的基本信息，包括卷的名称和驱动程序。以下是该命令的详细解释：

```bash
docker volume ls [OPTIONS]
```

- `OPTIONS`：可以包括一些选项，用于自定义列出卷的行为。

常用的选项包括：

- `-q` 或 `--quiet`：只显示卷的名称，而不显示其他详细信息。
- `--filter`：根据过滤条件来列出卷。你可以使用过滤条件来筛选所需的卷。

以下是一些示例和解释：

1. **列出所有卷**：

   ```bash
   docker volume ls
   ```

   这将列出系统中的所有卷，包括卷的名称、驱动程序和挂载点。

2. **只列出卷的名称**：

   如果你只对卷的名称感兴趣，可以使用 `-q` 或 `--quiet` 选项，这将只显示卷的名称，而不显示其他详细信息：

   ```bash
   docker volume ls -q
   ```

   这将只返回卷的名称，每行一个，方便用于脚本或其他自动化任务。

3. **使用过滤条件**：

   你可以使用 `--filter` 选项来根据过滤条件列出卷。例如，要列出具有特定标签的卷，可以使用：

   ```bash
   docker volume ls --filter label=my_label=my_value
   ```

   这将列出具有标签 `my_label=my_value` 的卷。
#### docker volume rm
`docker volume rm` 命令用于删除一个或多个 Docker 卷（volumes）。删除卷将永久删除与卷关联的数据，所以要谨慎操作。

以下是 `docker volume rm` 命令的详细解释：

```bash
docker volume rm VOLUME [VOLUME...]
```

- `VOLUME`：要删除的一个或多个卷的名称或 ID。

你可以指定一个或多个卷的名称或 ID，然后 `docker volume rm` 命令将删除它们。

以下是一些示例和说明：

1. **删除单个卷**：

   ```bash
   docker volume rm my_volume
   ```

   这将删除名为 `my_volume` 的卷。请注意，这会永久删除与卷关联的数据，所以要谨慎操作。

2. **删除多个卷**：

   你可以同时删除多个卷，只需在命令中列出它们的名称或 ID：

   ```bash
   docker volume rm volume1 volume2 volume3
   ```

   这将删除列出的多个卷。同样，请确保这是你的意图，因为相关数据将被永久删除。

3. **使用 `-f` 或 `--force` 选项**：

   如果卷正在被容器引用或无法删除，你可以使用 `-f` 或 `--force` 选项来强制删除卷。这将绕过错误和警告，并强制删除卷及其数据。要小心使用此选项，因为它可能导致数据丢失。

   ```bash
   docker volume rm -f my_volume
   ```

#### docker volume prune
`docker volume prune` 命令用于清理（删除）无用的 Docker 卷。这个命令可用于删除未被任何容器使用的卷，以释放磁盘空间并减小系统资源占用。

以下是 `docker volume prune` 命令的详细解释：

```bash
docker volume prune [OPTIONS]
```

- `OPTIONS`：可以包括一些选项，用于自定义清理卷的行为。

常用的选项包括：

- `-f` 或 `--force`：强制执行清理，不会要求用户确认。
- `--filter`：使用过滤条件来筛选要删除的卷。可以根据标签等过滤条件来删除特定的卷。

以下是一些示例和解释：

1. **清理未被使用的卷**：

   ```bash
   docker volume prune
   ```

   这将删除未被任何容器引用的卷。在执行命令之前，Docker 将要求你确认清理操作。

2. **强制清理**：

   如果你希望强制执行清理而不需要确认，可以使用 `-f` 或 `--force` 选项：

   ```bash
   docker volume prune -f
   ```

   这将直接执行清理操作，不会要求用户确认。

3. **使用过滤条件**：

   你可以使用 `--filter` 选项来根据过滤条件删除特定的卷。例如，你可以根据标签来删除具有特定标签的卷：

   ```bash
   docker volume prune --filter label=my_label=my_value
   ```

   这将删除具有标签 `my_label=my_value` 的卷。
#### docker run -v / --mount创建管理卷（重点）
在 Docker 中，你可以使用 `docker run` 命令的 `-v` 或 `--volume` 标志（也可以使用 `--mount` 标志）来创建和管理卷（volumes）。以下是创建和管理卷的语法示例：

使用 `-v` 标志：

```bash
docker run -v <volume_name>:<container_path> <image_name>
```

或者使用 `--volume` 标志：

```bash
docker run --volume <volume_name>:<container_path> <image_name>
```

或者使用 `--mount` 标志：

```bash
docker run --mount source=<volume_name>,target=<container_path> <image_name>
```

- `<volume_name>`：要创建或使用的卷的名称。
- `<container_path>`：容器内的路径，用于指定卷在容器内部的挂载点。
- `<image_name>`：要运行的 Docker 镜像的名称。

示例：

```bash
docker run -v my_volume:/container/path my_image
```

或者使用 `--mount` 标志（，不要加空格）：

```bash
docker run --mount source=my_volume,target=/container/path my_image
```

这将创建一个名为 `my_volume` 的卷，并将其挂载到容器内的 `/container/path`。容器内的应用程序可以读写这个卷，而且卷的数据将在容器销毁和重新创建时保持持久性。

要使用现有的卷，只需指定卷的名称，而不需要创建新卷：

```bash
docker run -v my_existing_volume:/container/path my_image
```

或者使用 `--mount` 标志：

```bash
docker run --mount source=my_existing_volume,target=/container/path my_image
```

这将使用名为 `my_existing_volume` 的现有卷并将其挂载到容器内的 `/container/path`。

启动一个docker容器并创建一个管理卷
```bash
docker run -d -v volnginx1:/usr/share/nginx/html --name mynginx nginx:1.24.0
```
查看该卷的详细信息
```bash
docker volume inspect volnginx1
```

![image.png](https://s2.loli.net/2023/10/27/EZ2dOasR5nXx1ti.png)

根据该管理卷在宿主机上的目录路径，查看该目录下的文件
![image.png](https://s2.loli.net/2023/10/27/4leWqzvwf2K9EnZ.png)
这些文件与容器中/usr/share/nginx/html目录下的文件相同
删除容器下对应目录的文件，宿主机该目录下的文件也将被删除

加入ro选项表示该目录只读，就无法在容器中删除该目录下的文件了，但是宿主机仍然可以删除该目录下的文件
```bash
docker run -d -v volnginx1:/usr/share/nginx/html:ro --name mynginx nginx:1.24.0
```
### 绑定卷
bind mount，和管理卷的区别就是：用户能够做到精确管理绑定卷，能够指定一个路径用来挂载宿主机的存储卷。但是容器也需要指定一个特定路径来建立绑定关系
缺点：指定的路径中不能存储无关数据
#### docker run -v创建绑定卷
使用 `docker run` 命令的 `-v` 标志可以创建绑定卷（bind mounts），将主机文件或目录挂载到容器中，以便容器可以访问主机上的文件或目录。以下是创建绑定卷的语法：

```bash
docker run -v <host_path>:<container_path> <image_name>
```

- `<host_path>`：主机（宿主机）上的路径，用于指定要绑定挂载到容器的文件或目录。
- `<container_path>`：容器内的路径，用于指定绑定挂载的目标路径。
- `<image_name>`：要运行的 Docker 镜像的名称。

示例：

```bash
docker run -v /host/path:/container/path my_image
```

这将创建一个绑定卷，将主机上的 `/host/path` 挂载到容器内的 `/container/path`。容器内的应用程序可以读写这个卷，并且对卷的更改会反映在主机文件系统上，反之亦然。

绑定卷非常适合共享数据和配置文件等用途，因为它们允许容器和主机之间的数据共享。

请确保主机路径存在并且具有适当的权限，以便容器能够访问和操作这些文件或目录。
#### docker run --mount创建绑定卷
使用 `docker run --mount` 命令创建绑定卷时，你需要提供一些参数来指定绑定卷的名称和绑定路径。以下是使用 `docker run --mount` 创建绑定卷的语法：

```bash
docker run --mount type=bind,source=<host_path>,target=<container_path> <image_name>
```

- `type=bind`：指定要创建的绑定卷类型（`bind,volume,tmpfs`），这里是 `bind`，表示绑定卷。
- `source=<host_path>`：主机（宿主机）上的路径，用于指定要绑定挂载到容器的文件或目录。
- `target=<container_path>`：容器内的路径，用于指定绑定挂载的目标路径。
- `<image_name>`：要运行的 Docker 镜像的名称。

示例：

```bash
docker run --mount type=bind,source=/host/path,target=/container/path my_image
```

这将创建一个绑定卷，将主机上的 `/host/path` 挂载到容器内的 `/container/path`。容器内的应用程序可以读写这个绑定卷，并且对卷的更改会反映在主机文件系统上，反之亦然。

绑定卷类型在这里指定为 `bind`，表示创建一个绑定卷。确保主机路径存在并且具有适当的权限，以便容器能够访问和操作这些文件或目录。
#### 结论
--mount创建绑定卷时，如果绑定卷所在的宿主机目录不存在，那么无法运行容器
-v则自动创建相应的空白目录
若创建绑定卷之前，宿主机与容器中存在同名文件，那么宿主机的同名文件将覆盖容器的同名文件
这一点两种创建方式的表现是一样的
### 临时卷
tmpfs mount，容器的存储卷映射到宿主机的内存（不再是文件系统）中，一旦容器停止运行，该卷就会被移除，用于高性能的临时数据存储

临时卷无法做到容器之间的共享，并且只能在Linux系统上使用
#### docker run --tmpfs创建临时卷
`docker run` 命令的 `--tmpfs` 选项用于创建一个临时文件系统（tmpfs）挂载，将一个内存文件系统挂载到容器的指定路径上。这可以用于在容器内创建临时文件或目录，将数据保存在内存中，而不是在磁盘上，从而提高读写速度。以下是 `docker run --tmpfs` 的语法和一些相关信息：

```bash
docker run --tmpfs <container_path>
```

- `<container_path>`：容器内的路径，用于指定要创建临时文件系统的目标路径。

示例：

```bash
docker run --tmpfs /container/temp_data my_image
```

使用 `--tmpfs` 选项创建的临时文件系统会在容器停止时自动销毁，因此它适用于需要在容器内部暂存数据的情况，而不需要将数据永久保存在磁盘上。一些常见的用例包括：

1. **缓存**：将缓存数据保存在内存中，以提高读取速度。

2. **临时文件**：创建临时文件或目录，用于容器内的临时操作，这些文件将在容器停止时被删除。

3. **临时数据**：将临时数据保存在内存中，以提高读写速度，而不必将数据写入持久存储。

需要注意的是，由于数据存储在内存中，临时文件系统的容量有限，取决于宿主机的可用内存。因此，在使用 `--tmpfs` 时，要确保不会占用过多的内存，以避免影响其他容器或宿主机的性能。此外，临时文件系统中的数据在容器停止时会丢失，因此不适合需要永久保存数据的情况。
#### docker run --mount创建临时卷
`docker run --mount` 命令可以用于创建临时卷，也就是挂载到容器的一个临时文件系统。这个文件系统在容器停止时会被销毁，因此适用于存储临时数据。以下是使用 `docker run --mount` 创建临时卷的语法和示例：

```bash
docker run --mount type=tmpfs,destination=<container_path> <image_name>
```

- `type=tmpfs`：指定要创建的卷类型为临时文件系统（tmpfs）。
- `destination=<container_path>`：容器内的路径，用于指定临时卷的目标路径。
- `<image_name>`：要运行的 Docker 镜像的名称。

示例：

```bash
docker run --mount type=tmpfs,destination=/container/temp_data my_image
```

这将创建一个临时卷，将一个临时文件系统挂载到容器内的 `/container/temp_data` 目录。任何数据写入这个目录都会保存在内存中，容器停止时会被销毁，数据会丢失。

在 Docker 中，`tmpfs-size` 和 `tmpfs-mode` 是用于配置临时文件系统（tmpfs）卷的两个重要参数。

1. **tmpfs-size**：用于指定 tmpfs 卷的大小限制，即可以在 tmpfs 卷上存储的最大数据量。这个参数非常有用，因为 tmpfs 卷存储在内存中，如果不加以限制，可能会占用过多的内存资源。

   语法：
   ```
   --mount type=tmpfs,destination=<container_path>,tmpfs-size=<size>
   ```

   - `<container_path>`：容器内的路径，用于指定 tmpfs 卷的目标路径。
   - `<size>`：指定卷的大小限制，可以使用大小单位（如 "100M" 表示100兆字节）。

   示例：
   ```
   --mount type=tmpfs,destination=/container/temp_data,tmpfs-size=100M
   ```

   在上面的示例中，我们为 tmpfs 卷设置了大小限制为100兆字节。

2. **tmpfs-mode**：用于指定 tmpfs 卷的访问权限模式，即它可以是只读（read-only）或读写（read-write）的。默认情况下，tmpfs 卷是读写的。

   语法：
   ```
   --mount type=tmpfs,destination=<container_path>,tmpfs-mode=<mode>
   ```

   - `<container_path>`：容器内的路径，用于指定 tmpfs 卷的目标路径。
   - `<mode>`：指定访问权限模式，可以是 "ro"（只读）或 "rw"（读写）。

   示例：
   ```
   --mount type=tmpfs,destination=/container/temp_data,tmpfs-mode=ro
   ```

   在上面的示例中，我们将 tmpfs 卷挂载为只读，这意味着容器只能读取其中的数据，不能写入或修改它。

这两个参数允许你更精细地控制 tmpfs 卷的大小和访问权限，以满足你的具体需求。需要根据应用程序的要求来选择合适的大小限制和权限模式。
#### 临时卷的一些结论
无论是--tmpfs还是--mount创建的临时卷，若临时卷所在的目录下存在文件，那么这些文件将被覆盖/删除
tmpfs将文件系统挂载到内存中，向临时卷写入数据，在宿主机上无法查看这些数据，因为要查看这些数据就意味着这些数据需要被持久化到**磁盘**上。显然临时卷依赖内存进行存储，不会将数据写入磁盘，所以容器向临时卷写入的数据对于宿主机来说不可见
侧面说明了安全性

### 灾难恢复
根据卷的持久化特性，误删容器后，将卷绑定到其他容器并启动容器即可恢复数据
模拟mysql容器被误删，此时恢复其中数据
创建mysql容器，并创建绑定卷，目录为mysql的数据存储目录
```bash
docker run --name testmysql -v /test/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123123 -d mysql:5.7
```
此时这个目录下就存储了mysql的一些数据
![image.png](https://s2.loli.net/2023/10/27/g4Q8ujY2NdvHSWf.png)
执行操作创建test数据库并创建stu表
![image.png](https://s2.loli.net/2023/10/27/6O1dQYleCPtbkK7.png)
删除mysql容器
![image.png](https://s2.loli.net/2023/10/27/bJ2a6mvOKoHN1GC.png)
使用同样的语句启动一个相同的mysql容器，该容器中仍然保存着被删除的数据
```bash
docker run --name testmysql -v /test/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123123 -d mysql:5.7
```
### 常见问题
什么时候使用volume，bind和tmpfs？
由于volume的目录由docker管理，所以一般用于不需要规划目录的场景，如测试
bind用于需要精准控制目录的场景，如某个应用需要很大的磁盘空间，此时需要将其挂载到大目录下
tmpfs通常用于敏感文件存储
### 存储卷在实际开发中带来的问题
跨主机使用：除了临时卷以外，其他卷使用的是本地宿主机的文件系统。而本地宿主机的文件系统没有共享给其他主机，所以使用本地主机文件系统的容器无法在其他主机上运行。解决方法：搭建一个共享nfs，或者使用其他分布式存储方案，使数据和存储分离

## docker网络
docker为什么需要网络？
docker的目的：打包软件，而软件本身就有很强的通信需要，所以docker也要支持便捷的通信，即网络
### docker的网络架构
CNM（container network model），docker网络架构的基础

Docker 的 CNM 架构是指容器网络模型（Container Network Model），它是 Docker 网络架构的基础。CNM 的目的是提供应用程序在不同的基础设施上的可移植性。CNM 通过一些抽象的接口来实现容器和网络之间的连接，这些接口包括

- 沙箱（Sandbox）：沙箱包含了一个容器的网络栈的配置，包括容器的接口，路由表，和 DNS 设置。一个沙箱可以包含多个来自不同网络的端点（Endpoint）。
- 端点（Endpoint）：端点将一个沙箱连接到一个网络（Network）。端点的存在是为了将实际的网络连接从应用程序中抽象出来，这有助于保持可移植性，使得一个服务可以使用不同类型的网络驱动（Network Driver）而不用关心它是如何连接到网络的。
- 网络（Network）：CNM 并没有按照 OSI 模型来定义网络，而是根据实现方式来区分。一个网络可以是一个 Linux 桥接（Bridge），一个 VLAN，等等。一个网络是一组可以相互通信的端点的集合。没有连接到网络的端点将无法在该网络上通信。

Docker 提供了一些内置的网络驱动，如 bridge, overlay, macvlan 等，来满足不同的网络需求和环境。Docker也支持**插件式**的网络驱动
### 常见网络模式
- bridge模式：这是默认的网络模式，它会在Docker主机上创建一个虚拟网桥，每个容器都会被连接到这个网桥上，并且分配一个和网桥同网段的IP地址。在同一主机上的容器可以相互通信，但不同主机的容器不能互相通信。这种模式适用于单主机上的容器网络。
- host模式：这种模式下，容器和主机共享网络命名空间，容器可以使用主机的IP地址和端口，容器之间也可以直接通信。这种模式适用于对网络性能有高要求的场景，或者需要配置主机网络的场景。
- container模式：这种模式下，容器和另外一个已经存在的容器共享网络命名空间，它们的网络配置是一样的。这种模式适用于多个容器之间需要紧密协作的场景，比如Kubernetes中的Pod。
- none模式：这种模式下，容器没有任何网络连接。这种模式适用于不需要联网的场景，或者需要自定义网络配置的场景。
- overlay模式：这种模式下，可以创建跨越多个Docker主机的网络，容器可以在不同主机上的网络中进行通信。这种模式适用于多主机或者集群环境下的容器网络。

最经常使用的网络为overlay网络
### docker run --network
启动容器时直接指定网络
```bash
docker run --network=mynetwork myimage
```
若不指定网络，默认连接到docker0网桥中
### docker network create
`docker network create` 命令用于在 Docker 中创建一个新的网络，以便容器可以在该网络上进行通信。下面是该命令的基本语法以及一些常用的可选参数以及相应的实例：

```bash
docker network create [OPTIONS] NETWORK
```

以下是一些常用的可选参数和相应的实例：

1. `--driver`：指定网络驱动程序，它定义了网络的行为。

   示例：
   ```bash
   docker network create --driver bridge mynetwork
   ```

2. `--subnet`：指定网络的子网。

   示例：
   ```bash
   docker network create --driver bridge --subnet=172.18.0.0/16 mynetwork
   ```

3. `--gateway`：指定网络的网关 IP 地址。

   示例：
   ```bash
   docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 mynetwork
   ```

4. `--attachable`：允许其他容器连接到该网络。

   示例：
   ```bash
   docker network create --driver bridge --attachable mynetwork
   ```

5. `--label`：为网络添加标签。

   示例：
   ```bash
   docker network create --driver bridge --label=mylabel=myvalue mynetwork
   ```

6. `--ipv6`：启用 IPv6 支持。

   示例：
   ```bash
   docker network create --driver bridge --ipv6 mynetwork
   ```

7. `--aux-address`：指定附加 IP 地址，这些地址可以分配给容器。

   示例：
   ```bash
   docker network create --driver bridge --aux-address="myrouter=192.168.1.1" mynetwork
   ```
### docker network inspect
`docker network inspect` 命令用于查看 Docker 网络的详细信息。您可以使用此命令来获取有关网络的配置、容器连接和其他相关信息。以下是该命令的基本语法以及一些常用的可选参数：

```bash
docker network inspect [OPTIONS] NETWORK [NETWORK...]
```

常用的可选参数包括：

1. `--format`：指定输出格式，您可以使用 Go 模板来自定义输出格式。

   示例：
   ```bash
   docker network inspect --format '{{.Name}}'
   ```

2. `--verbose`：显示更详细的信息，包括与网络相关的容器信息。

   示例：
   ```bash
   docker network inspect --verbose mynetwork
   ```

3. `--scope`：根据网络的作用域过滤结果。可选值包括 `local`（仅限本地）和 `global`（全局）。

   示例：
   ```bash
   docker network inspect --scope local mynetwork
   ```

4. `--type`：根据网络的类型过滤结果。可选值包括 `bridge`、`overlay`、`macvlan` 等。

   示例：
   ```bash
   docker network inspect --type bridge mynetwork
   ```

以下是一些实际示例：

1. 查看特定网络的详细信息：
   ```bash
   docker network inspect mynetwork
   ```

2. 使用自定义格式查看网络信息：
   ```bash
   docker network inspect --format '{{.Name}}: {{.Driver}}' mynetwork
   ```

3. 查看网络的容器连接信息：
   ```bash
   docker network inspect --verbose mynetwork
   ```

4. 查看指定网络类型的网络列表：
   ```bash
   docker network inspect --type bridge
   ```

### docker network connect
`docker network connect` 命令用于将容器连接到一个或多个 Docker 网络。这允许容器与其他容器在同一网络中进行通信。以下是该命令的基本语法以及一些常用的可选参数：

```bash
docker network connect [OPTIONS] NETWORK CONTAINER
```

常用的可选参数包括：

1. `--alias`：为连接的容器指定别名。这允许您在网络上访问容器时使用不同的名称。

   示例：
   ```bash
   docker network connect --alias myalias mynetwork mycontainer
   ```

2. `--ip`：指定连接容器的 IP 地址。这允许您为容器分配特定的 IP 地址。

   示例：
   ```bash
   docker network connect --ip 172.18.0.2 mynetwork mycontainer
   ```

下面是一些实际示例：

1. 连接容器到特定网络：
   ```bash
   docker network connect mynetwork mycontainer
   ```

2. 连接容器到网络并指定别名：
   ```bash
   docker network connect --alias myalias mynetwork mycontainer
   ```

3. 连接容器到网络并指定特定的 IP 地址：
   ```bash
   docker network connect --ip 172.18.0.2 mynetwork mycontainer
   ```
### docker network disconnect
`docker network disconnect` 命令用于从一个 Docker 网络中断开已连接的容器。这允许您从网络中移除容器，阻止它们在网络上进行通信。以下是该命令的基本语法以及一些常用的可选参数：

```bash
docker network disconnect [OPTIONS] NETWORK CONTAINER
```

常用的可选参数包括：

1. `--force`：强制断开容器与网络的连接，即使容器不是网络的一部分也可以断开。

   示例：
   ```bash
   docker network disconnect --force mynetwork mycontainer
   ```

下面是一些实际示例：

1. 从特定网络断开容器：
   ```bash
   docker network disconnect mynetwork mycontainer
   ```

2. 强制从网络断开容器：
   ```bash
   docker network disconnect --force mynetwork mycontainer
   ```

### docker network prune
`docker network prune` 命令用于清理不再使用的 Docker 网络，以释放未使用的网络资源。这有助于维护 Docker 主机的网络资源，尤其是在大规模的容器化环境中。以下是该命令的基本语法以及一些常用的可选参数：

```bash
docker network prune [OPTIONS]
```

常用的可选参数包括：

1. `--force`, `-f`：在运行命令时不需要进行确认提示。

   示例：
   ```bash
   docker network prune -f
   ```

2. `--filter`：根据过滤条件清理网络。

   示例：清理所有标签中包含"app"的网络。
   ```bash
   docker network prune --filter "label=app"
   ```

下面是一些示例：

1. 清理不再使用的网络（将显示确认提示）：
   ```bash
   docker network prune
   ```

2. 强制清理不再使用的网络（无需确认提示）：
   ```bash
   docker network prune -f
   ```

3. 清理具有特定标签的网络：
   ```bash
   docker network prune --filter "label=mylabel=myvalue"
   ```
### docker nwtwork rm
`docker network rm` 命令用于从 Docker 中删除一个或多个不再需要的网络。以下是该命令的基本语法以及一些常用的可选参数：

```bash
docker network rm [OPTIONS] NETWORK [NETWORK...]
```

常用的可选参数包括：

1. `--force`, `-f`：强制删除网络，即使网络上有容器仍在使用。

   示例：
   ```bash
   docker network rm -f mynetwork
   ```

下面是一些示例：

1. 删除单个不再需要的网络（将显示确认提示）：
   ```bash
   docker network rm mynetwork
   ```

2. 强制删除不再需要的网络（无需确认提示）：
   ```bash
   docker network rm -f mynetwork
   ```

3. 删除多个不再需要的网络：
   ```bash
   docker network rm network1 network2
   ```

### docker network ls
`docker network ls` 命令用于列出 Docker 中存在的网络。这可以让您查看当前配置的网络以及它们的一些基本信息。以下是该命令的基本语法以及一些常用的可选参数：

```bash
docker network ls [OPTIONS]
```

常用的可选参数包括：

1. `--filter`：根据过滤条件来列出网络。您可以使用标签、驱动程序等条件来筛选网络。

   示例：列出所有标签中包含"app"的网络。
   ```bash
   docker network ls --filter "label=app"
   ```

2. `--format`：指定输出格式，您可以使用 Go 模板来自定义输出格式。

   示例：仅列出网络的名称和驱动程序。
   ```bash
   docker network ls --format "table {{.Name}}\t{{.Driver}}"
   ```

下面是一些示例：

1. 列出所有网络：
   ```bash
   docker network ls
   ```

2. 列出具有特定标签的网络：
   ```bash
   docker network ls --filter "label=mylabel=myvalue"
   ```

3. 自定义网络列表的输出格式：
   ```bash
   docker network ls --format "table {{.Name}}\t{{.Driver}}"
   ```
### bridge桥接网络
docker0网桥由docker默认创建
docker bridge向上连接到宿主机的eth0网口，向下对**多台**容器提供了veth供容器的eth0进行连接。容器通过网口连接到网桥，网桥连接宿主机的网口，通过该网口访问互联网

创建一个网络，启动两个容器并使用该网络
```bash
docker run -dit --name b3 busybox
docker run -dit --name b4 busybox
docker network connect mynet b3
docker network connect mynet b4
```
与两个容器交互，获取两个容器的IP，使用其中一个容器ping另外一个容器
```bash
docker exec -it b3 sh
ping 172.18.0.3
```
### DNS解析
docker默认的桥接网络不支持dns解析
`ping 容器名`，容器名会被dns解析成IP地址，若启动容器时没有指定网络，那么处于默认桥接网络下的容器无法通过`ping 容器名`ping通其它容器
相反，启动容器时指定了网络（非默认桥接网络），那么就能通过`ping 容器名`ping通其它容器
### host主机网络
bridge桥接网络中，容器通过网桥间接连接到宿主机的网卡。而host主机网络中，容器直接连接到宿主机的网卡，即容器和宿主机共用一个Network Namespace
在使用host网络的容器中输入ifconfig，输出的信息和在宿主机中输入ifconfig得到的信息相同
优点：不经过网桥的转发，速度快
缺点：端口冲突

启动容器时使用`--network host`选项以使用host主机网络
### Container容器网络
和host主机网络类似，容器借用某个容器的Network Namespace，而不再借用宿主机的
启动容器时使用 `--network container:要共用Nerwork Namespace的容器名`选项以使用Container容器网络
容器的IP和宿主容器的IP地址相同，停止宿主容器时，容器的IP地址就会消失
此时按顺序重启两个容器即可恢复正常

优点：共享网络资源的容器之间实现了紧密协作，网络传输的效率大大提高，可以做到负载均衡
缺点：容器之间的耦合性增加，降低了容错率
### None网络
通常应用于机密性高的应用（不想让自己的网络信息被其他主机得到）
启动容器时使用`--network none`选项以使用None网络
## Docker Compose容器编排
定义和运行（管理）多个docker容器的应用
compost中的两个重要概念：服务和项目
服务：运行相同镜像的多个实例
项目：由一组相关联的服务组成项目

通过compose可以方便的管理多个服务/项目
docker compose解决**单机**的容器管理问题与编排问题，容器之间相互的依赖，配置信息的不同都能通过compost解决
### compose核心能力
- 启动、停止和重建服务
- 查看正在运行的服务状态
- 流式传输运行服务的日志输出
- 在服务上运行一次性命令
### compose使用场景
- 单主机部署环境
- 不同环境的隔离
### compose配置文件
使用docker-compose.yml定义构成应用程序的服务，执行docker-compose up命令启动并运行整个应用程序
文档：[Compose file version 3 reference | Docker Docs](https://docs.docker.com/compose/compose-file/compose-file-v3/)

只要在当前目录下编写了docker-compose.yml文件，那么就能创建服务
```bash
docker compose up -d // 以后台方式创建服务 
docker compose down  // 销毁服务
```
创建一个服务时，默认会创建一个网络，将容器加入该网络中
![image.png](https://s2.loli.net/2023/10/27/iBcKVu4dCO1kgXl.png)
![image.png](https://s2.loli.net/2023/10/27/68Y3BCybes9JdOU.png)
prj1为当前的目录名称，prj1-web-1为容器名称，prj1-default为网络名称
销毁服务时这些资源都会被销毁

执行
```bash
docker compose config
```
正常打印说明yml文件没问题，否则存在语法问题

image：指定镜像
command：覆盖镜像的启动命令
entrypoint：覆盖容器默认的entrypoint
environment：添加环境变量
nerworks：指定容器运行的网络
volumes：绑定卷
expose：将容器的端口进行暴露但不进行和宿主机之间的映射
```
version: "3.8"
services: 
  web: 
    image: nginx:1.24.0
    command: tail -f /etc/hosts
    entrypoint: tail -f /etc/os-realease
    environment:
      TEST: 1
    networks:
      - mynet1
      - mynet2
    volumes:
      - /test/compose:/usr/share/nginx/html
networks:
  mynet1:
  mynet2:
```

将容器的80端口映射到宿主机的8999端口
```
version: "3.8"
services: 
  web: 
    image: nginx:1.24.0
    ports:
      - 8999:80
```

depends_on：
web容器依赖mysql容器，mysql容器在healthcheck成功后，web容器才启动
确定依赖关系后，现启动mysql容器再启动web容器，停止服务时，先停止web容器再停止mysql容器

以下文件test命令中，-p需要和密码连到一起写，并且密码不能用单引号
-e指定的命令需要使用单引号
```
version: "3.8"
services: 
  web:
    image: nginx:1.24.0
    depends_on:
      mysql:
        condition: service_healthy
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: 112233
    healthcheck:
      test: mysql -u root -p112233 -e 'select 1;'
      interval: 10s
      timeout: 5s
      retries: 10
```
healthcheck：定义健康检查，当被依赖的容器通过健康检查，依赖容器才能启动
- `test`：定义用于健康检查的命令
- `interval`：定义了健康检查的时间间隔，每隔 10 秒进行一次检查。
- `timeout`：指定了每次健康检查的超时时间为 5 秒。
- `retries`：指定了在标记服务为不健康之前允许的重试次数，这里设置为 10。

一开始值启动了mysql容器，web容器只是创建了并等待mysql容器的健康检查
![image.png](https://s2.loli.net/2023/10/29/rW6ZeUciL1PHwzG.png)
当mysql容器健康检查完，web容器才会启动
![image.png](https://s2.loli.net/2023/10/29/I6k5TKCHPtSR2bz.png)

env_file：从文件中导入环境变量
```
version: "3.8"
services: 
  web: 
    image: nginx:1.24.0
    env_file:
      - ./commenv
      - ./webenv
```
### compose命令
```bash
docker compose [OPTIONS] COMMAND [ARGS...]
```
- -f, --file 指定使用的 Compose 模板文件，默认为 docker-compose.yml，可以多次指定
- -p, --project-name 指定项目名称，默认将使用所在目录名称作为项目名
- docker compose ps：列出以当前目录为项目名的项目中，启动的所有容器，可以使用-p指定项目名称

#### docker compose up
```bash
docker compose up [options] [SERVICE...]
```
默认创建项目下的所有服务，也可以通过SERVICE选项创建指定的服务
- -d 在后台运行服务容器， 推荐在生产环境下使用该选项
- --force-recreate 强制重新创建容器，不能与 --no-recreate 同时使用
- --no-recreate 如果容器已经存在了，则不重新创建，不能与 --force recreate 同时使用

若修改了yml配置文件，则需要使用--force-recreate选项来重新创建容器，使配置生效
#### docker compose down
停止所有容器，并删除容器和网络
```bash
docker compose down [options] [SERVICE...]
```
- -v, --volumes 用于停止并移除 Docker Compose 应用中的服务，并且还会移除与这些服务关联的卷（volumes）

不会删除绑定卷，只会删除管理卷
#### docker compose run
与docker run的区别：docker compose run通过服务启动容器，需要指定服务名称（yml文件中的服务名）。而docker run通过镜像创建容器

通常使用docker compose run启动一次性命令
```
docker-compose run [options] <service_name> [command] [args...]
```

以下是各个部分的详细说明：

- `options`：可选参数，用于控制 `docker-compose run` 命令的行为。一些常见的选项包括：
  - `-d`：在后台运行容器。
  - `--rm`：在容器退出时自动删除容器。
  - `--no-deps`：不启动依赖的服务。
  - `--service-ports`：将服务的端口映射到宿主机上。
  - `--entrypoint`：覆盖服务的默认入口点。
  - `-u, --user`：指定执行命令的用户。

- `service_name`：要运行的服务的名称，这是定义在 `docker-compose.yml` 文件中的服务名称。

- `command`：要在容器中运行的命令。这是可选的，如果省略，则将使用服务的默认入口点命令。

- `args`：要传递给命令的参数。这也是可选的。

以下是一些示例 `docker-compose run` 命令的用法：

1. 在名为 "web" 的服务中运行 Bash shell，并以交互模式运行：

   ```bash
   docker-compose run web bash
   ```

2. 在名为 "db" 的服务中运行 `psql` 命令，并连接到 PostgreSQL 数据库：

   ```bash
   docker-compose run db psql -U username -d database_name
   ```

3. 在名为 "app" 的服务中以后台模式运行自定义命令：

   ```bash
   docker-compose run -d app custom_command
   ```

4. 在名为 "worker" 的服务中使用自定义入口点命令和参数：

   ```bash
   docker-compose run --entrypoint /path/to/entrypoint.sh worker arg1 arg2
   ```

5. 在名为 "nginx" 的服务中映射端口并以后台模式运行：

   ```bash
   docker-compose run -d --service-ports nginx
   ```

### compose实现拓扑
创建yml文件，编写服务的配置信息
```
version: "3.8"
services: 
  web: 
    image: nginx:1.24.0
    ports:
      - 8888:80
    networks:
      - testnet
    volumes:
      - ./nginxhome:/usr/share/nginx/html
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
  mysql:
    image: mysql:5.7
    networks:
      - testnet
    volumes:
      - ./mysqldata:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: 112233
    healthcheck:
      test: mysql -u root -p112233 -e "select 1;"
      interval: 10s
      timeout: 5s
      retries: 10
  redis:
    image: redis:7
    networks:
      - testnet
    healthcheck:
      test: redis-cli ping
      interval: 10s
      timeout: 5s
      retries: 10

networks:
  testnet:  
```
该工程将通过nginx镜像创建web服务，web服务依赖于mysql和redis镜像启动的两个服务
被依赖的服务需要创建 