```toc
```

## Q&A
***
### 进程终止

#### Q：exit和_exit的区别？
A：
- \_eixt是一个系统调用函数，可以在man的2号手册中找到，而exit是一个C语言库函数，是语言级别的调用，可以在man的3号手册中找到
- \_exit只进行内核级别的清理工作：关闭文件描述符、释放内存、发送信号。而exit进行内核+语言级别的清理工作：关闭打开的流、刷新缓冲区、删除临时文件。比如
  - \_exit不清理语言缓冲区，进程退出后不会将其中的内容刷新出来
  - 而exit会清理语言缓冲区，进程退出后会将其中的内容刷新出来
- exit在底层调用了_exit，在清理完语言级别的数据后，再清理系统级别的数据

#### Q：内核是如何终止进程的？
A：一般情况下，内核会释放进程的数据结构和占用的内存资源。对于频繁使用的对象，内核会**假释放**该对象，将其放入到一个内存池中，当相同对象再次创建时，从内存池中取出一个对象并重置它的数据，给新进程使用。这有些类似STL的空间配置器。
***
### 进程等待

#### Q：为什么要等待子进程？
A：子进程退出，处于僵尸态，若父进程不回收子进程，过多僵尸进程占用PID资源，可能导致系统无法创建新进程。同时，父进程回收子进程也是为了获取子进程的运行情况、退出码等信息。

#### Q：如何等待子进程（wait/waitpid的区别）？
A：使用wait/waitpid等待子进程。两者的函数原型
```cpp
pid_t wait(int* status);
pid_t waitpid(pid_t pid, int* status, int options);
```
关于两者的输出型参数：status
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230405163535.png)

- 0 ~ 6：若进程被信号终止，该字段表示进程的退出信号。若没有被信号终止，该字段为全0。可以使用status & 0x7F获取该字段，或者使用EIFEXITED(status)判断进程是否正常退出
- 7 ：core dump标志位，进程运行出错时的调试开关
- 8 ~ 15：若进程正常退出，该字段表示其退出码。可以使用(status >> 8) & 0xFF或者WEXITSTATUS获取该字段

关于两函数的区别：
- wait默认阻塞，且等待的是任意子进程
- waitpid的参数options为0表示阻塞，为WNOHANG表示非阻塞
  - 非阻塞状态下，如果子进程没有退出，waitpid返回0。有子进程退出返回其pid
  - 且pid参数可以指定需要等待的子进程pid，pid为-1表示等待任意子进程
***
### 进程替换

#### Q：为什么要进程替换？
A：fork创建子进程后，父子进程数据和代码共享，此时子进程执行的代码和父进程是相似的。如果我们想让子进程执行全新的代码，完成其他的功能，就需要用到进程替换。替换后的程序可以是任何语言编写的，这也是不同语言的程序进行耦合的方式。

#### Q：进程替换的原理是什么？
A：fork创建子进程之后，若子进程没有进行数据修改，那么父子进程共享同样的页表，其映射的数据和代码完全一样。当进程替换发生，操作系统从磁盘加载新程序到内存中，然后修改子进程的页表与虚拟地址空间，使页表映射新程序的物理空间，不再与父进程共享同样的空间。要注意的是：进程替换没有产生新的进程，因为操作系统只是将新程序加载到内存，并没有为其创建进程控制块（*task_struct*）。

也可以这么理解，没有发生进程替换时，子进程的数据修改会触发数据的写时拷贝。发生进程替换时，子进程直接触发了**数据和代码**的写时拷贝。一般情况下，子进程的代码区数据和父进程相同，不会触发写时拷贝。而进程替换就是一个特殊情况，它会触发代码区的写时拷贝。

#### Q：如何替换进程？
A：关于一个程序，操作系统要执行它就要知道：1.程序所在的位置（*路径*），2.需要执行的程序名与执行该程序需要携带的选项。所以替换一个程序时，同样需要告知操作系统这两个信息：where + what。通常，我们使用系统调用，exec系列函数进行进程替换。

关于exec系列函数：
```cpp
int execl(const char *path, const char *arg, …); 
int execv(const char *path, char *const argv[]); 

int execlp(const char *file, const char *arg, …); 
int execvp(const char *file, char *const argv[]); 

int execle(const char *path, const char *arg, …, char * const envp[]); 
int execvpe(const char *file, char *const argv[], char *const envp[]); 
```
execl的函数参数：
- path：表示程序所在的路径，最好使用绝对路径
- arg：表示要替换的程序名称
- …： 可变参数列表，表示执行需要需要携带的选项，以NULL结尾
而execv使用指针数组argv\[\]代替arg和可变参数列表，指针数组具体使用见下图
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230405163414.png)

execlp和execvp函数涉及到PATH环境变量，使用echo $PATH可以查看PATH环境变量存储的路径，如果要替换的程序路径在这些路径下，就不需要写完整的绝对路径，只要写存在PATH路径下的文件名即可。比如
```cpp
execl("/usr/bin/pwd", "pwd", NULL);
execlp("pwd", "pwd", NULL);
```

execle和execvep可以携带环境变量，需要传入envp\[\]数组，具体的使用方式和argv\[\]数组一样，但环境变量的设置需要满足name = value的格式，比如
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230405163406.png)

直接设置环境变量会覆盖子进程从父进程继承下来的环境变量，若想在继承的环境变量上添加变量，可以使用environ指针，该指针指向一个字符串数组，比如
```cpp
extern char* environ[]; // 声明环境变量
execle("/usr/bin/pwd", "pwd", NULL, environ);
```
此时替换的进程就会继承父进程的环境变量。
***
### 环境变量

#### Q：为什么执行自己的程序需要"./"，而执行系统指令不需要"./"？
A：这个问题的本质是：为什么执行自己的程序需要带上路径，而执行系统指令只需要程序名？这是因为：当shell解析指令时，将指令分成指令名和选项之后会进行判断，如果指令名没有携带路径，shell会去PATH环境变量下存储的路径中查找该程序。PATH保存了多条路径，大多都是系统指令所在的路径，若将自己程序所在的路径添加到PATH中，执行该路径下的程序也不用携带路径名。但是这样会污染PATH环境变量。设置环境变量的方法
```cpp
export PATH=$PATH:要添加的路径
```
不同路径间用":"分割，export会覆盖原来的环境变量，$PATH的值是原来的PATH值，若不添加这条语句，原来的PATH值会被覆盖，不推荐这么做。