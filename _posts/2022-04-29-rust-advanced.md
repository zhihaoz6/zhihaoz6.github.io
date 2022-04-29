---
layout: post
title: Rust Programming-Advanced
date: 2022-4-29
categories: blog
tags: [Rust]
description:
---

# Rust高级进阶

## 生命周期

如果想要编译通过，也很简单，只要 'b 比 'a 大就好。总之，x 变量只要比 r 活得久，那么 r 就能随意引用 x 且不会存在危险：

```rust
{
    let x = 5;            // ----------+-- 'b
                          //           |
    let r = &x;           // --+-- 'a  |
                          //   |       |
    println!("r: {}", r); //   |       |
                          // --+       |
}                         // ----------+
```

根据之前的结论，我们重新实现了代码，现在 x 的生命周期 'b 大于 r 的生命周期 'a，因此 r 对 x 的引用是安全的。

生命周期的语法也颇为与众不同，以 ' 开头，名称往往是一个单独的小写字母，大多数人都用 'a 来作为生命周期的名称。 如果是引用类型的参数，那么生命周期会位于引用符号 & 之后，并用一个空格来将生命周期和引用参数分隔开:

```rust
&i32        // 一个引用
&'a i32     // 具有显式生命周期的引用
&'a mut i32 // 具有显式生命周期的可变引用
fn useless<'a>(first: &'a i32, second: &'a i32) {}
```

函数签名中的生命周期标注:

继续之前的 longest 函数，从两个字符串切片中返回较长的那个：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

需要注意的点如下：

1. 和泛型一样，使用生命周期参数，需要先声明 <'a>
2. x、y 和返回值至少活得和 'a 一样久(因为返回值要么是 x，要么是 y)

该函数签名表明对于某些生命周期 'a，函数的两个参数都至少跟 'a 活得一样久，同时函数的返回引用也至少跟 'a 活得一样久。实际上，这意味着返回值的生命周期与参数生命周期中的较小值一致：虽然两个参数的生命周期都是标注了 'a，但是实际上这两个参数的真实生命周期可能是不一样的(生命周期 'a 不代表生命周期等于 'a，而是大于等于 'a)。

在通过函数签名指定生命周期参数时，我们并没有改变传入引用或者返回引用的真实生命周期，而是告诉编译器当不满足此约束条件时，就拒绝编译通过。

因此 longest 函数并不知道 x 和 y 具体会活多久，只要知道它们的作用域至少能持续 'a 这么长就行。

当把具体的引用传给 longest 时，那生命周期 'a 的大小就是 x 和 y 的作用域的重合部分，换句话说，'a 的大小将等于 x 和 y 中较小的那个。由于返回值的生命周期也被标记为 'a，因此返回值的生命周期也是 x 和 y 中作用域较小的那个。

再来看一个例子，该例子证明了 result 的生命周期必须等于两个参数中生命周期较小的那个:

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

函数的返回值如果是一个引用类型，那么它的生命周期只会来源于：

1. 函数参数的生命周期
2. 函数体中某个新建引用的生命周期

若是后者情况，就是典型的悬垂引用场景：

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

主要问题就在于，result 在函数结束后就被释放，但是在函数结束后，对 result 的引用依然在继续。在这种情况下，没有办法指定合适的生命周期来让编译通过，因此我们也就在 Rust 中避免了悬垂引用。

那遇到这种情况该怎么办？最好的办法就是返回内部字符串的所有权，然后把字符串的所有权转移给调用者：

```rust
fn longest<'a>(_x: &str, _y: &str) -> String {
    String::from("really long string")
}

fn main() {
   let s = longest("not", "important");
}
```

至此，可以对生命周期进行下总结：生命周期语法用来将函数的多个引用参数和返回值的作用域关联到一起，一旦关联到一起后，Rust 就拥有充分的信息来确保我们的操作是内存安全的。

### 结构体中的生命周期

实际上，为具有生命周期的结构体实现方法时，我们使用的语法跟泛型参数语法很相似：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

其中有几点需要注意的：

1. impl 中必须使用结构体的完整名称，包括 <'a>，因为生命周期标注也是结构体类型的一部分！
2. 方法签名中，往往不需要标注生命周期，得益于生命周期消除的第一和第三规则

```rust
impl<'a: 'b, 'b> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

1. 'a: 'b，是生命周期约束语法，跟泛型约束非常相似，用于说明 'a 必须比 'b 活得久
2. 可以把 'a 和 'b 都在同一个地方声明（如上），或者分开声明但通过 where 'a: 'b 约束生命周期关系，如下：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &'b str
    where
        'a: 'b,
    {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

需要加一个约束。


### 静态生命周期：

在 Rust 中有一个非常特殊的生命周期，那就是 'static，拥有该生命周期的引用可以和整个程序活得一样久。

## 深入生命周期

@TODO

## &'static 和 T: 'static


'static 的用法：

```rust
use std::fmt::Display;
fn main() {
    let mark_twain = "Samuel Clemens";
    print(&mark_twain);
}

fn print<T: Display + 'static>(message: &T) {
    println!("{}", message);
}
```


T: 'static；

```rust
&'static 对于生命周期有着非常强的要求：一个引用必须要活得跟剩下的程序一样久，才能被标注为 &'static。


