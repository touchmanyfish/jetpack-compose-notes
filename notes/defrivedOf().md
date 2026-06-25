# derivedOf()

```kotlin
val state = derivedStateOf(lambda)
```

### lambda中不包含State
lambda中如果不包含State，则derivedStateOf的lambda参数只会在derivedStateOf第一次调用时执行
```kotlin
var price = 0

@Composable
fun DerivedDemo() {
    val age = remember { mutableIntStateOf(10) }
    val des = remember {
        derivedStateOf {
            price > 5
        }
    }
    Column {
        Text(
            modifier = Modifier.padding(top = 100.dp),
            text = "${des.value}"
        )
        Button(onClick = {
            price += 1
            age.value += 1
        }, content = { Text("add age") })
    }
}
```
点击Button修改age的值触发重组，间接导致derivedStateOf()执行，但是由于derivedStateOf的lambda参数中没有State，所以lambda只会执行一次。

### lambda中包含State
当State当值被修改时，使用到这个值当所有derivedStateOf(lambda)的lambda都会计算一次，如果lambda的结算结果发生变化，则触发所有(真的是所有吗？这里是感觉的)和derivedStateOf(lambda)返回State关联的可组合函数的重组
```kotlin
@Composable
f@Composable
fun DerivedDemo() {
    Column {
        val age = remember { mutableIntStateOf(10) }
        val des = remember {
            derivedStateOf {
                // 1.
                age.value > 15;
            }
        }
        
        Text(
            modifier = Modifier.padding(top = 100.dp),
            // 2.
            text = "${des.value}"
        )
        Button(
            onClick = { age.value += 1 },
            content = { Text("add age") }
        )
    }
}
```
当点击Button修改了age，由于(1.)处derivedOf(lambda)的lambda参数中使用了age，所以lambda会执行；当age.value的值被累加到16，lambda返回值发生变化，和des关联的可组合函数Column会触发重组。

### derivedOf使用场景
将上一个例子修改为不使用derivedOf
```kotlin
fun DerivedDemo2() {
    Column {
        // 1.
        var age by remember { mutableIntStateOf(10) }
        Text(
            // 2.
            modifier = Modifier.padding(top = 100.dp),
            // 3.
            text = "${age > 15}"
        )
        Button(
            onClick = { age += 1 },
            content = { Text("add age") }
        )
    }
}
```
为了显示(age > 5)这个值，在这个例子中每当age被修改时就会触发可组合函数Column的重组。虽然当age的值在[10,15]中时，可组合函数Text的重组可以被智能重组跳过，但是当执行可组合函数Column当重组时执行代码(1.)(2.)(3.)以及跳过Text可组合项前进行的参数对比仍然是开销(也许还有其他开销但是我不知道是什么)。而使用derivedOf()则可以跳过当age当值在[11,15]时的重组。·  

所以derivedOf的使用场景是:
```
某个值是基于一个或者多个其他State计算出来的，并且这些state更新频率很高。通过使用derivedOf来降低更新频率从而提升性能。
```

