# Enumerations

对于Swift的Enum来说，需要注意的知识点不多，下面来简单梳理一下。

## 关联值（associated value）和原始值（raw value）

我们可以定义 Swift 枚举来存储任意给定类型的关联值，如果需要的话不同枚举成员关联值的类型可以不同，例子如下：

```swift
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

var upcCode = Barcode.upc(8, 85909, 51226, 3)
var qrCode = .qrCode("ABCDEFGHIJKLMNOP")
```

而原始值（raw value）则是在定义枚举时，为每一个选项分配一个默认的值。例子如下：

```swift

enum ASCIIControlCharacter: Character {
    case tab = "\t"
    case lineFeed = "\n"
    case carriageReturn = "\r"
}
```

### 关联值（associated value）和原始值（raw value）有什么不同？

原始值与关联值*不同*。原始值是当你第一次定义枚举的时候，它们用来预先填充的值。关联值在你基于枚举成员的其中之一创建新的常量或变量时设定，并且在每次这么做的时候这些关联值可以是不同的。

## 枚举的遍历

我们只要让我们的enum实现`CaseIterable`协议，我们就能够像其他的`CollectionType`类似的语法来遍历所有的`case`。需要注意的是，这个枚举*不能*有关联值。例子如下：

``` swift
enum CompassDirection: CaseIterable {
    case north, south, east, west
}

print("There are \(CompassDirection.allCases.count) directions.")
// Prints "There are 4 directions."
let caseList = CompassDirection.allCases
                               .map({ "\($0)" })
                               .joined(separator: ", ")
// caseList == "north, south, east, west"
```

## if /guard case let

`if case let`是`swich case let`的语法糖（syntax sugar），可以让我们只想匹配其中`enumeration`中的一个`case`时。我们来看看例子：

``` swift
enum Media {
  case book(title: String, author: String, year: Int)
  case movie(title: String, director: String, year: Int)
  case website(urlString: String)
}

let m = Media.movie(title: "Captain America: Civil War", director: "Russo Brothers", year: 2016)

if case let Media.movie(title, _, _) = m {
  print("This is a movie named \(title)")
}
// This is a move named Captain America: Civil War
```

上面的代码与下面的代码是等价的

``` swift
...
switch m {
  case let Media.movie(title, _, _):
    print("This is a movie named \(title)")
  default: 
  	break
}
```

同理，`guard case let`也是类似的，这里就不举例子了，大家可以回忆一下`if let`和`guard let`。





