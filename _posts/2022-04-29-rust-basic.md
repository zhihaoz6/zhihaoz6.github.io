---
layout: post
title: Rust Programming-Basic
date: 2022-4-29
categories: blog
tags: [Rust]
description:
---

# Rust基础

## 基本类型

Rust 每个值都有其确切的数据类型，总的来说可以分为两类：基本类型和复合类型。 基本类型意味着它们往往是一个最小化原子类型，无法解构为其它类型(一般意义上来说)，由以下组成：

数值类型: 有符号整数 (i8, i16, i32, i64, isize)、 无符号整数 (u8, u16, u32, u64, usize) 、浮点数 (f32, f64)、以及有理数、复数

字符串：字符串字面量和字符串切片 &str

布尔类型： true和false

字符类型: 表示单个 Unicode 字符，存储为 4 个字节

单元类型: 即 () ，其唯一的值也是 ()

### 栈和堆

写入方面：入栈比在堆上分配内存要快，因为入栈时操作系统无需分配新的空间，只需要将新数据放入栈顶即可。相比之下，在堆上分配内存则需要更多的工作，这是因为操作系统必须首先找到一块足够存放数据的内存空间，接着做一些记录为下一次分配做准备。

读取方面：得益于 CPU 高速缓存，使得处理器可以减少对内存的访问，高速缓存和内存的访问速度差异在 10 倍以上！栈数据往往可以直接存储在 CPU 高速缓存中，而堆数据只能存储在内存中。访问堆上的数据比访问栈上的数据慢，因为必须先访问栈再通过栈上的指针来访问内存。

## 所有权与堆栈

当你的代码调用一个函数时，传递给函数的参数（包括可能指向堆上数据的指针和函数的局部变量）依次被压入栈中，当函数调用结束时，这些值将被从栈中按照相反的顺序依次移除。

因为堆上的数据缺乏组织，因此跟踪这些数据何时分配和释放是非常重要的，否则堆上的数据将产生内存泄漏 —— 这些数据将永远无法被回收。这就是 Rust 所有权系统为我们提供的强大保障。

### 所有权原则
1. Rust 中每一个值都 有且只有 一个所有者(变量)

2. 当所有者(变量)离开作用域范围时，这个值将被丢弃(drop)

### 转移所有权

```rust
let s1 = String::from("hello");
let s2 = s1;
```

当 s1 赋予 s2 后，Rust 认为 s1 不再有效，因此也无需在 s1 离开作用域后 drop 任何东西，这就是把所有权从 s1 转移给了 s2，s1 在被赋予 s2 后就马上失效了。

## 复合类型

### 字符串

切片(slice)：

对于字符串而言，切片就是对 String 类型中某一部分的引用，它看起来像这样：

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

hello 没有引用整个 String s，而是引用了 s 的一部分内容，通过 [0..5] 的方式来指定。

这就是创建切片的语法，使用方括号包括的一个序列: [开始索引..终止索引]，其中开始索引是切片中第一个元素的索引位置，而终止索引是最后一个元素后面的索引位置，也就是这是一个 右半开区间。在切片数据结构内部会保存开始的位置和切片的长度，其中长度是通过 终止索引 - 开始索引 的方式计算得来的

字符串切片的类型标识是`&str`。

str 类型是硬编码进可执行文件，也无法被修改，但是 String 则是一个可增长、可改变且具有所有权的 UTF-8 编码字符串，当 Rust 用户提到字符串时，往往指的就是 String 类型和 &str 字符串切片类型，这两个类型都是 UTF-8 编码。


将 &str 类型生成 String 类型的操作：

```rust
String::from("hello,world")
"hello,world".to_string()
```

将 String 类型转为 &str 类型，取引用即可。

### 字符串索引

Rust中不能使用索引的方式访问字符串的某个字符或者子串。

字符串的底层的数据存储格式实际上是[ u8 ]，一个字节数组。对于 let hello = String::from("Hola"); 这行代码来说， hello 的长度是 4 个字节，因为 "hola" 中的每个字母在 UTF-8 编码中仅占用 1 个字节。

但是对于`let hello = String::from("中国人")`，如果问你该字符串多长，你可能会说 3，但是实际上是 9 个字节的长度，因为大部分常用汉字在 UTF-8 中的长度是 3 个字节，因此这种情况下对 hello 进行索引，访问 &hello[0] 没有任何意义，因为你取不到 中 这个字符，而是取到了这个字符三个字节中的第一个字节，这是一个非常奇怪而且难以理解的返回值。

还有一个原因导致了 Rust 不允许去索引字符串：因为索引操作，我们总是期望它的性能表现是 O(1)，然而对于 String 类型来说，无法保证这一点，因为 Rust 可能需要从 0 开始去遍历字符串来定位合法的字符。

### 操作UTF-8字符串

如果你想要以 Unicode 字符的方式遍历字符串，最好的办法是使用 chars 方法，例如：

```rust
for c in "中国人".chars() {
    println!("{}", c);
}
```

这种方式是返回字符串的底层字节数组表现形式：

```rust
for b in "中国人".bytes() {
    println!("{}", b);
}
```

想要准确的从 UTF-8 字符串中获取子串是较为复杂的事情，例如想要从 holla中国人नमस्ते 这种变长的字符串中取出某一个子串，使用标准库你是做不到的。 你需要在 crates.io 上搜索 utf8 来寻找想要的功能。

### 字符串深度剖析

Rust 的开发者想出了一个无比惊艳的办法：变量在离开作用域后，就自动释放其占用的内存：

与其它系统编程语言的 free 函数相同，Rust 也提供了一个释放内存的函数： drop，但是不同的是，其它语言要手动调用 free 来释放每一个变量占用的内存，而 Rust 则在变量离开作用域时，自动调用 drop 函数: 上面代码中，Rust 在结尾的 } 处自动调用 drop。

### 元组

元组是由多种类型组合到一起形成的，因此它是复合类型，元组的长度是固定的，元组中元素的顺序也是固定的。

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```

用模式匹配解构元组:

```rust
fn main() {
    let tup = (500, 6.4, 1);

    let (x, y, z) = tup;

    println!("The value of y is: {}", y);
}
```

用 . 来访问元组:

模式匹配可以让我们一次性把元组中的值全部或者部分获取出来，如果只想要访问某个特定元素，那模式匹配就略显繁琐，对此，Rust 提供了 . 的访问方式：

```rust
fn main() {
    let x: (i32, f64, u8) = (500, 6.4, 1);

    let five_hundred = x.0;

    let six_point_four = x.1;

    let one = x.2;
}
```
和其它语言的数组、字符串一样，元组的索引从 0 开始。

### 结构体

一个结构体有几部分组成：

1. 通过关键字 struct 定义
2. 一个清晰明确的结构体 名称
3. 几个有名字的结构体 字段

```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
```

该结构体名称是 User，拥有 4 个字段，且每个字段都有对应的字段名及类型声明，例如 username 代表了用户名，是一个可变的 String 类型。


创建结构体实例:

为了使用上述结构体，我们需要创建 User 结构体的实例：

```rust
let mut user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};
user1.email = String::from("anotheremail@example.com");
```

有几点值得注意:

1. 初始化实例时，每个字段都需要进行初始化
2. 初始化时的字段顺序不需要和结构体定义时的顺序一致

需要注意的是，必须要将结构体实例声明为可变的，才能修改其中的字段，Rust 不支持将某个结构体某个字段标记为可变。

简化结构体创建:

下面的函数类似一个构建函数，返回了 User 结构体的实例：

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email: email,
        username: username,
        active: true,
        sign_in_count: 1,
    }
}
```