use std::fmt::Debug;

fn print_it<T: Debug + 'static>( input: T) {
    println!( "'static value passed in is: {:?}", input );
}

fn print_it1( input: impl Debug + 'static ) {
    println!( "'static value passed in is: {:?}", input );
}



fn main() {
    let i = 5;

    print_it(&i);
    print_it1(&i);
}
```


## 闭包 Closure

闭包是一种匿名函数，它可以赋值给变量也可以作为参数传递给其它函数，不同于函数的是，它允许捕获调用者作用域中的值:

```rust
fn main() {
   let x = 1;
   let sum = |y| x + y;

    assert_eq!(3, sum(2));
}
```

上面的代码展示了非常简单的闭包 sum，它拥有一个入参 y，同时捕获了作用域中的 x 的值，因此调用 sum(2) 意味着将 2（参数 y）跟 1（x）进行相加,最终返回它们的和：3。

Rust 闭包在形式上借鉴了 Smalltalk 和 Ruby 语言，与函数最大的不同就是它的参数是通过 |parm1| 的形式进行声明，如果是多个参数就 |param1, param2,...|， 下面给出闭包的形式定义：

```rust
|param1, param2,...| {
    语句1;
    语句2;
    返回表达式
}
```

如果只有一个返回表达式的话，定义可以简化为：

```rust
|param1| 返回表达式
```

### 闭包的类型推导

Rust 是静态语言，因此所有的变量都具有类型，但是得益于编译器的强大类型推导能力，在很多时候我们并不需要显式地去声明类型，但是显然函数并不在此列，必须手动为函数的所有参数和返回值指定类型，原因在于函数往往会作为 API 提供给你的用户，因此你的用户必须在使用时知道传入参数的类型和返回值类型。

与函数相反，闭包并不会作为 API 对外提供，因此它可以享受编译器的类型推导能力，无需标注参数和返回值的类型。

```rust
let sum = |x: i32, y: i32| -> i32 {
    x + y
}
```

下面展示了同一个功能的函数和闭包实现形式：

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

虽然类型推导很好用，但是它不是泛型，当编译器推导出一种类型后，它就会一直使用该类型：

```rust
let example_closure = |x| x;

