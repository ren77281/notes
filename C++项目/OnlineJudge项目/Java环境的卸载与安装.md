卸载现有的Java安装
```bash
sudo apt purge openjdk-\*      # 如果使用的是OpenJDK
sudo apt purge oracle-java\*   # 如果使用的是Oracle Java
```
无法使用通配符时，查找相关安装包并手动删除
```bash
dpkg -l | grep openjdk
```
假设有openjdk-11-jdk openjdk-11-jdk-headless openjdk-11-jre openjdk-11-jre-headless四个安装包
```bash
sudo apt purge openjdk-11-jdk openjdk-11-jdk-headless openjdk-11-jre openjdk-11-jre-headless
```
清理残留文件
```bash
sudo apt autoremove
```
安装Java
```bash
sudo apt update
sudo apt install default-jdk    # 或者选择其他OpenJDK版本
```
设置环境变量
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$JAVA_HOME/bin
```
验证安装
```bash
java -version
```
