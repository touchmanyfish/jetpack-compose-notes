## restarting的2种方式
Gemini回答的Restarting (重启)定义是:
```
在 Jetpack Compose 中，Restarting (重启) 是指 **Compose 运行时（Runtime）**在数据发生变化时，独立地重新运行一个 Composable 函数的过程。
```
不太清楚Restarting (重启)是特下面的第二种，还是两种都是。现在按特指下面两种来理解。

## 在compose中，可组项被重新执行有2种情况
### 1. 与自己关联的State发生变化，导致可组合项被重启，对应可组合函数重新执行
```kotlin
@Composable
fun RestartingDemo2(){
    Text("parent")
    ResChild2(System.currentTimeMillis())
}

@Composable
fun ResChild2(value:Long){
    println("value from parent:$value")
    var name by remember {
        mutableStateOf(System.currentTimeMillis().toString())
    }
    Button(onClick = { name = System.currentTimeMillis().toString() }) {
        Text("button")
    }

    println("reschild2 name:$name")
}
```

### 2. 当前可组合项的Parent被重启了，自己不符合跳过重组的条件，导致对应可组合函数被执行