或者使用缩略方式：

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}
```

结构体更新语法:

```rust
let user2 = User {
        email: String::from("another@example.com"),
        ..user1
    };
```

因为 user2 仅仅在 email 上与 user1 不同，因此我们只需要对 email 进行赋值，剩下的通过结构体更新语法 ..user1 即可完成。

.. 语法表明凡是我们没有显示声明的字段，全部从 user1 中自动获取。需要注意的是 ..user1 必须在结构体的尾部使用。

值得注意的是：username 所有权被转移给了 user2，导致了 user1 无法再被使用，但是并不代表 user1 内部的其它字段不能被继续使用，例如：

结构体的内存排列:


### 元组结构体(Tuple Struct)

结构体必须要有名称，但是结构体的字段可以没有名称，这种结构体长得很像元组，因此被称为元组结构体，例如：

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

元组结构体在你希望有一个整体名称，但是又不关心里面字段的名称时将非常有用。例如上面的 Point 元组结构体，众所周知 3D 点是 (x, y, z) 形式的坐标点，因此我们无需再为内部的字段逐一命名为：x, y, z。

### 枚举

枚举类型是一个类型，它会包含所有可能的枚举成员, 而枚举值是该类型中的具体某个成员的实例。

```rust
enum PokerCard {
    Clubs(u8),
    Spades(u8),
    Diamonds(u8),
    Hearts(u8),
}

fn main() {
   let c1 = PokerCard::Spades(5);
   let c2 = PokerCard::Diamonds(13);
}
```

直接将数据信息关联到枚举成员上，省去近一半的代码，这种实现是不是更优雅？

不仅如此，同一个枚举类型下的不同成员还能持有不同的数据类型，例如让某些花色打印 1-13 的字样，另外的花色打印上 A-K 的字样：

```rust
enum PokerCard {
    Clubs(u8),
    Spades(u8),
    Diamonds(char),
    Hearts(char),
}

fn main() {
   let c1 = PokerCard::Spades(5);
   let c2 = PokerCard::Diamonds('A');
}
```

任何类型的数据都可以放入枚举成员中: 例如字符串、数值、结构体甚至另一个枚举。

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let m1 = Message::Quit;
    let m2 = Message::Move{x:1,y:1};
    let m3 = Message::ChangeColor(255,255,0);
}
```

### Option 枚举用于处理空值

Rust 吸取了众多教训，决定抛弃 null，而改为使用 Option 枚举变量来表述这种结果。

Option 枚举包含两个成员，一个成员表示含有值：Some(T), 另一个表示没有值：None，定义如下：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

其中 T 是泛型参数，Some(T)表示该枚举成员的数据类型是 T，换句话说，Some 可以包含任何类型的数据。

```rust
let some_number = Some(5);
let some_string = Some("a string");

let absent_number: Option<i32> = None;
```

如果使用 None 而不是 Some，需要告诉 Rust Option<T> 是什么类型的，因为编译器只通过 None 值无法推断出 Some 成员保存的值的类型。

当有一个 Some 值时，我们就知道存在一个值，而这个值保存在 Some 中。当有个 None 值时，在某种意义上，它跟空值具有相同的意义：并没有一个有效的值。那么，Option<T> 为什么就比空值要好呢？

简而言之，因为 Option<T> 和 T（这里 T 可以是任何类型）是不同的类型，例如，这段代码不能编译，因为它尝试将 Option<i8>(Option<T>) 与 i8(T) 相加：

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);

let sum = x + y;
```

在对 Option<T> 进行 T 的运算之前必须将其转换为 T。通常这能帮助我们捕获到空值最常见的问题之一：期望某值不为空但实际上为空的情况。

为了拥有一个可能为空的值，你必须要显式的将其放入对应类型的 Option<T> 中。接着，当使用这个值时，必须明确的处理值为空的情况。只要一个值不是 Option<T> 类型，你就 可以 安全的认定它的值不为空。这是 Rust 的一个经过深思熟虑的设计决策，来限制空值的泛滥以增加 Rust 代码的安全性。

### 数组

在日常开发中，使用最广的数据结构之一就是数组，在 Rust 中，最常用的数组有两种，第一种是速度很快但是长度固定的 array，第二种是可动态增长的但是有性能损耗的 Vector，在本书中，我们称 array 为数组，Vector 为动态数组。

由于它的元素类型大小固定，且长度也是固定，因此数组 array 是存储在栈上，性能也会非常优秀。与此对应，动态数组 Vector 是存储在堆上，因此长度可以动态改变。当你不确定是使用数组还是动态数组时，那就应该使用后者。

在一些时候，还需要为数组声明类型，如下所示：

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

这里，数组类型是通过方括号语法声明，i32 是元素类型，分号后面的数字 5 是数组长度，数组类型也从侧面说明了数组的元素类型要统一，长度要固定。

还可以使用下面的语法初始化一个某个值重复出现 N 次的数组：

```rust
let a = [3; 5];
```

a 数组包含 5 个元素，这些元素的初始化值为 3，聪明的读者已经发现，这种语法跟数组类型的声明语法其实是保持一致的：[3; 5] 和 [类型; 长度]。

因为数组是连续存放元素的，因此可以通过索引的方式来访问存放其中的元素：
例如a[0].

## 流程控制

有同学可能会想到，如果我们想用 for 循环控制某个过程执行 10 次，但是又不想单独声明一个变量来控制这个流程，该怎么写？

```rust
for _ in 0..10 {
  // ...
}
```

可以用 _ 来替代 i 用于 for 循环中，在 Rust 中 _ 的含义是忽略该值或者类型的意思，如果不使用 _，那么编译器会给你一个 变量未使用的 的警告。

两种循环方式优劣对比

以下代码，使用了两种循环方式：

```rust
// 第一种
let collection = [1, 2, 3, 4, 5];
for i in 0..collection.len() {
  let item = collection[i];
  // ...
}

