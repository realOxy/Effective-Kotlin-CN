# Item 1: 限制可变性
在Kotlin中，我们使用模块（modules）设计程序，他们都由不同的元素组成，诸如类（classes）、对象（objects）、函数（functions），类型别名（type aliases）以及顶级属性（top-level properties）。有些元素可以持有状态值，例如持有可读写的属性`var`或者可变对象：
 
```kotlin
var a = 10
val list: MutableList<Int> = mutableListOf()
```
当元素持有状态时，他的行为不仅取决于你如何使用它，还有它的历史。一个典型的例子，带有状态的类是一个有一些货币余额的银行账户：

```kotlin
class BankAccount {
    var balance = 0.0
    	private set
    
    fun deposit(depositAmount: Double) {
        balance += depositAmount
    }
    
    @Throws(InsufficientFunds::class)
    fun withdraw(withdrawAmount: Double) {
        if (balance < withdrawAmount) {
            throw InsufficientFunds()
        }
        balance -= withdrawAmount
    }
}

class InsufficientFunds: Exception()
```

```kotlin
val account = BankAccount()
println(account.balance) // 0.0
account.deposit(100.0)
println(account.balance) // 100.0
account.withdraw(50.0)
println(account.balance) // 50.0
```
这里的`BankAccount`拥有一个代表当前账户中有多少余额的状态值。持有状态是一把双刃剑（double-edged sword）。一方面他非常有用，因为它可以表示随时间变化的元素，但另一方面表示状态管理却很难，因为：
    
1. 大量的可变点会使程序更加难以理解和调试。这些可变点之间的关系需要被理解清楚，并且很多时候我们难以追踪它们是怎样改变的。拥有许多相互依赖的可变点的类往往很难理解和调整。在出现意外或者是抛出异常的时候尤其成问题。

2. 使得对代码的推理变得更加困难。不可变元素的状态很清楚，可变状态更难理解。很难推断它的值是多少，因为它随时可能发生变化，因为我们仅仅某个时刻检查了并不意味着它仍然是一样的。

3. 在多线程程序中进行适当的同步。 每一个可变点都有潜在的冲突。

