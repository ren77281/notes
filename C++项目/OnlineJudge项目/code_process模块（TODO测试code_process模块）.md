对用户提交的代码进行处理，即编译并运行

compiler.hpp，命名空间为ns_compiler
- Compiler类，提供静态方法Compile
  - 返回值类型为bool，表示编译成功/失败
  - 参数类型为string，表示源文件名
  - 函数具体行为：创建子进程，通过子进程编译源文件，父进程等待子进程编译完成，检查是否编译成功并返回（通过可执行文件是否存在检查编译是否成功）

run.hpp
- Runer类，提供静态方法Run
  - 返回值为int，小于0表示程序运行发生严重错误而主动终止，等于0表示程序正确运行，大于0表示程序运行异常（收到新号而退出）
  - 

code_process.hpp
- CodeProcess类，提供静态方法Start
  - 返回值为void
  - 参数为两个Json格式的string串，inJson与outJson，分别为输入型参数与输出型参数
  - 函数具体行为：反序列化Json串，得到用户代码（标准输入、资源限制），编译并执行用户代码。将运行结果序列化为Json串，最后删除编译运行时产生的临时文件

Start函数涉及到的其他细节：
如何生成编译运行产生的临时文件名，并保证其唯一性？
使用毫秒级时间+原子计数器来保证文件名的唯一性

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202402290927676.png)

使用httplib，将code_process_server.hpp包装成一个网络服务，将服务部署在对应主机的对应端口上
用户将发送的Json串含有属性：
code：用户提交的代码
input：用户的标准输入
memLimit：该题的内存限制
cpuLimit：该题的时间限制