// 第二种
for item in collection {

}
```

第一种方式是循环索引，然后通过索引下标去访问集合，第二种方式是直接循环集合中的元素，优劣如下：

1. 性能：第一种使用方式中 collection[index] 的索引访问，会因为边界检查(Bounds Checking)导致运行时的性能损耗 —— Rust 会检查并确认 index 是否落在集合内，但是第二种直接迭代的方式就不会触发这种检查，因为编译器会在编译时就完成分析并证明这种访问是合法的

2. 安全：第一种方式里对 collection 的索引访问是非连续的，存在一定可能性在两次访问之间，collection 发生了变化，导致脏数据产生。而第二种直接迭代的方式是连续访问，因此不存在这种风险（这里是因为所有权吗？是的话可能要强调一下）
3. 
由于 for 循环无需任何条件限制，也不需要通过索引来访问，因此是最安全也是最常用的，通过与下面的 while 的对比，我们能看到为什么 for 会更加安全。

loop(无条件循环)

```rust
fn main() {
    let mut n = 0;

    loop {
        if n > 5 {
            break
        }
        println!("{}",n);
        n+=1;
    }

    println!("我出来了！");
}
```

## 模式匹配

### match

先来看一个关于 match 的简单例子：

```rust
enum Direction {
    East,
    West,
    North,
    South,
}

fn main() {
    let dire = Direction::South;
    match dire {
        Direction::East => println!("East"),
        Direction::North | Direction::South => {
            println!("South or North");
        },
        _ => println!("West"),
    };
}
```

这里我们想去匹配 dire 对应的枚举类型，因此在 match 中用三个匹配分支来完全覆盖枚举变量 Direction 的所有成员类型，有以下几点值得注意：

1. match 的匹配必须要穷举出所有可能，因此这里用 _ 来代表未列出的所有可能性
2. match 的每一个分支都必须是一个表达式，且所有分支的表达式最终返回值的类型必须相同
3. X | Y，类似逻辑运算符 或，代表该分支可以匹配 X 也可以匹配 Y，只要满足一个即可

其实 match 跟其他语言中的 switch 非常像，_ 类似于 switch 中的 default.

match 的通用形式：

```rust
match target {
    模式1 => 表达式1,
    模式2 => {
        语句1;
        语句2;
        表达式2
    },
    _ => 表达式3
}
```

### if let 匹配

有时会遇到只有一个模式的值需要被处理，其它值直接忽略的场景，

我们只想要对 Some(3) 模式进行匹配, 不想处理任何其他 Some<u8> 值或 None 值。但是为了满足 match 表达式（穷尽性）的要求，写代码时必须在处理完这唯一的成员后加上 _ => ()，这样会增加不少无用的代码。

杀鸡焉用牛刀，可以用 if let 的方式来实现：

```rust
if let Some(3) = v {
    println!("three");
}
```

这两种匹配对于新手来说，可能有些难以抉择，但是只要记住一点就好：当你只要匹配一个条件，且忽略其他条件时就用 if let ，否则都用 match。

### matches!宏

Rust 标准库中提供了一个非常实用的宏：matches!，它可以将一个表达式跟模式进行匹配，然后返回匹配的结果 true or false。

```rust
let foo = 'f';
assert!(matches!(foo, 'A'..='Z' | 'a'..='z'));

let bar = Some(4);
assert!(matches!(bar, Some(x) if x > 2));
```
### 匹配Option<T>

使用 Option<T>，是为了从 Some 中取出其内部的 T 值以及处理没有值的情况，为了演示这一点，下面一起来编写一个函数，它获取一个 Option<i32>，如果其中含有一个值，将其加一；如果其中没有值，则函数返回 None 值：

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

plus_one 接受一个 Option<i32> 类型的参数，同时返回一个 Option<i32> 类型的值(这种形式的函数在标准库内随处所见)，在该函数的内部处理中，如果传入的是一个 None ，则返回一个 None 且不做任何处理；如果传入的是一个 Some(i32)，则通过模式绑定，把其中的值绑定到变量 i 上，然后返回 i+1 的值，同时用 Some 进行包裹。


### 模式适用场景

```rust
while let 条件循环：

// Vec是动态数组
let mut stack = Vec::new();

// 向数组尾部插入元素
stack.push(1);
stack.push(2);
stack.push(3);

// stack.pop从数组尾部弹出元素
while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

对于以下代码，编译器会报错：

```rust
let Some(x) = some_option_value;
```

因为右边的值可能不为 Some，而是 None，这种时候就不能进行匹配，也就是上面的代码遗漏了 None 的匹配。

类似 let 和 for、match 都必须要求完全覆盖匹配，才能通过编译。

但是对于 if let，就可以这样使用：

```rust
if let Some(x) = some_option_value {
    println!("{}", x);
}
```

因为 if let 允许匹配一种模式，而忽略其余的模式.

## 方法Method

### 定义方法

