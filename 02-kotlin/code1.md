## requireFound与requestRequire

### requireFound

```kotlin
inline fun <T : Any> requireFound(value: T?, lazyMessage: () -> Any): T {
    contract {
        returns() implies (value != null)
    }

    if (value == null) {
        val message = lazyMessage()
        throw NotFound(message.toString())
    } else {
        return value
    }
}
```

inline 内联函数，是指编译的时候会插入到调用处，而不是作为函数调用执行

为什么要写内联函数？

> - 消除了函数调用的额外开销，对于经常使用的小函数有很大好处
> - 消除了对象的创建
> - 编译器可以进行更准确的函数类型推断

调用的时候就是把lambda参数

### requestRequire

```kotlin
inline fun requestRequire(expression: Boolean, lazyMessage: () -> Any) {
    contract {
        returns() implies expression
    }
    if (!expression) {
        val message = lazyMessage()
        throw BadRequest(message.toString())
    }
}
```



调用的时候：

```kotlin
requestRequire(api.edsId == apiVo.edsId){
            "不是同一个eds的api"
        }
```

## 调用函数的时候给lambda参数传参

当函数的最后一个参数是一个lambda表达式时，有多种方式来传递这个lambda。

### 1. 基础用法

最简单的方式是直接作为参数传入：

```kotlin
fun someFunction(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}

// 调用方法
someFunction(2, 3, { x, y -> x + y })
```

### 2. 尾随lambda（Trailing Lambda）

如果lambda是函数的最后一个参数，你可以将其移到括号外部：

```kotlin
someFunction(2, 3) { x, y -> x + y }
```

### 3. 多个Lambda参数

如果一个函数有多个lambda参数，你不能将所有的都移到外面，只有最后一个可以。其他的需要按照普通的方式传递。

```kotlin
fun multipleLambdas(a: Int, first: (Int) -> Int, second: (Int) -> Int): Int {
    return second(first(a))
}

// 调用方法
multipleLambdas(2, { it * 2 }, { it + 3 })  // 结果是 7
```

或者你也可以使用命名参数的方式来明确地标识哪个lambda做什么：

```kotlin
multipleLambdas(2, first = { it * 2 }, second = { it + 3 })  // 结果还是 7
```

## `firstOrNull()` & `?.` & `.let`

这三个都是具有安全调用的性质，都是为了避免空判断

- `firstOrNull（）`：对于一个list，如果不为空就取第一个，如果为空就返回null
- `?. ` : 安全调用，只有不为空的时候才调用，不然就返回null
- `?.let` ：只有当调用者不为空的时候，let代码块内的代码才会被执行

```kotlin
val schemaFields = requireFound(
            edsService.listEds(listOf(apiVo.edsId)).firstOrNull()?.schema
                ?.let { schema -> schema.fields.map { it.name to it.type.typeClass } }
        ){"eds或者schema不存在"}
```

### firstOrNull

```kotlin
/**
 * Returns the first element, or `null` if the list is empty.
 */
public fun <T> List<T>.firstOrNull(): T? {
    return if (isEmpty()) null else this[0]
}

/**
 * Returns the first element matching the given [predicate], or `null` if element was not found.
 */
public inline fun <T> Iterable<T>.firstOrNull(predicate: (T) -> Boolean): T? {
    //对于所有的在这个集合里的元素，返回第一个符合条件的
    for (element in this){
    	if (predicate(element)) 
    		return element        
    } 
    return null
}
```



## 函数重载

```kotlin
 private fun checkParamConfig(
        inputs: List<InputParam>,
        outputs: List<OutputParam>,
        schemaFields: List<Pair<String, FieldTypeClass>>,
    ) {
        requestRequire(checkParamConfig(inputs, schemaFields)
        { "${it.bindingName}-${(it as InputParam).queryOp}" }) { "新增API：入参配置有误" }
        requestRequire(checkParamConfig(outputs, schemaFields) { it.name }) { "新增API：出参配置有误" }
    }

private fun checkParamConfig(
        paramConfig: List<Param>,
        schemaFields: List<Pair<String, FieldTypeClass>>,
        distinctFunc: (Param) -> String,
    ): Boolean {
        val noDup = paramConfig.distinctBy(distinctFunc)
        return noDup.size == paramConfig.size &&
                paramConfig.all { param ->
                    param.name.isNotBlank() &&
                                 //至少有一个schema中的字段名与其对应，并且类型相匹配
                            schemaFields.firstOrNull { (fieldName, fieldType) ->
                                param.bindingName == fieldName && param.type match fieldType
                            } != null
                }
    }
infix fun ParamType.match(dbType: FieldTypeClass): Boolean {
    return when (this) {
        ParamType.STRING -> when (dbType) {
            FieldTypeClass.VARCHAR, FieldTypeClass.CHAR, FieldTypeClass.TEXT -> true
            else -> false
        }
        ParamType.INT -> when (dbType) {
            FieldTypeClass.INT_2, FieldTypeClass.INT_4, FieldTypeClass.INT_8, FieldTypeClass.DECIMAL -> true
            else -> false
        }
        ParamType.FLOATING_POINT,
        -> when (dbType) {
            FieldTypeClass.FLOAT_4, FieldTypeClass.DOUBLE_8 -> true
            else -> false
        }
        ParamType.LIST -> true
        ParamType.BOOL -> dbType == FieldTypeClass.BOOLEAN
        ParamType.DATE -> dbType == FieldTypeClass.DATE
        ParamType.TIME -> dbType == FieldTypeClass.TIME
        ParamType.DATETIME -> dbType == FieldTypeClass.DATETIME
    }
}
```

