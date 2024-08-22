```toc
```
## 什么是虚拟化容器化？
虚拟化：通过虚拟化技术将一台物理机虚拟为多台逻辑计算机，一台物理机上可用运行多台逻辑计算机，应用程序可以在独立的空间中运行而不相互影响，提高计算机的工作效率
容器化：一种虚拟化技术，特指操作系统层面的虚拟化。将用户软件实例分割成多个实例，在内核中运行
容器化是虚拟化技术的一种，而docker是一种容器化技术，docker是现今容器化技术的标准
## 为什么要虚拟化容器化？
- 资源利用率高：将物理机虚拟为多台逻辑计算机，充分利用物理机的资源。如云服务器厂商对外出售服务器时通常使用虚拟化技术将资源进行切分
- 环境标准化：实现程序的执行环境的标准化发布、部署和测试。docker的镜像提供了除内核外的完整运行时环境，确保应用运行环境的一致。实现一次构建，随处执行
- 资源弹性收缩：能够根据实际情况，动态调整软件资源
- 提供差异化环境：一台物理机上的多个服务依赖于不同的操作系统，容器化此时就能提供多种不同的环境
- 沙箱安全：一个容器出问题并不影响宿主机
- 容器对比虚拟化更轻量，启动更快：传统的虚拟机需要加载内核，而docker不需要内核，启动docker就像启动应用程序一样容易
- 维护和扩展十分容易：基于基础镜像可以扩展出不同的定制化镜像，

## 虚拟化的常见类别
**虚拟机**：虚拟机通过伪造一个硬件抽象接口，将操作系统和操作系统之上的系统嫁接到虚拟硬件上，实现和真实物理机一样的功能
具体实现有两种类型：
Vmware Workstation作为应用程序直接运行在宿主机的操作系统上
Vmware ESX不运行在宿主机的操作系统，而是直接运行在硬件上，能够直接控制硬件资源

**容器**：容器通过伪造操作系统的接口，将函数库层以上的功能嫁接到虚拟操作系统上。docker就基于Linux的namespace和Cgroup功能实现的隔离容器，可以模拟操作系统的功能。虚拟化出的逻辑计算机共用同一内核
欺骗应用程序的软件资源信息，使不同的应用程序看到不同的目录（根目录发生变化），产生资源隔离
限制应用程序的硬件资源信息，使应用程序只能使用一部分的资源
通过对硬件和软件的限制，完成虚拟化

**JVM**：JVM通过伪造函数库层，将应用程序层嫁接到函数库层上。在函数层与应用程序层中建立一层抽象层，对上提供统一的调用接口，对下适应不同的操作系统函数库

## namespace
类比C++的namespace和Java的package
不同命名空间下可以拥有相同的函数名（类名）
Linux的namespace可以隔离的资源：
- UTS：主机名与域名
- IPC：信号量、消息队列和共享内存--进程间通信
- PID：进程编号
- Network：网络设备、网络栈、端口
- Mount：文件系统挂载点
- User：用户和用户组

## unshare
### 常用命令
**dd**：
`dd`命令在Linux和类Unix操作系统上用于复制和转换文件，它具有多个参数，允许你控制数据的输入、输出、块大小等。以下是`dd`命令的一些常见参数和它们的详细说明：

1. `if`：用于指定输入文件的路径。if表示"input file"。
2. `of`：用于指定输出文件的路径。of表示"output file"。
3. `bs`：指定块大小，这是数据传输的基本单位。例如，`bs=4k`表示块大小为4KB（ibs/obs指定读取/写入的块大小）。
4. `count`：指定要复制的块数，即复制多少个块。通常，如果你想要整个输入文件，可以将`count`设置为文件的总块数。
5. `skip`：用于跳过输入文件的前几块。可以在组合`skip`和`count`参数时用于选择输入文件的特定部分。
6. `seek`：用于跳过输出文件的前几块。与`skip`一样，可以在组合`seek`和`count`参数时用于定位输出文件的特定部分。
7. `iflag`：用于设置输入文件的标志，例如`direct`用于绕过文件系统缓存，`sync`用于强制同步输入。
8. `oflag`：用于设置输出文件的标志，例如`direct`用于绕过文件系统缓存，`sync`用于强制同步输出。
9. `status`：指定处理完毕后输出的信息级别，包括`none`、`noxfer`和`progress`。
10. `conv`：用于执行不同的数据转换操作，例如`conv=notrunc`用于防止截断输出文件，`conv=sync`用于用零填充输出的末尾。
11. `count_bytes`：指定要复制的字节数，而不是块数。

