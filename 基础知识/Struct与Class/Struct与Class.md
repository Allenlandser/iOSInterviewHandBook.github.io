# Struct与Class

结构体与类是Swift中比较重点的知识点，概念也比较多，下面我们来慢慢梳理一下。

## Sturct和Class的异同

### 相同点

- 定义属性用来存储值；
- 定义方法用于提供功能；
- 定义下标脚本用来允许使用下标语法访问值；
- 定义初始化器用于初始化状态；
- 可以被扩展来默认所没有的功能；
- 遵循协议来针对特定类型提供标准功能。

### 不同点

- 类能够继承允许一个类继承另一个类的特征;
- 类型转换（Type casting）允许你在运行检查和解释一个类实例的类型；
- `deinit`允许一个类实例释放任何其所被分配的资源；
- 引用计数（Reference counting）允许不止一个对类实例的引用。

### Struct和Class相比的优势

- 结构较小，适用于复制操作，相比于一个 class 的实例被多次引用在线程上更加安全；
- 无须担心内存 memory leak 或者多线程冲突问题。

## 值类型（Value Type)和引用类型（Reference Type）

在Swift中，除了Class是引用类型之外，其他的包括Struct，Enum等都是值类型，而Int，Float，Array，Set，Dict等都是用Struct来实现的。下面来看看两者的对比：

|               | Reference Type                                               | Value Type                                           |
| ------------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| 声明为`let`时 | 不能改变变量指向的实例，但是可以更改指向的实例存储的属性的值 | 不管实例中的属性被定义为`let`还是`var`，都不能被修改 |
| 比较时        | 使用`===`                                                    | 使用`==`                                             |
| 何时使用？    | 当我们想要实例被其他实例共享时                               | 当我们想要每一个实例相互独立，或者在多线程中使用时   |
| 变量指向？    | 变量指向内存中的存储指针的地址，该指针指向存储数据的地址     | 变量直接指向存储数据的地址                           |

下面我们来看一个Reference Type和Value Type混用的例子：

``` swift
class Person {
  var name: String = ""
}

struct Address {
  let person: Person
}

let person = Person()
person.name = "Allen"

let address = Address(person: person)

print(address.person.name)  
//"Allen"

address.person.name = "Jack"

print(address.person.name)
// "Jack"

address.person = Person()
// error: cannot assign to property: 'person' is a 'let' constant
```

这里我们可以看到，因为在`struct Address`中`person`被定义为`let`，所以改变其指向的实例会报错；但是因为`person`是`reference type`，即便是被`let`修饰，我们依旧可以修改它的存储的数据`name`。

## Copy-On-Write

`Copy-on-Write`是Swift对`Value Type`进行拷贝时的一种优化机制，尤其是对Collection进行拷贝的时候。

在对`Collection Type`进行拷贝的时候，Swift一开始会将被复制的变量和原本变量的的指针指向同一处内存地址。只有对被复制的变量进行修改时，Swift才会将数据进行拷贝，并将被复制的变量指向新的拷贝的地址。我们来看一个例子。

``` swift
var array1: [Int] = [1, 2, 3]
var array2: [Int] = array1

print(address(o: &array1))
print(address(o: &array2))
//105553118847712
//105553118847712

array2.append(2)
print(address(o: &array1))
print(address(o: &array2))
//105553118847712
//105553116870016


func address(o: UnsafePointer<Void>) -> Int {
  return unsafeBitCast(o, to: Int.self)
}
```

## 属性（Property）

### `lazy`关键字

对于Struct和Class的`Stored Property`来说，比较重要的概念就是`lazy`关键字，这也是面试时面试官非常喜欢问的一个问题，不管是Startup还是大公司。

根据定义，`lazy property`只有在初次被调用的时候才会初始化赋值，而不是在Struct和Class的实例初始化时。需要注意的是，`lazy property`只能被定义成`var`，因为只有对于常量（let）来说，它在实例的初始化结束前必须被赋值，所以并不能够被定义成`lazy`。

还有一点，`lazy`并**不是**线程安全的，如果被标记为 lazy 修饰符的属性同时被多个线程访问并且属性还没有被初始化，则无法保证属性只初始化一次。

### Computed Property

`Computed Property`并不储存值，相反地，它提供一个`get`和`set`方法来间接获取值和设置其他属性的值。

当然，我们也可以为`Computed Property`只定义一个`set`方法，这样我们会得到一个只读（`readonly`）的`Computed Property`。

最后需要注意的是，必须用 var 关键字定义`Computed Property`，因为它们的值不是固定的。 

### Property Observer

`Porperty Observer`会观察并对属性值的变化做出回应。每当一个属性的值被设置时，属性观察者都会被调用，即使这个值与该属性当前的值**相同**。Swift主要提供了两个方法：`willSet`和`didSet`。前者会在新的值被存储之前调用，后者则会在新的值被存储之后调用。一个简单的例子如下：

``` swift
var title: String {
  willSet {
    print("将标题从\(title)设置到\(newValue)")
  }
  didSet {
    print("已将标题从\(oldValue)设置到\(title)")
  }
}
```

下面是几点需要注意的地方：

* 在初始化方法中初对属性的赋值，以及在 willSet 和 didSet 中对属性的再次更改都不会触发`Property Observer`的调用。

* 将某一个属性传入被`inout`修饰的参数时，`willSet`和`didSet`一定会被调用，这和之前提到的`inout`关键字的实现原理有关
* 父类（`Super Class`）属性的 willSet 和 didSet 观察者会在子类（`Sub Class`）初始化器中设置时被调用。它们不会在类的父类初始化器调用中设置其自身属性时被调用。

