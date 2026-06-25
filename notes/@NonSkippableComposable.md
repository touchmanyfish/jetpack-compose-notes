
## 被@NonSkippableComposable标记的函数
被@NonSkippableComposable标记的函数的parent函数重组时，无论parent传递给它的参数是否变化，它总是被重组。

## 例子
```kotlin
var count by mutableIntStateOf(0)

@Composable
fun NonSkippableComposable1() {
    // 1.
    println("parent:$count")

    Button(onClick = { count++ }) {
        Text(text = "button")
    }
    NonSkChild1(value = 100)
}

@Composable
@NonSkippableComposable
fun NonSkChild1(value: Int) {
    // 2.
    println("NonSkChild1:$value")
}

// 点击按钮以后输出
// parent:0
// NonSkChild1:100
```
1. 点击按钮修改count触发NonSkippableComposable1函数重组，所以（1.）处println()执行
2. 虽然此时再次给NonSkChild1()传递参数100，但是由于它被@NonSkippableComposable注解，所以不它的重组不被跳过，所以（2.）处println()执行

## 使用场景
？？？
库里面用的多，不是很理解