**mkfs**：
`mkfs`命令用于创建文件系统在Linux和类Unix操作系统上。这个命令的作用是将一个设备或分区格式化为特定类型的文件系统，使其可以用于存储文件。以下是`mkfs`命令的详细介绍：

```
mkfs [options] device [filesystem-type]
```

- `options`：可选参数，用于控制`mkfs`的行为，可以包括以下一些常见选项：
  - `-t` 或 `--type`：用于指定要创建的文件系统的类型，例如ext4、xfs、ntfs等。
  - `-c`：检查设备上的坏块（仅适用于某些文件系统）。
  - `-v`：显示详细信息，以便跟踪文件系统创建的进度。
  - 其他特定于文件系统的选项：不同文件系统可能支持特定的选项，可以使用`man mkfs`命令来查看`mkfs`命令的手册页面，以获取更多信息。

- `device`：要格式化为文件系统的设备或分区的路径。通常是块设备文件，例如`/dev/sda1`。

- `filesystem-type`：可选参数，用于指定要创建的文件系统类型。如果不提供此参数，`mkfs`将尝试根据设备的类型和已安装的文件系统工具来确定正确的文件系统类型。

常见的`filesystem-type`包括：
  - `ext2`：Linux的第二版扩展文件系统。
  - `ext3`：Linux的第三版扩展文件系统，支持日志记录。
  - `ext4`：Linux的第四版扩展文件系统，具有更多功能和性能改进。
  - `xfs`：SGI开发的高性能文件系统。
  - `btrfs`：新一代的COW（写时复制）文件系统。
  - `ntfs`：Windows NT文件系统。
  - `fat`：FAT文件系统，包括FAT12、FAT16和FAT32。

示例用法：

1. 创建ext4文件系统：
   ```
   mkfs -t ext4 /dev/sda1
   ```

2. 创建xfs文件系统：
   ```
   mkfs -t xfs /dev/sdb1
   ```

3. 创建NTFS文件系统：
   ```
   mkfs -t ntfs /dev/sdc1
   ```

**df**：
`df`（磁盘空间使用情况）是一个常用的命令行工具，用于显示计算机上各个文件系统的磁盘空间使用情况。它提供了关于磁盘空间的信息，包括已使用的空间、可用空间和文件系统的挂载点。以下是`df`命令的详细解释：

```
df [options] [filesystem]
```

- `options`：`df`命令的一些常见选项包括：
  - `-h`：以人类可读的格式显示磁盘空间使用情况，以便更容易理解。
  - `--total`：在输出底部显示所有文件系统的总计。
  - `-T`：显示文件系统的类型。
  - `-t`：仅显示指定类型的文件系统。
  - `-x`：排除指定类型的文件系统。

- `filesystem`：可选参数，用于指定要显示信息的特定文件系统。如果省略此参数，`df`将显示所有已挂载的文件系统。

`df`命令的输出通常包括以下列信息：

1. **文件系统**：文件系统的设备文件或路径，如`/dev/sda1`。

2. **1K-块总数**：文件系统的总块数，以1K字节为单位。一个块通常是文件系统的最小分配单元。

3. **已使用**：文件系统中已使用的块数。

4. **可用**：文件系统中可用的块数。

5. **已用%**：已使用空间的百分比。

6. **挂载点**：文件系统的挂载点，也就是它在文件系统层次结构中的位置，如`/`、`/home`等。

示例用法：

- 显示所有已挂载文件系统的磁盘空间使用情况（以人类可读的格式）：
  ```
  df -h
  ```

- 显示特定类型的文件系统的磁盘空间使用情况（例如，只显示ext4文件系统）：
  ```
  df -t ext4
  ```

- 显示指定文件系统的磁盘空间使用情况（例如，只显示`/dev/sda1`文件系统）：
  ```
  df /dev/sda1
  ```

`df`命令是系统管理员和用户用于监视磁盘空间的重要工具，以确保不会耗尽磁盘空间，从而导致系统性能问题。

**mount**：
挂载命令：Windows系统自动完成挂载，插入u盘时，我的电脑处理C/D盘，还有一个新的盘
而Linux需要手动挂载系统

`mount`命令在Linux和类Unix操作系统中用于将文件系统挂载到指定的挂载点，使文件系统中的内容在挂载点处可访问。以下是`mount`命令的详细解释：

```
mount [options] device directory
```

