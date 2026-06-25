## 参与智能重组的前提

## enableStrongSkippingMode

### 官方文档
[官方文档:Strong skipping mode](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping#avoid-memoization)

### 开启和关闭
从Kotlin 2.0.20开始默认开启Strong skipping.

### 开启以后效果
* Composables with unstable parameters become skippable
* Lambdas with unstable captures are remembered 

## Composables with unstable parameters become skippable
Strong skipping mode开启以前，如果可组合函数有类型不稳定的参数，它的重组总是不跳过。  

Strong skipping mode开启以后，将使用===判定是否跳过重组。

### 开启Strong skipping mode以后，跳过重组的条件
可组合函数的参数满足如下所有条件时，可以跳过重组
* 对于不稳定参数:（本次 === 上次）== true
* 对于稳定参数: (本次 === 上次) || (本次.equals(上次))  

```kotlin
// 不稳定类型，跳过重组的例子

private data class UnstableCount(var count: Int) {
    override fun equals(other: Any?) :Boolean{
        throw RuntimeException("run equals!")
    }
}

private val ustableCount = UnstableCount(0)
private var count by mutableIntStateOf(0)

@Composable
fun StrongSkippingDemo2() {
    Button(onClick = { count++ }) { Text("count1+1") }
    println("trigger:${count}")
    SkChild1(ustableCount)
}

@Composable
private fun SkChild1(usCount: UnstableCount) {
    println("SkChild1 count:${usCount.count}")
}
```
* UnstableCount为不稳定类型
* 点击按钮以后总是给SkChild1传递同一个值，=== 结果为true，所以SkChild1的重组跳过
* UnstableCount的equals方法中的异常异常不会被抛出

```kotlin
// 稳定类型跳过重组的例子

private data class MyStableCount(val count: Int) {
    override fun equals(other: Any?): Boolean {
        println("run MyStableCount equals")

        // override equals以后编译器就不会帮我们生成equals方法了
        // 所以这里自己实现一下

        if (other == null) return false
        if (other === this) return true
        if (other !is MyStableCount) return false

        return this.count == other.count
    }
}

private var count by mutableIntStateOf(0)

@Composable
fun StrongSkippingDemo3() {
    Button(onClick = { count++ }) { Text("count1+1") }
    println("trigger:${count}")
    SkChild1(MyStableCount(0))
}

@Composable
private fun SkChild1(msCount: MyStableCount) {
    println("SkChild1 count:${msCount.count}")
}

// 点击按钮以后输出
// trigger:9
// run MyStableCount equals

```
点击按钮以后，导致需要判断是否跳过SkChild1的重组：
* 
* MyStableCount为稳定类型，所以使用equals比较2次的参数(所以看到输出：“run MyStableCount equals”)
* equals方法结果为true，所以SkChild1的重组被跳过(所以看不到输出:"SkChild1 count:${msCount.count}")

### 例外
当可组合函数被@NonSkippableComposable或者@NonRestartableComposable注解时，它的重组总是不被跳过

### 解释文档里的话
[官方文档:Strong skipping mode](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping#avoid-memoization)
```
you might want a restartable but non-skippable composable. In this case, use the @NonSkippableComposable annotation.
```
看如下例子
```kotlin
@Composable
fun ParentCompose(){
    //..

    // 1.
    println("parent")
    NonSkippableDemoChild(
        value = System.currentTimeMillis().toString()
    )
}

@Composable
@NonRestartableComposable
fun ChildCompose(value:String){
    //..
    val count by remember { mutableIntStateOf(0) }

    // 2.
    println("count:$count")
    // ..
}
```
* “restartable”指的是当count被修改时，ChildCompose的函数体可以在ParentCompose函数体不执行的情况下独立执行。观察控制台只会看到（2.）处的输出，看不到（1.）处的。

## Lambda memoization

### 要解决的问题
```kotlin
var countLC by mutableIntStateOf(0)

@Composable
fun LambdaComposable() {
    val valueToCapture = "good!"
    val lambda = {
        println("value:${valueToCapture}")
    }

    Button(onClick = {countLC++}) {
        Text("button")
    }
    println("count:$countLC")
    
}
```
假设没有“Lambda memoization”特性，每当LambdaComposable重组时，就会新创建一个lambda。当lambda的执行结果没有变化时这完全没必要。

### 解决办法
开启了Strong skipping以后，编译器会将lambda包裹在remeber中
```kotlin
// 伪代码
fun LambdaComposable() {
    val valueToCapture = "good!"
    val lambda = remember(valueToCapture){
        {
             println("value:${valueToCapture}")
        }
    }

    //..
}
```

* 这里的“remember”和普通的使用的略有不同，它使用"[开启strong-skipping-mode以后跳过重组的条件](#开启strong-skipping-mode以后跳过重组的条件)"中规则进行key比较。普通的remember总是使用equals方法进行key比较。
* 通过伪代码可以看出，当lambda捕获的值发生了变化(使用===或equals比较结果为false)时才会创建新的lambda实例，否则就复用旧的，这提升了性能。

### 例子1:重组时创建新的lambda
```kotlin
// 不稳定类型
private data class UnstableName(var name: String) {
    // equal总是不相等
    override fun equals(other: Any?) = false
}

private val unstableName = UnstableName("unstableName")
private var count = mutableIntStateOf(0)

@Composable
fun StrongSkippingLambdaDemo21() {
    val name = UnstableName("${System.currentTimeMillis()}")
    val lambda = {
        "name:${name}"
    }

    // 每次打印的lambda不同
    println("lambda:${lambda}")
    
    Button(onClick = { count.value++ }) { Text("count1+1") }
    println("trigger:${count.value}")
}
```

### 例子1:重组时创建新的lambda
```kotlin
// 不稳定类型
private data class UnstableName(var name: String) {
    // equal总是不相等
    override fun equals(other: Any?) = false
}

private val unstableName = UnstableName("unstableName")
private var count = mutableIntStateOf(0)

@Composable
fun StrongSkippingLambdaDemo22() {
    val name = unstableName
    val lambda = {
        "name:${name}"
    }

    // 每次打印的lambda相同
    println("lambda:${lambda}")

    Button(onClick = { count.value++ }) { Text("count1+1") }
    println("trigger:${count.value}")
}

```

### Avoid memoization
要关闭Lambda memization，使用如下方式
```kotlin
val lambda = @DontMemoize {
    ...
}
```

## APK Size

When compiled, skippable Composables result in more generated code than composables that are not skippable. With strong skipping enabled, the compiler marks nearly all composables as skippable and wraps all lambdas in a remember{...}. Due to this, enabling strong skipping mode has a very small impact on the APK size of your application.

Enabling strong skipping in Now In Android increased the APK size by 4kB. The difference in size largely depends on the number of previously unskippable composables that were present in the given app but should be relatively minor.

## 资料
* [官方文档:Strong skipping mode](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping)