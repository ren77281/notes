```toc
```
javac可以指定编码格式，默认以GBK格式编码
```
javac -encoding utf-8 xxx.java
```
标识符：数字、字母、下划线与美元符号$可组成标识符，但是不能以数字开头，并且严格区分大小写

类名：大驼峰
方法名与变量名：小驼峰

基本数据类型
- 整数：byte、short、int、long
- 小数：float、double
- 布尔：boolean
- 字符：char
引用数据类型：数组、字符串、String、接口、类、枚举

局部变量不初始化无法通过编译
初始化时，若字面值超过类型的数据范围同样无法通过编译
java中的实数字面值默认是double类型，11.5为double，11.5f为float
java使用Unicode表示字符，一个字符占用两个字节，可以表示中文
boolean只能有两个值，true和false，不能用数字表示。若int a = 0，不能写`if(a)`
boolean没有明确指定大小

java的隐式类型转换只能大范围转小范围，否则将报错

1. 如果一个操作数是 `float` 类型，而另一个操作数是整数类型（`byte`、`short`、`int` 或 `long`），则整个表达式中的所有操作数都会被提升为 `float` 类型。
2. 如果一个操作数是 `float` 类型，而另一个操作数是 `double` 类型，那么整个表达式中的所有操作数都会被提升为 `double` 类型。
整型提升时，所有小于4字节的数据都会被提升为4字节（如byte和short）

| 数据类型          | 取值范围                                   |
| ---------------- | ---------------------------------------- |
| byte             | -128 到 127                               |
| short            | -32,768 到 32,767                         |
| int              | -2^31 到 2^31 - 1                         |
| long             | -2^63 到 2^63 - 1                         |
| float            | IEEE 754 单精度浮点，约 -3.40282347E+38 到 3.40282347E+38 |
| double           | IEEE 754 双精度浮点，约 -1.7976931348623157E+308 到 1.7976931348623157E+308 |
| char             | 0 到 65535                                |
| boolean          | true 或 false                             |

java的int一定是4个字节，与平台无关（和C/C++不同）
java整数类型的第一位都是符号位
字符串拼接：从左往后运算，只要之前进行了字符串拼接，之后的数据都会被转换成字符串
```java
int a = 10;
int b = 20;
System.out.println("a = " + a + " b = " + b);
System.out.println("a + b = " + a + b);
```
程序将输出：
```
a = 10 b = 20
a + b = 1020
```

```java
System.out.println("a + b = " + (a + b));
```
此时程序输出:`a + b = 30`

```java
System.out.println(a + b " = a + b");
```
此时程序输出:`30 = a + b`

整数和字符串之间转换两种方法：
```java
int a = 10;  
String s1 = a + "";  
System.out.println(s1);  
String s2 = String.valueOf(a);
System.out.println(s2);
```

```java
String s = "123";  
int a = Integer.parseInt(s);  
int b = Integer.valueOf(s);
```

java除0会报异常
增量运算符不用进行强制类型转换，如
```java
short s = 0;
int a = 10;
s = (short)a + s;
s += a;
```
取模运算：
```
10 % 3 = 1
10 % -3 = 1
-10 % 3 = -1
-10 % -3 = -1
```

关系运算符的结果为true或false
逻辑反`!`只能用于布尔表达式，若a为int，不能写`!a`

`|`和`&`的左右两边表达式结果为布尔类型时，表示逻辑运算，同时不支持短路运算
不是布尔表达式，表示位运算

右移`>>`补符号位，java存在无符号右移`>>>`补0
switch的括号内不能是long类型，可以是byte、char、short、int、String、枚举

快捷键：
- psvm(main)：生成main函数
- sout：生成打印语句

输入输出
```java
System.out.println(msg); // 输出一个字符串, 带换行
System.out.print(msg); // 输出一个字符串, 不带换行
System.out.printf(format, msg); // 格式化输出，和c语言一样
```

```java
import java.util.Scanner; // 需要导入 util 包
Scanner sc = new Scanner(System.in);
System.out.println("请输入你的姓名：");
String name = sc.nextLine();
System.out.println("请输入你的年龄：");
int age = sc.nextInt();
System.out.println("请输入你的工资：");
float salary = sc.nextFloat();
System.out.println("你的信息如下：");
System.out.println("姓名: "+name+"\n"+"年龄："+age+"\n"+"工资："+salary);
sc.close();
```

JVM的内存布局
.java程序由javac编译为字节码文件，由jvm运行该文件
jvm为该程序分配了空间，和C语言的进程地址空间类似，包括方法区、堆虚拟机栈、本地方法栈、程序计数器。其中的方法区和堆是被所有线程共享的空间，其他则是线程隔离的数据区
虚拟机栈等价于栈，用来存储局部变量，而Java本地方法栈运行一些由C/C++代码编写的程序
只要是Java中的对象，都使用堆上的空间，数组就是一个对象


java数组
```java
int[] a1 = { 1, 2, 3 };
int[] a2 = new int[]{ 1, 2, 3 };
int[] a3 = new int[3];
```
a1、a2、a3的数据类型为int\[\]，表示int数组，括号中不能写数字
a1.length将输出数组的长度，编译器会自动推导数组的长度。编译器会检查数组越界
java的数组都使用堆上的空间，a1和a2本质是相同的，只是省略了`new int[]`