Rust 使用 impl 来定义方法，例如以下代码：

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    // new是Circle的关联函数，因为它的第一个参数不是self
    // 这种方法往往用于初始化当前结构体的实例
    fn new(x: f64, y: f64, radius: f64) -> Circle {
        Circle {
            x: x,
            y: y,
            radius: radius,
        }
    }

    // Circle的方法，&self表示借用当前的Circle结构体
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}
```

其它语言中所有定义都在 class 中，但是 Rust 的对象定义和方法定义是分离的，这种数据和使用分离的方式，会给予使用者极高的灵活度。

在一个 impl 块内，Self 指代被实现方法的结构体类型，self 指代此类型的实例，换句话说，self 指代的是 Rectangle 结构体实例，这样的写法会让我们的代码简洁很多，而且非常便于理解：我们为哪个结构体实现方法，那么 self 就是指代哪个结构体的实例。

需要注意的是，self 依然有所有权的概念：

1. self 表示 Rectangle 的所有权转移到该方法中，这种形式用的较少
2. &self 表示该方法对 Rectangle 的不可变借用
3. &mut self 表示可变借用

### 关联函数

接受几个参数，然后构造并返回该结构体的实例:

这种定义在 impl 中且没有 self 的函数被称之为关联函数： 因为它没有 self，不能用 f.read() 的形式调用，因此它是一个函数而不是方法，它又在impl 中，与结构体紧密关联，因此称为关联函数。

因为是函数，所以不能用 . 的方式来调用，我们需要用::来调用，例如 let sq = Rectangle::new(3,3);。这个方法位于结构体的命名空间中：:: 语法用于关联函数和模块创建的命名空间。

### 多个 impl 定义

Rust 允许我们为一个结构体定义多个 impl 块，目的是提供更多的灵活性和代码组织性，例如当方法多了后，可以把相关的方法组织在同一个 impl 块中，那么就可以形成多个 impl 块，各自完成一块儿目标

## 泛型与特征

### 泛型Generics

 T 就是泛型参数，实际上在 Rust 中，泛型参数的名称你可以任意起，但是出于惯例，我们都用 T ( T 是 type 的首字母)来作为首选，这个名称越短越好，除非需要表达含义，否则一个字母是最完美的。

 使用泛型参数，有一个先决条件，必需在使用前对其进行声明：

```rust
fn largest<T>(list: &[T]) -> T {
```

该泛型函数的作用是从列表中找出最大的值，其中列表中的元素类型为 T。首先 largest<T> 对泛型参数 T 进行了声明，然后才在函数参数中进行使用该泛型参数 list: &[T] （还记得 &[T] 类型吧？这是数组切片）。

总之，我们可以这样理解这个函数定义：函数 largest 有泛型类型 T，它有个参数 list，其类型是元素为 T 的数组切片，最后，该函数返回值的类型也是 T。

不是所有 T 类型都能进行相加操作，因此我们需要用 std::ops::Add<Output = T> 对 T 进行限制：

```rust
fn add<T: std::ops::Add<Output = T>>(a:T, b:T) -> T {
    a + b
}
```

进行如上修改后，就可以正常运行。

### 结构体中使用泛型

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```
这里有两点需要特别的注意：

1. 提前声明，跟泛型函数定义类似，首先我们在使用泛型参数之前必需要进行声明 Point<T>，接着就可以在结构体的字段类型中使用 T 来替代具体的类型
2. x 和 y 是相同的类型

如果想让 x 和 y 即能类型相同，又能类型不同，需要使用不同的泛型参数：

```rust
struct Point<T,U> {
    x: T,
    y: U,
}
fn main() {
    let p = Point{x: 1, y :1.1};
}
```

### 枚举中使用泛型

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

这个枚举和 Option 一样，主要用于函数返回值，与 Option 用于值的存在与否不同，Result 关注的主要是值的正确性。

### 方法中使用泛型

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```

使用泛型参数前，依然需要提前声明：impl<T>，只有提前声明了，我们才能在Point<T>中使用它，这样 Rust 就知道 Point 的尖括号中的类型是泛型而不是具体类型。需要注意的是，这里的 Point<T> 不再是泛型声明，而是一个完整的结构体类型，因为我们定义的结构体就是 Point<T> 而不再是 Point.


为具体的泛型类型实现方法:

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

这段代码意味着 Point<f32> 类型会有一个方法 distance_from_origin，而其他 T 不是 f32 类型的 Point<T> 实例则没有定义此方法。


### const 泛型

const 泛型，也就是针对值的泛型，

```rust
fn display_array<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(arr);

    let arr: [i32; 2] = [1, 2];
    display_array(arr);
}
```

如上所示，我们定义了一个类型为 [T; N] 的数组，其中 T 是一个基于类型的泛型参数，这个和之前讲的泛型没有区别，而重点在于 N 这个泛型参数，它是一个基于值的泛型参数！因为它用来替代的是数组的长度。

N 就是 const 泛型，定义的语法是 const N: usize，表示 const 泛型 N ，它基于的值类型是 usize。

### 泛型的性能

Rust 通过在编译时进行泛型代码的 单态化(monomorphization)来保证效率。单态化是一个通过填充编译时使用的具体类型，将通用代码转换为特定代码的过程。

### 特征 Trait

特征定义了一个可以被共享的行为，只要实现了特征，你就能使用该行为。

如果不同的类型具有相同的行为，那么我们就可以定义一个特征，然后为这些类型实现该特征。定义特征是把一些方法组合在一起，目的是定义一个实现某些目标所必需的行为的集合。

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

这里使用 trait 关键字来声明一个特征，Summary 是特征名。在大括号中定义了该特征的所有方法，在这个例子中是： fn summarize(&self) -> String。

特征只定义行为看起来是什么样的，而不定义行为具体是怎么样的。因此，我们只定义特征方法的签名，而不进行实现，此时方法签名结尾是 ;，而不是一个 {}。

接下来，每一个实现这个特征的类型都需要具体实现该特征的相应方法，编译器也会确保任何实现 Summary 特征的类型都拥有与这个签名的定义完全一致的 summarize 方法。

### 为类型实现特征

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
pub struct Post {
    pub title: String, // 标题
    pub author: String, // 作者
    pub content: String, // 内容
}

impl Summary for Post {
    fn summarize(&self) -> String {
        format!("文章{}, 作者是{}", self.title, self.author)
    }
}

pub struct Weibo {
    pub username: String,
    pub content: String
}

impl Summary for Weibo {
    fn summarize(&self) -> String {
        format!("{}发表了微博{}", self.username, self.content)
    }
}
```

实现特征的语法与为结构体、枚举实现方法很像：impl Summary for Post，读作“为 Post 类型实现 Summary 特征”，然后在 impl 的花括号中实现该特征的具体方法。

特征定义与实现的位置(孤儿规则):

关于特征实现与定义的位置，有一条非常重要的原则：如果你想要为类型 A 实现特征 T，那么 A 或者 T 至少有一个是在当前作用域中定义的！例如我们可以为上面的 Post 类型实现标准库中的 Display 特征，这是因为 Post 类型定义在当前的作用域中。同时，我们也可以在当前包中为 String 类型实现 Summary 特征，因为 Summary 定义在当前作用域中。

但是你无法在当前作用域中，为 String 类型实现 Display 特征，因为它们俩都定义在标准库中，其定义所在的位置都不在当前作用域，跟你半毛钱关系都没有，看看就行了。

该规则被称为**孤儿规则**，可以确保其它人编写的代码不会破坏你的代码，也确保了你不会莫名其妙就破坏了风马牛不相及的代码。


### 使用特征作为函数参数

现在，先定义一个函数，使用特征用做函数参数：

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

impl Summary，只能说想出这个类型的人真的是起名鬼才，简直太贴切了，故名思义，它的意思是 实现了Summary特征 的 item 参数。

你可以使用任何实现了 Summary 特征的类型作为该函数的参数，同时在函数体内，还可以调用该特征的方法，例如 summarize 方法。

### 特征约束(trait bound)

虽然 impl Trait 这种语法非常好理解，但是实际上它只是一个语法糖：

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

真正的完整书写形式如上所述，形如 T: Summary 被称为特征约束。

### 多重约束

除了单个约束条件，我们还可以指定多个约束条件，例如除了让参数实现 Summary 特征外，还可以让参数实现 Display 特征以控制它的格式化输出：

```rust
pub fn notify(item: &(impl Summary + Display)) {
```

除了上述的语法糖形式，还能使用特征约束的形式：

```rust
pub fn notify<T: Summary + Display>(item: &T) {}
```

通过这两个特征，就可以使用 item.summarize 方法，以及通过 println!("{}", item) 来格式化输出 item。

### Where 约束

