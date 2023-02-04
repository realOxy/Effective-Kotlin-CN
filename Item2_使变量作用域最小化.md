# Item 2: 使变量作用域最小化

当我们定义状态时，我们更希望通过以下方式收紧变量和属性的作用域：

- 使用局部变量代替属性
- 尽量在最窄的作用域使用变量。例如：如果某个变量仅在循环中使用，那么就该将其定义在循环内部

对象的作用域是程序中对象保持可见性的区域。在 Kotlin 中，这个区域通常是使用花括号创建的，我们能直接对外层作用域的对象进行访问。例如：
```kotlin
val a = 1
fun fizz() {
    val b = 2
    print(a + b)
}
val buzz = {
    val c = 3
    print(a + c)
}
// 此处我们能拿到 a，但是拿不到 b 和 c
```
上面的例子，在函数`fizz`和`buzz`的作用域中，我们能访问外层作用域的变量。但是在外层作用域中，我们无法访问那些定义在函数内部的变量。下面是关于如何限制变量作用域的例子：
```kotlin
// Bad
var user: User
for(i in users.indices) {
    user = users[i]
    print("User at $i is $user")
}

// Better
for (i in users.indices) {
    val user = users[i]
    print("User at $i is $user")
}

// Same variables scope, nicer syntax
for ((i, user) in users.withIndex()) {
    print("User at $i is $user")
}
```
第一个例子中，user变量不仅能被`for-循环`内部访问，还能被外部访问。在第二、三个例子中，我们明确限制了user变量的作用域只能控制在`for-循环`中。
同样，我们可能会遇到很多内部作用域（很可能是由lambda表达式中的lambda表达式创建的），最好在尽可能狭窄的范围内定义变量。

我们倾向于这种规则是有很多原因的，其中最重要的原因之一是：**收紧变量的作用域，我们的程序就更加简单从而更加易于跟踪和管理**。当我们分析代码的时候，我们需要弄清楚这个对象在这里代表着什么含义。要处理的对象越多，编程也就越难。你的软件越简单，它崩溃的几率就越小。这与我们更喜欢不可变属性或对象而不是可变属性或对象的原因相似。

**考虑到可变属性，当它们只能在较小的范围内修改时，更容易跟踪它们如何变化**。更容易对它们进行推理并改变它们的行为。

另一个问题是，**范围更广的变量可能会被其他开发人员过度使用**。例如，可以推断，如果使用变量来分配迭代中的下一个元素，则一旦循环完成，列表中最后一个元素应该保留在该变量中。这种推理可能导致可怕地滥用，例如在迭代后使用此变量对最后一个元素做一些事情。这真的很糟糕，因为另一个试图了解这个值的含义的开发人员需要理解整个推理。这将是一个不必要的并发症。

**无论变量是只读还是读写，我们总是更喜欢在定义变量时对其进行初始化**。不要强迫开发人员查看定义的位置。这可以通过流程控制结构实现，例如`if`,`when`,`try-catch`或用作表达式的Elvis运算符：
```kotlin
// Bad
val user: User
if (hasValue) {
    user = getValue()
} else {
    user = User()
}

// Better
val user: User = if(hasValue) {
    getValue()
} else {
    User()
}
```
如果我们需要设置多个属性，解构声明可以帮助我们：
```kotlin
// Bad
fun updateWeather(degrees: Int) {
    val description: String
    val color: Int
    if (degrees < 5) {
        description = "cold"
        color = Color.BLUE
    } else if (degrees < 23) {
        description = "mild"
        color = Color.YELLOW
    } else {
        description = "hot"
        color = Color.RED
    }
    // ...
}

// Better
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 5 -> "cold" to Color.BLUE
        degrees < 23 -> "mild" to Color.YELLOW
        else -> "hot" to Color.RED
    }
    // ...
}
```
最后，过宽的可变范围可能是危险的情况。让我们来描述一个常见的危险情况。

## 捕捉
在我讲授 Kotlin 协程时，我布置的练习之一是使用序列生成器实现埃拉托色尼筛法以查找质数。该算法在概念上很简单：
1. 我们拿来一个从2开始的自然数列表。 
2. 我们取其第一个，它是一个质数。
3. 从其余的数字中，我们删除第一个，然后我们过滤掉所有能被这个质数整除的数字。

该算法的一个非常简单的实现如下所示：
```kotlin
var numbers = (2..100).toList()
val primes = mutableListOf<Int>()
while (numbers.isNotEmpty()) {
    val prime = numbers.first()
    primes.add(prime)
    numbers = numbers.filter { it % prime != 0 }
}
print(primes) // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31,
// 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
```
挑战在于：让它产生一个可能无限的质数序列。如果你想挑战自己，现在就停下来尝试去实现它。

解决方案可以是这个样子：
```kotlin
val primes: Sequence<Int> = sequence { 
    var numbers = generateSequence(2) { it + 1 }
    while (true) { 
        val prime = numbers.first()
        yield(prime)
        numbers = numbers
            .drop(1)
            .filter { it % prime != 0 } 
    } 
}
print(primes.take(10).toList())
// [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```
几乎每个小组中都有一个人试图“优化”它，为了不在每个循环中创建变量，他/她都会将 `prime` 提取为可变变量：
```kotlin
val primes: Sequence<Int> = sequence { 
    var numbers = generateSequence(2) { it + 1 } 
    var prime: Int
    while (true) { 
        prime = numbers.first()
        yield(prime)
        numbers = numbers
            .drop(1)
            .filter { it % prime != 0 } 
    }
}
```
问题是这个实现不再能正常工作。这些是前 10 个产生的数字：
```kotlin
print(primes.take(10).toList())
// [2, 3, 5, 6, 7, 8, 9, 10, 11, 12]
```
请读者现在停下来，尝试解释这个结果。

之所以有这样的结果，是因为我们捕获了变量 `prime`。`filter` 是延迟完成的，因为我们使用的是 `Sequence`。在每一步中，我们都会添加越来越多的 `filter`。 在“优化”中，我们总是为可变属性 `prime` 的引用添加 `filter`。因此我们总是过滤质数的最后一个值。这就是此过滤无法正常工作的原因。除非 `drop` 生效，否则我们最终就会得到一串连续的数字（除了 4，当质数仍设置为 2 时被过滤掉了）。

我们应该谨慎留意无意捕捉的问题，因为上面这类情况可能会发生。为了防止它们，我们应该避免可变性，尽可能改用更窄的变量范围。

## 总结
出于多种原因，我们更愿意为尽可能接近的范围定义变量。 此外，对于局部变量，我们更喜欢 val 而不是 var。 我们应该始终意识到变量是在 lambda 中捕获的。 这些简单的规则可以为我们省去很多麻烦。