- `options`：`mount`命令的选项，可以包括以下一些常见选项：
  - `-t`：指定要使用的文件系统类型，例如ext4、xfs、ntfs等，通常自动识别不用填写该字段。
  - `-o`：用于指定挂载选项，如`rw`（读写），`ro`（只读），`uid`（指定用户ID），`gid`（指定组ID）等。
  - `--bind`：将一个目录挂载到另一个目录，创建目录的一个副本。
  - `--make-private`：将挂载点设置为私有，不共享挂载点。
  - `--make-shared`：将挂载点设置为共享，可以被其他挂载点共享。

- `device`：要挂载的设备或分区的路径，通常是块设备文件，例如`/dev/sda1`。也可以是NFS共享或其他网络文件系统的路径。

- `directory`：挂载点的目录路径，文件系统将被挂载到这个目录下。

直接输入`mount`，将输出系统中已经挂载的文件系统，以及它们的挂载点

`mount`命令的主要功能是将文件系统挂载到文件系统层次结构中的指定位置，使文件系统中的文件和目录可在挂载点处访问。以下是一些示例用法：

1. 挂载分区到挂载点：
   ```
   mount /dev/sda1 /mnt/mydisk
   ```
   这将/dev/sda1分区挂载到/mnt/mydisk挂载点，使分区中的内容在/mnt/mydisk处可访问。

2. 挂载NFS共享：
   ```
   mount -t nfs 192.168.1.100:/shared /mnt/nfsshare
   ```
   这将远程NFS共享（在IP地址192.168.1.100上的/shared目录）挂载到/mnt/nfsshare挂载点。

3. 挂载ISO映像文件：
   ```
   mount -o loop diskimage.iso /mnt/iso
   ```
   这将一个ISO映像文件挂载到/mnt/iso挂载点，使映像文件中的内容可访问。

4. 使用特定挂载选项：
   ```
   mount -t ext4 -o rw,uid=1000,gid=1000 /dev/sdb1 /mnt/mydisk
   ```
   这将/ext4文件系统分区挂载到/mnt/mydisk，以读写方式，并设置所有者用户ID为1000，所有者组ID为1000。

`mount`命令是系统管理员和用户在Linux系统中管理文件系统的关键工具之一，允许他们将不同类型的文件系统挂载到系统中，以便访问文件和数据。

**unshare**：
`unshare`是一个Linux命令，它用于创建一个新的命名空间（namespace，与父进程不共享），在这个命名空间中，可以隔离一些系统资源，如文件系统、网络、进程等，使其独立于主机系统。`unshare`命令通常与`fork`或`exec`等命令结合使用，以便在新的命名空间中运行一个独立的进程。这种隔离和独立性对于容器化和虚拟化等场景非常有用。

`unshare`命令的基本语法如下：

```
unshare [options] [command [arguments]]
```

- `options`：`unshare`命令的选项，可以包括以下一些常见选项：
  - `-m`：创建一个新的挂载命名空间，允许在其中挂载文件系统。
  - `-u`：创建一个新的用户命名空间，允许在其中重新映射用户和组ID。
  - `-i`：创建一个新的网络命名空间，允许在其中配置网络接口和防火墙规则。
  - `-p`：创建一个新的PID（进程ID）命名空间，使新的进程在其中运行。
  - `-n`：创建一个新的IPC（进程间通信）命名空间，允许在其中使用System V IPC机制（如消息队列、信号量）。
  - `-U`：创建一个新的UTS（Unix Timesharing System）命名空间，允许在其中独立设置主机名。
  - `-C`：创建一个新的cgroup命名空间，允许在其中分配和管理cgroup。
  - `-f`：在后台运行新的命名空间。

- `command`：可选参数，指定要在新的命名空间中运行的命令。

- `arguments`：命令的参数，如果指定了`command`的话。

`unshare`命令的主要作用是创建一个新的命名空间，将命令运行在该命名空间中，从而实现资源的隔离。这对于实验、测试、容器化和虚拟化等用途非常有用。例如，你可以使用`unshare`命令创建一个新的挂载命名空间，然后在其中挂载一个文件系统，以测试文件系统的行为而不影响主机文件系统。

以下是一个示例用法，创建一个新的挂载命名空间并在其中挂载一个临时文件系统：

```bash
unshare -m /bin/bash
mount -t tmpfs none /mnt/temp
```