let s = example_closure(String::from("hello"));
let n = example_closure(5);
```

### 结构体中的闭包

假设我们要实现一个简易缓存，功能是获取一个值，然后将其缓存起来，那么可以这样设计：

一个闭包用于获取值
一个变量，用于存储该值
可以使用结构体来代表缓存对象，最终设计如下：

```rust
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    query: T,
    value: Option<u32>,
}
```

可以看得出这一长串是 T 的特征约束，再结合之前的已知信息：query 是一个闭包，大概可以推测出，Fn(u32) -> u32 是一个特征，用来表示 T 是一个闭包类型。T: Fn(u32) -> u32 意味着 query 的类型是 T，该类型必须实现了相应的闭包特征 Fn(u32) -> u32。

特征 Fn(u32) -> u32 从表面来看，就对闭包形式进行了显而易见的限制：该闭包拥有一个u32类型的参数，同时返回一个u32类型的值。

接着，为缓存实现方法：

```rust
impl<T> Cacher<T>
where
    T: Fn(u32) -> u32,
{
    fn new(query: T) -> Cacher<T> {
        Cacher {
            query,
            value: None,
        }
    }

    // 先查询缓存值 `self.value`，若不存在，则调用 `query` 加载
    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.query)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}
```

### 捕获作用域中的值

```rust
fn main() {
    let x = 4;

    let equal_to_x = |z| z == x;

    let y = 4;

    assert!(equal_to_x(y));
}
```

上面代码中，x 并不是闭包 equal_to_x 的参数，但是它依然可以去使用 x，因为 equal_to_x 在 x 的作用域范围内。

### 三种 Fn 特征

闭包捕获变量有三种途径，恰好对应函数参数的三种传入方式：转移所有权、可变借用、不可变借用，因此相应的 Fn 特征也有三种：

FnOnce，该类型的闭包会拿走被捕获变量的所有权。Once 顾名思义，说明该闭包只能运行一次：

```rust
fn fn_once<F>(func: F)
where
    F: FnOnce(usize) -> bool,
{
    println!("{}", func(3));
    println!("{}", func(4));
}

fn main() {
    let x = vec![1, 2, 3];
    fn_once(|z|{z == x.len()})
}
```

仅实现 FnOnce 特征的闭包在调用时会转移所有权，所以显然不能对已失去所有权的闭包变量进行二次调用：

如果你想强制闭包取得捕获变量的所有权，可以在参数列表前添加 move 关键字，这种用法通常用于闭包的生命周期大于捕获变量的生命周期时，例如将闭包返回或移入其他线程。

```rust
use std::thread;
let v = vec![1, 2, 3];
let handle = thread::spawn(move || {
    println!("Here's a vector: {:?}", v);
});
handle.join().unwrap();
```

FnMut，它以可变借用的方式捕获了环境中的值，因此可以修改该值：

```rust
fn main() {
    let mut s = String::new();

    let update_string =  |str| s.push_str(str);
    update_string("hello");

    println!("{:?}",s);
}
```

Fn 特征，它以不可变借用的方式捕获环境中的值

```rust
fn main() {
    let s = "hello, ".to_string();

    let update_string =  |str| println!("{},{}",s,str);

    exec(update_string);

    println!("{:?}",s);
}

fn exec<'a, F: Fn(String) -> ()>(f: F)  {
    f("world".to_string())
}
```

在这里，因为无需改变 s，因此闭包中只对 s 进行了不可变借用，那么在 exec 中，将其标记为 Fn 特征就完全正确。

**一个闭包实现了哪种 Fn 特征取决于该闭包如何使用被捕获的变量，而不是取决于闭包如何捕获它们。**

实际上，一个闭包并不仅仅实现某一种 Fn 特征，规则如下：

1. 所有的闭包都自动实现了 FnOnce 特征，因此任何一个闭包都至少可以被调用一次
2. 没有移出所捕获变量的所有权的闭包自动实现了 FnMut 特征
3. 不需要对捕获变量进行改变的闭包自动实现了 Fn 特征

### 闭包作为函数返回值

```rust
fn factory() -> Fn(i32) -> i32 {
    let num = 5;

    |x| x + num
}

