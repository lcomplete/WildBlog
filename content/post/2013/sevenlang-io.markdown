---
layout: post
title: "七周七语言之用 Io 编写领域特定语言"
date: 2013-06-05T05:56:08+08:00
comments: true
tags: ["Io", "编程", "编程语言"]
categories: ["技术"]
---

Io 语言是在 2002 年创造出来的，虽然距今已经有 11 个年头了，但是对于一门编程语言来说，它还只能算是一门年轻的语言。Io 并不是主流编程语言，没有什么名气，就连名字取的也并不适合推广，io 有太多其他的含义了，用 google 直接搜索 io 的话，很难找到关于这门编程语言的资料，英文推荐搜索“io language”，中文则建议搜索“io 编程语言”。

## Io 的优缺点

Io 是个小众语言，文档不足，这是其中的一个缺点，但是这门语言吸收了许多语言的特点，具有很高的学习价值。Io 的设计借鉴了 SmallTalk、Self、NewtonScript、Act1、LISP 和 Lua 的一些特点，它是一门基于原型式设计的编程语言，具有面向对象特性，语法概念极少，它没有关键字，所以能够在很短的时间内熟悉语法，Io 还有出色的并发库，利用它能够以简单直观的方式编写并发程序。

Io 有很少的语法规则和语言特性，这似乎在某种程度上束缚了它的能力，但是使用 Io 可以很容易地扩展这门语言自身的语法，我们可以自定义运算符或改变运算符的行为，也可以使用元编程在任何时候改变任何对象的行为。正是这些高扩展性和可定制性使得 Io 在创造**领域特定语言**（Domain Specific Language，简称 DSL）方面具有非常强大的能力，我们可以利用它创造出自己喜欢的语法，编写出**能够编写程序的程序**，学习 Io 颇具启发性，怎样利用 Io 打造出能够提高编程效率的 DSL 更是一个值得深入研究的课题。

## 什么是领域特定语言？

Martin Flower 在《领域特定语言》一书中给出的定义是：针对某一特定语言，具有受限表达性的一种计算机程序设计语言。DSL 不是图灵完备的，所以它的表达能力有限；由于它是领域特定的，它的出现往往是为了消除业务领域的复杂性，所以 DSL 一般是整洁的、易于使用且接近自然语言的。我们平时在编码的过程中其实会接触到很多领域特定语言，比如 CSS、HTML、XML、JSON、正则表达式、XAML，他们都是针对某一领域的，“求专不求全”，CSS 用于定义 Web 页面样式；HTML 用于定义 Web 界面元素；XML、JSON 用于描述数据，这些数据可以作为纯数据，也可作为某种行为的配置。

DSL 从实现角度分为内部 DSL 和外部 DSL，用 XML 来描述某种行为就是一种常见的外部 DSL 的形式，利用编程语言自带的语法结构定义出来的 DSL 称为内部 DSL，比如时下流行的流畅接口（Fluent Interface）编码方式就是一种内部 DSL。上面说到 Io 编程语言可以自定义运算符且具有元编程特性，因此 Io 可以方便的创造出内部 DSL，下面我们就通过几个示例来看一看 Io 在这方面的能力。

## 使用 Io 创造内部 DSL

先看一个比较简单的例子，可以说在所有的编程语言中都有列表和字典（一些语言中也称为映射）这两种数据结构，在 io 中也不例外，io 中创建列表对象可以使用 list 函数，比如 foo := list(1,2) ，这很方便，然而创建一个字典的写法却比较繁琐，你需要 clone 一个 Map 对象，然后依次调用 atPut 方法向其中添加键值对，作为一个常用的数据结构，这样写不仅会影响效率而且会使代码变的臃肿，纵观其他编程语言，大部分都有相应的字典初始化语法糖，常用的写法为 {"key":"value"} 结构，下面我们就在 io 中来创造这种语法糖。

```ruby dict.io
OperatorTable addAssignOperator(":", "atPutNumber")

Map atPutNumber := method(
  self atPut(
    call evalArgAt(0) asMutable removePrefix("\"") removeSuffix("\""),
    call evalArgAt(1)
  )
)

curlyBrackets := method(
  data := Map clone
  call message arguments foreach(arg,
    data doMessage(arg))
  data
)

squareBrackets := method(
    arr := list()
    call message arguments foreach(arg,
        arr push(call sender doMessage(arg))
    )
    arr
)

doFile("dict_sample.io")
```

