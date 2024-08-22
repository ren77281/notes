超文本标记语言
超文本：文本、语音、图片...
标记：使用标签对数据类型进行标记

vscode插件：
- auto rename tag
- view in browser 
- live server


html：html文件根标签
head：编写页面相关属性
title：页面标题
body：页面展示的内容

标签就是对象，通过代码拿到对象进行修改
!+tab/回车快速生成
`<!DOCTYPE html>`表示html的版本为5
`<html lang="en">`告知浏览器html的语言
`content = "IE=edge"`IE浏览器的渲染效果按照最高IE版本渲染
`<meta name="viewport" content="width=device-width, initial-scale=1.0">`移动端适配

input标签可以提交文件
为单选框设置id值，创建lable使其for属性等于id，即可将lable标签与单选框关联
无语义标签，span不独占一行，div独占一行

（内部样式表）css写在style标签中，标签名 + {}
（行内样式表）直接对标签添加style属性，优先级比内部样式表高
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202403081025807.png)
通过link标签引入外部样式表

选择器：
标签选择器将选择所有标签
元素可以有多个类名，类名之间用空格隔开
元素的id值唯一，#id选择元素
通配符选择器用来消除浏览器的默认样式

块元素独占一行，行内元素可以转化成块元素
盒模型：margin、border、padding
padding留白，将改变div的大小，可以设置box-sizing不撑大div大小

弹性布局的设置，需要加载父级元素上
行内元素的大小将根据内容进行改变，display:flex将强行改变其大小

JS中，单引号和双引号可以混合使用
== 和 ===
比较内容 比较内容和类型（不进行隐式类型转换）
定义变量不指定类型，将被视为全局变量
数组用`[]`创建，不用`{}`创建
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202403100836566.png)
数组的元素不要求类型相同
可以直接打印数组，访问越界元素，将得到undefined
splice删除数组元素(要删除的下标，从该下标往后几个元素)
函数调用时，参数不匹配将导致形参undefined，但不报错
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202403100904456.png)
通过构造函数定义类
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202403100907278.png)

通过类定义对象
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202403100909865.png)
静态成员属于类，不同通过对象调用