let f = factory();

let answer = f(1);
assert_eq!(6, answer);
```

上面代码编译器报错：Rust 要求函数的参数和返回类型，必须有固定的内存大小，例如 i32 就是 4 个字节，引用类型是 8 个字节，总之，绝大部分类型都有固定的大小，但是不包括特征，因为特征类似接口，对于编译器来说，无法知道它后面藏的真实类型是什么，因为也无法得知具体的大小。同样，我们也无法知道闭包的具体类型。

impl Trait 可以用来返回一个实现了指定特征的类型，那么这里 impl Fn(i32) -> i32 的返回值形式，说明我们要返回一个闭包类型，它实现了 Fn(i32) -> i32 特征。

```rust
fn factory(x:i32) -> impl Fn(i32) -> i32 {

    let num = 5;

    if x > 1{
        move |x| x + num
    } else {
        move |x| x - num
    }
}
```

上面代码编译器报错: if 和 else 分支中返回了不同的闭包类型.
只需要用 Box 的方式即可实现：

```rust
fn factory(x:i32) -> Box<dyn Fn(i32) -> i32> {
    let num = 5;

    if x > 1{
        Box::new(move |x| x + num)
    } else {
        Box::new(move |x| x - num)
    }
}
```

## 迭代器

对于 for 如何遍历迭代器，还有一个问题，它如何取出迭代器中的元素？

先来看一个特征：

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // 省略其余有默认实现的方法
}
```

迭代器之所以成为迭代器，就是因为实现了 Iterator 特征，要实现该特征，最主要的就是实现其中的 next 方法，该方法控制如何从集合中取值，最终返回值的类型是关联类型 Item。

### 例子：模拟实现 for 循环

```rust
let values = vec![1, 2, 3];

{
    let result = match IntoIterator::into_iter(values) {
        mut iter => loop {
            match iter.next() {
                Some(x) => { println!("{}", x); },
                None => break,
            }
        },
    };
    result
}
```

IntoIterator::into_iter 是使用完全限定的方式去调用 into_iter 方法，这种调用方式跟 values.into_iter() 是等价的。

同时我们使用了 loop 循环配合 next 方法来遍历迭代器中的元素，当迭代器返回 None 时，跳出循环

### into_iter, iter, iter_mut

我们统一使用了 into_iter 的方式将数组转化为迭代器，除此之外，还有 iter 和 iter_mut，聪明的读者应该大概能猜到这三者的区别：

1. into_iter 会夺走所有权
2. iter 是借用
3. iter_mut 是可变借用

into_ 之类的，都是拿走所有权，_mut 之类的都是可变借用，剩下的就是不可变借用。

### 迭代器适配器

迭代器适配器，顾名思义，会返回一个新的迭代器，这是实现链式方法调用的关键：v.iter().map().filter()...。

迭代器适配器是惰性的，意味着你需要一个消费者适配器来收尾，最终将迭代器转换成一个具体的值：

```rust
let v1: Vec<i32> = vec![1, 2, 3];

v1.iter().map(|x| x + 1);
```

上述代码报错，如下修改:

这里的 map 方法是一个迭代者适配器，它是惰性的，不产生任何行为，因此我们还需要一个消费者适配器进行收尾：

```rust
let v1: Vec<i32> = vec![1, 2, 3];

let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

assert_eq!(v2, vec![2, 3, 4]);
```

### collect

上面代码中，使用了 collect 方法，该方法就是一个消费者适配器，使用它可以将一个迭代器中的元素收集到指定类型中，这里我们为 v2 标注了 Vec<_> 类型，就是为了告诉 collect：请把迭代器中的元素消费掉，然后把值收集成 Vec<_> 类型，至于为何使用 _，因为编译器会帮我们自动推导。

为何 collect 在消费时要指定类型？是因为该方法其实很强大，可以收集成多种不同的集合类型，Vec<T> 仅仅是其中之一，因此我们必须显式的告诉编译器我们想要收集成的集合类型。

