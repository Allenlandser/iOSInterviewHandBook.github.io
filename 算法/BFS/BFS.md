# BFS（低频）

BFS（广度优先搜索）在具体的面试中问到的概率较低。一般来说，能够使用BFS来解决的问题，都能够使用DFS来解决，唯一的例外就是BFS能够搜索**最短路径**。所以如果题目要求需要搜索最短路径时，那么我们则要优先考虑BFS。下面我们来先来看一看BFS的模板，然后看看如何利用模板来解决一些高频问题。

## BFS模板

和之在`Tree`一节中提到的递归三要素一样，BFS也是有模板可循，我们来看一下BFS的解题框架：

``` swift
func bfs() {
  // some work to do before BFS
  var visited: Set<Node> = [] // 一个set，用于记录所有已经搜索过的node，具体到不同的题目可能有不同的数据结构
  var queue: [Node] = [] // 一个先进先出的队列FIFO，用于记录所有待检查的node。
  queue.append(startNode) //将初始node放入队列中
  while !queue.isEmpty { // 当我们在待检查队列中还有元素时
    let currentNode = queue.removeFirst()
		// check the currentNode
    visited.insert(currentNode) // 将当前检查的node标记为『已检查』
    
    // 循环当前节点的所有邻接节点，如果该邻接节点还没有被检查，则将其加入到待检查的队列中
    for next in currentNode.nextNodes {
      if !visited.contains(next) {
        queue.append(next)
      }
    }
  }
	// some work to do after BFS
  return result // 返回搜索的结果
}
```

以上就是基本的BFS解题框架和思路了，如果记住上面的这个模板的话，在解一般的BFS的题目时，我们就简单地套用就可以了。下面我们来看一看一些面试中出现的高频例题，顺便看看如何使用这个模板。

## [Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/)(Google电面)

这道题目很有意思，我们完全可以套用上面的模板来解题，不过需要考虑两点：

* 因为是`Tree`的`level order`遍历，所以我们不需要使用一个`set`来记录搜索过的节点，因为已经遍历过的节点在之后的遍历中我们不会二度检查。

* 第二，因为是『之』字形的遍历，所以一个比较符合直觉的想法就是在遍历完当前层的节点后，将下一层未遍历的节点做一个`reverse`操作。如此循环，我们就能得到需要的结果。但是，这样我们就需要额外的操作，最终会使时间复杂度翻倍，显然这是一个可以优化的解法。

  但是这个思路是正确的，显然，我们的模板中使用的是一个先进先出的队列，而上一个方法中的`reverse`操作，也是为了能够从队列的末尾优先取出节点。那么，我们只要修改逻辑，让这个队列变为双向队列，不就能够解决我们的需求了吗？归纳一下，『之』字形遍历的核心逻辑可以归纳为：

  > * 当前层为**偶数层**， 我们从队列的**开头**取出节点，并**先左后右**地从队列的**末尾**加入下一层的节点。
  > * 当前层为**奇数层**，我们从队列的末尾取出节点，并**先右后左**的从队列的**开头**加入下一层的节点。

有了核心逻辑和上面的模板，我们来看看具体的代码：

``` swift
 func zigzagLevelOrder(_ root: TreeNode?) -> [[Int]] {
    var res: [[Int]] = []
    guard let root = root else { return res }

    var nodes: [TreeNode] = [] // 双向队列

    nodes.append(root)

    var level = 0

    while !nodes.isEmpty {
        var currVal: [Int] = []
        let size = nodes.count // 当前层的节点数目
        if level % 2 == 0 { // 偶数层
            for _ in 0 ..< size {
                let curr = nodes.removeFirst()
                currVal.append(curr.val)
                if let leftNode = curr.left {
                    nodes.append(leftNode)
                }
                if let rightNode = curr.right {
                    nodes.append(rightNode)
                }
            }
        } else {
            for _ in 0 ..< size {
                let curr = nodes.removeLast()
                currVal.append(curr.val)
                if let rightNode = curr.right {
                    nodes.insert(rightNode, at: 0)
                }
                if let leftNode = curr.left {
                    nodes.insert(leftNode, at: 0)
                }
            }
        }
        level += 1
        res.append(currVal)
    }
    return res
}
```

大家可以和上面的模板进行一下对比，主要多了这几行：

``` swift
let size = nodes.count // 当前层的节点数目
for _ in 0 ..< size {
  ...
}
```