当特征约束变得很多时，函数的签名将变得很复杂：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {
```

严格来说，上面的例子还是不够复杂，但是我们还是能对其做一些形式上的改进，通过 where：

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

### 函数返回中的 impl Trait

可以通过 impl Trait 来说明一个函数返回了一个类型，该类型实现了某个特征：

```rust
fn returns_summarizable() -> impl Summary {
    Weibo {
        username: String::from("sunface"),
        content: String::from(
            "m1 max太厉害了，电脑再也不会卡",
        )
    }
}
```

因为 Weibo 实现了 Summary，因此这里可以用它来作为返回值。要注意的是，虽然我们知道这里是一个 Weibo 类型，但是对于 returns_summarizable 的调用者而言，他只知道返回了一个实现了 Summary 特征的对象，但是并不知道返回了一个 Weibo 类型。

### 通过 derive 派生特征

在本书中，形如 #[derive(Debug)] 的代码已经出现了很多次，这种是一种特征派生语法，被 derive 标记的对象会自动实现对应的默认特征代码，继承相应的功能。

例如 Debug 特征，它有一套自动实现的默认代码，当你给一个结构体标记后，就可以使用 println!("{:?}", s) 的形式打印该结构体的对象。

再如 Copy 特征，它也有一套自动实现的默认代码，当标记到一个类型上时，可以让这个类型自动实现 Copy 特征，进而可以调用 copy 方法，进行自我复制。

总之，derive 派生出来的是 Rust 默认给我们提供的特征，在开发过程中极大的简化了自己手动实现相应特征的需求，当然，如果你有特殊的需求，还可以自己手动重载该实现。

### 特征对象

```rust
pub trait Draw {
    fn draw(&self);
}
```

只要组件实现了 Draw 特征，就可以调用 draw 方法来进行渲染。假设有一个 Button 和 SelectBox 组件实现了 Draw 特征：

@TODO

### Self 与 self

在 Rust 中，有两个self，一个指代当前的实例对象，一个指代特征或者方法类型的别名：

```rust
trait Draw {
    fn draw(&self) -> Self;
}

#[derive(Clone)]
struct Button;
impl Draw for Button {
    fn draw(&self) -> Self {
        return self.clone()
    }
}

fn main() {
    let button = Button;
    let newb = button.draw();
}
```

上述代码中，self指代的就是当前的实例对象，也就是 button.draw() 中的 button 实例，Self 则指代的是 Button 类型。

当理解了 self 与 Self 的区别后，我们再来看看何为对象安全。

### 特征对象的限制

@TODO

### 特征进阶

@TODO

## 集合类型

### Vector

动态数组允许你存储多个值，这些值在内存中一个紧挨着另一个排列，因此访问其中某个元素的成本非常低。动态数组只能存储相同类型的元素，如果你想存储不同类型的元素，可以使用之前讲过的枚举类型或者特征对象。

使用 Vec::new 创建动态数组是最 rusty 的方式，它调用了 Vec 中的 new 关联函数：

```rust
let v: Vec<i32> = Vec::new();
```

还可以使用宏 vec! 来创建数组，与 Vec::new 有所不同，前者能在创建同时给予初始化值：

```rust
let v = vec![1, 2, 3];
```

同样，此处的 v 也无需标注类型，编译器只需检查它内部的元素即可自动推导出 v 的类型是 Vec<i32>

从 Vector 中读取元素
读取指定位置的元素有两种方式可选：

通过下标索引访问。
使用 get 方法。

```rust
let v = vec![1, 2, 3, 4, 5];

let third: &i32 = &v[2];
println!("第三个元素是 {}", third);

match v.get(2) {
    Some(third) => println!("第三个元素是 {}", third),
    None => println!("去你的第三个元素，根本没有！"),
}
```

和其它语言一样，集合类型的索引下标都是从 0 开始，&v[2] 表示借用 v 中的第三个元素，最终会获得该元素的引用。而 v.get(2) 也是访问第三个元素，但是有所不同的是，它返回了 Option<&T>，因此还需要额外的 match 来匹配解构出具体的值。

下标索引与 .get 的区别:

这两种方式都能成功的读取到指定的数组元素，既然如此为什么会存在两种方法？何况 .get 还会增加使用复杂度，让我们通过示例说明：

```rust
let v = vec![1, 2, 3, 4, 5];

let does_not_exist = &v[100];
let does_not_exist = v.get(100);
```

运行以上代码，&v[100] 的访问方式会导致程序无情报错退出，因为发生了数组越界访问。 但是 v.get 就不会，它在内部做了处理，有值的时候返回 Some(T)，无值的时候返回 None，因此 v.get 的使用方式非常安全。

### 迭代遍历 Vector 中的元素

如果想要依次访问数组中的元素，可以使用迭代的方式去遍历数组，这种方式比用下标的方式去遍历数组更安全也更高效（每次下标访问都会触发数组边界检查）：

```rust
let v = vec![1, 2, 3];
for i in &v {
    println!("{}", i);
}
```

也可以在迭代过程中，修改 Vector 中的元素：

```rust
let mut v = vec![1, 2, 3];
for i in &mut v {
    *i += 10
}
```

### 存储不同类型的元素

通过使用枚举类型和特征对象来实现不同类型元素的存储。先来看看通过枚举如何实现：

```rust
#[derive(Debug)]
enum IpAddr {
    V4(String),
    V6(String)
}
fn main() {
    let v = vec![
        IpAddr::V4("127.0.0.1".to_string()),
        IpAddr::V6("::1".to_string())
    ];

    for ip in v {
        show_addr(ip)
    }
}

fn show_addr(ip: IpAddr) {
    println!("{:?}",ip);
}
```

数组 v 中存储了两种不同的 ip 地址，但是这两种都属于 IpAddr 枚举类型的成员，因此可以存储在数组中。

再来看看特征对象的实现：

```rust
trait IpAddr {
    fn display(&self);
}

struct V4(String);
impl IpAddr for V4 {
    fn display(&self) {
        println!("ipv4: {:?}",self.0)
    }
}
struct V6(String);
impl IpAddr for V6 {
    fn display(&self) {
        println!("ipv6: {:?}",self.0)
    }
}

fn main() {
    let v: Vec<Box<dyn IpAddr>> = vec![
        Box::new(V4("127.0.0.1".to_string())),
        Box::new(V6("::1".to_string())),
    ];

    for ip in v {
        ip.display();
    }
}
```

比枚举实现要稍微复杂一些，我们为 V4 和 V6 都实现了特征 IpAddr，然后将它俩的实例用 Box::new 包裹后，存在了数组 v 中，需要注意的是，这里必需手动的指定类型：Vec<Box<dyn IpAddr>>，表示数组 v 存储的是特征 IpAddr 的对象，这样就实现了在数组中存储不同的类型。

在实际使用场景中，特征对象数组要比枚举数组常见很多，主要原因在于特征对象非常灵活，而编译器对枚举的限制较多，且无法动态增加类型。

### HashMap

@TODO

## 类型转换

### 内存地址转换为指针

Rust 中内置了一些基本类型之间的转换，这里使用 as 操作符来完成:

```rust
let mut values: [i32; 2] = [1, 2];
let p1: *mut i32 = values.as_mut_ptr();
let first_address = p1 as usize; // 将p1内存地址转换为一个整数
let second_address = first_address + 4; // 4 == std::mem::size_of::<i32>()，i32类型占用4个字节，因此将内存地址 + 4
let p2 = second_address as *mut i32; // 访问该地址指向的下一个整数p2
unsafe {
    *p2 += 1;
}
assert_eq!(values[1], 3);
```

### TryInto

在一些场景中，使用 as 关键字会有比较大的限制。如果你想要在类型转换上拥有完全的控制而不依赖内置的转换，例如处理转换错误，那么可以使用 TryInto ：

```rust
use std::convert::TryInto;

fn main() {
   let a: u8 = 10;
   let b: u16 = 1500;

   let b_: u8 = b.try_into().unwrap();

   if a < b_ {
     println!("Ten is less than one hundred.");
   }
}
```

最主要的是 try_into 转换会捕获大类型向小类型转换时导致的溢出错误：

```rust
fn main() {
    let b: i16 = 1500;

    let b_: u8 = match b.try_into() {
        Ok(b1) => b1,
        Err(e) => {
            println!("{:?}", e.to_string());
            0
        }
    };
}
```

运行后输出如下 "out of range integral type conversion attempted"，在这里我们程序捕获了错误，编译器告诉我们类型范围超出的转换是不被允许的，因为我们试图把 1500_i16 转换为 u8 类型，后者明显不足以承载这么大的值。

## 返回值和错误处理

### panic! 与不可恢复错误

既然是不可恢复错误，那么一旦发生，只需让程序崩溃即可。对此，Rust 为我们提供了 panic! 宏，当调用执行该宏时，程序会打印出一个错误信息，展开报错点往前的函数调用堆栈，最后退出程序。

Result<T, E> 这个枚举类型，它是用来表示函数的返回结果：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

当没有错误发生时，函数返回一个用 Result 类型包裹的值 Ok(T)，当错误时，返回一个 Err(E)。对于 Result 返回我们有很多处理方法，最简单粗暴的就是 unwrap 和 expect，这两个函数非常类似，我们以 unwrap 举例：

```rust
use std::net::IpAddr;
let home: IpAddr = "127.0.0.1".parse().unwrap();
```

上面的 parse 方法试图将字符串 "127.0.0.1" 解析为一个 IP 地址类型 IpAddr，它返回一个 Result<IpAddr, E> 类型，如果解析成功，则把 Ok(IpAddr) 中的值赋给 home，如果失败，则不处理 Err(E)，而是直接 panic。

因此 unwrap 简而言之：成功则返回值，失败则 panic，总之不进行任何错误处理.

需要快速地搭建代码，错误处理会拖慢编码的速度，也不是特别有必要，因此通过 unwrap、expect 等方法来处理是最快的。

同时，当我们回头准备做错误处理时，可以全局搜索这些方法，不遗漏地进行替换。

### 可恢复的错误 Result

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("Problem opening the file: {:?}", error)
        },
    };
}
```

代码很清晰，对打开文件后的 Result<T, E> 类型进行匹配取值，如果是成功，则将 Ok(file) 中存放的的文件句柄 file 赋值给 f，如果失败，则将 Err(error) 中存放的错误信息 error 使用 panic 抛出来，进而结束程序，这非常符合上文提到过的 panic 使用场景。

直接 panic 还是过于粗暴，因为实际上 IO 的错误有很多种，我们需要对部分错误进行特殊处理，而不是所有错误都直接崩溃：

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => panic!("Problem opening the file: {:?}", other_error),
        },
    };
}
```

上面代码在匹配出 error 后，又对 error 进行了详细的匹配解析，最终结果：

1. 如果是文件不存在错误 ErrorKind::NotFound，就创建文件，这里创建文件File::create 也是返回 Result，因此继续用 match 对其结果进行处理：创建成功，将新的文件句柄赋值给 f，如果失败，则 panic
2. 剩下的错误，一律 panic.


在不需要处理错误的场景，例如写原型、示例时，我们不想使用 match 去匹配 Result<T, E> 以获取其中的 T 值，因为 match 的穷尽匹配特性，你总要去处理下 Err 分支。那么有没有办法简化这个过程？有，答案就是 unwrap 和 expect。

它们的作用就是，如果返回成功，就将 Ok(T) 中的值取出来，如果失败，就直接 panic，真的勇士绝不多 BB，直接崩溃。

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").unwrap();
}
```