还有一点值得注意，map 会对迭代器中的每一个值进行一系列操作，然后把该值转换成另外一个新值，该操作是通过闭包 |x| x + 1 来完成：最终迭代器中的每个值都增加了 1，从 [1, 2, 3] 变为 [2, 3, 4]。

再来看看如何使用 collect 收集成 HashMap 集合：

```rust
use std::collections::HashMap;
fn main() {
    let names = ["sunface", "sunfei"];
    let ages = [18, 18];
    let folks: HashMap<_, _> = names.into_iter().zip(ages.into_iter()).collect();

    println!("{:?}",folks);
}
```

zip 是一个迭代器适配器，它的作用就是将两个迭代器的内容压缩到一起，形成 Iterator<Item=(ValueFromA, ValueFromB)> 这样的新的迭代器，在此处就是形如 [(name1, age1), (name2, age2)] 的迭代器。

然后再通过 collect 将新迭代器中(K, V) 形式的值收集成 HashMap<K, V>，同样的，这里必须显式声明类型，然后 HashMap 内部的 KV 类型可以交给编译器去推导，最终编译器会推导出 HashMap<&str, i32>，完全正确！

### 闭包作为适配器参数

之前的 map 方法中，我们使用闭包来作为迭代器适配器的参数，它最大的好处不仅在于可以就地实现迭代器中元素的处理，还在于可以捕获环境值：

```rust
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}
```

filter 是迭代器适配器，用于对迭代器中的每个值进行过滤。 它使用闭包作为参数，该闭包的参数 s 是来自迭代器中的值，然后使用 s 跟外部环境中的 shoe_size 进行比较，若相等，则在迭代器中保留 s 值，若不相等，则从迭代器中剔除 s 值，最终通过 collect 收集为 Vec<Shoe> 类型。

### 实现 Iterator 特征

首先，创建一个计数器：

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}
```

我们为计数器 Counter 实现了一个关联函数 new，用于创建新的计数器实例。下面我们继续为计数器实现 Iterator 特征：

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

首先，将该特征的关联类型设置为 u32，由于我们的计数器保存的 count 字段就是 u32 类型， 因此在 next 方法中，最后返回的是实际上是 Option<u32> 类型。

每次调用 next 方法，都会让计数器的值加一，然后返回最新的计数值，一旦计数大于 5，就返回 None。

最后，使用我们新建的 Counter 进行迭代：

```rust
let mut counter = Counter::new();

assert_eq!(counter.next(), Some(1));
assert_eq!(counter.next(), Some(2));
assert_eq!(counter.next(), Some(3));
assert_eq!(counter.next(), Some(4));
assert_eq!(counter.next(), Some(5));
assert_eq!(counter.next(), None);
```

可以看出，实现自己的迭代器非常简单，但是 Iterator 特征中，不仅仅是只有 next 一个方法，那为什么我们只需要实现它呢？因为其它方法都具有默认实现，所以无需像 next 这样手动去实现，而且这些默认实现的方法其实都是基于 next 方法实现的。

下面的代码演示了部分方法的使用：

```rust
let sum: u32 = Counter::new()
    .zip(Counter::new().skip(1))
    .map(|(a, b)| a * b)
    .filter(|x| x % 3 == 0)
    .sum();
