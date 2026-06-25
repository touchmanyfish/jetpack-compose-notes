# @NonRestartableComposeable

## 被@NonRestartableComposeable标记的函数
1. parent重组时不管该函数参数是否变化，它总是被执行，没有检测参数是否变化的步骤
2. 编译器不会给该函数生成重启相关代码代码，该函数只能通过parent的重组来导致该函数的重启

## 不检测参数变化的例子
### 1. parent重组，参数不变，该函数不跳过重组
```kotlin
@Composable
fun NonRestartableDemo1() {
    var count by remember {
        mutableIntStateOf(0)
    }
    
    // 1.
    println("parent run! count:$count")
    Button(onClick = { count++ }) { Text("button") }

    // 每次都传固定的"good!"
    NonRChild("good!")
}

@Composable
@NonRestartableComposable
fun NonRChild(name: String) {
    // 2.
    println("child run!")
    Text("non r chile")
}

// 点击Button以后输出
// parent run! count:1
// child run!
```

1. 点击Button触发可组合项NonRestartableDemo1的重组，对应函数体再次执行；所以(1.)处println()执行。
2. 接着调用(这么说准确吗？不知道反编译代码漏掉了多少编译器处理以后的细节)NonRChild("good!"),由于它被NonRestartableComposable标记，它的重组不跳过。所以（2.）处println()执行。

### 2. parent重组，参数变化，该函数不跳过重组
```kotlin
@Composable
fun NonRestartableDemo2() {
    var count by remember {
        mutableIntStateOf(0)
    }

    // 1.
    println("parent run! count:$count")
    Button(onClick = { count++ }) { Text("button") }

    // 每次传递不同参数
    NonRChild2(System.currentTimeMillis().toString())
}

@Composable
@NonRestartableComposable
fun NonRChild2(name: String) {
    // 2.
    println("child run!")
    Text("non r chile")
}

// 点击Button以后输出
// parent run! count:1
// child run!
```
parent重组了，所以（1.）处println()执行，由于可组合函数NonRChild被NonRestartableComposable标记，所以它对应的重组不被跳过，所以（2.）处println()执行

## 不生成重启相关代码的说明
```kotlin
@Composable
@NonRestartableComposable
fun NonRestartableDemo3(value:String){
    Text("$value")
}
```
反编译代码如下
```java
public final class NonRestartableDemo3Kt {
   @Composable
   public static final void NonRestartableDemo3(@NotNull String value, @Nullable Composer $composer, int $changed) {
    // ..
      $composer.startReplaceGroup(1376374039);
      ComposerKt.sourceInformation($composer, "C(NonRestartableDemo3)10@299L14:NonRestartableDemo3.kt#ili49k");
      // ..

      TextKt.Text--4IGK_g(//..);
      
      // ..

      $composer.endReplaceGroup();
   }
}
```
如果去掉@NonRestartableComposable注解，反编译如下
```java
public final class NonRestartableDemo3Kt {
   @Composable
   public static final void NonRestartableDemo3(@NotNull String value, @Nullable Composer $composer, int $changed) {

      // ..
      $composer = $composer.startRestartGroup(1376374039);
      ComposerKt.sourceInformation($composer, "C(NonRestartableDemo3)10@301L14:NonRestartableDemo3.kt#ili49k");
      int $dirty = $changed;
      if (($changed & 14) == 0) {
         $dirty |= $composer.changed(value) ? 4 : 2;
      }

      if (($dirty & 11) == 2 && $composer.getSkipping()) {
         $composer.skipToGroupEnd();
      } else {
         //..

         TextKt.Text--4IGK_g(//..);

         //..
      }

      // 这里就是重启相关代码；
      // 当 NonRestartableDemo3的重启作用域重启时，会使用相同参数再次调用NonRestartableDemo3()方法
      // 注意！反编译以后有些细节对不上:
      //  1.updateScope()方法参数列表和NonRestartableDemo3Kt::NonRestartableDemo3$lambda$0对不上
      //  2.编译器会将NonRestartableDemo3函数体和传递给它的参数包裹到匿名函数中，这里也没体现出来
      //  3.等等
      ScopeUpdateScope var4 = $composer.endRestartGroup();
      if (var4 != null) {
         var4.updateScope(NonRestartableDemo3Kt::NonRestartableDemo3$lambda$0);
      }

   }

   private static final Unit NonRestartableDemo3$lambda$0(String $value, int $$changed, Composer $composer, int $force) {
      Intrinsics.checkNotNullParameter($value, "$value");
      NonRestartableDemo3($value, $composer, RecomposeScopeImplKt.updateChangedFlags($$changed | 1));
      return Unit.INSTANCE;
   }
}
```
对比上述反编译后到代码可以验证，加上@NonRestartableComposeable注解以后，编译器不会在对应可组合函数内部生成重启作用域相关代码。

## 读取State，导致parent重组
```kotlin
var count by mutableIntStateOf(0)

@Composable
fun NonRParent2() {
    println("parent2")
    NonRParent1()
}

@Composable
fun NonRParent1() {
    println("parent1")
    NonRChild3()
    Button(onClick = { count += 1 }) {
        Text("button")
    }
}

@Composable
@NonRestartableComposable
fun NonRChild3() {
    println("NonRChild3")
    NonRChild4()
}

@Composable
@NonRestartableComposable
fun NonRChild4() {
    println("NonRChild3:count:$count")
}

// 点击Button以后输出：
// parent1
// NonRChild3
// NonRChild3:count:1
```
根据Gemini的解释:
1. NonRChild4中读取了State count，count需要关联到重启作用域但是NonRChild4没有对应的重启作用域
2. 于是向上查找，查看NonRChild3，发现也没有
3. 继续向上查找找到NonRParent1有重启作用域，将count关联到这个重启作用域
4. 点击Button以后修改count到值，NonRParent1的作用域与count关联，导致这个重启作用域重启,于是可组合函数NonRChild3被执行
5. 由于NonRChild3/NonRChild4被NonRestartableComposable注解了，所以它俩总是执行

### 使用场景

这部分不是很理解