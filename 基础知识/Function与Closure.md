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

对于Closure来说，Capture List是其中非常重要的知识点，因为它和Swift与Objective-C中的重点考察内容retain cycle有关。

与Objective-C不同，Swift的closure会自动捕获上下文中需要在closure中使用的变量，即使在原本的作用域中中这些变量已经不存在了。而这一个自动捕获的默认机制是使用Strong Reference来捕获这些变量的。当然，这也不是必然的，如果某个值(value)在closure的执行过程中没有改变，那么编译器会使用这个值(value)的拷贝而不是引用。

因为Capture List的默认机制是进行一个强引用，所以如果我们在一个类的实例中将一个Closure赋值给了它的某个属性，那么这么Closure就会自动捕获当前类的实例，这样我们就有了一个retain cycle。

这里我们来看另外的一个例子可能会造成问题的：

```
func presentDelayedConfirmation(in presenter: UIViewController) {
    let queue = DispatchQueue.main

    queue.asyncAfter(deadline: .now() + 3) {
        let alert = UIAlertController(
            title: "...",
            message: "...",
            preferredStyle: .alert
        )
        presenter.present(alert, animated: true)
    }
}
```

在这里虽然一眼看上去没有问题，但是在closure中执行`presenter.present(alert, animated: true)`时，我们可能会碰到`present`已经被移除了。虽然这不会引起严重的问题，但是我们也应该避免以上的这种状况。

当然，解决以上两种情况，我们只要使用`capture list`并且将其中的变量用`weak`来修饰就可以了；

``` swift
func presentDelayedConfirmation(in presenter: UIViewController) {
    let queue = DispatchQueue.main
    queue.asyncAfter(deadline: .now() + 3) { [weak presenter] in
        // 我们先检查presenter依旧在内存中，否则就直接返回
        guard let presenter = presenter else { return }

        let alert = UIAlertController(
            title: "...",
            message: "...",
            preferredStyle: .alert
        )

        presenter.present(alert, animated: true)
    }
}
```

下面我们来看看`[weak self]`。诚然，在`capture list`中使用`[weak self]`是没有大问题的，但是`[weak self]`也并不是必要的，比如下面的一个例子：

```swift
class ImageLoader {
    private let cache = Cache<URL, Image>()

    func loadImage(from url: URL,
                   then handler: @escaping (Result<Image, Error>) -> Void) {
        request(url) { [cache] result in
            do {
                let image = try result.decodedAsImage()
                cache.insert(image, forKey: url)
                handler(.success(image))
            } catch {
                handler(.failure(error))
            }
        }
    }
}
```

这里我们并没有直接使用`[weak self]`，而是使用通过`capture list`捕获了类中的`cache`属性，这样我们就避免了因为捕获了`self`而形成的`retain cycle`。

### Unowned关键字

`unowned`关键字和`weak`关键字类似，被`unowned`关键字修饰的变量不会形成`retain cycle。`唯一的不同就是`unowned`的和`optional`中的强制解析(Force Unwraping)有些类似：我们在使用被`unwoned`关键字修饰的变量时，不需要进行nil检查，我们可以将这个`weak-reference`的变量当做`non-optional`来使用，但是如果该引用的变量已经被释放了，在使用该变量的时候就会导致崩溃。

所以说如果我们确定在一个被捕捉的变量在`Closure`中调用时绝对不会是`nil`时，我们可以用`unowned`关键字来修饰它，来简化我们的代码。在实际应用中，因为`weak`能够覆盖`unowned`的情况，所以我们会优先使用`weak` 关键字，即便我们需要做额外的检查。

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

## @autoclosure

`autoclosure`关键字能够自动地为被当做参数传入的closure建立一个新的`wrap closure`。传入的代码片段并不会立刻执行，而是当我们显示地调用这个`closure`时才会执行。当它被执行时，它会调用被包装的`closure`并返回被包装的`closure`的返回值。这其实是一个简单的语法糖，能够让我们的代码更加简洁。来看一个例子：

``` swift
func isValid(_ lhs: Bool, _ rhs: @autoclosure () -> Bool) -> Bool {
    guard lhs else {
        return false
    }
    return rhs()
}
if isValid(numbers.count > 0, numbers.last == 10) {
    // perform some work
}
```

这里`rhs`中的代码不会立刻被执行，而是当`lhs`被返回值为`true`时才会执行`rhs`这个`closure`。

如果不使用`@autoclosure`的话，代码会变成这样：

``` swift
func isValid(_ lhs: Bool, _ rhs: () -> Bool) -> Bool {
    guard lhs else {
        return false
    }
    return rhs()
}
if isValid(numbers.count > 0, { numbers.last == 10 }) {
    // perform some work
}
```

或者

```swift
func isValid(_ lhs: Bool, _ rhs: Bool) -> Bool {
    guard lhs else {
        return false
    }
    return rhs
}
if isValid(numbers.count > 0, numbers.last == 10) {
    // perform some work
}
```

这样的话`numbers.last == 10`不论如何都会执行，这无疑会造成系统的浪费。而和上述例子类似，Swift的`Foundation`中的`&&`,`||`等操作符，都用了`@autoclosure`关键字。