assert_eq!(18, sum);
```

其中 zip，map，filter 是迭代器适配器：

zip 把两个迭代器合并成一个迭代器，新迭代器中，每个元素都是一个元组，由之前两个迭代器的元素组成。例如将形如 [1, 2, 3] 和 [4, 5, 6] 的迭代器合并后，新的迭代器形如 [(1, 4),(2, 5),(3, 6)]
map 是将迭代器中的值经过映射后，转换成新的值
filter 对迭代器中的元素进行过滤，若闭包返回 true 则保留元素，反之剔除
而 sum 是消费者适配器，对迭代器中的所有元素求和，最终返回一个 u32 值 18。


### enumerate

针对 for 循环，我们提供了一种方法可以获取迭代时的索引：

```rust
let v = vec![1u64, 2, 3, 4, 5, 6];
for (i,v) in v.iter().enumerate() {
    println!("第{}个值是{}",i,v)
}
```

首先 v.iter() 创建迭代器，其次 调用 Iterator 特征上的方法 enumerate，该方法产生一个新的迭代器，其中每个元素均是元组 (索引，值)。

因为 enumerate 是迭代器适配器，因此我们可以对它返回的迭代器调用其它 Iterator 特征方法：

```rust
let v = vec![1u64, 2, 3, 4, 5, 6];
let val = v.iter()
    .enumerate()
    // 每两个元素剔除一个
    // [1, 3, 5]
    .filter(|&(idx, _)| idx % 2 == 0)
    .map(|(idx, val)| val)
    // 累加 1+3+5 = 9
    .fold(0u64, |sum, acm| sum + acm);

println!("{}", val);
```

### 迭代器的性能

要完成集合遍历，既可以使用 for 循环也可以使用迭代器，那么二者之间该怎么选择呢，性能有多大差距呢？

理论分析不会有结果，直接测试最为靠谱：

```rust
#![feature(test)]

extern crate rand;
extern crate test;

fn sum_for(x: &[f64]) -> f64 {
    let mut result: f64 = 0.0;
    for i in 0..x.len() {
        result += x[i];
    }
    result
}

fn sum_iter(x: &[f64]) -> f64 {
    x.iter().sum::<f64>()
}

#[cfg(test)]
mod bench {
    use test::Bencher;
    use rand::{Rng,thread_rng};
    use super::*;

    const LEN: usize = 1024*1024;

    fn rand_array(cnt: u32) -> Vec<f64> {
        let mut rng = thread_rng();
        (0..cnt).map(|_| rng.gen::<f64>()).collect()
    }

    #[bench]
    fn bench_for(b: &mut Bencher) {
        let samples = rand_array(LEN as u32);
        b.iter(|| {
            sum_for(&samples)
        })
    }

    #[bench]
    fn bench_iter(b: &mut Bencher) {
        let samples = rand_array(LEN as u32);
        b.iter(|| {
            sum_iter(&samples)
        })
    }
}
```

上面的代码对比了 for 循环和迭代器 iterator 完成同样的求和任务的性能对比，可以看到迭代器还要更快一点。

```bash
test bench::bench_for  ... bench:     998,331 ns/iter (+/- 36,250)
test bench::bench_iter ... bench:     983,858 ns/iter (+/- 44,673)
```

迭代器是 Rust 的 零成本抽象（zero-cost abstractions）之一，意味着抽象并不会引入运行时开销，这与 Bjarne Stroustrup（C++ 的设计和实现者）在 Foundations of C++（2012） 中所定义的 零开销（zero-overhead）如出一辙：

## 深入类型

### newtype

何为 newtype？简单来说，就是使用元组结构体的方式将已有的类型包裹起来：struct Meters(u32);，那么此处 Meters 就是一个 newtype。

#### 为外部类型实现外部特征

如果在外部类型上实现外部特征必须使用 newtype 的方式，否则你就得遵循孤儿规则：要为类型 A 实现特征 T，那么 A 或者 T 必须至少有一个在当前的作用范围内。

如果想使用 println!("{}", v) 的方式去格式化输出一个动态数组 Vec，以期给用户提供更加清晰可读的内容，那么就需要为 Vec 实现 Display 特征，但是这里有一个问题： Vec 类型定义在标准库中，Display 亦然，这时就可以祭出大杀器 newtype 来解决：

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

如上所示，使用元组结构体语法 struct Wrapper(Vec<String>) 创建了一个 newtype Wrapper，然后为它实现 Display 特征，最终实现了对 Vec 动态数组的格式化输出。

### 类型别名(Type Alias)

除了使用 newtype，我们还可以使用一个更传统的方式来创建新类型：类型别名

```rust
type Meters = u32
```

嗯，不得不说，类型别名的方式看起来比 newtype 顺眼的多，而且跟其它语言的使用方式几乎一致，但是： 类型别名并不是一个独立的全新的类型，而是某一个类型的别名，因此编译器依然会把 Meters 当 u32 来使用：

```rust
type Meters = u32;

