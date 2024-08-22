```toc
```

## 序列化概念
打电话时，一方将声音转换成光、电信号进行传输，一方接受后将其重新转换成声音信号。这个过程涉及到了序列化与反序列化

网络传输中，发送方将信息转换成二进制（字节）序列，接收方将二进制序列解析（恢复）成原始信息，这是数据的序列化与反序列化过程

数据持久化存储时，先将数据序列化成二进制序列，再进行持久化。读取数据时，反序列化这些数据，以获取原始数据

序列化的实现，目前主流协议有：JSON，XML，ProtoBuf
ProtoBuf：将结构化数据进行序列化的一种方式

官方文档给出的ProtoBuf特点：
- 语言无关、平台无关：支持多种语言（C++、Python...)多个平台（Windows、Linux...）
- 高效：（序列化结果）比XML更小，更快，更为简单
- 扩展性、兼容性好：更新原有数据结构，但不影响与破坏原有的旧程序

对于C++来说：**ProtoBuf需要依赖编译生成的头文件与源文件使用**
以下是C++中某个类的伪定义，该类含有序列化方法
```cpp
class xxx{
	// 1. 类的属性字段
	// 2. 处理数据的方法
	// 3. 序列化、反序列方法
};
```
以下是ProtoBuf中某个类的伪定义，该类也含有序列化方法
```cpp
message xxx{
	// 1. 类的属性字段
	// 不需要定义C++需要定义的2、3
};
```
PB中的类能够通过编译器自动生成C++需要手动定义的2、3方法，节省了大量开发时间