这将创建一个新的挂载命名空间，启动一个交互式Shell，并在其中挂载一个临时tmpfs文件系统。随后，所有的文件操作都将在这个命名空间中进行，不会影响主机系统。
### 空间隔离unshare的使用
进程A创建了一个新的命名空间，A不会被放入新的命名空间，其所有子进程将被放入新的进程空间
![image.png](https://s2.loli.net/2023/10/23/LZPAmrSgv2nBJea.png)
报错的原因：
总所周知，bash作为一个解释程序，启动时将fork几个子进程以完成一些初始工作，当这些工作完成后，这些子进程就会退出

用于unshare是一个外建命令，由旧空间bash的子进程执行。该子进程创建了新的命名空间后，启动/bin/bash，以其作为新命名空间的1号进程
但是/bin/bash是在当前进程启动的，而当前进程是旧空间bash的一个子进程，属于旧命名空间，所以该子进程启动的/bin/bash也属于旧空间。其PID不为1（旧空间的PID为1的进程已经被使用）

而当前进程启动/bin/bash时，bash要创建几个子进程以执行用户输入的外建命令。**当前进程创建的子进程将被放入新命名空间**，PID的分配按照创建进程的先后时间按顺序分配

所以bash创建的第一个子进程将成为新命名空间PID为1的进程，当该子进程退出（PID为1的进程）时，将引起系统崩溃。因为1号进程是一个特殊的进程，负责孤儿进程的资源回收，至于为什么报的错有关内存分配，这就和Linux内核的实现逻辑有关

要使/bin/bash成为1号进程，需要加上-f选项。表示当前进程执行完unshare之后，创建出一个子进程，以执行/bin/bash程序

解释：当前进程执行完unshare之后，仍然属于旧命名空间，创建子进程后，子进程属于新命名空间，并且该子进程的PID为1（在此之前，新命名空间中没有进程）。让该子进程执行/bin/bash程序，那么该bash程序就能成为新命名空间的1号进程

此外，加上"--mount-proc"选项，表示将新的/proc文件系统挂载到新的命名空间中。由于unshare -fp只隔离了命名空间，仍然共享了很多资源，如文件系统。--mount-proc将自动执行mount -t proc proc /proc命令，用新命名空间的文件系统与旧命名空间的文件系统隔离
![image.png](https://s2.loli.net/2023/10/23/LDrmH63ocvS5hUu.png)
### 挂载点隔离mount的使用
unshare创建文件系统隔离的命名看见
```bash
unshare -fm /bin/bash
```
创建一个80MB的镜像文件
```bash
dd if=/dev/zero of=data.img bs=8k count=10240
```
根据data.img进行文件创建ext4文件系统
```bash
mkfs -t ext4 ./data.img
```
创建文件挂载目录
![image.png](https://s2.loli.net/2023/10/23/gZGma4bQreIjL7p.png)
将镜像文件data.img挂载到目录/data/datamount
```bash
mount -t ext4 ./data.img /data/datamount
```


退出该命名空间后，无法看到该文件系统的挂载，说明实现了unshare对文件系统的隔离
但是可以空间创建的目录
这是因为mount隔离后，根目录没有发生改变，所以整个文件系统并没有因为mount隔离发生变化。但是新旧命名空间的挂载点视图不同，mount隔离后的新空间在挂载点创建了新文件后，对于其他命名空间是不可见/不可访问的

目录可见，而文件不可见：因为mount隔离的是挂载点，不是**文件系统**

![image.png](https://s2.loli.net/2023/10/23/cgJ7XKEfm61OpVN.png)
![image.png](https://s2.loli.net/2023/10/23/IQC7elGn3ouSLsP.png)
但是当新命名空间向mount隔离的挂载点写入数据/创建文件时，其他命名空间无法看见这些数据/文件
## Cgroup技术
Cgroup（Control Groups）是Linux内核提供的一种机制，可以将系统任务整合/划分，从而划分成不同的资源组（每组的资源按照等级划分）
为什么要使用**Cgroup**？为了限制容器使用的资源，防止某一容器使用的资源过多，其他容器无法使用资源的情况发生
### 使用
**pidstat**：作为sysstat的一个命令，用于监控全部或者指定进程的CPU、内存、设备IO等资源的占用情况，并且实时统计以报表的形式展示出来

```
pidstat [options] [interval] [count]
```

- `options`：`pidstat`命令的选项，可以包括以下一些常见选项：
  - `-u`：显示CPU使用情况。
  - `-r`：显示内存使用情况。
  - `-d`：显示磁盘I/O活动。
  - `-w`：显示上下文切换。
  - `-t`：显示线程统计信息。
  - `-p`：指定要监视的进程ID。-p ALL表示监控所有进程，默认只监控活动进程。
  - `-C`：指定要监视的**进程名**。
  - `-h`：以人类可读的格式显示输出。
  - `-V`：显示`pidstat`的版本信息。

每秒统计1000次，该命令将不停地打印
```bash
pidstat 1 1000
```
安装：
```bash
apt install sysstat -y
```

`pidstat -r`显示的字段：

1. **UID**：这是用户标识符（User ID）的缩写，表示运行进程的用户。每个进程都由一个用户启动并属于一个用户。
2. **PID**：这是进程标识符（Process ID），是每个进程在系统中唯一的标识符。
3. **minflt/s**：这表示每秒的次要缺页错误（minor page faults）的数量。次要缺页错误通常是由于页面不在内存中而导致的，但它们通常可以通过磁盘或内核中的其他页面来解决，而无需从磁盘读取。
4. **majflt/s**：这表示每秒的重要缺页错误（major page faults）的数量。重要缺页错误通常涉及到从磁盘读取页面到内存中，因为所需的页面不在内存中。
5. **VSZ**：这是虚拟内存大小（Virtual Memory Size），表示进程所使用的虚拟内存的大小。虚拟内存包括了进程的代码、数据、堆栈等。
6. **RSS**：这是驻留内存大小（Resident Set Size），表示进程当前在物理内存中占用的内存大小。
7. **%MEM**：这是进程使用的物理内存占系统物理内存总量的百分比。
8. **Command**：这是进程的命令行，表示进程的启动命令和参数。

这些信息用于监视和分析正在运行的进程的性能和资源使用情况。这对于系统管理、性能优化和故障排除非常有用。

**stress**：
stress命令是一个Linux系统下的系统压力测试工具，它可以模拟系统负载较高时的场景，比如消耗CPU、内存、磁盘IO等资源stress命令的基本语法是：

```bash
stress <options>
```

其中，options可以指定不同的参数来控制资源的使用量和时间。常用的参数有：

- -c, --cpu N：产生N个进程，每个进程都反复不停地计算随机数的平方根。
- -i, --io N：产生N个进程，每个进程反复调用sync()函数，将内存上的内容写到硬盘上。
- -m, --vm N：产生N个进程，每个进程不断分配和释放内存。
- –vm-bytes B：指定分配内存的大小。
- –vm-stride B：不断地给部分内存赋值，让COW（Copy On Write）发生。
- –vm-hang N：指示每个消耗内存的进程在分配到内存后转入睡眠状态N秒，然后释放内存，一直重复执行这个过程。
- –vm-keep：一直占用内存，区别于不断地释放和重新分配（默认是不断释放并重新分配内存）。
- -d, --hdd N：产生N个不断执行write和unlink函数的进程（创建文件，写入内容，删除文件）。
- –hdd-bytes B：指定文件大小。
- -t, --timeout N：在N秒后结束程序。
- –backoff N：等待N微秒后开始运行。
- -q, --quiet：程序在运行的过程中不输出信息。
- -n, --dry-run：输出程序会做什么而并不实际执行相关的操作。
- –version：显示版本号。
- -v, --verbose：显示详细的信息。

通过使用stress命令，可以帮助我们更好地理解系统瓶颈并掌握性能检测工具的用法。

在一个会话下执行：
```bash
pidstat -C stress -p ALL -u 2 1000
```
在另一个会话下执行：
```bash
stress -c 1
```
![image.png](https://s2.loli.net/2023/10/23/ldDEV1ntQAXFhM7.png)
对于pidstat -u显示的字段：
- %usr：表示进程在用户态运行所占CPU时间的百分比。
- %system：表示进程在内核态运行所占CPU时间的百分比。
- %guest：表示进程在虚拟机上运行所占CPU时间的百分比。
- %wait：表示进程等待运行所占CPU时间的百分比。
- %CPU：表示进程总共占用的CPU时间的百分比。

查看当前Linux系统发行版
```bash
cat /etc/*release*
```
**查看Cgroup的版本信息**
```bash
cat /proc/filesystems | grep cg
```
![image.png](https://s2.loli.net/2023/10/24/9Tt3mbH1PEUYDBv.png)
出现cgroup2表示当前系统支持cgroup v2的版本

查看Cgroup支持哪些资源的控制（子系统查看）
```bash
cat /proc/cgroups
```
![image.png](https://s2.loli.net/2023/10/24/SDdtZ9I1EJOH7jN.png)

有关cgroup的挂载信息查看
```bash
mount | grep cgroup
```
![image.png](https://s2.loli.net/2023/10/24/fXxO3KqZG6ECARM.png)
输出显示了各种cgroup控制器，如cquest、cpu、memory等，以及它们的挂载点
通过这些挂载点可以修改资源限制信息

显示当前进程所属的cgroup信息，\$\$表示Shell进程本身的PID
```bash
cat /proc/$$/cgroup
```
![image.png](https://s2.loli.net/2023/10/24/2UCK8EycGDuzPFO.png)
cat /proc/\$\$/cgroup 的输出格式是这样的：

hierarchy-ID:controller-list:cgroup-path

其中，hierarchy-ID 是一个数字，表示 cgroup 的层级编号；controller-list 是一个逗号分隔的列表，表示该层级上挂载的 cgroup 控制器的名称；cgroup-path 是一个相对路径，表示该进程所在的 cgroup 在该层级上的位置。

通过`mount | grep cgroup`查看cgroup的挂载点，再通过`cat /proc/\$\$/cgroup`查看当前进程的cgroup信息，两者结合就能找到限制资源的配置文件
### 使用cgroup对内存进行控制
通过`mount | grep cg`找到有关内存的cgroup文件路径：
![image.png](https://s2.loli.net/2023/10/24/neYic9kXHFANWRJ.png)

进入memory目录并创建一个目录用来测试我们的程序
![image.png](https://s2.loli.net/2023/10/24/ivcWQpLUxYTtI6B.png)
目录创建后，系统自动会将关于内存的控制信息初始化
进入该目录，`ll`发现已经存在许多配置文件
![image.png](https://s2.loli.net/2023/10/24/7CIgAW3otZaQNjk.png)

将"20M"（20971520）写入memeory.limit_in_bytes文件中
```bash
echo "20971520" > memory.limit_in_bytes
```
![image.png](https://s2.loli.net/2023/10/24/Wen8zphyVGRJPET.png)

创建两个会话，在其中的一个会话：用stress生成一个进程
```bash
stress -m 1 --vm-bytes 50m
```
`-m 1`表示只生成一个进程，`--vm-bytes`表示该进程将占用50m的内存

在另一个会话中输入，每2秒监控一次stress进程所有信息
```bash
pidstat -C stress -p ALL -r 2 10000
```
![image.png](https://s2.loli.net/2023/10/24/4FtLJH8P5S3vR9j.png)

在最开始的会话中执行，将stress创建的用来压力测试的进程的PID写入tasks文件中，使cgroup对其进行监控
```bash
echo "339923" > tasks
```
添加完监控后，可以看到该压力程序要么直接退出，要么占用的内存减少到20m以下
![image.png](https://s2.loli.net/2023/10/24/bqTmKfp4QP6IVDj.png)
可以看到虚拟内存的大小不变，但是虚拟内存使用的物理内存减少到20m以下
### 使用cgroup对cpu进行控制
通过`mount | grep cp`找到cpu控制组的挂载点
![image.png](https://s2.loli.net/2023/10/24/VvlirHPe76ApJaS.png)

进入该挂载点，创建cpulimit目录用来测试我们的程序
![image.png](https://s2.loli.net/2023/10/24/SBRaiLQ4I23F8G6.png)

创建两个会话，一个执行
```bash
stress -c 1
```
创建一个占用一个cpu核心的进程

一个执行：
```bash
pidstat -C stress -p ALL -u 3 10000
```
监控该进程的资源使用

设置 cproup 的 cpu 使用率为 30%，cpu 使用率的计算公式 cfs_quota_us/cfs_period_us
1. cfs_period_us 表示一个 cpu 带宽，单位为微秒。系统总 CPU 带宽 ，默认值 100000
2. cfs_quota_us 表示 Cgroup 可以使用的 cpu 的带宽，单位为微秒。cfs_quota_us 为-1，表示使用的 CPU 不受 cgroup 限制。cfs_quota_us 的最小值为1ms(1000)，最大值为 1s

![image.png](https://s2.loli.net/2023/10/24/w1dX9SZA6lbYNWx.png)
Cgroup默认不限制进程的cpu使用率

执行：修改cpu.cfs_quota_us文件，将进程的PID写入tasks文件中，表示添加监控
```bash
echo "20000" > cpu.cfs_quota_us
echo "347450" >> tasks
```
可以看到该进程的cpu使用率下降到20%
![image.png](https://s2.loli.net/2023/10/24/fgWFKnvNpsIAydV.png)