let x: u32 = 5;
let y: Meters = 5;

println!("x + y = {}", x + y);
```

上面的代码将顺利编译通过，但是如果你使用 newtype 模式，该代码将无情报错，简单做个总结：

1. 类型别名仅仅是别名，只是为了让可读性更好，并不是全新的类型，newtype 才是！
2. 类型别名无法实现为外部类型实现外部特征等功能，而 newtype 可以

### !永不返回类型

! 用来说明一个函数永不返回任何值。

```rust
fn main() {
    let i = 2;
    let v = match i {
       0..=3 => i,
       _ => println!("不合规定的值:{}", i)
    };
}
```

上面函数，会报出一个编译错误,原因很简单: 要赋值给 v，就必须保证 match 的各个分支返回的值是同一个类型，但是上面一个分支返回数值、另一个分支返回元类型 ()，自然会出错。

既然 println 不行，那再试试 panic

```rust
fn main() {
    let i = 2;
    let v = match i {
       0..=3 => i,
       _ => panic!("不合规定的值:{}", i)
    };
}
```

神奇的事发生了，此处 panic 竟然通过了编译。难道这两个宏拥有不同的返回类型？

你猜的没错：panic 的返回值是 !，代表它决不会返回任何值，既然没有任何返回值，那自然不会存在分支类型不匹配的情况。

### Sized 和不定长类型 DST

如果从编译器何时能获知类型大小的角度出发，可以分成两类:

1. 定长类型( sized )，这些类型的大小在编译时是已知的
2. 不定长类型( unsized )，与定长类型相反，它的大小只有到了程序运行时才能动态获知，这种类型又被称之为 DST

### 动态大小类型 DST

之前学过的几乎所有类型，都是固定大小的类型，包括集合 Vec、String 和 HashMap 等，而动态大小类型刚好与之相反：编译器无法在编译期得知该类型值的大小，只有到了程序运行时，才能动态获知。对于动态类型，我们使用 DST(dynamically sized types)或者 unsized 类型来称呼它。

上述的这些集合虽然底层数据可动态变化，感觉像是动态大小的类型。但是实际上，*这些底层数据只是保存在堆上，在栈中还存有一个引用类型*，该引用包含了集合的内存地址、元素数目、分配空间信息，通过这些信息，编译器对于该集合的实际大小了若指掌，最最重要的是：栈上的引用类型是固定大小的，因此它们依然是固定大小的类型。

正因为编译器无法在编译期获知类型大小，若你试图在代码中直接使用 DST 类型，将无法通过编译。

试图创建动态大小的数组:

```rust
fn my_function(n: usize) {
    let array = [123; n];
}
```

以上代码就会报错(错误输出的内容并不是因为 DST，但根本原因是类似的)，因为 n 在编译期无法得知，而数组类型的一个组成部分就是长度，长度变为动态的，自然类型就变成了 unsized 。

str是一个动态类型，同时还是 String 和 &str 的底层数据类型。

由于 str 是动态类型，因此它的大小直到运行期才知道，下面的代码会因此报错：

```rust
// error
let s1: str = "Hello there!";
let s2: str = "How's it going?";

