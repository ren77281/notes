## Ward安装
下载源码
```bash
git clone https://github.com/AntonyLeons/Ward.git
```
在主目录下运行，通过Dockerfile构建镜像
```bash
docker build . --tag ward
```
实例化镜像，构建容器
```bash
docker run -d --name ward -p 4000:4000 \  
-p 4001:4001 \  
--privileged=true \  
--restart always \  
ward:latest
```

查看Swap内存使用情况
```bash
swapon --show
```
这个命令会显示当前系统中已启用的交换设备的信息。如果有任何交换设备启用并使用中，它们会列在输出中。
```bash
cat /proc/swaps
```
其中，`Used` 列表示当前使用的交换空间大小。如果该值大于0，则说明系统正在使用交换空间。

```bash
free -h
```
显示系统的内存使用情况，包括交换空间。`Swap` 一行显示交换空间的总大小和使用情况。如果 `used` 列大于0，则说明系统正在使用交换空间。
## cat命令使用
当cat不指定文件时，将从标准输入中读取，并复制标准输入中的字符
当cat不指定文件，并将输出重定向到某一文件后，我们就能通过键盘输入文件内容。最后用Ctrl+D终止键盘的输入即可

（修改docker守护进程的配置文件：
  限制最大的日志大小、数量
  开启IPV6功能，指定IPV6内网地址）
```bash
cat > /etc/docker/daemon.json <<EOF
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "20m",
        "max-file": "3"
    },
    "ipv6": true,
    "fixed-cidr-v6": "fd00:dead:beef:c0::/80",
    "ip6tables":true
}
EOF
```
`<<EOF`：指定Here Document的标记，两个标记之间的内容将被传递给`cat`命令

### npm反向代理

docker-compose文件：
```
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'  # 保持默认即可，不建议修改左侧的80
      - '81:81'  # 冒号左边可以改成自己服务器未被占用的端口
      - '443:443' # 保持默认即可，不建议修改左侧的443
    volumes:
      - ./data:/data # 冒号左边可以改路径，现在是表示把数据存放在在当前文件夹下的 data 文件夹中
      - ./letsencrypt:/etc/letsencrypt  # 冒号左边可以改路径，现在是表示把数据存放在在当前文件夹下的 letsencrypt 文件夹中
```
## C++环境镜像
dockerfile文件（安装C++编译环境，以及运行测评程序依赖的动态库）
使用相对路径时，注意工作目录的切换
沙箱环境为`Ubuntu 20.04`，与宿主机相同，否则可执行文件的动态链接将存在问题
对于交互式提示，使用默认选项，否则构建镜像的过程将被卡住
```
# 第一阶段：构建环境
FROM ubuntu:20.04 

# 避免交互式提示
ARG DEBIAN_FRONTEND=noninteractive

# 更新apt包管理器并安装构建环境和依赖
RUN apt-get update && \
    apt-get install -y \
    g++ \
    libjsoncpp1 \
    libboost-system1.71.0 \
    libboost-filesystem1.71.0 \
    libboost-thread1.71.0 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /home/work

# 将当前目录下的code_process_server可执行文件复制到容器的/home/work目录下
COPY code_process_server /home/work/code_process_server

# 创建/home/code文件夹
RUN mkdir -p /home/work/code
```

## 容器编排
创建docker-compose.yml文件
```yml
version: '3.8'

services:
  container1:
    image: code_process_server
    # 容器创建后执行的命令
    command: ["/home/work/code_process_server", "8050"]
    # 与宿主机端口号之间的映射
    ports:
      - "8050:8050"

  container2:
    image: code_process_server
    command: ["/home/work/code_process_server", "8051"]
    ports:
      - "8051:8051"

  container3:
    image: code_process_server
    command: ["/home/work/code_process_server", "8052"]
    ports:
      - "8052:8052"

```
## 执行命令
```
# 进入源码及docker相关文件所在目录
cd code_process
# 根据Dockerfile构建沙箱的镜像
docker build -t code_process_server .
# 使用容器编排技术，实例化多个容器并运行
docker compose up -d
```

## 其他
通过curl检测端口是否开放
```bash
curl -X POST -d '{"key": "value"}' http://localhost:8050/code_process
```
查看发行版
```bash
lsb_release -a
```