这也是树的层级遍历的核心代码之一，在别的BFS中遇到的概率较小。

## [Number of Islands](https://leetcode.com/problems/number-of-islands/)

这也是一道高频题，LeetCode上类似的题目有很多，解法也有很多，包括DFS， BFS，Union-Find。不过在这一节中，我们来讨论一下BFS的解法。

这道题目核心逻辑十分简单：

> 我们遍历整个二维数组，每当我们碰到一个内有被检查过的『1』时，我们就从这个坐标开始，将所有相连的『1』都标记为`visited`，并且将`island counter`加1。而这个搜索的过程，既可以用BFS，也可以用DFS，这里我们直接套用上面的模板就可以了。

有了核心逻辑，直接上代码：

```swift
let dirs: [(Int, Int)] = [(-1, 0), (1, 0), (0, -1), (0, 1)]

func numIslands(_ grid: [[Character]]) -> Int {
    if grid == nil || grid.count == 0 { return 0 }

    var grid = grid
    var islandCounter = 0
    let n = grid.count
    let m = grid[0].count

    for row in 0..<n {
        for col in 0..<m {
            if grid[row][col] == "1" {
                bfsSearchIsland(row, col, &grid)
                islandCounter += 1
            }
        }
    }

    return islandCounter
}
    
func bfsSearchIsland(_ row: Int, _ col: Int, _ grid: inout [[Character]]) {
    var queue = [(row: Int, col: Int)]()
    queue.append((row: row, col: col))

    while !queue.isEmpty {
        let currPos = queue.removeFirst()
        for dir in dirs {
            let newPos = (row: currPos.row + dir.0, col: currPos.col + dir.1)
            if newPos.row >= 0 
                && newPos.row < grid.count 
                && newPos.col >= 0 
                && newPos.col < grid[0].count 
                && grid[newPos.row][newPos.col] == "1" {
                	queue.append(newPos)
               		grid[newPos.row][newPos.col] = "-1" //将其修改为『1』以外的任何值都可以, 目的是为了确保检查过的节点不会再进入queue中
            }
        }
    }
}
```



`bfsSearchIsland`中就是核心的BFS代码，可以看到，代码逻辑和上面的模板没有本质的区别。

## [Open the Lock](https://leetcode.com/problems/open-the-lock/)

这道题目也是一道非常有趣的BFS题。如果我们直接套用上面的模板，在Leetcode上会抱LTE（超时）的错误。这里就需要使用另外的一个技巧——**双向BFS**。

> 所谓的双向BFS，就是我们同时从开始点和结束点开始我们的BFS搜索，直到两个搜索相遇。
>
> 双向BFS的最大优势就在于能够对搜索树进行极大的剪枝，极大地优化算法的时间复杂度和空间复杂度。

我们直接来看代码：

```swift
func openLock(_ deadends: [String], _ target: String) -> Int {
    var start: Set<String> = [] // 从起始数字开始搜索，即"0000"， 将其作为set能够加快双向BFS相遇检测的速度
    var end: Set<String> = [] // 从traget数字开始搜索
    let deadends: Set<String> = Set(deadends)

    start.insert("0000")
    end.insert(target)

    var steps = 0

    while !start.isEmpty && !end.isEmpty {
        var temp: Set<String> = []
        for s in start {
            if end.contains(s) { return steps }
            if deadends.contains(s) { continue } // 如果当前的数字是deadend， 我们就不再检查该数字的后续数字
            for next in getNext(s) {
                if !deadends.contains(next) {
                    temp.insert(next)
                }
            }
        }
        steps += 1
      	// 交换start和end，下一轮搜索从另一端开始
        start = end 
        end = temp
    }

    return -1
}

private func getNext(_ s: String) -> Set<String> {
    var s: [Int] = Array(s).map{ return Int(String($0))! }
    var res: Set<String> = []

    for i in  0 ..< 4 {
        var t = s
        t[i] = (s[i] + 1) % 10
        res.insert(t.reduce("", { return $0 + String($1) }))
        t[i] = (s[i] - 1 + 10) % 10
        res.insert(t.reduce("", { return $0 + String($1) }))
    }
    return res
}
```



可以看到，即使使用了双向BFS，核心代码还是跳不过我们的模板。所以只要牢记我们的模板，一般的BFS都能够做出解答。

