## 一个关于Kotlin版本的error

在引入关于mongo的依赖之后，报错：

> Class 'com.mongodb.kotlin.client.coroutine.MongoClient' was compiled with an incompatible version of Kotlin. The binary version of its metadata is 1.8.0, expected version is 1.6.0.

在吗pom文件中把kotlin-version改了之后报错变成需要1.8，但是是1.6.很奇怪

把kotlin的版本直接改成1.9 然后

查看idea的projectStructrure 发现版本是这样的：

![image-20230926145318166](https://typoraryne.oss-cn-beijing.aliyuncs.com/undefinedimage-20230926145318166.png)



API version是1.6  Language Version是1.9

不止是这个有点蹊跷，在settings的编译器里也有这个版本相关的东西

![image-20230926165842346](https://typoraryne.oss-cn-beijing.aliyuncs.com/undefinedimage-20230926165842346.png)

这个报错根本就没看懂，到底是谁的版本是1.8，谁的是1.6。看上去很像是依赖关系的问题。这个`mongodb.kotlin.client.coroutine`依赖的版本可能是更老的或者是更新的，但是我把父模块的pom文件中的`kotlin.version`改成1.8.0 `mvn clear`  `incalidate caches` 一套下来。重新建立索引，

> 也不知道language version和api version 有什么✓8用， 
>
> 这两个都可以在maven的plugin项目中进行设置，但是几次尝试过后，发现api version 和这个错误相关性并不大，就写成1.6就行了大概是为了对以前的版本进行支持，而1.5版本已经是deprecated了

然后在mongodb.kotlin.client.coroutine的依赖中看到：

![image-20230926172235181](https://typoraryne.oss-cn-beijing.aliyuncs.com/undefinedimage-20230926172235181.png)

它依赖的kotlin版本是1.8啊，我不懂兼容不兼容，也不知道我现在的环境是多少了

既然觉得是mongodb.kotlin.client.coroutine与kotlin版本之间的关系的问题，那么就找到之前为了引入mongo官方驱动写的一个demo，里边的kotlin版本是最新版本1.9。那就先换成1.9试一下（此时都是在父模块的pom文件更改的），把pom文件、Project Structure里的版本、Setting Compiler中的全和Demo调整成一致，但是还是和原来一样的报错。

根本没有提1.9的事情。又茫然了，因为我项目里已经没有和kotlin 1.8 任何相关的东西了。

我也忘了我改了啥了，随后，又来一个报错，发现我自己写的一个类，也报错：

> Class 'com.cgs.deh.eds.service.config.EdsProps' was compiled with an incompatible version of Kotlin. The binary version of its metadata is 1.9.0, expected version is 1.6.0. 

那可能就不是版本与版本之间的关系了。

之后发现有些子模块并没有随着父项目的kotlin版本的改变而改变，看了一眼，是在子模块又写了一个kotlin.version。给他改了。此时父项目里的仍然是1.9.0 但是报错没有了。又回到了mongo那

![image-20230926172822883](https://typoraryne.oss-cn-beijing.aliyuncs.com/undefinedimage-20230926172822883.png)

好

把虽然mongo那边没有重写这个版本，明显是没有继承到父项目pom文件中的版本信息。

改了之后报错没有了。

舒服



## 原因

寻找一下原因：

对于`expected version is 1.6.0.` 说明这个是报错处所在的项目依赖的kotlin版本是1.6.0，但是什么元数据二进制的版本比它高，并不匹配。

看下面几个例子：

##### **1.**

> <kotlin.version>1.7.0</kotlin.version>
>
>  报错：
>
> Module was compiled with an incompatible version of Kotlin. The binary version of its metadata is 1.9.0, expected version is 1.7.1.

此时好像能直接检查出来，整个module与编译器环境不匹配

#### **2.**

> <kotlin.version>1.6.10</kotlin.version>
>
> 报错
>
> Class 'com.mongodb.kotlin.client.coroutine.MongoClient' was compiled with an incompatible version of Kotlin. The binary version of its metadata is 1.8.0, expected version is 1.6.0.

此时，不报整个module的错，但是会出现com.mongodb.kotlin.client.coroutine包下的报错

##### **3.** 

> <kotlin.version>1.8.0</kotlin.version>
>
> **success**

是不是说明在运行环境为1.9.0的时候，1.8的kotli代码是可以运行的

#### Kotlin Complier version 是 1.6.0 的时候

它有时候会变，当我把ides设置里的kotlin compiler version更改后，mvn compile后会改变，可能是又加载到了父项目中plugin的配置：

```xml
                <artifactId>kotlin-maven-plugin</artifactId>
                <groupId>org.jetbrains.kotlin</groupId>
                <version>${kotlin.version}</version>
                <configuration>
                    <compilerPlugins>
                        <plugin>all-open</plugin>
                        <plugin>spring</plugin>
                        <plugin>no-arg</plugin>
                    </compilerPlugins>
```

现在把它改成1.6.0（能找到的最老的）

可是真的很烦，idea设置的编译器版本是1.6，父项目写的是1.6

把子项目中的版本改成1.7后，去编译

`[INFO] --- kotlin-maven-plugin:1.7.0:compile (compile) @ deg-service ---` 成功

改成1.8之后：

`[INFO] --- kotlin-maven-plugin:1.8.0:compile (compile) @ deg-service ---` 成功

改成1.6之后

`[INFO] --- kotlin-maven-plugin:1.6.0:compile (compile) @ deg-service ---`

`[ERROR] Failed to execute goal org.jetbrains.kotlin:kotlin-maven-plugin:1.6.0:compile (compile) on project deg-service: Compilation failure: Compilation failure: `

终于报错了,但是这时候看一眼傻逼idea的运行环境变成了1.9.0 也不知道是不是真的变成了1.9

改成1.6之后就是会报错。



所以根据以上可以猜测，Mongo依赖的是1.8.0，在编译器1.7、1.8、1.9版本的环境下，都可以运行

唯独1.6.0不可以，说到底还是依赖的版本之间的问题



此时可以解释一下这个报错：

**Class 'com.mongodb.kotlin.client.coroutine.MongoClient' was compiled with an incompatible version of Kotlin. The binary version of its metadata is 1.8.0, expected version is 1.6.0.**

这个类被是一个不匹配的Kotlin版本编译的。这个分支（指的是MongoClient这个分支）的编译器版本是1.8.0，我们需要它是1.6.0（因为我现在在1.6.0的环境上运行它）



所以问题的关键在于，我该如何控制我这一个子项目使用比1.6.0版本更高的编译器，

如果我不在子项目中重写Kotlin-version的话，它将会使用1.6版本，可能是因为之前项目结构有变动的原因，或者是idea抽风。至于idea的compilier设置和ProjectStructure，不可靠。看日志里的才是对的

可以猜测，如果pom文件没有通过配置plugin来约束编译器版本，单独编译这个项目的时候，项目使用的编译器版本会和它的kotlin语言的版本保持一致或者对应。

>  至于在 **1.** 中报错1.7.1的，本身应该是可以成功的，只是 意外 这个整个module的语言被识别成了1.9版本的，但是编译器使用的是1.7.1。看来语言版本和编译器可能只是临近的才会兼容。