### 代码解读

这两个函数都名为`checkParamConfig`，但有不同的签名（即不同的参数类型或数量），这在Kotlin（也在许多其他编程语言中）是完全合法的。这叫做函数重载。

### 第一个`checkParamConfig`

这个函数接收三个参数：`inputs`（类型为`List<InputParam>`），`outputs`（类型为`List<OutputParam>`）和`schemaFields`（类型为`List<Pair<String, FieldTypeClass>>`）。

这个函数调用了两次第二个`checkParamConfig`函数：

1. 第一次是用来检查输入参数`inputs`。它传递了一个lambda表达式，用于生成一个由`bindingName`和`queryOp`组成的唯一字符串，这个字符串用于检查是否有重复的输入参数。

   ```kotlin
   requestRequire(checkParamConfig(inputs, schemaFields)
   { "${it.bindingName}-${(it as InputParam).queryOp}" }) { "新增API：入参配置有误" }
   ```

2. 第二次用来检查输出参数`outputs`。这次，lambda表达式只使用了`name`字段来检查是否有重复的输出参数。

   ```kotlin
   requestRequire(checkParamConfig(outputs, schemaFields) { it.name }) { "新增API：出参配置有误" }
   ```

`requestRequire`可能是一个自定义函数，用于抛出异常或其他处理，如果其内部函数调用（`checkParamConfig`）返回`false`。

### 第二个`checkParamConfig`

这个函数接收三个参数：一个`List<Param>`，一个`List<Pair<String, FieldTypeClass>>`和一个lambda函数（`distinctFunc`）。

- `distinctBy(distinctFunc)`: 这一行用于检查是否有重复的参数，使用`distinctFunc`来生成一个唯一标识。
  
- 接下来，它检查两件事：

    1. 检查参数列表`paramConfig`中是否有重复项。
    
    2. 使用`all`函数来检查每一个`param`是否满足以下条件：
        - `name`不能为空。
        - `bindingName`和`type`应该与`schemaFields`中的一个字段匹配。

这个函数最终返回一个布尔值，表示所有检查是否通过。

这两个函数大致做的事情就是：确保输入和输出参数与预定义的schema匹配，并且没有重复的参数。如果任何检查失败，可能会通过`requestRequire`抛出一个异常（或其他处理方式）。





### 重载
在Kotlin中，你可以拥有多个同名函数，只要它们的参数列表（参数类型和数量）不同。这称为函数重载（Function Overloading）。

在你提供的代码中，两个`checkParamConfig`函数有不同的参数列表：

1. 第一个`checkParamConfig`函数接受三个参数：
   - `inputs: List<InputParam>`
   - `outputs: List<OutputParam>`
   - `schemaFields: List<Pair<String, FieldTypeClass>>`
2. 第二个`checkParamConfig`函数也接受三个参数，但类型和名称不同：
   - `paramConfig: List<Param>`
   - `schemaFields: List<Pair<String, FieldTypeClass>>`
   - `distinctFunc: (Param) -> String`

由于这两个函数的参数列表不同，Kotlin编译器可以根据参数类型和数量来决定应调用哪个版本，因此这是合法的。

注意：函数重载并不仅仅是基于参数名称的，而是基于参数的类型和数量。即使参数名称不同，如果类型和数量相同，那么这两个函数不能共存，除非它们有不同的返回类型（但这也不是推荐的做法）。





# 类的构造

### 加val与不加val的区别

```kotlin
class  DEGCatalogController(
    @Autowired
    degCatalogService : DEGCatalogService
){...}

class  DEGCatalogController(
    @Autowired
    val degCatalogService : DEGCatalogService
){...}
```

不加的时候，degCatalogService仅仅作为构造函数的参数被注入，这样的话，你不能在类的其他方法和代码块中访问到它，因为它不是类的属性



加的时候，degCatalogService 作为类的制度属性，是类的成员变量



总结：

- 如果需要在`DEGCatalogController`类的其他部分访问`degCatalogService`，你应该使用第二种方式。
- 如果`degCatalogService`只在构造函数中用于一些初始化操作，并不需要在类的其他地方使用，那么第一种方式可能就足够了。



























