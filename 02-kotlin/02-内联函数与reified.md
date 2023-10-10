1. 泛型
2. 泛型的类型擦除
3. reified关键字
4. json的反序列化方法
5. 内联函数



## intro

### reified

```kotlin
data class Vo(
        val json: String,
        val signature: ByteArray,
    ) {
        /** 尝试反序列化[json]字段为[T]类型 */
        inline fun <reified T> into(): T {
            return jacksonObjectMapper().readValue(json)
        }
    }
```

> 上述代码中，使用了关键字**reified**，是因为在编译的时候，有泛型类型擦除的机制



`reified` 是 Kotlin 中的一个关键字，通常与内联函数（inline functions）和泛型一起使用。它的主要作用是允许在内联函数中获取泛型类型参数的实际类型信息，而不受泛型类型擦除的限制。

在 Java 中，由于泛型类型擦除，你不能在运行时访问泛型类型的具体信息。但是，在某些情况下，你可能希望在运行时获得泛型类型参数的实际类型。这就是 `reified` 的用武之地。

在 Kotlin 中，你可以在内联函数中使用 `reified` 关键字来声明泛型类型参数，从而允许你在函数内部获取具体的类型信息。这在以下情况下特别有用：

1. **类型检查和转换：** 你可以在函数内部进行类型检查和转换，而无需考虑类型擦除。例如，你可以检查一个泛型类型是否是某个特定类的子类。

2. **创建泛型类型的实例：** 你可以在函数内部创建泛型类型的实例，而不必传递 `Class` 对象或使用工厂函数。

以下是一个示例，演示了如何在 Kotlin 中使用 `reified` 关键字：

```kotlin
inline fun <reified T> exampleFunction(item: Any) {
    if (item is T) {
        println("Item is of type ${T::class.simpleName}")
    } else {
        println("Item is not of type ${T::class.simpleName}")
    }
}

fun main() {
    val str = "Hello, World!"
    val num = 42

    exampleFunction<String>(str) // 输出：Item is of type String
    exampleFunction<Int>(num)    // 输出：Item is of type Int
}
```

在上面的示例中，`reified T` 允许我们在 `exampleFunction` 中访问 `T` 的实际类型信息，以进行类型检查。这使得代码更加类型安全和灵活。

总之，`reified` 关键字允许你在内联函数中获取泛型类型参数的实际类型信息，而不受类型擦除的限制，从而提高了代码的类型安全性和表达能力。





**为什么泛型要编译的时候要擦除？**

