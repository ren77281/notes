在JS中，对象作为一种数据结构，它可以包含键值对（即属性），

通过CDN引入Vue框架，这是传统开发模式
```html
<script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
```

```cpp
    <div id="app">
        {{ msg }}
    </div>
    <script>
        Vue.createApp({
            setup(){
                return {
                    msg:"success"
                }
            }
        }).mount("#app")
    </script>
```
`Vue.createApp()`：创建Vue根组件，可以包含子组件，数据，方法
`setup()`：用于设置组件的响应式数据和方法
`DOM`：文档对象模型（Document Object Model）的缩写，对文档结构进行抽象，是一种表达方式。在Web开发中，DOM是html文档的编程接口，将文档解析成一颗树，树中的节点为可操作对象，通过编程改变节点，进而改变网页内容、结构、样式
`{{}}`：差值表达式，将其中的数据绑定到html模板中（渲染到DOM中），实现动态数据显示
`{}`：JS中的语法，用于创建一个对象字面量`{属性名:属性值}`，createapp需要一个对象作为参数，将setup用`{}`，包含，表示它是一个对象

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202405181401201.png)
```html
<div id="app">
        {{ msg }}
        <h2>{{ web.title }}</h2>
        <h2>{{ web.url }}</h2>
    </div>
    <script>
        Vue.createApp({
            setup(){
                const web = Vue.reactive({
                    title:"哈哈哈哈哈哈",
                    url:"chenwen.cfd"
                })

                return {
                    msg:"success",
                    web
                }
            }
        }).mount("#app")
```
`Vue.reactive`：创建响应式（复杂）的数据对象，更改对象的属性时，依赖这些属性的组件将自动更新
`const {createApp, reactive} = Vue`：解构，将Vue对象中提取`createApp, reactive`两个属性，并赋值给新的常量`createApp, reactive`，之后使用`createApp, reactive`，等价于使用`Vue.createApp, Vue.reactive`，如果对象中不存在这样的属性，那么常量值为undefined

 `Vue.ref`：创建单个基本类型数据，如数字、字符串。可以理解为数据的引用，需要通过`.value`修改被引用的数据，而reactive创建的数据可以直接修改
 
 使用ES6模块化开发，此时需要从Vue的ES module构建版本中导入函数
```html
<script type="module"> 
import {createApp, reactive} from './vue.esm-browser.js'
</script>
```

为按钮添加**绑定事件**
```html
<body>
    <div id="app">
        {{ msg }}
        <h2>{{ web.title }}</h2>
        <h2>{{ web.url }}</h2>
        <button v-on:click="edit">修改</button>
    </div>
    <script type="module">
        import {createApp, reactive, ref } from './vue.esm-browser.js'
        createApp({
            setup(){
                const web = reactive({
                    title:"哈哈哈哈哈哈",
                    url:"chenwen.cfd"
                })
                const edit = () =>{
                    web.url = "www.chenwen.cfd"
                }

                return {
                    msg:"success",
                    edit,                    
                    web
                }
            }
        }).mount("#app")
    </script>
</body>
```

`const edit = () => {}`：箭头符号，用来定义箭头函数（ES6语法）
`()`：函数的参数列表
可以用`@`替换`v-on:` ：v-on表示监听DOM事件
`v-show`：为标签添加该属性，当属性值为true时，显示该标签，否则将隐藏
可以通过按钮，实现属性值的变换：`web.show = !web.show`
v-if也能达到相同效果，但是显式/隐藏标签时，应该使用v-show，效率更高

通过v-if、v-else-if、v-else，实现条件判断，在不同条件下显式不同标签

动态渲染：在属性名前加上`v-bind:`/`:`即可
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202405181957146.png)
`{textColor:web.fontStatus}`：当`web.fontStatus`为true，那么`textColor`将被添加到`class`属性中。`{}`表示JS对象，当一个属性动态绑定JS对象时，需要用`{}`修饰