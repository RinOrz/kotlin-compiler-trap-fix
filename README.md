# kotlin-compiler-trap-fix
记录开发 Kotlin 编译器插件时的踩坑日常

## Old Frontend

### SyntheticResolveExtension
> `BindingContext.getType` = null

<details><summary><b>原因</b></summary>

`SyntheticResolveExtension` 的执行时机很早，此时类型尚未被解析也并未绑定到上下文当中，因此无法获取绑定的类型信息，详细参考我与 [@demiurg906](https://github.com/demiurg906) 的对话：https://kotlinlang.slack.com/archives/C7L3JB43G/p1643646855085129
  
</details><br/>

> `Recursion detected on input: actualType under LockBasedStorageManager@30057fbe (TopDownAnalyzer for JVM)`

<details><summary><b>原因</b></summary>

不允许在 `SyntheticResolveExtension` 中调用 `unsubstitutedMemberScope.getContributed...`，详细参考我与 [@demiurg906](https://github.com/demiurg906) 的对话：https://kotlinlang.slack.com/archives/C7L3JB43G/p1643648810215179
  
</details><br/>

## IR Backend

> `No mapping for symbol: VALUE_PARAMETER INSTANCE_RECEIVER name:<this> type:<root>.MyClass`

<details><summary><b>解决方式</b></summary>

```diff
- val dispatchReceiver = parentAsClass.thisReceiver!!.copyTo(constructor)
+ val dispatchReceiver = parentAsClass.thisReceiver!!
```
原因为 JVM 字节码中的 `this` 在构造函数中是直接访问的，而在常规函数中将会为 `this` 创建一个临时变量，因此 `IrConstructor.body` 中不需要使用 [IrValueParameter.copyTo](https://github.com/JetBrains/kotlin/blob/1.6.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/ir/IrUtils.kt#L113) 来复制 `class.thisReceiver`。（浪费了俩小时🤬西内！！！！）

</details><br/>

> `Null argument in ExpressionCodegen for parameter VALUE_PARAMETER SYNTHETIC_MARKER_PARAMETER name:$constructor_marker index:2 type:kotlin.jvm.internal.DefaultConstructorMarker?`

<details><summary><b>解决方式</b></summary>

```diff
- addConstructor(origin = IrDeclarationOriginImpl("SYNTHETIC_DECLARATION", isSynthetic = true))
+ addConstructor(origin = IrDeclarationOriginImpl("SYNTHETIC_DECLARATION", isSynthetic = false))
```
原因为带有合成 `origin` 的构造函数会从 `constructor(p0: kotlin.String)` 转换为 `($this: <root>.MyClass, p0: kotlin.String, $constructor_marker: kotlin.jvm.internal.DefaultConstructorMarker?)`，因此创建构造函数时 `origin` 的 `isSynthetic` 参数不能设置为 `true`

</details><br/>
