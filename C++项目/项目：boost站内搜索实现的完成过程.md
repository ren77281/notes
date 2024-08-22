## 预备工作
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230409181813.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230409181949.png)
下载boost_xxx_tar.gz压缩包，

Linux没有自带rz（*将本地文件上传至远端服务器的工具*），需要手动安装，centos环境下yum命令的安装是
```cpp
sudo yum install -y lrzsz
```
然后在命令行中输入rz，会跳出选择窗口，选择下载好的压缩包，将其上传至远端。注意：默认的上传路径是执行rz时所在的路径，这里创建boost_searcher目录，进入该目录，将压缩包上传。上传完成后解压
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230409183638.png)

搜索需要使用到的文件资源在boost_1_81_0/doc/html这个路径下，创建boost_searcher/data/input目录，input目录用来存储搜索需要的原始html文件。使用cp命令将所有文件拷贝
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230409185253.png)
parser.cc将所有html文件去标签，写入到同一文件中，html文件以\\3区分

### 为什么要用iter->path().extension()而不是iter->extension()？
这是因为iter的类型是recursive_directory_iterator，指向的对象类型不是path类型，而是director_entry。extension方法只有path才有
- const &：只读参数，函数只用到参数的值
- \*：输出型参数，函数中不会用到参数的值，只会修改参数
- &：输入输出型参数，函数不仅会用到参数的值，还会修改参数

## parser.cc实现
- 该文件将所有html文件拆分，提取出有效信息（*title、content、url*），将有效信息存储到目标文件中，以/3对文件html文件进行划分
1. 将源路径下的所有带路径的html文件名存储到files_list中
  - 先判断根目录是否存在
  - 使用递归迭代器recursive_directory_iterator，遍历根目录下的所有文件
  - 跳过非普通文件（*html为普通文件*）和后缀不是.html的文件
  - 将html文件push_back到files_list中
2. 解析files_list中的所有文件，将文件信息保存到results中
  - 首先是要读取文件，调用util.hpp中的ReadFIle函数，该函数将文件的内容读取到某个字符串中
  - 然后分别解析title、content、url到结构体doc_info中，注意若解析失败则continue
  - 最后注意使用移动构造，push_back类doc_infor的对象到results中
3. 将results中的信息输出到目标文件中
  - 打开目标文件，以'\\3'为内容分隔符，以'\\n'为文件之间的分隔符
  - 将所有信息，title、content、url都输入到文件中
-  这些函数返回bool值，需要判断函数是否执行成功

用ifstream类需要包含ifstream而不是iostream头文件，这是文件操作的不熟悉导致的开发效率低。

其次需要弄清楚的是string substr(size_t pos, size_t len)这个接口与size_t find之间的关系：find返回一个整数，是要查找的内容在字符串中第一次出现的下标，若要查找的内容长度大于1，那么find返回的是其第一个字符出现的下标。那么，下标的本质是是什么？由于下标从0开始，所以以i为下标的字符其实是第i + 1个字符。字符串中从下标0开始的长度为i的字符串不包括str\[i\]，而是\[0, i - 1\]之间的字符。

EnumFile实现后，带路径的文件名被存储到容器中
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230415142606.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230415161634.png)

jieba库函数介绍

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230416095506.png)

坑点：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230416103026.png)
需要把cppjieba/deps/limonp目录拷贝到cppjieba/include/cppjieba目录下

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230416184427.png)

## index.hpp
索引的过程：倒排+正排，先通过关键字进行倒排索引，找文档id对应的倒排信息，依据权值进行排序。接着根据倒排信息中的文档id进行正排索引，将文档的内容返回。

首先是两个结构struct DocInfo和InventedElem，分别存储了文档的信息（*title、content、url、id*）和倒排信息（*id、word、weight*）

利用线性表进行正排索引的建立，文档id对应下标，下标对应位置存储的是文档内容struct DocInfo，正排索引的建立需要将parser后的数据（*每个html文件以'\\n'分隔，内部的数据用'\\3'分隔*）输入到DocInfo中，然后push_back进线性表中

利用unordered_map进行倒排索引的建立，每个word映射一组InventedElem，也就是vector\<InventedElem\>，因为有些文章出现的word重复，一个word可能在多个文章中出现，所以word会映射多个InventedElem。

我用InventedElems_t表示vector\<InventedElem\>。然后是index类的编写，首先这个类是个单例，其成员有：用来进行正排的_forward_index和倒排的_invented_index与为了创建单例的锁、该类的指针，当然了成员函数都要私有化，删除拷贝构造和赋值重载

