# Function与Closure

Swift作为function programming的语言，function与closure是其中非常重要的一环，里面涉及的知识点也非常广泛。相关的面试题也可以考的比较深入。下面我们来根据官方文档详细地解读一下其中比较重要的概念和知识点。

## First-Class Function

什么是First-class function？First-class funciton是指在Swift中，`function`和其他的类型，如基础类型`Int`，`Double`，`Bool`，`Tuple`或者高级类型`Struct`, `Class`一样，能够被当做参数传入其他的`function`中，被`function`返回，被赋值，被存储在其他的数据结构中。这一点`closure`也是一样的，Swift的`function`，甚至也是一种特殊的`closure`。根据官方文档：

- 全局函数(Global Function)是一个有名字但不会捕获任何值的闭包；
- 内嵌函数(Nested Function)是一个有名字且能从其上层函数捕获值的闭包；
- 闭包表达式是一个轻量级语法所写的可以捕获其上下文中常量或变量值的没有名字的闭包。

## Inout关键字

`inout`是Swift中常见的关键字，需要注意的是，inout的实现方法和其他语言中的`call by reference`是有本质的区别的，我们不能将其等同。

`inout` 的工作流程如下：

* 当function被调用的时候，`inout`参数的值被复制。
* 在function在执行时，复制的变量被修改。
* 当function结束时，将复制的变量赋值给原来的变量。

所以说，`inout`的工作方式和`call by reference`是you本质的区别的，我们可以看一下下面的代码：

``` swift
var myInteger: Int {
    get {
        print("Get")
        return 0
    }
    set {
        print("Set")
    }
}

func myFunction(count: inout Int) {
}

myFunction(count: &myInteger) 
```

在这里，我们会输出

> Get
>
> Set

因为`myInteger`是一个计算属性，在`copy-in`的时候会进行一次`get`调用，在`copy-out`时会进行一次`set`的调用。

不过需要注意的一点是，Swift的编译器在某些情况下会使用`call-by-reference`的方法来优化实现`inout`关键字，但是我们并不能知道编译器究竟会在何种情况下进行这个操作。所以我们并不能够将`inout`简单地理解为`call-by-reference`。考虑下面的一个例子：

```swift
func exampleFunc() -> () -> Void {
    var i = 0
    return increase(&i)
}

func increase(inout x:Int) -> () -> Void {
    return {
        x += 1
        return
    }
}

let closure = exampleFunc()
closure()
```

在上面的例子中，如果单纯地将`inout`理解为`call-by-reference`的话，变量`i`在`exampleFunc`调用结束之后就已经被释放了，之后再执行`x += 1`的时候就会出现错误。为了避免以上的情况，swift才引入了`copy-in-and-cop-out`的机制。

## Capture List

## Pure Function

纯函数(Pure Function)是function programming中的一个概念，定义如下：

> 纯函数是一种不会产生任何副作用的函数，它的执行也不依赖任何外界的状况。
>
> A function is considered *pure* when it doesn’t produce any side effects and when it doesn’t rely on any external state.

换句话说，纯函数在执行时只依赖它传入的数据，每次传入相同的数据，那么函数的输出就是相同的。

那为什么要在Swift中使用纯函数呢？

最大的优点就在于者能够极大地调高函数的复用率，降低编写测试的难度和提高线程的安全，这与Swift中提倡的优先使用值类型是一脉相承的。举一个例子

```swift
extension String {
    mutating func greetMe(_ str: String) {
        append(str)
    }
}

var name = "Writer"
name.greetMe(", Hey")
```

上面的这个function就不是一个`pure function`，因为在函数的调用的过程中，它修改了自身的值，这并不是一个好的习惯。在Swift中，我们更倾向于把它写成

``` swift
extension String {
    func greetMe(_ str: String) {
        return append(str)
    }
}
```