补充：PB规定，message必须定义在`.proto`文件中，`protoc编译器`编译`.proto`文件生成message的一系列接口方法
## PB的Win安装
### Protoc编译器
github链接：[Releases · protocolbuffers/protobuf · GitHub](https://github.com/protocolbuffers/protobuf/releases)
![image.png](https://s2.loli.net/2023/10/13/eki3jfgGxmdlSXb.png)
随便选择一个安装，解压完有两个文件
![image.png](https://s2.loli.net/2023/10/13/S8fDkuWUrahPV3X.png)
protoc.exe编译器在bin目录下，找到path环境变量，添加新的变量填入protoc.exe所在路径

搜索环境变量
![image.png](https://s2.loli.net/2023/10/13/ugBsTXxGDmvkcFo.png)
找到path，点击编辑
![image.png](https://s2.loli.net/2023/10/13/yLwi1o5gDkeBIC6.png)
点击新建，添加protoc.exe所在路径名
![image.png](https://s2.loli.net/2023/10/13/9onfl3VyUm6LTCJ.png)
一路点击确定即可
在cmd下输入`protoc --version`出现版本号，说明安装成功
![image.png](https://s2.loli.net/2023/10/13/1dmvyVClAU7kiao.png)

***
## 快速上手PB
实现通讯录1.0版本
- 联系人包含：姓名、年龄
- 对一个联系人的信息使用PB进行序列化，将序列化的结果打印出来
- 对序列化的结果进行PB反序列化，将反序列化的结果打印出来
### .proto文件编写
所有**类的实现**都放在.proto文件中，注意：命名必须以.proto为后缀，并且文件名最好是小写字母，文件名也可以用下划线分割
变量命名也最好是小写字母，也可以用下划先分割

**字段编号**：
字段编号范围：1~$2^{29}$-1，其中19000~19999不可用（PB的源码中，使用了这些字段）
序列化时，字段编号也会被序列化，编号1~15只会占用1Byte，编号16~2047占用2Byte，因此对于使用频繁的字段，应该使用（预留）较小的编号，使最终的序列化结果尽可能小
#### .proto文件编译
```cpp
syntax = "proto3"; // 首行：语法指定行，指定PB的语法，不指定默认proto2语法
package contacts;  // 编译的c++代码将生成contacts命名空间，定义的类都在该命名空间中

// 联系人message定义
message PeopleInfo{
    string name = 1; // 姓名，字段编号为1
    int32 age = 2;   // 年龄，字段编号为2，必须使用字段编号，且字段编号不能相同，否则会报错
} // 没有分号
```

```
protoc --cpp_out=存储编译文件的路径 需要编译的proto文件
protoc --cpp_out=. contacts.proto
```
protoc：PB提供的编译命令
cpp_out：指定编译生成cpp可用的文件
.：.表示当前目录
contacts.proto：原proto文件
![image.png](https://s2.loli.net/2023/10/16/GIi239eMYTSpWsP.png)
编译后将生成两个cpp可用文件，一个是源文件.pb.cc，一个是头文件.pb.h

若携带`-I 路径`：表示指定.proto文件的搜索目录，若有多个proto文件，则可用有多个路径
不携带`-I`选项，默认从当前路径搜索proto文件

#### 源文件编译
main.cc：
```cpp
#include <iostream>
#include <string>
#include "contacts.pb.h"

int main()
{
    // 将联系人信息序列化，并用string保存二进制序列，打印结果
    std::string people_str;
    contacts::PeopleInfo p1;
    p1.set_name("李四");
    p1.set_age(18);
    if (!p1.SerializeToString(&people_str))
    {
        std::cerr << "序列化失败!\n";
        return -1;
    }
	for (int i = 0; i < people_str.length(); i++)
	{
		std::cout << std::hex << (int)(unsigned char)people_str[i] << ' ';
	}
	std::cout << "\n";
    // 对序列化结果进行反序列化，并打印结果
    contacts::PeopleInfo p2;
    if (!p2.ParseFromString(people_str))
    {
        std::cerr << "反序列化失败\n";
        return -1;
    }
    std::cout << "反序列化结果为:" << people_str << "\n"
            << "姓名: " << p2.name() << "\n"
            << "年龄: " << p2.age() << "\n";
    return 0;
}
```
.proto文件定义了PeopleInfo类，其包含两个字段name和age，main.cc引用.proto生成的.pb.h头文件就能使用PeopleInfo类
main.cc初始化一个PeopleInfo对象并赋值。将该对象进行序列化并打印，最后将其反序列化并打印

编译指令：需要将main.cc和.proto生成的.pb.cc一起编译
```
g++ -o test test.cc contacts.pb.cc -lprotobuf -std=c++11
```
运行结果：
![image.png](https://s2.loli.net/2023/10/16/xnMAWRNe5aykDOU.png)
PB的序列化结果为二进制序列，因为二进制的破解成本更大，所以相较于xml和Json而言，PB更加安全
***
## proto3语法
### 通讯录1.0实现
- 将通讯录序列化后并写入文件中（通讯录含有多个联系人）
- 从文件解析出通讯录，并打印到标准输出
- 新增联系人属性：姓名、年龄、电话

PB允许定义嵌套类型(message中定义message)，嵌套message中的字段编号不与外层message冲突
repeated关键字：用来修饰数组
一个.proto文件使用另一个.proto文件中的message时，需要`import 文件名.proto`
使用具体message时，在类名之前需要带上命名空间
![image.png](https://s2.loli.net/2023/10/16/y1IOuKE5AcrNYvl.png)

**write.cc的修改**：从标准输入中获取联系人的信息，将其序列化到输出文件中
```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <cstdio>
#include "contacts.pb.h"
using namespace std;

void AddPeopleInfo(contacts2::PeopleInfo* people)
{
    cout << "-------------新增联系⼈-------------" << endl;
    cout << "输入联系人姓名: ";
    string name; cin >> name;
    people->set_name(name);

    cout << "输入联系人年龄: ";
    int age; cin >> age;
    people->set_age(age);
    cin.ignore(256, '\n');

    for (int i = 0; ; ++ i)
    {
        cout << "输入联系人电话，次数: " << i + 1 << "(只输入回车完成新增): ";
        string number; getline(cin, number);
        // cin.ignore(256, '\n');
        if (number.empty()) break;
        contacts2::PeopleInfo_Phone* phone = people->add_phone_number();
        phone->set_number(number);
    }
    
    cout << "-----------添加联系⼈成功-----------" << endl;
}

int main()
{
    contacts2::Contacts contacts;
    // 读取本地已经存在的通讯录文件
    fstream input("contacts.bin", ios::in | ios::binary);
    if (!input)
        cout << "文件未找到，创建了一个新的文件\n";
    else if (!contacts.ParseFromIstream(&input))
    {
        cout << "反序列化文件失败\n";
        input.close();
        return -1;
    }
    // 向通讯录中添加一个联系人
    AddPeopleInfo(contacts.add_contacts());
    // 将通讯录写入本地文件中
    fstream output("contacts.bin", ios::out | ios::trunc | ios::binary);
    if (!contacts.SerializeToOstream(&output))
    {
        cout << "序列化通讯录失败\n";
        input.close();
        output.close();
        return -1;
    }
    cout << "序列化通讯录成功\n";
    input.close();
    output.close();
    return 0;
}
```
对象.ParserFromIstream(&input)：从标准输入（输入对象）中反序列化数据到对象中，读取文件+反序列化
对象.SerializeToOstream(&output)：将数据序列化到输出文件（输出对象）中，序列化+写入文件
两者的参数为标准输入/输出对象的地址，成功返回true，失败返回false

对象.add_'repeated修饰的字段名'()：在数组中尾插一个数组成员，函数返回新增成员的地址，用repeated修改的字段类型的指针接收。代码用PeopleInfo\*接收
![image.png](https://s2.loli.net/2023/10/16/sxO6ybSrlJAFf27.png)

对象.set_字段名(字段值)，修改该对象的字段

`cin.ignore(256, '\n')`，最少忽略缓冲区的256个字符，直到遇到`\n`（会将`\n`忽略）
代码中使用，是为了在读取字符串时，减少`\n`的干扰

cin无法读取`\n`，cin将其作为分割符并读取其之前的有效字符，所以无法通过cin输入一个空串
一直输入回车无法终止cin
使用`getline(cin, str)`可以使cin成为空串

运行结果：通过hexdump工具查看二进制文件（-C选项）
![image.png](https://s2.loli.net/2023/10/16/DryFL2jMHo15lKv.png)

**read.cc的修改**：读取文件中的联系人信息，并将其反序列化到标准输出中
```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <cstdio>
#include "contacts.pb.h"
using namespace std;

void PrintContacts(contacts2::Contacts& contacts)
{
    for (int i = 0; i < contacts.contacts_size(); ++ i)
    {
        cout << "联系人" << i + 1 << ": " << "\n";
        const contacts2::PeopleInfo people = contacts.contacts(i);
        cout << "姓名: " << people.name() << "\n";
        cout << "年龄: " << people.age() << "\n";
        for (int j = 0; j < people.phone_number_size(); ++ j )
        {
            const contacts2::PeopleInfo_Phone number = people.phone_number(j);
            cout << "联系人电话" << j + 1 << ": " << number.number() << "\n";
        }
    }
}

int main()
{
    contacts2::Contacts contacts;
    // 读取已经存在的通讯录文件
    fstream input("contacts.bin", ios::in | ios::binary);
    if (!contacts.ParseFromIstream(&input))
    {
        cout << "反序列化文件失败\n";
        input.close();
        return -1;
    }

    // 打印通讯录列表
    PrintContacts(contacts);
    return 0;
}

```

数组成员的访问：若一个对象中含有数组类型，那么
对象.数组名(i)，访问数组的第i+1个成员，用对应的成员类型接收（需要const修饰）
对象.数组名_size()，返回数组的大小

访问对象的某个成员：
对象.成员名()，返回该成员的左值引用，用const修饰，所以无法修改

运行结果：
![image.png](https://s2.loli.net/2023/10/17/ioHcfrUB57lV1J9.png)

### decode选项
用标准输出中读取指定类型的二进制message，并以文本格式将内容输出到标准输出中
即，将二进制数据转换成文本数据
```
protoc --decode=类型名 定义该类型的.proto文件 < 二进制文件名
protoc --decode=contact2.Contacts contacts.proto < contacts.bin
```
结果：
![image.png](https://s2.loli.net/2023/10/17/mZ1Js7PG2iUXWlC.png)
（一个汉字由三个字符表示）

### 通讯录2.1
相比于2.0，.proto文件的PeopleInfo_Phone类添加了一个枚举类型type，该枚举有两个常量值，0表示移动电话，1表示固定电话
write.cc从标准输入获取信息以新增联系人时，需要告知用户输入电话类型并读取
read.cc反序列化数据后，需要打印电话类型

contacts.proto
```cpp
syntax = "proto3";
package contacts2;

enum PhoneType
{
	MP = 0;  // 移动电话
	TEL = 1; // 固定电话
}
    
message PeopleInfo_Phone
{
    string number = 1;    
    PhoneType type = 2;
}

message PeopleInfo
{
    string name = 1;
    int32 age = 2;   
    repeated PeopleInfo_Phone phone_number = 3;
    
}

message Contacts
{
    repeated PeopleInfo contacts = 1;
}
```
write.cc：
```cpp
void AddPeopleInfo(contacts2::PeopleInfo* people)
{
    cout << "-------------新增联系⼈-------------" << endl;
    cout << "输入联系人姓名: ";
    string name; cin >> name;
    people->set_name(name);

    cout << "输入联系人年龄: ";
    int age; cin >> age;
    people->set_age(age);
    cin.ignore(256, '\n');

    for (int i = 0; ; ++ i)
    {
        cout << "输入联系人电话，次数: " << i + 1 << "(只输入回车完成新增): ";
        string number; getline(cin, number);
        if (number.empty()) break;
        contacts2::PeopleInfo_Phone* phone = people->add_phone_number();
        phone->set_number(number);

        cout << "输入电话类型(1、移动 2、固定): ";
        int type; cin >> type;
        cin.ignore(256, '\n');
        if (type == 1)
            phone->set_type(contacts2::PhoneType::MP);
        else if (type == 2)
            phone->set_type(contacts2::PhoneType::TEL);
        else 
            cout << "输入有误!\n" << "\n";
    }
    
    cout << "-----------添加联系⼈成功-----------" << endl;
}
```
对象.set_字段名()，设置枚举变量的值，用宏作为参数：命名空间::枚举类型名::枚举常量名
若嵌套定义enum，那么枚举类型名会变得很长，由外层message名_枚举类型名拼接而成

read.cc的修改：
```cpp
void PrintContacts(contacts2::Contacts& contacts)
{
    for (int i = 0; i < contacts.contacts_size(); ++ i)
    {
        cout << "-----------联系人" << i + 1 << "---------" << "\n";
        const contacts2::PeopleInfo people = contacts.contacts(i);
        cout << "姓名: " << people.name() << "\n";
        cout << "年龄: " << people.age() << "\n";
        for (int j = 0; j < people.phone_number_size(); ++ j )
        {
            const contacts2::PeopleInfo_Phone number = people.phone_number(j);
            cout << "联系人电话" << j + 1 << ": " << number.number() 
                    << "   (" << contacts2::PhoneType_Name(number.type())
                    << ")" << "\n";
        }
    }
}
```
enum类有一个方法：
枚举类型名_Name()，传入常量值将返回该常量值对应的枚举常量
常量值通过：枚举变量.type()获取

运行结果：
![image.png](https://s2.loli.net/2023/10/17/wAPMnse1TKymNV6.png)
### 通讯录2.2
在contacts.proto中，为联系人添加地址信息，修改write.cc和read.cc

```cpp
syntax = "proto3";
package contacts2;

import "google/protobuf/any.proto";

enum PhoneType
{
    MP = 0;  // 移动电话
    TEL = 1; // 固定电话
}

message Address
{
    string home_address = 1; // 家庭地址
    string unit_address = 2; // 单位地址
}

message PeopleInfo_Phone
{
    string number = 1;    
    PhoneType type = 2;
}

message PeopleInfo
{
    string name = 1;
    int32 age = 2;   
    repeated PeopleInfo_Phone phone_number = 3;
    google.protobuf.Any data = 4; // 地址
}

message Contacts
{
    repeated PeopleInfo contacts = 1;
}
```
contacts.proto实现Address类，定义两个字段家庭地址和单位地址
同时PeopleInfo添加Any字段，使用Any需要使用头文件`import "google/protobuf/ang/proto"`
同时需要使用命名空间google.protobuf.Any

```cpp
void AddPeopleInfo(contacts2::PeopleInfo* people)
{
    cout << "-------------新增联系⼈-------------" << endl;
    cout << "输入联系人姓名: ";
    string name; cin >> name;
    people->set_name(name);

    cout << "输入联系人年龄: ";
    int age; cin >> age;
    people->set_age(age);
    cin.ignore(256, '\n');

    for (int i = 0; ; ++ i)
    {
        cout << "输入联系人电话，次数: " << i + 1 << "(只输入回车完成新增): ";
        string number; getline(cin, number);
        if (number.empty()) break;
        contacts2::PeopleInfo_Phone* phone = people->add_phone_number();
        phone->set_number(number);

        cout << "输入电话类型(1、移动 2、固定): ";
        int type; cin >> type;
        cin.ignore(256, '\n');
        if (type == 1)
            phone->set_type(contacts2::PhoneType::MP);
        else if (type == 2)
            phone->set_type(contacts2::PhoneType::TEL);
        else 
            cout << "输入有误!\n" << "\n";
    }
    
    contacts2::Address address;
    cout << "输入联系人家庭住址: ";
    string home_address; getline(cin, home_address);
    address.set_home_address(home_address);
    cout << "输入联系人单位地址: ";
    string unit_address; getline(cin, unit_address);
    address.set_unit_address(unit_address);
    people->mutable_data()->PackFrom(address);
    cout << "-----------添加联系⼈成功-----------" << endl;
}
```
对象.mutable_字段名()，在该对象中开辟一块空间给该字段使用，返回这段空间的起始地址
Any有一个函数：Any对象.PackFrom()，将需要保存到Any中的对象作为参数，调用这个函数，该对象就会被保存到Any对象中

```cpp
void PrintContacts(contacts2::Contacts& contacts)
{
    for (int i = 0; i < contacts.contacts_size(); ++ i)
    {
        cout << "-----------联系人" << i + 1 << "---------" << "\n";
        const contacts2::PeopleInfo people = contacts.contacts(i);
        cout << "姓名: " << people.name() << "\n";
        cout << "年龄: " << people.age() << "\n";
        for (int j = 0; j < people.phone_number_size(); ++ j )
        {
            const contacts2::PeopleInfo_Phone number = people.phone_number(j);
            cout << "联系人电话" << j + 1 << ": " << number.number() 
                    << "   (" << contacts2::PhoneType_Name(number.type())
                    << ")" << "\n";
        }
        if (people.has_data() && people.data().Is<contacts2::Address>())
        {
            contacts2::Address address;
            people.data().UnpackTo(&address);
            if (address.home_address().size())
                cout << "联系人家庭住址: " << address.home_address() << "\n";
            if (address.unit_address().size())
                cout << "联系人单位住址: " << address.unit_address() << "\n";
        }
    }
}
```
has_'Any对象名'()：返回该Any对象是否被设置
Any对象.Is<\T>()：判断Any对象是否为T类型
Any对象.UnpackTo()：将Any对象提取为**原类型对象**，将原类型对象的地址作为参数传入

运行结果：
![image.png](https://s2.loli.net/2023/10/17/UwMTkOxnegzuv7Z.png)

![image.png](https://s2.loli.net/2023/10/17/nFPGNeiH5arE9Uq.png)

### 通讯录2.3
新增其他联系方式：qq或者wechat，只设置其中一种联系方式
```cpp
message PeopleInfo
{
    string name = 1;
    int32 age = 2;   
    repeated PeopleInfo_Phone phone_number = 3;
    google.protobuf.Any data = 4; // 地址
    oneof other_contact
    {
        string qq = 5;
        string wechat = 6;
    }
}
```

write.cc：
```cpp
void AddPeopleInfo(contacts2::PeopleInfo* people)
{
    cout << "-------------新增联系⼈-------------" << endl;
    cout << "输入联系人姓名: ";
    string name; cin >> name;
    people->set_name(name);

    cout << "输入联系人年龄: ";
    int age; cin >> age;
    people->set_age(age);
    cin.ignore(256, '\n');

    for (int i = 0; ; ++ i)
    {
        cout << "输入联系人电话，次数: " << i + 1 << "(只输入回车完成新增): ";
        string number; getline(cin, number);
        if (number.empty()) break;
        contacts2::PeopleInfo_Phone* phone = people->add_phone_number();
        phone->set_number(number);

        cout << "输入电话类型(1、移动 2、固定): ";
        int type; cin >> type;
        cin.ignore(256, '\n');
        if (type == 1)
            phone->set_type(contacts2::PhoneType::MP);
        else if (type == 2)
            phone->set_type(contacts2::PhoneType::TEL);
        else 
            cout << "输入有误!\n" << "\n";
    }
    
    contacts2::Address address;
    cout << "输入联系人家庭住址: ";
    string home_address; getline(cin, home_address);
    address.set_home_address(home_address);
    cout << "输入联系人单位地址: ";
    string unit_address; getline(cin, unit_address);
    address.set_unit_address(unit_address);
    people->mutable_data()->PackFrom(address);

    cout << "选择要添加的其他联系方式(1、qq  2、wechat): ";
    int other_contact; cin >> other_contact;
    cin.ignore(256, '\n');
    if (other_contact == 1)
    {
        cout << "输入联系人qq号码: ";
        string qq; cin >> qq;
        people->set_qq(qq);
    }
    else if (other_contact == 2)
    {
        cout << "输入联系人wechat号码: ";
        string wechat; cin >> wechat;
        people->set_wechat(wechat);
    }
    else 
        cout << "选择有误!\n";
    
    cout << "-----------添加联系⼈成功-----------" << endl;
}
```
oneof变量的设置：set_字段名()即可

read.cc的修改：
```cpp
void PrintContacts(contacts2::Contacts& contacts)
{
    for (int i = 0; i < contacts.contacts_size(); ++ i)
    {
        cout << "-----------联系人" << i + 1 << "---------" << "\n";
        const contacts2::PeopleInfo people = contacts.contacts(i);
        cout << "姓名: " << people.name() << "\n";
        cout << "年龄: " << people.age() << "\n";
        for (int j = 0; j < people.phone_number_size(); ++ j )
        {
            const contacts2::PeopleInfo_Phone number = people.phone_number(j);
            cout << "联系人电话" << j + 1 << ": " << number.number() 
                    << "   (" << contacts2::PhoneType_Name(number.type())
                    << ")" << "\n";
        }
        if (people.has_data() && people.data().Is<contacts2::Address>())
        {
            contacts2::Address address;
            people.data().UnpackTo(&address);
            if (address.home_address().size())
                cout << "联系人家庭住址: " << address.home_address() << "\n";
            if (address.unit_address().size())
                cout << "联系人单位住址: " << address.unit_address() << "\n";
        }

        switch(people.other_contact_case())
        {
            case contacts2::PeopleInfo::OtherContactCase::kQq:
                cout << "联系人qq: " << people.qq() << "\n";
                break;
            case contacts2::PeopleInfo::OtherContactCase::kWechat:
                cout << "联系人wechat: " << people.wechat() << "\n";
                break;
            default:
                break;
        }
    }
}
```
定义oneof的message中有一个方法：oneof名_case()，该函数将返回一个常量值，用来表示哪个oneof字段被设置了
将返回的常量值与oneof生成的enum常量进行判断，得到哪个字段被设置
如何获取oneof生成的enum？该enum的名字为oneof名_case，将首字母与下划线后的字母大写，并删除下划线即可
入：名为other_contact_case的oneof生成的enum名为：OtherContactCase
通过::返回enum下的常量即可

获取oneof字段：对象.oneof字段名()

运行结果：
![image.png](https://s2.loli.net/2023/10/17/GEAFijvR8XfOect.png)

### 通讯录2.4
使用map类型，为联系人信息添加备注字段
.proto文件的修改
```cpp
 message PeopleInfo
{
    string name = 1;
    int32 age = 2;   
    repeated PeopleInfo_Phone phone_number = 3;
    google.protobuf.Any data = 4; // 地址
    oneof other_contact
    {
        string qq = 5;
        string wechat = 6;
    }
    map<string, string> remark = 7; // 备注信息
}
```

write.cc文件的修改
```cpp
void AddPeopleInfo(contacts2::PeopleInfo* people)
{
    cout << "-------------新增联系⼈-------------" << endl;
    cout << "输入联系人姓名: ";
    string name; cin >> name;
    people->set_name(name);

    cout << "输入联系人年龄: ";
    int age; cin >> age;
    people->set_age(age);
    cin.ignore(256, '\n');

    for (int i = 0; ; ++ i)
    {
        cout << "输入联系人电话，次数: " << i + 1 << "(只输入回车完成新增): ";
        string number; getline(cin, number);
        if (number.empty()) break;
        contacts2::PeopleInfo_Phone* phone = people->add_phone_number();
        phone->set_number(number);

        cout << "输入电话类型(1、移动 2、固定): ";
        int type; cin >> type;
        cin.ignore(256, '\n');
        if (type == 1)
            phone->set_type(contacts2::PhoneType::MP);
        else if (type == 2)
            phone->set_type(contacts2::PhoneType::TEL);
        else 
            cout << "输入有误!\n" << "\n";
    }
    
    contacts2::Address address;
    cout << "输入联系人家庭住址: ";
    string home_address; getline(cin, home_address);
    address.set_home_address(home_address);
    cout << "输入联系人单位地址: ";
    string unit_address; getline(cin, unit_address);
    address.set_unit_address(unit_address);
    people->mutable_data()->PackFrom(address);

    cout << "选择要添加的其他联系方式(1、qq  2、wechat): ";
    int other_contact; cin >> other_contact;
    cin.ignore(256, '\n');
    if (other_contact == 1)
    {
        cout << "输入联系人qq号码: ";
        string qq; cin >> qq;
        people->set_qq(qq);
    }
    else if (other_contact == 2)
    {
        cout << "输入联系人wechat号码: ";
        string wechat; cin >> wechat;
        people->set_wechat(wechat);
    }
    else 
        cout << "选择有误!\n";
    
    cin.ignore(256, '\n');
    for (int i = 0; ; ++ i)
    {
        cout << "输入备注" << i + 1 << "标题(只输入回车完成备注新增): ";
        string remark_key; getline(cin, remark_key);
        if (remark_key.empty()) break;
        cout << "输入备注" << i + 1 << "内容: ";
        string remark_value; getline(cin, remark_value);
        people->mutable_remark()->insert({remark_key, remark_value});
    }
    
    cout << "-----------添加联系⼈成功-----------" << endl;
}
```
mutable_'map类型的字段名'()：返回新增键值对的地址
map类型的对象拥有一个insert方法，和c++一样使用即可

read.cc文件的修改：
```cpp
void PrintContacts(contacts2::Contacts& contacts)
{
    for (int i = 0; i < contacts.contacts_size(); ++ i)
    {
        cout << "-----------联系人" << i + 1 << "---------" << "\n";
        const contacts2::PeopleInfo people = contacts.contacts(i);
        cout << "姓名: " << people.name() << "\n";
        cout << "年龄: " << people.age() << "\n";
        for (int j = 0; j < people.phone_number_size(); ++ j )
        {
            const contacts2::PeopleInfo_Phone number = people.phone_number(j);
            cout << "联系人电话" << j + 1 << ": " << number.number() 
                    << "   (" << contacts2::PhoneType_Name(number.type())
                    << ")" << "\n";
        }
        if (people.has_data() && people.data().Is<contacts2::Address>())
        {
            contacts2::Address address;
            people.data().UnpackTo(&address);
            if (address.home_address().size())
                cout << "联系人家庭住址: " << address.home_address() << "\n";
            if (address.unit_address().size())
                cout << "联系人单位住址: " << address.unit_address() << "\n";
        }

        switch(people.other_contact_case())
        {
            case contacts2::PeopleInfo::OtherContactCase::kQq:
                cout << "联系人qq: " << people.qq() << "\n";
                break;
            case contacts2::PeopleInfo::OtherContactCase::kWechat:
                cout << "联系人wechat: " << people.wechat() << "\n";
                break;
            default:
                break;
        }

        if (people.remark_size())
        {
            cout << "备注信息: \n";
            for (auto it = people.remark().begin(); it != people.remark().end(); ++ it)
                cout << "    " << it->first << ": " << it->second << "\n";
        }
    }
}
```
似乎这map不能用范围for...
'map类型的字段名'\_size()：返回map含有的键值对数量
用迭代器遍历该map即可

运行结果：
![image.png](https://s2.loli.net/2023/10/17/whefYxtLd9pN4FH.png)
### 通讯录3.0 网络版
支持操作
- 客户端支持操作
  - 新增联系人
  - 删除联系人
  - 查询通讯录列表
  - 查询联系人详细信息
- 服务端支持增删查能力，并能持久化数据库

**协议约定**：
```cpp
syntax = "proto3";
package add_contact;

enum PhoneType
{
    MP = 0;  // 移动电话
    TEL = 1; // 固定电话
}

message Phone
{
    string number = 1;    
    PhoneType type = 2;
}

message AddContactRequuest
{
    string name = 1;
    int32 age = 2;   
    repeated Phone phone_number = 3;
}

message AddcontactResponse
{
    bool success = 1;        // 服务调用是否成功
    string error_desc = 2;   // 错误信息描述
    string uid = 3;
}
```

## 数据类型

|类型|解释|对应C++类型|
|-------|------|---------|
|double||double|
|float||float|
|int32|使用变长编码，负数的编码效率低，对于负数推荐使用sint32|int32|
|int64|使用变长编码，负数的编码效率低，对于负数推荐使用sint64|int64|
|uint32|使用变长编码|uint32|
|uint64|使用变长编码|uint64|
|sint32|使用变长编码，符号整型，负数的编码效率高|int32|
|sint64|使用变长编码，符号整型，负数的编码效率高|int64|
|fix32|定长4字节，若值比$2^{28}$大，则比uint32更高效|uint32|
|fix64|定长8字节，若值比$2^{56}$大，则比uint64更高效|uint64|
|sfix32|定长4字节|int32|
|sfix64|定长8字节|int64|
|bool||bool|
|string|包含utf8和ASCII编码的字符串，长度不超过$2^{32}$|string|
|bytes|可以包含任意的字节序列但长度不超过$2^{32}$|string|


使用变长编码，可能从原字节数转换成其他字节数可大可小
### enum
enum的命名推荐使用大驼峰命名，所定义的常量推荐使用全大写，可以用下划线分割
**0值常量**必须存在，其他常量值必须在32位整数范围内，不能设置负值
enum可以定义在message中

同级枚举类型下，常量名不能相同，即使枚举名不相同，以下程序是错误的：
```
enum P1
{
	MP = 0;
}
enum P2
{
	MP = 0;
}
```
当.proto文件依赖了其他.proto，也不能出现常量名相同的情况（除非定义了不同的命名空间）
枚举类型的**默认值**为0，PB在反序列化时，发现枚举类型没有设置值，会默认设置其常量值为0
### Any
Any可以存储任意类型的数据，是一个泛型
通过PackFrom()将原对象保存到Any对象中
通过UnpackTo()恢复原对象
### oneof
![image.png](https://s2.loli.net/2023/10/17/iuRIUEOZLcSbwQ9.png)

oneof作为类型，一定被定义在message中
oneof里，定义的字段编号不能与message的其他字段编号冲突
不能使用repeated修饰oneof中的任何类型，oneof中的类型应该是相同的
若是进行了多次设置，只保留最后一次设置的成员
被编译过后，oneof会转换成enum，并且多出一个常量值为0的常量表示没有设置oneof
### map
map的key可以是除了float和byte之外的任意标量类型
map不能用repeated修饰
map中的记录顺序为无序

### 默认值
反序列化消息时，如果被反序列化的二进制序列中不包含某个字段，反序列化对象中相应字段时，就会用默认值填充该字段。不同的类型对应的默认值不同：
- 字符串，默认值为空字符串
- 字节，默认值为空字节
- 布尔值，默认值为 false
- 数值类型，默认值为 0.0
- 枚举，默认值是第⼀个定义的枚举值
- 消息字段，默认值依赖于语言
- 对于设置了repeated 的字段的默认值是空的（ 通常是相应语言的一个空列表 ）
- 消息字段 、oneof字段和any字段 ，C++ 和 Java 语言中都有has_方法来检测当前字段
是否被设置

**在proto3语法中，标准数据类型没有has_字段名()用来检测数据是否被设置**

### 更新消息
新增字段时，只需要注意新字段名不要和老字段名冲突，字段编号不冲突即可

新增消息时需要遵循如下规则：
- 禁止修改任何已有字段的字段编号
- 若是移除老字段，正确的做法是保留字段编号（reserved），以确保该编号将不能被重复使用。不要直接删除或注释掉字段
![image.png](https://s2.loli.net/2023/10/17/FXk1nv23C5czG64.png)
保留一个，多个，某个范围内的编号，保留字段名
若使用保留项，将导致编译出错
- int32， uint32， int64， uint64 和 bool 是完全兼容的。可以两两转换，但是可能发生截断
- sint32 和 sint64 相互兼容但不与其他的整型兼容
- string 和 bytes 在合法 UTF-8 字节前提下也是兼容的
- bytes 包含消息编码版本的情况下，嵌套消息与 bytes 也是兼容的
- fixed32 与 sfixed32 兼容， fixed64 与 sfixed64兼容
- enum 与 int32，uint32， int64 和 uint64 兼容（注意：若值不匹配会被截断）。但要注意当反序列化消息时会根据语言采用不同的处理方案：例如，未识别的 proto3 枚举类型会被保存在消息中，但是当消息反序列化时如何表示是依赖于编程语言的。整型字段总是会保持其的值。
- oneof：
  - 将⼀个单独的值更改为新oneof类型成员之一是安全和二进制兼容的
  - 若确定没有代码⼀次性设置多个值那么将多个字段移⼊⼀个新 oneof 类型也是可行的
  - 将任何字段移入已存在的 oneof 类型是不安全的

### 未知字段
若服务端的.prote文件更新了新的字段，但客户端的.proto文件未更新该字段，那么客户端在反序列化数据时，会将服务端更新的字段放入未知字段中
未知字段：保存无法**识别**的数据
在3.5以及更高的版本中，PB重新引入了未知字段，未知字段会保留在序列化的结果中
```cpp
void PrintContacts(contacts2::PeopleInfo& people)
{

    cout << "-----------联系人" << "---------" << "\n";
    cout << "姓名: " << people.name() << "\n";
    cout << "年龄: " << people.age() << "\n";
    const Reflection* reflection = contacts2::PeopleInfo::GetReflection();
    const UnknownFieldSet& s = reflection->GetUnknownFields(people);
    for (int j = 0; j < s.field_count(); ++ j)
    {
        const UnknownField& unknown_field = s.field(j);
        cout << "未知字段" << j + 1 << ": "
            << "  编号: " << unknown_field.number();
        switch(unknown_field.type()) 
        {
            case UnknownField::Type::TYPE_VARINT:
                cout << "  值: " << unknown_field.varint() << "\n";
                break;
            case UnknownField::Type::TYPE_LENGTH_DELIMITED:
                cout << "  值: " << unknown_field.length_delimited() << "\n";
                break;
            case UnknownField::Type::TYPE_FIXED32:
                cout << "  值: " << unknown_field.fixed32() << "\n";
                break;
            case UnknownField::Type::TYPE_FIXED64:
                cout << "  值: " << unknown_field.fixed64() << "\n";
                break;
        }

    }
}
```

server保存了生日字段
![image.png](https://s2.loli.net/2023/10/17/AnkxCrQbN74FmVI.png)

client无法识别server发送的生日字段，该字段将被放入未知字段中
![image.png](https://s2.loli.net/2023/10/17/pRLJbjso1v7FPgN.png)

### 前后兼容性
老模块能够识别新模块的序列化数据（向前兼容）
新模块能够识别后模块的序列化数据（向后兼容）
无法识别的字段将被存储在未知字段中
当一个庞大的分布式系统中，存在无法更新所有设备的情况，若协议没有前后兼容性，那么整个业务将被迫停止

### 选项option
```
option optimize_for = LITE_RUNTIME;
```
optimize_for：该选项可以设置编译后的.proto文件的优化级别，分别为 SPEED 、
CODE_SIZE 、 LITE_RUNTIME 

- SPEED：PB的默认选项，生成的代码运行效率极高，但代码占用空间多
- CODE_SIZE： proto 编译器生成最少的类，占用空间少，依赖基于反射的代码来
实现序列化、反序列化和各种其他操作。但运行效率极低。这种⽅式适合用于含有大量的.proto文件，但对速度没有要求的应用中
- LITE_RUNTIME：生成代码的效率高，同时占用空间少，但牺牲了PB提供的反射功能，可以理解为阉割版本，并且不链接protobuf库，而链接libprotobuf-lite库。通常用于资源有限的平台

- allow_alias ： 允许将相同的常量值分配给不同的枚举常量，用来定义别名。该选项为枚举选项
比如：
```cpp
enum PhoneType {
	option allow_alias = true;
	MP = 0;
	TEL = 1;
	LANDLINE = 1; // 若不加 option allow_alias = true; 这⼀⾏会编译报错
}
```