**<遇到个问题：私有静态成员变量可以在类外访问？定义初始化？与公有静态成员变量的区别？>**

单例写完，接着是索引的建立，打开parser后的文档，读取每个html文件，用boost的split将title、content与url进行拆分，存储到_forward_index中，这是正排索引的建立。然后再建立倒排

关于倒排索引的建立：将_forward_index中的title和content用jieba进行分词，创建一个unordered_map保存每个word的词频，需要统计title和content中的单词词频。然后是建立_invented_index，根据word插入一个InventedElems_t对象，接收该对象的引用，创建一个InventElem对象，将word，id写入，计算权值后将其push_back到InventedElems_t对象后。

### search.hpp
编写搜索相关的逻辑：首先是初始化搜索对象，获取index单例+构建索引（*ps：构建索引需要大概十分钟*）

关于搜索：需要返回一个能够进行打印的对象，这些对象一般都是文章。用SearchResult结构体，存储文章id，文章中word的词频，以及文章中出现的word。当然了不是所有的word，而是分词用户搜索语句后得到的word

搜索相关的逻辑：对用户输入的句子进行分词，存储到search_words中，接着遍历search_words根据word获取倒排索引的InventedElems_t，该结构存储InventedElem结构，InventedElem存储了word所在的文档id、word本身以及word在文章中的权重。注意：搜索的目标是构建SearchResult对象，返回给用户。这里用vector\<SearchResult\> searchResults存储需要返回的结果，因为要返回的文档不止一个。然后用unordered_map\<uint64_t, SearchResult\> mapIdResult将文档id和对应的结果进行映射。为什么要有这个映射表呢？首先需要明白搜索结果依赖的权重不是由一个word权重决定的，而是由多个word（*前提是你输入了多个word*）的权值相加决定的，所以需要根据不同word的倒排索引结果InventedItem的文档id，映射SearchResult对象，对该对象的权值进行累加

写的有些啰嗦：总之就是有一个映射表，映射文档id和SearchResult之间的关系，我们需要根据用户的分词结果进行倒排索引，查找word在一些文章中的权重，以及文章的id。然后根据这些信息构建SearchResult，将id和构建好的SearchResult作为pair插入到映射表中。

映射表构建完成后，将id映射的SearchResult对象插入searchResults中，接着根据searchResults中的SearchResult对象的权重进行升序排序。最后是构建Json对象，

胜利就在眼前！！！写完这个就剩服务端的http了！！！而且瞅了一眼，代码很少！！！
***
突然想到：关于C++的命名规范，自己还没有好好学习过，在写项目的过程中也没有这样的规范

Json的安装，直接yum -y install jsoncpp-devel

Json::FastWriter用于网络传输，更快。Json::StyledWriter用于调式，格式更好看

遇到个问题：索引建立的太慢，怎么解决？代码逻辑和源码一样的，现在估计是阿里云做恶，明天再说


大概找出问题了，应该是建立索引的逻辑有问题，换了服务器+原html文件，问题锁定在倒排索引的创建

...无解了，感觉是服务器配置太低，30秒的程序我跑10分钟？先不管了

运行后台进程，&，若进程正在运行，ctrl+z，bg 作业号，fg 作业号将其放入前台

将enable命令添加到~/.bash_profile中，每次登录用户，都会执行这条命令(*这是g++的使用*)
```bash
yum -y install centos-release-scl
yum -y install devtoolset-8-gcc devtoolset-8-gcc-c++ devtoolset-8-binutils
scl enable devtoolset-8 bash
```

有map和unordered_map两个头文件！

text/plain是content-type对照表中的内容

### server_http.cc
调用第三方库启用http服务，该库有一个Server类，其Get方法可以针对客户的请求设置指定的回调函数。该回调函数有两个参数，一个是用户请求，一个是服务端的响应。对于客户请求可以用has_prama方法检测请求中是否含有指定key值，用get_prama_value可以返回key的value对象，用string接收。因为前端的搜索结果会使请求中含有key值——word，所以这里要获取word的value，这是后端用来搜索的query。调用search方法，将得到的retString对象作为服务端的响应内容，然后给用户

在此之前需要设置一个默认根路径，用户访问的路径必须在该路径下，否则将被视为无效路径

sed -n '行号p' filename 打印文件的指定行
### html
\<button onclick="Search()" \>是一个按钮，用于触发Js函数
netstat -nalp
### 项目部署
nohup ./server > log/log.txt 2>&1 &