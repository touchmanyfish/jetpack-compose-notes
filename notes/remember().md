# remember()

remember()用于记住一个存在于所在可组合项生命周期内的值(在进入组合以后，退出组合之前)。分2种情况讨论

### 1.不带key的情况
```kotlin
remember{
   value
}
```
* 当所在可组合项进入组合时(Initial Composition)执行lambda，remember(..)返回calculation执行结果，结果还会被被缓存
* 当所在可组合项重组时(Recomposition)返回calculation第一次执行时结果的缓存
* 当所在可组合项退出组合时(Dispose)缓存会被清除

### 2.带key的情况
```kotlin
remember(key1){
    value
}

remember(key1,key2,calculation){
    value
}
```
* 当所在可组合项进入组合时(Initial Composition)执行lambda，remember(..)返回calculation执行结果，结果还会被被缓存
* 当所在可组合项重组时(Recomposition),先依次对比key，如果key(一个或者多个)和上次传递给remember()函数的key发生了变，则执行lambda重新计算，remember(..)返回执行结果并缓存这个结果；如果所有key都没有发生变化则remember(..)直接返回上次缓存结果
* 当所在可组合项退出组合时(Dispose)缓存会被清除