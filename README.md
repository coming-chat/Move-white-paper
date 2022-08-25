# Move 语言中文白皮书
## 翻译自： [Move language whiter paper](https://github.com/coming-chat/Move-white-paper/blob/main/Move%202020-05-26.pdf)

## Move: 一种使用可编程**resources**的语言

## 摘要
我们为 Libra 区块链[1][2] 提供了一种安全且灵活的编程语言，它的名字叫做Move。Move也是一种可执行的字节码语言，用于实现自定义交易和智能合约。

Move 的关键特性是能够使用受线性逻辑启发的语义定义自定义**resources**类型 [3]：**resources**永远不能被复制或隐式丢弃，只能在程序存储位置之间移动。这些安全保证由 Move 的类型系统静态强制执行。虽然有这些特殊保护，但不会局限resource这一类型的功能，它可以像传统的类型一样，可以存储在数据结构中，也可以作为参数传递。

**First-class resources**是一个非常笼统的概念，程序员不仅可以使用它来实现安全的数字资产，还可以编写正确的业务逻辑来包装资产以及对数值不同权限的访问和操作。
Move 的安全性和表现力使我们能够在 Move 中实现 Libra 协议的重要部分，包括 Libra 币、交易处理和验证者管理。