先简单说明一下上面这段代码，OperatorTable 中保存了可用的操作符号表，我们可以使用 addAssignOperator 方法向其中添加用于赋值的操作符号，并将其映射到第二个参数所指定的方法中，第一行代码的意思为将 : 符号映射到 atPutNumber 方法中，当在代码中碰到 **:** 符号时，将调用 **atPutNumber** 方法；**curlyBrackets**、**squareBrackets** 为 io 中的特殊变量，将方法赋值给它们相当于改变 {} 和 [] 符号的行为，在我们的代码中，指定了当碰到 { 符号时初始化 Map 对象，接着循环处理参数，这里的参数是用 , 符号分段的，data doMessage 方法相当于以 data 为上下文执行参数中的代码，参数代码中若碰到 : 符号则会调用 atPutNumber 方法，将符号左边的参数当作键，右边的参数当作值存入 data 中，最后返回 data。

注意，最后一行的 **doFile** 方法是执行另一文件中的代码的意思，为什么这个例子要分成两个文件呢？主要是因为虽然我们在操作符号表里面添加了一个符号，但是在这个文件的执行上下文中，是不会起效果的，只有调用 doString 或 doFile 才会起效，所以将创建字典的代码放在了下面这个文件中。

```ruby dict_sample.io
dict := {
  "name": "lcomplete",
  "array": [1, "str"],
  "nested": {
    "hello" : "io"
  }
}
```

很简洁吧，而且还支持嵌套定义哦。

通过第一个例子，相信各位已经能看出 io 在创造内部 dsl 上的灵活性了，接下来我们看一个更强大的例子，使用 io 让编写 html 变的更有禅意，何谓禅意，其实是受到 Sublime 上的 ZenCode 插件的启发，使用 ZenCode 语法可以快速的编写 html 片段。这次我们先看一下使用 dsl 的方式。

```ruby enhance_html.io
zencode(Html html>head+body>div!content$id@class*3)
```

不难发现，这个语法已经跟宿主语言几乎没有任何关系，是一门彻底的内部 dsl，上面这段代码生成的 html 如下所示：

```html
<html>
  <head> </head>
  <body>
    <div id="id" class="class">content</div>
    <div id="id" class="class">content</div>
    <div id="id" class="class">content</div>
  </body>
</html>
```

很强大吧，一行代码就可以生成出这么多的 html，现在我们再看看这个 dsl 是如何定义出来的。

```ruby zencode.io
forward := method(
    call message name
)

root := nil

Html := Object clone do(
    OperatorTable addOperator("@", 15)
    OperatorTable addOperator("$", 15)
    OperatorTable addOperator(">", 15)
    OperatorTable addOperator("+", 15)
    OperatorTable addOperator("*", 15)
    OperatorTable addOperator("!", 15)

    @ := method(classname,
        attrs append(" class=\"#{classname}\"" interpolate)
        self
    )

    $ := method(id,
        attrs append(" id=\"#{id}\"" interpolate)
        self
    )

    > := method(tag,
        html := Html clone
        html tag := tag
        html parent := self
        self childs append(html)
        html
    )

    + := method(tag,
        html := nil
        if(self parent,
            html = Html clone
            html tag := tag
            self parent childs append(html)
        )
        html
    )

    * := method(count,
        self count := count asNumber
        self
    )

    ! := method(content,
        self content := content
        self
    )

    forward := method(
        name := call message name
        html := Html clone
        html tag := name
        root = html
        html
    )

    init := method(
        self childs := list()
        self attrs := list()
        self count := 1
        self parent := nil
        self content := nil
    )
)

Html flush := method(
    root render("")
)

Html render := method(indent,
    self count repeat(
        writeln(indent,"<",self tag,self attrs join(""),">")
        childIndent := indent .. "  "
        if(self content,
            writeln(childIndent,self content)
        )
        self childs foreach(index,arg,
            arg render(childIndent)
        )
        writeln(indent,"</",self tag,">")
    )
)


zencode := method(html,
    html flush
)

doFile("enhance_html.io")
```

有几处需要说明一下，forward 是一个特殊方法，相当于 ruby 中的 method_missing ，当调用的方法不存在时，系统将会把调用转向到 forward 方法，这就给了我们很大的灵活性，我们无需定义 html、head 等方法，这些未定义的方法会自动转向到 forward 中处理；注意，一共有两个 forward 方法，分别应用在 Object 和 Html 对象上，只有 Html html 会返回 Html 对象，其他的未定义方法会转向到 Object 的 forward 方法上，这里直接将方法名作为字符串返回，新添加的 @、$ 等操作符，将会把接受到的字符串进行处理，最后再返回一个 Html 对象以进行链式调用。

## 总结

在 Io 中可以轻松地定义操作符、实现动态方法或更改所有对象的行为，这些能力是强大的，也是危险的，使用之前需要仔细衡量。当然，如果只是用它来做某一件事情，比如用来编写上面的 ZenCode 时，那么可以大胆发挥自己的想象力，在它的基础上构建一门属于自己的语言。
