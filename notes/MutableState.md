## MutableState
MutableState类型内部保存一个内部值，
当在可组合函数中读取这个内部值时，compose会将这个MutableState和当前的可组合函数关联到一起。

当修改这个内部值时，如果新的值和旧的内部值不同时(通过==判断)，会找到和这个MutableState有关联的的所有可组合函数最顶层的那个进行重组。  

例如MutableIntState内部的“内部值”指的是intValue，“读取”指的是调用intValue的get()方法；“写入”值的是调用intValue的set()方法。

### 1.通过.value使用MutableState
```kotlin
var count = mutableIntStateOf(0)

// 赋值
count.value = 100

// 读取值
val z = count.value
```
### 2.通过解构使用MutableState
(注意MutableState的创建方式有很多，这里只分析mutableIntStateOf())
```kotlin
val (value, setValue) = mutableIntStateOf(0)

// 继承链
// ↑
// MutableState
// ↑
// MutableIntState
// ↑
// SnapshotMutableIntStateImpl
// ↑
// ParcelableSnapshotMutableIntState
```
mutableStateOf(0)实际返回ParcelableSnapshotMutableIntState类型，它的父类SnapshotMutableIntStateImpl实现了在MutableState中定义的解构方法，(value, setValue)实际上是对ParcelableSnapshotMutableIntState的解构。
```kotlin
// SnapshotMutableIntStateImp类源码

override var intValue: Int
    get() = next.readable(this).value
    set(value) =
        next.withCurrent {
            if (it.value != value) {
                next.overwritable(this, it) { this.value = value }
            }
        }

// 解构方法
override fun component1(): Int = intValue

// 解构方法
override fun component2(): (Int) -> Unit = { intValue = it }
```
intValue为MutableState对应(这种关系如何表达？)的值，会被赋值给(value, setValue)中的value；{ intValue = it }这个lambda执行以后可以设置intValue的值并触发compose的重组(就是触发一个通知吗？这里是试出来的)，它将被赋值给(value, setValue)中的setValue.  

### 3.通过by关键字使用MutableState
```kotlin
var count by mutableIntStateOf(0)
```
mutableStateOf(default)实际返回MutableState类型，在SnapshotIntState.kt中实现了MutableState对属性代理的支持
```kotlin
// SnapshotIntState.kt

inline operator fun IntState.getValue(thisObj: Any?, property: KProperty<*>): Int = intValue

inline operator fun MutableIntState.setValue(thisObj: Any?, property: KProperty<*>, value: Int) {
    intValue = value
}
```
intValue为MutableState对应的值，当读取count的值时，会调用上述getValue()方法获取intValue；当给count赋值时会调用上述setValue()方法来修改intValue的值。MutableIntState的子类还会重写intValue的set()方法，让intValue当值被修改时能触发重组.  

## 从标记(对这个标记的理解有点模糊)的角度分析3种使用方式的区别

### 1.通过.value使用MutableState
```kotlin
var count = mutableIntStateOf(0)
```
count为MutableIntState类型，任何能访问到它的可组合函数都可以通过读取count.value来将自己和count关联到一起，以便当count内部值改变时参与到重组流程当中(因为可能读取count内部值的可组合函数有多个，它不是最顶层的那个)；

也可以通过count.value = 100,来修改这个内部值，并可能触发读取它的可组合函数的重组。  

所以这种使用方式的特点是：
1. 任何能访问到count的可组合函数都有关联的机会
2. 任何能访问到count的地方都能过修改内部值，并触发可组合函数的重组

### 2.通过解构使用MutableState
```kotlin
val (value, setValue) = mutableIntStateOf(0)
```
特点：
   
1. 只有在解构时才会调用get()方法读取内部值，所以只有在解构所在的可组合函数才会关联到MutableIntState。如果在非组合函数中执行解构，这个MutableIntState就永远失去了关联和触发重组的机会。
2. 任何能访问到setValue的地方都能通过调用它来触发解构所在可组合函数们的重组

### 3.通过by关键字使用MutableState

#注意，下面这块不知道怎么写了，先跳过

```kotlin
var count by mutableIntStateOf(0)
```
当使用by关键字进行属性代理时，编译器会在底层生成一个代理对象，这个对象没法传递到by关键字所在作用域之外。例如通过如下方式
```kotlin
val z = count
```
因为当你读取count的值时，实际上会调用getValue()返回内部的实际值intValue。所以对count的读取和修改只能在by关键字所在kotlin作用于中进行。