// ok
let s3: &str = "on?"
```

Rust 需要明确地知道一个特定类型的值占据了多少内存空间，同时该类型的所有值都必须使用相同大小的内存。如果 Rust 允许我们使用这种动态类型，那么这两个 str 值就需要占用同样大小的内存，这显然是不现实的: s1 占用了 12 字节，s2 占用了 15 字节，总不至于为了满足同样的内存大小，用空白字符去填补字符串吧？

所以，我们只有一条路走，那就是给它们一个固定大小的类型：&str。那么为何字符串切片 &str 就是固定大小呢？因为它的引用存储在栈上，具有固定大小(类似指针)，同时它指向的数据存储在堆中，也是已知的大小，再加上 &str 引用中包含有堆上数据内存地址、长度等信息，因此最终可以得出字符串切片是固定大小类型的结论。

正是因为 &str 的引用有了底层堆数据的明确信息，它才是固定大小类型。假设如果它没有这些信息呢？那它也将变成一个动态类型。因此，将动态数据固定化的秘诀就是使用引用指向这些动态数据，然后在引用中存储相关的内存位置、长度等信息。

### 特征对象

```rust
fn foobar_1(thing: &dyn MyThing) {}     // OK
fn foobar_2(thing: Box<dyn MyThing>) {} // OK
fn foobar_3(thing: MyThing) {}          // ERROR!
```

如上所示，只能通过引用或 Box 的方式来使用特征对象，直接使用将报错！

总结：只能间接使用的 DST:
Rust 中常见的 DST 类型有: str、[T]、dyn Trait，它们都无法单独被使用，必须要通过引用或者 Box 来间接使用 。

### Sized 特征

在使用泛型时，Rust 如何保证我们的泛型参数是固定大小的类型呢？例如以下泛型函数：

```rust
fn generic<T>(t: T) {
    // --snip--
}
```

该函数很简单，就一个泛型参数 T，那么如何保证 T 是固定大小的类型？奥秘在于编译器自动帮我们加上了 Sized 特征约束：

```rust
fn generic<T: Sized>(t: T) {
    // --snip--
}
```

在上面，Rust 自动添加的特征约束 T: Sized，表示泛型函数只能用于一切实现了 Sized 特征的类型上，而所有在编译时就能知道其大小的类型，都会自动实现 Sized 特征.

每一个特征都是一个可以通过名称来引用的动态大小类型。因此如果想把特征作为具体的类型来传递给函数，你必须将其转换成一个特征对象：诸如 &dyn Trait 或者 Box<dyn Trait> (还有 Rc<dyn Trait>)这些引用类型。

现在还有一个问题：假如想在泛型函数中使用动态数据类型怎么办？可以使用 ?Sized 特征:

```rust
fn generic<T: ?Sized>(t: &T) {
    // --snip--
}
```

?Sized 特征用于表明类型 T 既有可能是固定大小的类型，也可能是动态大小的类型。还有一点要注意的是，函数参数类型从 T 变成了 &T，因为 T 可能是动态大小的，因此需要用一个固定大小的指针(引用)来包裹它。

### Box

使用 Box 可以将一个动态大小的特征变成一个具有固定大小的特征对象，能否故技重施，将 str 封装成一个固定大小类型？

首先，Box<str> 使用了一个引用来指向 str，嗯，满足了第一个条件。但是第二个条件呢？Box 中有该 str 的长度信息吗？显然是 No。那为什么特征就可以变成特征对象？其实这个还蛮复杂的，简单来说，对于特征对象，编译器无需知道它具体是什么类型，只要知道它能调用哪几个方法即可，因此编译器帮我们实现了剩下的一切。

### 整数转换为枚举

在 Rust 中，从枚举到整数的转换很容易，但是反过来，就没那么容易

在实际场景中，从枚举到整数的转换有时还是非常需要的，例如你有一个枚举类型，然后需要从外面传入一个整数，用于控制后续的流程走向，此时就需要用整数去匹配相应的枚举.

https://course.rs/advance/into-types/enum-int.html

## 智能指针

@TODO