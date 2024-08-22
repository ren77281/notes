## 生命周期
防止悬空引用
生命周期的作用就是告诉编译器，多个引用之间的关系。这是一种模糊的说明
生命周期'a表示大于等于'a
返回值的生命周期等于两个参数生命周期较小的那个
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408062051314.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408062051589.png)

没有限制的生命周期，默认是'static
```rust
// `print_refs` 有两个引用参数，它们的生命周期 `'a` 和 `'b` 至少得跟函数活得一样久
fn print_refs<'a, 'b>(x: &'a i32, y: &'b i32) {
    println!("x is {} and y is {}", x, y);
}

/* 让下面的代码工作 */
fn failed_borrow<'a>() {
    let _x = 12;

    // ERROR: `_x` 活得不够久does not live long enough
    let y: &'a i32 = &_x;

    // 在函数内使用 `'a` 将会报错，原因是 `&_x` 的生命周期显然比 `'a` 要小
    // 你不能将一个小的生命周期强转成大的
}

fn main() {
    let (four, nine) = (4, 9);
    

    print_refs(&four, &nine);
    // 这里，four 和 nice 的生命周期必须要比函数 print_refs 长
    
    failed_borrow();
    // `failed_borrow`  没有传入任何引用去限制生命周期 `'a`，因此，此时的 `'a` 生命周期是没有任何限制的，它默认是 `'static`
}
```

生命周期的消除规则：
1. 每个引用参数都将被标注上生命周期
2. 如果只有一个输入生命周期，那么其生命周期将被赋予给所有的输出生命周期，所有返回值的生命周期等于输入生命周期
3. 若存在多个输入生命周期，其中一个是&self或者&mut self，则&self的生命周期将被赋给所有输出生命周期
最后检查所有的引用是否具有生命周期
## 错误处理
map将获取Result中的Ok, 你需要传入一个闭包
## 包和模块
包是一个独立**可编译**的单元，编译后生成可执行文件，或是一个库。
相关联的功能打包
同一包下，不能有同名类型(特征，结构体，枚举)。不同包下可以有相同类型
package具有多个crate。只能包含一个库(library)类型的包，以及多个二进制可执行类型的crate
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408071507564.png)

二进制包可以直接运行，有main.rs文件（根文件），crate name和package name相同
library不能直接运行，没有main.rs文件，只有lib.rs文件（根文件）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408071507835.png)

module将包中的代码按照功能性重组

同一包内的代码可以访问