expect 跟 unwrap 很像，也是遇到错误直接 panic, 但是会带上自定义的错误提示信息，相当于重载了错误打印的函数：

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

### 传播错误

例如以下函数从文件中读取用户名，然后将结果进行返回：

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    // 打开文件，f是`Result<文件句柄,io::Error>`
    let f = File::open("hello.txt");

    let mut f = match f {
        // 打开文件成功，将file句柄赋值给f
        Ok(file) => file,
        // 打开文件失败，将错误返回(向上传播)
        Err(e) => return Err(e),
    };
    // 创建动态字符串s
    let mut s = String::new();
    // 从f文件句柄读取数据并写入s中
    match f.read_to_string(&mut s) {
        // 读取成功，返回Ok封装的字符串
        Ok(_) => Ok(s),
        // 将错误向上传播
        Err(e) => Err(e),
    }
}
```

有几点值得注意：

1. 该函数返回一个 Result<String, io::Error> 类型，当读取用户名成功时，返回 Ok(String)，失败时，返回 Err(io:Error)
2. File::open 和 f.read_to_string 返回的 Result<T, E> 中的 E 就是 io::Error

可以使用？来简化代码：

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

其实 ? 就是一个宏，它的作用跟上面的 match 几乎一模一样：

```rust
let mut f = match f {
    // 打开文件成功，将file句柄赋值给f
    Ok(file) => file,
    // 打开文件失败，将错误返回(向上传播)
    Err(e) => return Err(e),
};
```

如果结果是 Ok(T)，则把 T 赋值给 f，如果结果是 Err(E)，则返回该错误，所以 ? 特别适合用来传播错误。

虽然 ? 和 match 功能一致，但是事实上 ? 会更胜一筹.

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```

? 还能实现链式调用，File::open 遇到错误就返回，没有错误就将 Ok 中的值取出来用于下一个方法调用，简直太精妙了.

```rust
use std::fs;
use std::io;

fn read_username_from_file() -> Result<String, io::Error> {
    // read_to_string是定义在std::io中的方法，因此需要在上面进行引用
    fs::read_to_string("hello.txt")
}
```

从文件读取数据到字符串中，是比较常见的操作，因此 Rust 标准库为我们提供了 fs::read_to_string 函数，该函数内部会打开一个文件、创建 String、读取文件内容最后写入字符串并返回.

### ? 用于 Option 的返回:

? 不仅仅可以用于 Result 的传播，还能用于 Option 的传播，再来回忆下 Option 的定义：

Result 通过 ? 返回错误，那么 Option 就通过 ? 返回 None：

```rust
fn first(arr: &[i32]) -> Option<&i32> {
   let v = arr.get(0)?;
   Some(v)
}
```

上面的函数中，arr.get 返回一个 Option<&i32> 类型，因为 ? 的使用，如果 get 的结果是 None，则直接返回 None，如果是 Some(&i32)，则把里面的值赋给 v。

其实这个函数有些画蛇添足，我们完全可以写出更简单的版本：

```rust
fn first(arr: &[i32]) -> Option<&i32> {
   arr.get(0)
}
```

新手用 ? 常会犯的错误:

初学者在用 ? 时，老是会犯错，例如写出这样的代码：


```rust
fn first(arr: &[i32]) -> Option<&i32> {
   arr.get(0)?
}
```