将数组转换成字符串进行输出
```java
int[] a = { 11, 22, 33 };
String ret = Arrays.toString(a);
```
输出带有中括号，这也是一种打印数组的方法
![image.png](https://s2.loli.net/2023/11/15/8hNkCQpaWBU1vsY.png)
直接输出数组名，将得到一串数字，`@`后面的数字为经过哈希的数组首元素地址，`@`之前的字符为数组类型
```java
int[] a = { 11, 22, 33 };  
System.out.println(a);
```
![image.png](https://s2.loli.net/2023/11/15/HbJUC12Xmy53lwY.png)

所以我们叫数组为引用变量，即存储了地址的变量被称为引用
引用类型的初值为null，表示不指向任何对象，此时打印数组的长度将触发空指针异常
```java
int[] a = null;  
System.out.println(a);  
System.out.println(a.length);
```
![image.png](https://s2.loli.net/2023/11/15/R3NYt7aD84GSe6d.png)
```java
public class TestDemo {  
    public static void fun1(int[] array) {  
        array = new int[]{ 11, 22, 33 };  
    }  
    public static void fun2(int[] array) {  
        array[0] = 111;  
    }  
    public static void main(String[] args) {  
        int[] a = { 1, 2, 3};  
        fun1(a);  
        System.out.println(Arrays.toString(a));  
        fun2(a);  
        System.out.println(Arrays.toString(a));  
    }  
}
```
fun2修改了main函数中的数组的值，fun1没有修改
形参array接收main函数中数组的地址，fun2通过地址修改了数组中的值
而fun1直接修改array保存的地址，使之指向一个新的数组，这不会影响main函数中的数组，因为fun1没有通过该数组的地址进行修改，只是修改了保存数组地址的变量
![image.png](https://s2.loli.net/2023/11/15/4jQylg5p72PhZnV.png)

数组的深拷贝
```java
int[] a = { 1, 2, 3 };  
int[] copy = Arrays.copyOf(a, a.length);  
System.out.println(Arrays.toString(a));
```
copyOf的两个参数：第一个为原数组，第二个为拷贝长度，从第一个元素开始要拷贝几个元素。若大于原数组长度，则拷贝0

```java
public static native void arraycopy(Object src,  int  srcPos,  
                                    Object dest, int destPos,  
                                    int length);
```
src、srcPos，原数组，从原数组的哪个位置开始拷贝
dest、destPos，目标数组，从目标数组的哪个位置开始存储
length，拷贝的长度

native修饰：本地方法，由C/C++编写，无法查看源代码，由JVM实现

调用数组的copy方法也可以
```java
int[] copy = array.copy();
```
经常使用Arrays.copyOf()或者Arrays.copyOfRange()

Arrays.binarySearch()：二分查找

二维数组定义方式
```java
int[][] a1 = { {1,2,3}, {4, 5, 6} };  
int[][] a2 = new int[][]{ {1,2,3}, {4, 5, 6} };  
int[][] a3 = new int[2][3];
```

可以使用下标的方式进行遍历，也可以使用deepToString的方式进行打印
```java
System.out.println(Arrays.deepToString(a1));
```

```java
int[][] a = new int[2][3];  
for (int i = 0; i < a.length; ++ i) {  
    for (int j = 0; j < a[i].length; ++ j) {  
        System.out.print(a[i][j]);  
    }  
    System.out.println();  
}
```
![image.png](https://s2.loli.net/2023/11/15/6zwo7dHJY5FPnAD.png)
将`int[][] a = new int[2][3]`中的3去掉，程序将报空指针异常
Java的数组类似于C++的`vector<vector<int>>`，数组中的成员是`vector<int>`没有为其分配空间怎么能直接访问它？C++中，该成员可能是一个野指针，Java中该成员的值为null，所以引发了空指针异常

## 类和对象
Java作为一种纯面向对象的语言（Object Oriented Program，继承OOP），认为一切皆对象
这是一种解决问题的思想，依靠对象之间的交互解决问题。这样做更符合人们对事物的认知

而面向对象的关键在于：抽象对象。将对象抽象出来并完成封装（实现类），进而使用对象

修改类名时，若文件中只有一个类，那么用IDEA的重构功能，修改文件名（类名），此时所有的类名都会被修改（若文件中有多个类，那么IDEA只会修改文件名）


Java通常使用new实例化类，创建一个对象
通过`.`访问对象的属性。若没有为对象的属性赋初值时，此时所有的成员变量都是自己的零值（引用为null，boolean为false，char为空格`\u0000`）

```java
public class Person {
    private String name;
    private int age;
    private String address;

    // 构造方法、其他方法等...

    // 设置 name、age 和 address 的方法
    public void setPersonDetails(String name, int age, String address) {
        // 方法参数与类字段同名，但它们位于不同的作用域
        this.name = name;
        this.age = age;
        this.address = address;
    }
}
```
形参name、age、address作为局部变量与对象属性的命名冲突了，在方法setPersonDetails中，无法访问对象的属性，因为它们被形参“覆盖”了，只能使用`this.`访问对象的属性，同理C++也能使用`this->`访问对象的属性

Java中`this`表示当前对象的引用，是所有方法的第一个隐藏形参，而C++的`this`为指向该对象的指针，也是所有方法的第一个隐藏形参，本质还是一样的
Java和C++相同，编译器编译代码后，会在相应的地方加上this

建议：只要访问自己的属性或者方法时，使用`this`（好习惯）

```java
public class MyClass {
    private int x;
    private int y;

    // 默认构造函数
    public MyClass() {
        // 调用带参数的构造函数，参数使用默认值
        this(0, 0);
        System.out.println("Default Constructor");
    }

    // 带参数的构造函数
    public MyClass(int x, int y) {
        this.x = x;
        this.y = y;
        System.out.println("Parameterized Constructor with x = " + x + " and y = " + y);
    }

    public static void main(String[] args) {
        MyClass obj1 = new MyClass();          // 调用默认构造函数
        MyClass obj2 = new MyClass(42, 24);    // 调用带参数的构造函数
    }
}
```
Java可以在构造方法中的**第一行**使用this引用调用其他构造方法（重载）
```cpp
#include <iostream>

class MyClass {
public:
    // 带参数的构造函数
    MyClass(int x, int y) {
        initialize(x, y);
        std::cout << "Parameterized Constructor with x = " << x << " and y = " << y << std::endl;
    }

    // 默认构造函数，通过委托给带参数的构造函数实现
    MyClass() : MyClass(0, 0) {
        std::cout << "Default Constructor" << std::endl;
    }

private:
    void initialize(int x, int y) {
        // 在成员函数中进行初始化工作
        // ...
        std::cout << "Initialization done." << std::endl;
    }
};

int main() {
    MyClass obj1;          // 调用默认构造函数
    MyClass obj2(42, 24);  // 调用带参数的构造函数

    return 0;
}
```
C++中，可以在初始化列表中直接调用构造函数（不通过this指针），实现类似的效果

但是这样的调用关系必须满足拓扑序，即不存在循环调用
同样，Java在定义类时，可以直接为类的成员赋予一个初值

使用`println`打印一个对象，将输出`类名@哈希值`或者`null`，如果类定义了toString方法，那么`println`将调用该方法打印对象的信息
在IDEA中，可以自动生成`toString`方法

使用访问限定符修饰类的成员：

| 访问限定符    | 同一类 | 同一包 | 子类 | 任意位置 |
|---------------|--------|--------|------|----------|
| public        | √      | √      | √    | √        |
| protected     | √      | √      | √    | ×        |
| default (无修饰符) | √ | √   | ×    | ×        |
| private       | √      | ×      | ×    | ×        |

默认权限：包访问权限（不需要显式地写出`default`）
什么是包访问权限？
包是一种组织和管理类的机制，可以理解为目录，目录管理着很多类
同一工程中允许存在相同名称的类，只要它们位于不同的包中

使用`import`关键字指定包中的类并将该类导入当前文件中，对比`include`关键字
如`impore java.util.Arrays;`表示将java.util文件夹（包）下的Arrays类导入
`impore java.util.*;`表示将java.util包下的所有类导入
当使用到其中的类时，才会去加载类，不会直接加载所有的类（C++则直接加载/编译所有的类）

当使用的类需要导包时，鼠标放到类名上，`Alt+回车`，IDEA将弹出需要导入的包，选择你需要的即可
还能在不导包的前提下通过包名直接使用类，如`java.util.Date`表示java.util包下的Date类。这又有点像C++的命名空间

创建一个自己的包：包的命名一般使用逆域名，如`com.baidu.www`
![image.png](https://s2.loli.net/2023/11/16/rVCneJq5aIYcEjh.png)

此时src文件夹下出现一个路径
![image.png](https://s2.loli.net/2023/11/16/KTox5tidBP8GMA4.png)

去掉下图的选项后，就能在IDEA看到目录结构
![image.png](https://s2.loli.net/2023/11/16/P2KrNS4WlXTGJqk.png)

![image.png](https://s2.loli.net/2023/11/16/5ZW2yhmoF8ersla.png)
在com、xxx、demo这三个文件夹下都能创建class
在demo文件夹下创建一个类后，第一行会出现`package com.xxx.demo;`的声明，声明这个类所属于的包
如何在默认包的类中使用自定义包的类？不推荐使用默认包，推荐将所有类都定义在自定义包下，只要import自定义包即可使用里面的类

这些访问限定符是修饰类成员的，而类只有两个访问限定符`public`和`default`
- public：其他类可以访问public修饰的类，无论是否在同一个包中。但一个Java文件最多只能有一个public类，并且类的名字必须和文件名相同。其他类则是default，数量不限
- default：只有在同一个包下才能访问default修饰的类

private和protected不能修饰外部类，因为外部类无法被继承和封装。只能修饰内部类

static修饰的成员存储在方法区中，JVM将方法区实现在了堆中，虽然方法区和堆是同级的关系，但是方法区使用了堆的一部分空间

在类中直接创建代码块，其中包含初始化成员变量的代码。调用构造方法之前会先执行实例代码块中的代码（编译程序后，实例代码块中的代码会被插入到所有构造方法的最前面）
```java
{
	// 代码
}
```

需要将Java的实例代码块和C++的初始化列表进行区别，初始化列表是为成员分配空间的地方，如果存在一旦定义就必须赋初值的变量，就必须使用初始化列表（引用、const常量以及无默认构造函数的自定义变量）
而Java的实例代码块则没有这个作用，只是为了统计所有构造方法（重载）的逻辑，使每个构造方法先执行实例代码块中的内容。可以说是一种代码复用吧

在代码块之前加上static，为静态代码块，用例初始化静态成员，实例代码块无法初始化静态成员，必须使用静态代码块初始化静态成员
```java
static{
	// 初始化静态成员
}
```
而静态代码块比实例代码块更早执行。并且只会执行一次，只在JVM加载类时被执行
（如果定义了多个静态代码块，这些代码块将被合并成一个代码块，根据前后顺序执行）

### 内部类
实例内部类的实例化
```java
public class OuterClass {
    // 实例内部类
    public class InnerClass {
        public void display() {
            System.out.println("Inside InnerClass");
        }
    }
}

public class Main {
    public static void main(String[] args) {
        // 实例化外部类
        OuterClass outerObj = new OuterClass();
        // 实例化内部类
        OuterClass.InnerClass innerObj = outerObj.new InnerClass();
        // 调用内部类的方法
        innerObj.display();
    }
}
```

首先，不能直接用内部类的类名作为类型，这点和C++一样，需要使用`外部类.内部类`作为内部类的类型。并且实例化内部类需要通过外部类对象，这一点和C++不同，比C++繁琐：
```cpp
#include <iostream>

class OuterClass {
public:
    class InnerClass {
    public:
        void display() {
            std::cout << "Inside InnerClass" << std::endl;
        }
    };
};

int main() {
    // 实例化外部类
    OuterClass outerObj;
    // 实例化内部类
    OuterClass::InnerClass* innerObj = new OuterClass::InnerClass();
    // 调用内部类的方法
    innerObj->display();
    // 记得释放内存
    delete innerObj;
    return 0;
}
```

实例内部类无法定义静态成员，而C++的内部类可以定义静态成员
```java
public class OuterClass {
    private int outerField;

    public class InnerClass {
        private int innerField;

        // 实例内部类不能包含静态字段或方法
        // public static int staticField; // 错误
        // public static void staticMethod() {} // 错误
        public static final int staticField; // 正确
    }
}
```
但是可以定义一个在编译期间就确定的常量，Java中的常量由`final`修饰，不能改变变量的值
不能在方法中定义常量，只能在类的成员中定义常量，加载类时，常量也会被加载
所以Java中的常量是类级别，而不是方法级别的

和C++一样，内部类天生是外部类的“友元”（虽然Java没有友元）
内部类的成员与外部类成员冲突时，默认访问内部类的成员
```java
public class OuterClass {
    private int outerVariable = 10;

    public class InnerClass {
        private int outerVariable = 20;
        private int innerVariable = 30;

        public void accessVariables() {
            // 访问内部类的变量
            System.out.println("InnerClass's outerVariable: " + this.outerVariable); // 内部类的 outerVariable
            System.out.println("InnerClass's innerVariable: " + this.innerVariable); // 内部类的 innerVariable

            // 访问外部类的变量
            System.out.println("OuterClass's outerVariable: " + OuterClass.this.outerVariable); // 外部类的 outerVariable
        }
    }

    public static void main(String[] args) {
        OuterClass outerObj = new OuterClass();
        OuterClass.InnerClass innerObj = outerObj.new InnerClass();
        innerObj.accessVariables();
    }
}
```
所以实例内部类的方法中，不仅包含了内部类的this引用，还包含了外部类的this引用
外部类要访问内部类的成员，只能new一个内部类对象，通过对象访问

Java，创建外部类时，实例内部类不会自动创建
C++的内部类是独立于外部类的，内部类的创建不依赖于外部类的实例，只是需要通过外部类名访问内部类

为什么叫实例内部类？因为它和外部类的其他实例成员的等级相同，也受到访问限定符的修饰
### 静态内部类
如何实例化？
```java
public class OuterClass {
    private static int outerStaticField = 42;

    public static class StaticInnerClass {
        private int innerField;

        public StaticInnerClass(int innerValue) {
            this.innerField = innerValue;
        }

        public void display() {
            System.out.println("OuterStaticField: " + outerStaticField);
            System.out.println("InnerField: " + innerField);
        }
    }

    public static void main(String[] args) {
        // 直接使用外部类的名称实例化静态内部类
        OuterClass.StaticInnerClass innerObj = new OuterClass.StaticInnerClass(10);

        // 调用静态内部类的方法
        innerObj.display();
    }
}
```
不需要通过外部类的对象实例化静态内部类，此时的实例化方法和C++相同
静态内部类只能访问外部类的静态成员，访问非静态成员需要new一个外部类对象
```java
public class OuterClass {
    private int outerNonStaticField = 42;
    private static int outerStaticField = 100;

    public static class StaticInnerClass {
        private int innerField;

        public StaticInnerClass(int innerValue) {
            this.innerField = innerValue;
        }

        public void display() {
            // 静态内部类可以访问外部类的静态成员
            System.out.println("OuterStaticField: " + outerStaticField);

            // 不能直接访问外部类的非静态成员
            // System.out.println("OuterNonStaticField: " + outerNonStaticField); // 错误

            // 如果需要访问外部类的非静态成员，需要通过外部类的实例来访问
            OuterClass outerInstance = new OuterClass();
            System.out.println("OuterNonStaticField: " + outerInstance.outerNonStaticField);
        }
    }

    public static void main(String[] args) {
        // 创建静态内部类的实例
        StaticInnerClass innerObj = new StaticInnerClass(10);

        // 调用静态内部类的方法
        innerObj.display();
    }
}
```
## 继承
继承的本质是对共性进行抽取，达到代码的复用
```java
// 父类
class Animal {
    String name;

    Animal(String name) {
        this.name = name;
    }

    void eat() {
        System.out.println(name + " is eating.");
    }
}

// 子类继承父类
class Dog extends Animal {
    Dog(String name) {
        // 调用父类的构造方法
        super(name);
    }

    // 子类可以添加自己的方法
    void bark() {
        System.out.println(name + " is barking.");
    }

    // 子类可以重写父类的方法
    @Override
    void eat() {
        System.out.println(name + " is eating bones.");
    }
}

public class InheritanceExample {
    public static void main(String[] args) {
        // 创建子类的实例
        Dog myDog = new Dog("Buddy");

        // 调用继承的方法
        myDog.eat();

        // 调用子类自己的方法
        myDog.bark();
    }
}
```
子类访问父类的同名成员
```java
class ParentClass {
    int field = 10;
}

class ChildClass extends ParentClass {
    int field = 20;

    void display() {
        // 访问父类字段
        System.out.println("ChildClass field: " + this.field);   // 子类的字段
        System.out.println("ParentClass field: " + super.field);  // 父类的字段
    }
}

public class Example {
    public static void main(String[] args) {
        ChildClass obj = new ChildClass();
        obj.display();
    }
}
```
super是this的一部分，其中父类和子类的同名变量被子类隐藏，无法通过this访问，只能通过super访问
`super()`调用父类的构造方法。注意：super只能在非静态方法中使用，因为静态方法中没有this引用，而super是this引用的一部分
个人理解：super并不是父类对象的引用，而是子类对象中父类成员的引用，本质是子类引用的一部分

Java中，子类的方法名和父类的方法名相同，但参数列表不同时，也构成重载。而不是C++中的重定义（隐藏）

当父类的构造方法需要传递参数并且父类没有提供无参的构造方法，那么子类需要在自己的构造方法的**第一行**，显式地调用父类的构造方法
C++只能在初始化列表调用父类的构造函数，而不能在函数体的第一行调用父类的构造函数
`super()`和`this()`不能同时出现在子类的构造方法中，因为它们必须出现在方法的第一行
super并不是方法的隐藏参数，而this是，super只是this的一部分

和C++一样，若没有显式地调用父类的构造方法，那么编译器会自动调用父类的构造方法`super()`

当子类和父类都有静态代码块和实例代码块时，在子类的构造方法中显式地调用父类的构造方法，此时程序的执行顺序是怎样的？
首先，第一次调用子类的构造方法时，静态代码块一定先于实例代码块执行
用于构造子类对象需要先实例化父类对象，所以父类的静态代码块先调用，子类的静态代码块后调用
用于子类的构造方法中，第一行就调用了父类的构造方法，所以程序执行父类的实例代码块以及父类的构造方法。再执行子类的实例代码块与子类的构造方法
**子类的实例代码块后于父类的构造方法执行**

protected权限，不同包下的子类也能访问父类的protected成员，前提是父类由public修饰
Java不支持多继承，即一个类继承多个类，从而避免了C++复杂的菱形继承问题

用`final`修饰的类（密封类）不能被继承，修饰方法，表示不能被重写

### 多态
同一个方法，因为调用该方法的引用，所引用的对象不同，导致了结果不同
向上转型是多态的前提，C++中，向上转型有一个更形象的叫法：切片
用父类引用引用子类对象叫做向上转型，三种情况：1. 直接引用 2. 作为参数 3. 作为函数返回值（返回值类型为父类，但是返回子类对象）

多态的条件：切片+重写
重写：方法的返回值、参数列表以及名字都相同
1. 就算满足“三同”，静态方法不能进行重写
2. private方法也不能进行重写
3. 子类的方法权限必须大于等于父类（这点和C++不同）

还有Java的方法不需要用virtual修饰，也没有这个关键字，动态绑定是Java的默认行为，所有非static，非final，非private的方法都可以被认为是虚方法。C++则需要使用virtual触发动态绑定

用父类引用接收子类对象，通过父类引用调用重写方法。程序在编译期间依然是调用父类的方法，但是在运行期间却调用了子类的方法（运行时决议/动态绑定）
而重载则是编译时决议

可以在子类的重写方法上一行添加`@Override`检查重写
Java中也允许协变构成重载

甚至Java还有向下转型
```java
class Animal {
    void eat() {
        System.out.println("Animal is eating");
    }
}

class Dog extends Animal {
    void bark() {
        System.out.println("Dog is barking");
    }
}

public class DowncastingExample {
    public static void main(String[] args) {
        // 向上转型
        Animal animal = new Dog();

        // 检查类型并向下转型
        if (animal instanceof Dog) {
            Dog dog = (Dog) animal;  // 向下转型
            dog.bark();  // 调用子类特有的方法
        }

        // 注意：如果animal并不是一个Dog的实例，直接进行强制转换会导致ClassCastException
    }
}
```
向下转型发生在向上转型后，需要再次获取子类的方法或属性时，用子类引用接收父类引用。但是需要使用`instanceof`关键字进行类型的合法性判定
```java
object instanceof Class
```
`object`是要检查的对象，如果`object`是`Class`类的实例（或者其子类的实例），返回ture，否则返回false

多态的好处：
1. 减少代码的圈复杂度（if-else、循环的出现次数）
2. 可扩展性强

## 抽象类
用`abstract`修饰类名，表示抽象类。该类的信息无法描述一个具体的对象，使其成为抽象类
用`abstract`修饰方法名，表示抽象方法

抽象类无法实例化，也无法继承抽象类（抽象类能继承抽象类），只能重写父类抽象方法才能继承抽象类

`final`、`private`和`abstract`不能同时出现
有抽象方法的类一定是抽象类，但抽象类不一定有抽象方法
## 接口
接口是一种行为的规范和标准
接口和`class`同级，可以定义成员变量与成员方法，所有的方法都是抽象方法，默认由`public abstract`修饰。如果想实现接口中的方法，必须由default修饰，使之成为默认方法。或者由`public static`修饰，使之成为静态方法
成员默认由`public static final`修饰，必须初始化
接口不能实例化，和抽象类一样，只能被继承，通过`implements`继承接口并重写非静态方法
接口不能有静态代码块和构造方法
编译接口后，文件后缀为`.class`

子类可以先继承父类，再继承一个/多个接口 
```java
// 父类
class ParentClass {
    void parentMethod() {
        System.out.println("ParentClass method");
    }
}

// 接口1
interface Interface1 {
    void method1();
}

// 接口2
interface Interface2 {
    void method2();
}

// 子类继承父类并实现两个接口
class ChildClass extends ParentClass implements Interface1, Interface2 {
    // 实现接口1的方法
    public void method1() {
        System.out.println("Implemented Interface1 method");
    }

    // 实现接口2的方法
    public void method2() {
        System.out.println("Implemented Interface2 method");
    }
}

public class Main {
    public static void main(String[] args) {
        ChildClass obj = new ChildClass();

        // 调用父类的方法
        obj.parentMethod();

        // 调用实现的接口方法
        obj.method1();
        obj.method2();
    }
}
```

通过接口也能实现多态
```java
// 定义一个接口
interface Shape {
    void draw();
}

// 实现接口的两个类
class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Circle");
    }
}

class Square implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Square");
    }
}

public class Main {
    public static void main(String[] args) {
        // 使用接口类型的引用，实现多态
        Shape circle = new Circle();
        Shape square = new Square();

        // 调用 draw 方法，具体的实现取决于对象的类型
        circle.draw(); // 调用 Circle 的 draw 方法
        square.draw(); // 调用 Square 的 draw 方法
    }
}
```

接口和接口之间用`extends`继承

自定义类型的比较规则，类似于`operator<`：
```java
import java.util.Arrays;

class Student implements Comparable<Student> {
    private String name;
    private int age;
    private double grade;

    public Student(String name, int age, double grade) {
        this.name = name;
        this.age = age;
        this.grade = grade;
    }

    // Getter and Setter methods

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public double getGrade() {
        return grade;
    }

    public void setGrade(double grade) {
        this.grade = grade;
    }

    @Override
    public int compareTo(Student otherStudent) {
        if (this.age > otherStudent.age) 
	        return 1;
	    else if (this.age > otherStudent.age) 
		    return 0;
		else
			return -1;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", grade=" + grade +
                '}';
    }
}

public class Main {
    public static void main(String[] args) {
        Student[] students = new Student[3];
        students[0] = new Student("Alice", 20, 85.5);
        students[1] = new Student("Bob", 22, 78.0);
        students[2] = new Student("Charlie", 19, 92.3);

        // 使用Arrays.sort进行排序
        Arrays.sort(students);

        // 输出排序结果
        for (Student student : students) {
            System.out.println(student);
        }
    }
}
```
Java不支持引用的大小写比较，所以不能用比较符号比较两对象的引用
必须调用compareTo进行比较
```java
return this.age - o.age;
```
通常正数表示前一个数大于后一个数，负数表示前一个数小于后一个数，零表示两数相等

用比较器实现：
```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;

class Student {
    private String name;
    private int age;
    private double grade;

    public Student(String name, int age, double grade) {
        this.name = name;
        this.age = age;
        this.grade = grade;
    }

    // Getter and Setter methods

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public double getGrade() {
        return grade;
    }

    public void setGrade(double grade) {
        this.grade = grade;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", grade=" + grade +
                '}';
    }
}

class AgeComparator implements Comparator<Student> {
    @Override
    public int compare(Student student1, Student student2) {
        return Integer.compare(student1.getAge(), student2.getAge());
    }
}

public class Main {
    public static void main(String[] args) {
        List<Student> studentList = new ArrayList<>();
        studentList.add(new Student("Alice", 20, 85.5));
        studentList.add(new Student("Bob", 22, 78.0));
        studentList.add(new Student("Charlie", 19, 92.3));

        // 使用 AgeComparator 进行排序
        Collections.sort(studentList, new AgeComparator());

        // 输出排序结果
        for (Student student : studentList) {
            System.out.println(student);
        }
    }
}
```

Java对象之间的赋值操作，本质是改变引用的指向，不会触发浅拷贝
要触发浅拷贝就要调用clone()函数
```java
Arrays.clone()
```

自定义类型需要继承`clonable`接口，才能调用`clone()`方法
而`clonable`接口是一个空接口，只是用来标记类是否能被clone
若没有继承该接口，则不能被clone

clone是Object类的一个默认的native方法，由C/C++实现，将进行浅拷贝（和C++的默认赋值函数的行为相同）
```java
class MyClass implements Cloneable {
    private int value;

    public MyClass(int value) {
        this.value = value;
    }

    public int getValue() {
        return value;
    }

    public static void main(String[] args) {
        MyClass original = new MyClass(42);

        try {
            // 调用继承自Object类的clone方法
            Object clonedObject = original.clone();

            // 需要强制类型转换
            MyClass cloned = (MyClass) clonedObject;

            System.out.println("Original Value: " + original.getValue());
            System.out.println("Cloned Value: " + cloned.getValue());
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
    }
}
```
若不重写clone方法，将调用Object类的clone方法，返回值是Object类型的对象，此时注意强转
重写clone方法，实现深拷贝
```java
import java.util.Objects;

class MyCustomType implements Cloneable {
    private int customValue;

    public MyCustomType(int customValue) {
        this.customValue = customValue;
    }

    public int getCustomValue() {
        return customValue;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}

class MyClass implements Cloneable {
    private int value;
    private MyCustomType customType;

    public MyClass(int value, MyCustomType customType) {
        this.value = value;
        this.customType = customType;
    }

    public int getValue() {
        return value;
    }

    public MyCustomType getCustomType() {
        return customType;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        MyClass cloned = (MyClass) super.clone();

        // 对于自定义类型，需要递归调用其clone方法或其他深拷贝方式
        cloned.customType = (MyCustomType) customType.clone();

        return cloned;
    }

    public static void main(String[] args) {
        MyCustomType customType = new MyCustomType(100);
        MyClass original = new MyClass(42, customType);

        try {
            // 调用继承自Object类的clone方法
            MyClass cloned = (MyClass) original.clone();

            System.out.println("Original Value: " + original.getValue());
            System.out.println("Original Custom Value: " + original.getCustomType().getCustomValue());
            System.out.println("Cloned Value: " + cloned.getValue());
            System.out.println("Cloned Custom Value: " + cloned.getCustomType().getCustomValue());
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
    }
}
```

定义的所有类，都隐式地继承Object类，该类为Java的根类

对象之间支持`==`运算，将进行引用（地址）之间的比较。可以重写equals方法，使之进行所有元素的比较
```java
// 默认的equals
public boolean equals(Object obj) {
    return (this == obj);
}
```

进行重写
```java
public class MyClass {
    private int value;
    private String name;

    // 构造函数、getter等方法

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        MyClass myClass = (MyClass) obj;
        return value == myClass.value && Objects.equals(name, myClass.name);
    }

    // hashCode方法也需要相应覆盖
    @Override
    public int hashCode() {
        return Objects.hash(value, name);
    }
}
```
getClassi将得到用来判定类型的数据，该方法也是Object的方法
### String
String作为Java中的一个类，通常包含两个成员对象：
1. private final char value\[\] ：用于存储字符串的数组
2. private int hash：缓存的哈希值，首次计算后缓存，以便在后续检索中被使用

用得最多的赋值方式：
```java
String s1 = "hello";
String s2 = new String("world");
char value[] = { 'a', 'b', 'c' };
String s3 = new String(value);
System.out.println(s1);
System.out.println(s2.length());
```
直接输出字符串，将输出它的长度。也能通过`.length()`方法获取，注意这里的length是一个方法，而数组的length是一个变量，不需要使用`()`调用
String保存的是字符串常量，无法进行修改 

通过引用值进行String之间的比较，而String**重写**了`equals`方法，调用该方法进行内容之间的比较`s1.equals(s2)`，`equalsIgnoreCase`将忽略大小写进行比较
也能通过`compareTo`方法进行每个字符的字典序比较

| 方法签名                             | 描述                           | 示例                                    | 输出  |
| ------------------------------------ | ------------------------------ | --------------------------------------- | ----- |
| `char charAt(int index)`             | 返回指定索引位置的字符，不合法将编译报错。       | `String str = "Hello";`<br>`char result = str.charAt(2);` | `l`   |
| `int indexOf(int ch)`                | 返回指定字符第一次出现的索引，不存在返回-1。 | `String str = "Hello";`<br>`int result = str.indexOf('l');` | `2`   |
| `int indexOf(int ch, int fromIndex)` | 返回指定字符从指定索引开始的第一次出现的索引。 | `String str = "Hello";`<br>`int result = str.indexOf('l', 3);` | `3`  |
| `int indexOf(String str)`            | 返回指定子字符串第一次出现的索引。 | `String str = "Hello";`<br>`int result = str.indexOf("lo");` | `3`   |
| `int indexOf(String str, int fromIndex)` | 返回指定子字符串从指定索引开始的第一次出现的索引。 | `String str = "Hello";`<br>`int result = str.indexOf("lo", 2);` | `3`   |
| `int lastIndexOf(int ch)`             | 返回指定字符最后一次出现的索引。 | `String str = "Hello";`<br>`int result = str.lastIndexOf('l');` | `3`   |
| `int lastIndexOf(int ch, int fromIndex)` | 返回指定字符从指定索引开始的最后一次出现的索引。 | `String str = "Hello";`<br>`int result = str.lastIndexOf('l', 2);` | `2`   |
| `int lastIndexOf(String str)`         | 返回指定子字符串最后一次出现的索引。 | `String str = "Hello";`<br>`int result = str.lastIndexOf("lo");` | `3`   |
| `int lastIndexOf(String str, int fromIndex)` | 返回指定子字符串从指定索引开始的最后一次出现的索引。 | `String str = "Hello";`<br>`int result = str.lastIndexOf("lo", 2);` | `2`   |

将任意类型的数据转换成字符串
```java
public class NumberToStringDemo {
    public static void main(String[] args) {
        // 使用 String.valueOf() 方法
        int number1 = 123;
        String str1 = String.valueOf(number1);
        System.out.println("使用 String.valueOf(): " + str1);

        // 直接使用字符串拼接
        double number2 = 45.67;
        String str2 = "" + number2;
        System.out.println("直接使用字符串拼接: " + str2);
    }
}
```
String的valueOf重载了很多的基本类型，这些类型可以直接转换成字符串
```java
public class StringToIntDemo {
    public static void main(String[] args) {
        // 使用 Integer.parseInt()
        String str = "123";
        int number = Integer.parseInt(str);
        System.out.println("转换后的整数: " + number);
    }
}
```
大小写转换
```java
String original = "Hello";
String uppercase = original.toUpperCase();
System.out.println("转换为大写: " + uppercase);
String original = "Hello";
String lowercase = original.toLowerCase();
System.out.println("转换为小写: " + lowercase);
```
由于String存储的是常量字符串，无法直接修改原数组，所以方法将返回一个新的String

String转char类型数组
```java
public class StringToCharArrayDemo {
    public static void main(String[] args) {
        String str = "Hello";
        char[] charArray = str.toCharArray();

        // 遍历字符数组并输出每个字符
        for (char c : charArray) {
            System.out.println(c);
        }
    }
}
```
String的格式化转换
```java
String formattedString = String.format("Name: %-10s, Age: %5d, Height: %.2f meters", name, age, height);
System.out.println(formattedString);
```
String的replace
```java
public class StringReplaceDemo {
    public static void main(String[] args) {
        String originalString = "Hello, World! Hello, Java!";
        
        // 将 "Hello" 替换为 "Hi"
        String replacedString = originalString.replace("Hello", "Hi");
        
        System.out.println("原始字符串: " + originalString);
        System.out.println("替换后的字符串: " + replacedString);
    }
}
```
replaceFirst将替换第一个匹配的字符串
字符串分割
```java
public class StringSplitDemo {
    public static void main(String[] args) {
        String sentence = "Hello, World! How are you today?";

        // 使用正则表达式分割字符串，以逗号或叹号为分隔符
        String[] parts = sentence.split("[,\\s!]+");
        
        // 输出分割后的部分
        for (String part : parts) {
            System.out.println(part);
        }
    }
}
```
特殊字符需要用`\\`来转义，`\\s`表示空白字符，若分割符有多个，还能用`|`连接。四个`\`转义成一个`\`
还有一个重载版本，第二个参数为limit，表示最多分割的组（分割几次）
字符串截取
```java
public class SubstringExample {
    public static void main(String[] args) {
        String original = "Hello, World!";
        
        // 从索引为 7 开始截取到末尾
        String substring1 = original.substring(7);
        System.out.println(substring1);
    }
}

public class SubstringExample {
    public static void main(String[] args) {
        String original = "Hello, World!";
        
        // 从索引为 7 开始截取到索引为 12（不包含）的部分
        String substring2 = original.substring(7, 12);
        System.out.println(substring2);
    }
}
```
截取区间为左闭右开

去除空格
```java
public class TrimExample {
    public static void main(String[] args) {
        String original = "   Hello, World!   ";
        
        // 使用 trim() 方法去除首尾空格
        String trimmed = original.trim();
        
        System.out.println("原始字符串: '" + original + "'");
        System.out.println("去除空格后的字符串: '" + trimmed + "'");
    }
}
```

```
原始字符串: '   Hello, World!   '
去除空格后的字符串: 'Hello, World!'
```
字符串常量池
```java
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");
```
s1 == s2但s1 != s3，这里指的是引用之间的比较
双引号引起的字符串将保存在字符串常量池中，本质是一个存储在堆区的哈希表。保存之前会先检查是否相同相同字符串，若有则直接引用，若没有则存储一个新的对象
包括了value和hash，s1和s2存储了该对象的引用
而new存储的对象保存在堆上，该String对象的value和s1，s2的value相同，引用了常量池中的字符串，但是s3保存的是value和hash的引用
```java
String s1 = "hello";
String s2 = "he" + "llo";
String s3 = "he";
String s4 = "llo";
String s5 = s3 + s4;
```
s1 == s2，"he" + "llo"是常量，将直接被编译成"hello"
此时s1 != s5

intern方法手动入池（intern是一个native方法，由C/C++完成）
```java
String s1 = new String("abc");
String s2 = "abc";
```
s1 != s2，执行s1.intern()后，将堆上的字符串存储进行常量池中，此时s1 == s2

String的拼接将产生多个对象
```java
String s1 = "hello";
String s2 = s1 + "world";
```
hello和world存储进常量池，再创建一个StringBulider对象调用两次append方法，生成"helloworld"，将该StringBulider对象转成String存储进常量池中

不建议使用String的拼接，String的拼接将被优化成StringBuild的拼接

StringBuffer是线程安全的，而StringBuild不是线程安全的
### 异常
Java中，所有不正常的行为都被称为异常 

异常大概分为两类：
1. 受查异常（编译时异常）
2. 非受查异常（运行时异常）

![image.png](https://s2.loli.net/2023/11/23/UlsiEToVLQtJ41Y.png)
Error和Exception继承于Throwable，Throwable是继承体系中的底层类
Error是Java虚拟机无法解决的严重问题，比如：JVM的内部错误和资源耗尽。常见的Error有：StackOverflowError和OutOfMemoryError，当Error出现时，我们必须修改程序解决Error，因为Error将导致程序无法运行
而Exception可能会使程序的运行结果出错，但是没有Error严重，产生异常后可以通过提前写好的处理程序解决异常

而受查异常将在编译时被编译器检查出来，必须解决才能通过编译

`printStackTrace`是`Thorwable`类的一个方法，用于追踪出现异常的栈信息，包括异常发生的位置以及调用链
若对throw的异常不进行捕捉，那么JVM将终止程序
```java
try {
    // 一些可能抛出异常的代码
} catch (IOException | SQLException e) {
    // 捕获 IOException 或 SQLException 异常
    e.printStackTrace();
}

try {
    // 一些可能抛出异常的代码
} catch (IOException e) {
    // 捕获 IOException 异常
    e.printStackTrace();
} catch (SQLException e) {
    // 捕获 SQLException 异常
    e.printStackTrace();
} catch (Exception e) {
    // 捕获其他类型的异常
    e.printStackTrace();
}
```
Java中，用一个catch捕捉多个异常时，类型用`|`分割
捕获多个异常时，父类异常必须放在最后

在Java中，`throws` 是一个用于声明方法可能抛出的异常的关键字。当一个方法可能抛出异常，但不处理这些异常时，可以使用 `throws` 关键字在方法声明中列出可能被抛出的异常类型。这通常用于将异常传递给调用该方法的地方，让调用者处理这些异常
```java
public void methodName() throws ExceptionType1, ExceptionType2, ... {
    // 方法体
}
```
在main函数后声明异常，意味着把异常处理交给JVM，此时发生异常将终止程序

只要抛出了受查异常，那么程序必须catch处理该异常，否则无法通过编译
而抛出非受查异常，可以不catch该异常，但是发生异常将导致程序终止


`finally` 是一个关键字，用于定义在异常处理结束后无论是否发生异常都必须执行的代码块。`finally` 块通常用于释放资源、关闭文件、网络连接等清理操作，确保这些操作在程序执行的任何情况下都会**被执行**
```java
try {
    // 一些可能抛出异常的代码
} catch (ExceptionType1 e1) {
    // 捕获 ExceptionType1 异常
} catch (ExceptionType2 e2) {
    // 捕获 ExceptionType2 异常
} finally {
    // 在无论是否发生异常的情况下都会执行的代码块
}
```
不要在finally中写return语句，它将代替原函数的return语句