用户请求某一道题目时（点击某一道题目），将该题的in和out读取到内存中
model.hpp：
负责与后端数据的交互
后端数据分为两个模块，一个是所有题目的简要信息，展示`题号 标题 难度 时空限制`，使用结构体BriefPuzzle存储
一个是题目的详细信息，包括`题号 时间限制 内存限制 题目描述（使用markdown编写） 输入 输出（vector<string>占用较大空间）`，使用Puzzle存储

Model类拥有两个私有成员，unordered_map<uint64_t, Puzzle>存储当前系统已经加载的题目（详细信息），map<uint64_t, BriefPuzzle>存储当前系统所有的题目信息（简要）

control.hpp：调用model.hpp获取题目信息，再将题目信息序列化成Json串，通过网络将Json发送给部署编译模块的主机

control.hpp
LoadBalancer类：提供有关负载均衡的方法，包括加载编译机器的配置，离线编译服务，上线编译服务，选择负载最少的服务器
