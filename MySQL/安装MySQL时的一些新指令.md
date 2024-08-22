`systemctl`是一个脚本，用来控制系统中正在运行的程序，常用参数用`stop`，`start`，`restart`，需要输入进程名，比如`systemctl stop mysqld`，`d`表示该进程是一个守护进程

`rpm -qa`查找系统中所有安装包
卸载安装包，`yum remove 参数名`，其中参数名是`rpm -qa`查找出的安装包名字

`rpm -qa | grep mysqld | xargs yum -y remove`，其中`xargs`的作用是将标准输入中的数据作为程序的参数，喂给程序

`ls etc/yum.repos.d/`目录下存储了系统安装的yum源
`cat /etc.rehat-release`查找Linux发行版本
`rpm -ivh 参数名`安装rpm包
为什么要安装yum源？可以提供一键部署环境
`yum list | grep mysql`查看repo是否正常工作
`yum install -y mysql-community-server`，安装mysql

如何验证？
`which mysql`
`which mysqld`
`ls /etc/my.cnf`
`mysql -v`

`mysqld`是服务端，`mysql`是客户端
`systemctl start mysqld
`netstat -nltp`可以查看网络端口，其中有mysql的服务
`grep 'temporary password' /var/log/mysqld.log`查找临时密码
`mysql -uroot -p`用临时密码登录
有些版本可以用空密码登录
`vim /etc/my.cnf`，在`[mysqld]`中添加`skip-grant-tables`，设置免密登录
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230611113215.png)

数据的存储路径，即使mysql被删除，数据也不会被删除