### Global Property

`Global Property`是定义在任何函数、方法、闭包或者类型环境之外的变量。`Local Property`是定义在函数、方法或者闭包环境之中的变量。

`Global Property`可以是`Computed Property`，也能够为其添加`Property Observer`。

`Global Property`永远都是`lazy`的，但是不需要用`lazy`关键字修饰。

## 方法（Methods）

### mutating关键字

`mutating`关键字主要用于修饰Struct和Enum中定义的方法。因为在Swift中，Struct和Enum都是值类型，默认情况下不能修改其中的属性。但是Swift支持用`mutating`修饰的方法来修改Struc和Enum中存储的属性，甚至是实例本身。

``` swift
struct Point {
    var x = 0.0, y = 0.0
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        x += deltaX
        y += deltaY
    }
}
var somePoint = Point(x: 1.0, y: 1.0)
somePoint.moveBy(x: 2.0, y: 3.0)
print("The point is now at (\(somePoint.x), \(somePoint.y))")
// prints "The point is now at (3.0, 4.0)"
```

但是需要注意的是如果一个Struct的实例被定义为let，我们不能够调用`mutating`方法。

``` swift
let fixedPoint = Point(x: 3.0, y: 3.0)
fixedPoint.moveBy(x: 2.0, y: 3.0)
// cannot use mutating member on immutable value: 'fixedPoint' is a 'let' constant
```

而且被`mutating`方法修改的属性也不能是`let`

``` swift
struct Point {
    var x = 0.0
    let y = 0.0
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        x += deltaX
        y += deltaY
    }
}
// change 'let' to 'var' to make it mutable
```

上面也说到，`mutating`方法也可以修改自身实例

``` swift
struct Point {
    var x = 0.0, y = 0.0
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        self = Point(x: x + deltaX, y: y + deltaY)
    }
}
```

## 初始化(Initialization)

与`Objective-C`的`init`方法不同，`Swift`的`init`方法不返回值，它的主要功能是确保实例在初次使用前能够被正确地初始化。

### 两段式初始化

Swift 的类初始化是一个两段式过程。

* 在第一个阶段，我们需要为每一个属性分配一个初始值。一旦每个存储属性的初始状态被确定，初始化就会转入第二阶段。
* 在第二个阶段中，我们可以为类的实例订制属性的值。

两段式初始化过程的使用让初始化更加安全，同时在每个类的层级结构给与了完备的灵活性。两段式初始化过程可以防止属性值在初始化之前被访问，还可以防止属性值被另一个初始化器意外地赋予不同的值。

### Designated Initializer 与 Convenience Initializer（convenience关键字）

`Designated Initializer` 并不是`Swift`的特性，其他的语言也有类似的定义。但是Swift中显示地定义了`convenience`关键字，并将`Designated Initializer` 与 `Convenience Initializer`作为语言的特色包括在了其中。

我们先来看一下什么是`Designated Initializer`：

`Designated Initializer`是类的主要初始化器。`Designated Initializer`可以初始化所有那个类引用的属性并且调用合适的父类初始化器来继续这个初始化过程给父类链。每一个类必须有**至少**一个`Designated Initializer`，因为不管我们调用类的那一个`Initializer`，最终我们都必须要调用`Designated Initializer`来初始化类的实例。

与之对应的，`Convenience Initializer`则是用`convenience`关键字修饰的`Initializer`。我们可以在一个类中定义任意数目的`Convenience Initializer`来为类的初始化提供比较简单的接口，比如为一些参数定义预设值。需要注意的是，`Convenience Initializer` 并不是必须的。

关于`Designated Initializer`和`Convenience Initializer`以及父类的`Designated Initializer`的规则可以用下面的图来概括：

![pic1](./pic1.png)

简单来说，规则可以囊括为下面两点：

* `Convenience Initializer`必须调用自身类的`Designated Initializer`。

* 类的`Designated Initializer`必须调用父类的`Designated Initializer`。

### Failable Initializer

虽然说`Swift`的`init`方法不返回任何值，但是在某些情况下，比如用`rawValue`初始化`enum`时，我们可能会面临`rawValue`不符合任何一个`case`的情况，这个时候，我们就需要`enum`自动生成的`init`方法返回`nil`。而这也是`Swift`中的一个特殊的`init`方法，`Failable Initializer`。语法上，我们只要将`init`方法的签名改为`init?()`即可。

`Failable Initializer`创建了一个初始化类型的*可选*值。通过在`Failable Initializer`中写 `return nil` 语句，来表明`Failable Initializer`在何种情况下会触发初始化失败。但是，对于Swift来说，严格来讲，`Initializer`不会有返回值。相反，它们的角色是确保在初始化结束时， `self` 能够被正确初始化。虽然你写了 `return nil` 来触发初始化失败，但是不能使用 `return` 关键字来表示初始化成功了。同时，不能定义`Failable Initializer`和`None-Failable Initializer`为相同的形式参数类型和名称。

一个简单的例子：

``` swift
struct Animal {
    let species: String
    init?(species: String) {
        if species.isEmpty { return nil }
        self.species = species
    }
}
```

## deinit

关于deinit的需要注意的知识点不多，总结起来you一下几点：

* 大多数情况下ARC会为我们清理内存，如果需要额外的清理工作，就将这些工作放入到`deinit`中。
* 每个类当中只能有一个反初始化器。反初始化器不接收任何形式参数，也不需要写圆括号。
* 子类的`deinit`实现结束之后父类`deinit`的会被调用。父类的`deinit`总会被调用，就算子类没有定义`deinit`。