这段代码无法通过编译，切记：? 操作符需要一个变量来承载正确的值，这个函数只会返回 Some(&i32) 或者 None，只有错误值能直接返回，正确的值不行，所以如果数组中存在 0 号元素，那么函数第二行使用 ? 后的返回类型为 &i32 而不是 Some(&i32)。

## 包和模块

二进制 Package

让我们来创建一个二进制 Package：

```bash
$ cargo new my-project
     Created binary (application) `my-project` package
$ ls my-project
Cargo.toml
src
$ ls my-project/src
main.rs
```

这里，Cargo 为我们创建了一个名称是 my-project 的 Package，同时在其中创建了 Cargo.toml 文件，可以看一下该文件，里面并没有提到 src/main.rs 作为程序的入口，原因是 Cargo 有一个惯例：src/main.rs 是二进制包的根文件，该二进制包的包名跟所属 Package 相同，在这里都是 my-project，所有的代码执行都从该文件中的 fn main() 函数开始。

库 Package
再来创建一个库类型的 Package：

```bash
$ cargo new my-lib --lib
     Created library `my-lib` package
$ ls my-lib
Cargo.toml
src
$ ls my-lib/src
lib.rs
```

首先，如果你试图运行 my-lib，会报错：

```bash
$ cargo run
error: a bin target must be available for `cargo run`
```

原因是库类型的 Package 只能作为三方库被其它项目引用，而不能独立运行，只有之前的二进制 Package 才可以运行。

与 src/main.rs 一样，Cargo 知道，如果一个 Package 包含有 src/lib.rs，意味它包含有一个库类型的同名包 my-lib，该包的根文件是 src/lib.rs。

一个真实项目中典型的 Package，会包含多个二进制包，这些包文件被放在 src/bin 目录下，每一个文件都是独立的二进制包，同时也会包含一个库包，该包只能存在一个 src/lib.rs：

```
.
├── Cargo.toml
├── Cargo.lock
├── src
│   ├── main.rs
│   ├── lib.rs
│   └── bin
│       └── main1.rs
│       └── main2.rs
├── tests
│   └── some_integration_tests.rs
├── benches
│   └── simple_bench.rs
└── examples
    └── simple_example.rs
```

1. 唯一库包：src/lib.rs
2. 默认二进制包：src/main.rs，编译后生成的可执行文件与 Package 同名
3. 其余二进制包：src/bin/main1.rs 和 src/bin/main2.rs，它们会分别生成一个文件同名的二进制可执行文件
4. 集成测试文件：tests 目录下
5. 基准性能测试 benchmark 文件：benches 目录下
6. 项目示例：examples 目录下

### 模块 Module

创建嵌套模块:

使用 cargo new --lib restaurant 创建一个小餐馆，注意，这里创建的是一个库类型的 Package，然后将以下代码放入 src/lib.rs 中：

```rust
// 餐厅前厅，用于吃饭
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

以上的代码创建了三个模块，有几点需要注意的：

1. 使用 mod 关键字来创建新模块，后面紧跟着模块名称
2. 模块可以嵌套，这里嵌套的原因是招待客人和服务都发生在前厅，因此我们的代码模拟了真实场景
3. 模块中可以定义各种 Rust 类型，例如函数、结构体、枚举、特征等
4. 所有模块均定义在同一个文件中

用路径引用模块：

想要调用一个函数，就需要知道它的路径，在 Rust 中，这种路径有两种形式：

绝对路径，从包根开始，路径名以包名或者 crate 作为开头
相对路径，从当前模块开始，以 self，super 或当前模块的标识符作为开头
让我们继续经营那个惨淡的小餐馆，这次为它实现一个小功能： 文件名：src/lib.rs

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();

    // 相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

上面的代码为了简化实现，省去了其余模块和函数，这样可以把关注点放在函数调用上。eat_at_restaurant 是一个定义在包根中的函数，在该函数中使用了两种方式对 add_to_waitlist 进行调用。

### 使用 super 引用模块

super 代表的是父模块为开始的引用方式，非常类似于文件系统中的 .. 语法：../a/b 文件名：src/lib.rs

```rust
fn serve_order() {}

// 厨房模块
mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```

### 结构体和枚举的可见性

1. 将结构体设置为 pub，但它的所有字段依然是私有的
2. 将枚举设置为 pub，它的所有字段也将对外可见
   
原因在于，枚举和结构体的使用方式不一样。如果枚举的成员对外不可见，那该枚举将一点用都没有，因此枚举成员的可见性自动跟枚举可见性保持一致，这样可以简化用户的使用。

而结构体的应用场景比较复杂，其中的字段也往往部分在 A 处被使用，部分在 B 处被使用，因此无法确定成员的可见性，那索性就设置为全部不可见，将选择权交给程序员。

### 模块与文件分离

现在，把 front_of_house 前厅分离出来，放入一个单独的文件中 src/front_of_house.rs：

```rust
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

然后，将以下代码留在 src/lib.rs 中：

```rust
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

1. mod front_of_house; 告诉 Rust 从另一个和模块 front_of_house 同名的文件中加载该模块的内容
2. 使用绝对路径的方式来引用 hosting 模块：crate::front_of_house::hosting;

需要注意的是，和之前代码中 mod front_of_house{..} 的完整模块不同，现在的代码中，模块的声明和实现是分离的，实现是在单独的 front_of_house.rs 文件中，然后通过 mod front_of_house; 这条声明语句从该文件中把模块内容加载进来。因此我们可以认为，模块 front_of_house 的定义还是在 src/lib.rs 中，只不过模块的具体内容被移动到了 src/front_of_house.rs 文件中。

在这里出现了一个新的关键字 use，该关键字用来将外部模块中的项引入到当前作用域中来，这样无需冗长的父模块前缀即可调用：hosting::add_to_waitlist().

### as 别名引用

对于同名冲突问题，还可以使用 as 关键字来解决，它可以赋予引入项一个全新的名称：

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

### 受限的可见性

在 Rust 中，包是一个模块树，我们可以通过 pub(crate) item; 这种方式来实现：item 虽然是对外可见的，但是只在当前包内可见，外部包无法引用到该 item。

所以，如果我们想要让某一项可以在整个包中都可以被使用，那么有两种办法：

1. 在包根中定义一个非 pub 类型的 X(父模块的项对子模块都是可见的，因此包根中的项对模块树上的所有模块都可见)
2. 在子模块中定义一个 pub 类型的 Y，同时通过 use 将其引入到包根

```rust
mod a {
    pub mod b {
        pub fn c() {
            println!("{:?}",crate::X);
        }

        #[derive(Debug)]
        pub struct Y;
    }
}

#[derive(Debug)]
struct X;
use a::b::Y;
fn d() {
    println!("{:?}",Y);
}
```

通过 pub(in crate::a) 的方式，我们指定了模块 c 和常量 J 的可见范围都只是 a 模块中，a 之外的模块是完全访问不到它们的。如下：

```rust
pub mod a {
    pub const I: i32 = 3;

    fn semisecret(x: i32) -> i32 {
        use self::b::c::J;
        x + J
    }

    pub fn bar(z: i32) -> i32 {
        semisecret(I) * z
    }
    pub fn foo(y: i32) -> i32 {
        semisecret(I) + y
    }

    mod b {
        pub(in crate::a) mod c {
            pub(in crate::a) const J: i32 = 4;
        }
    }
}
```

pub(crate) 或 pub(in crate::a) 就是限制可见性语法，前者是限制在整个包内可见，后者是通过绝对路径，限制在包内的某个模块内可见，总结一下：

1. pub 意味着可见性无任何限制
2. pub(crate) 表示在当前包可见
3. pub(self) 在当前模块可见
4. pub(super) 在父模块可见
5. pub(in <path>) 表示在某个路径代表的模块中可见，其中 path 必须是父模块或者祖先模块

## 注释和文档 

@TODO

## 格式化输出

print!，println!，format!：

1. print! 将格式化文本输出到标准输出，不带换行符
2. println! 同上，但是在行的末尾添加换行符
3. format! 将格式化文本输出到 String 字符串


在实际项目中，最常用的是 println! 及 format!，前者常用来调试输出，后者常用来生成格式化的字符串：

```rust
fn main() {
    let s = "hello";
    println!("{}, world", s);
    let s1 = format!("{}, world", s);
    print!("{}", s1);
    print!("{}\n", "!");
}
```

eprint!，eprintln!输出到标准错误输出：

```rust
eprintln!("Error: Could not complete task")
```

与 {} 类似，{:?} 也是占位符：

1. {} 适用于实现了 std::fmt::Display 特征的类型，用来以更优雅、更友好的方式格式化文本，例如展示给用户
2. {:?} 适用于实现了 std::fmt::Debug 特征的类型，用于调试场景

其实两者的选择很简单，当你在写代码需要调试时，使用 {:?}，剩下的场景，选择 {}

### Display 特征

与大部分类型实现了 Debug 不同，实现了 Display 特征的 Rust 类型并没有那么多，往往需要我们自定义想要的格式化方式.

为自定义类型实现 Display 特征：

```rust
struct Person {
    name: String,
    age: u8,
}

use std::fmt;
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(
            f,
            "test,{},{}",
            self.name, self.age
        )
    }
}
fn main() {
    let p = Person {
        name: "sunface".to_string(),
        age: 18,
    };
    println!("{}", p);
}
```

### 指定位置参数:

除了按照依次顺序使用值去替换占位符之外，还能让指定位置的参数去替换某个占位符，例如 {1}，表示用第二个参数替换该占位符(索引从 0 开始)：

```rust
fn main() {
    println!("{}{}", 1, 2); // =>"12"
    println!("{1}{0}", 1, 2); // =>"21"
    // => Alice, this is Bob. Bob, this is Alice
    println!("{0}, this is {1}. {1}, this is {0}", "Alice", "Bob");
    println!("{1}{}{0}{}", 1, 2); // => 2112
}
```

### 带名称的变量:

除了像上面那样指定位置外，我们还可以为参数指定名称：

```rust
fn main() {
    println!("{argument}", argument = "test"); // => "test"
    println!("{name} {}", 1, name = 2); // => "2 1"
    println!("{a} {c} {b}", a = "a", b = 'b', c = 3); // => "a 3 b"
}
```

需要注意的是：带名称的参数必须放在不带名称参数的后面，

### 格式化参数

格式化输出，意味着对输出格式会有更多的要求，例如只输出浮点数的小数点后两位：

```rust
fn main() {
    let v = 3.1415926;
    // Display => 3.14
    println!("{:.2}", v);
    // Debug => 3.14
    println!("{:.2?}", v);
}
```

### Rust 中的格式化参数

宽度:

字符串格式化默认使用空格进行填充，并且进行左对齐。

```rust
fn main() {
    //-----------------------------------
    // 以下全部输出 "Hello x    !"
    // 为"x"后面填充空格，补齐宽度5
    println!("Hello {:5}!", "x");
    // 使用参数5来指定宽度
    println!("Hello {:1$}!", "x", 5);
    // 使用x作为占位符输出内容，同时使用5作为宽度
    println!("Hello {1:0$}!", 5, "x");
    // 使用有名称的参数作为宽度
    println!("Hello {:width$}!", "x", width = 5);
    //-----------------------------------

    // 使用参数5为参数x指定宽度，同时在结尾输出参数5 => Hello x    !5
    println!("Hello {:1$}!{}", "x", 5);
}
```

数字格式化默认也是使用空格进行填充，但与字符串左对齐不同的是，数字是右对齐。

```rust
fn main() {
    // 宽度是5 => Hello     5!
    println!("Hello {:5}!", 5);
    // 显式的输出正号 => Hello +5!
    println!("Hello {:+}!", 5);
    // 宽度5，使用0进行填充 => Hello 00005!
    println!("Hello {:05}!", 5);
    // 负号也要占用一位宽度 => Hello -0005!
    println!("Hello {:05}!", -5);
}
```

对齐

```rust
fn main() {
    // 以下全部都会补齐5个字符的长度
    // 左对齐 => Hello x    !
    println!("Hello {:<5}!", "x");
    // 右对齐 => Hello     x
    println!("Hello {:>5}!", "x");
    // 居中对齐 => Hello   x  !
    println!("Hello {:^5}!", "x");

    // 对齐并使用指定符号填充 => Hello x&&&&!
    // 指定符号填充的前提条件是必须有对齐字符
    println!("Hello {:&<5}!", "x");
}
```

进制
可以使用 # 号来控制数字的进制输出：

1. #b, 二进制
2. #o, 八进制
3. #x, 小写十六进制
4. #X, 大写十六进制
5. x, 不带前缀的小写十六进制

```rust
fn main() {
    // 二进制 => 0b11011!
    println!("{:#b}!", 27);
    // 八进制 => 0o33!
    println!("{:#o}!", 27);
    // 十进制 => 27!
    println!("{}!", 27);
    // 小写十六进制 => 0x1b!
    println!("{:#x}!", 27);
    // 大写十六进制 => 0x1B!
    println!("{:#X}!", 27);

    // 不带前缀的十六进制 => 1b!
    println!("{:x}!", 27);

    // 使用0填充二进制，宽度为10 => 0b00011011!
    println!("{:#010b}!", 27);
}
```

指数:

```rust
fn main() {
    println!("{:2e}", 1000000000); // => 1e9
    println!("{:2E}", 1000000000); // => 1E9
}
```

指针地址:

```rust
let v= vec![1, 2, 3];
println!("{:p}", v.as_ptr()) // => 0x600002324050
```

### 在格式化字符串时捕获环境中的值（Rust 1.58 新增）

```rust
fn get_person() -> String {
    String::from("sunface")
}
fn main() {
    let person = get_person();
    println!("Hello, {person}!");
}
```

甚至还可以将环境中的值用于格式化参数:

```rust
let (width, precision) = get_format();
for (name, score) in get_scores() {
  println!("{name}: {score:width$.precision$}");
}
```