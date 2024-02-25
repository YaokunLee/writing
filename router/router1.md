# 自定义路由框架详解（一）前置知识 Groovy进阶语法



groovy是开发Router框架的基础，我们先介绍一些Groovy的进阶语法（Groovy的简答语法和java kotlin很像，10分钟就能学完，请自行搜索）。

### 闭包

Groovy中的闭包是一个开放的、匿名的代码块，它可以接受参数、返回值，并且可以被赋值给变量。闭包在Groovy中非常强大，可以用于实现函数式编程范式。它们可以访问并修改其外部作用域中的变量。

闭包的基本语法如下：

```Groovy
{ [参数列表] -> [闭包体] }
```

闭包可以没有参数，也可以有多个参数。如果闭包有参数，参数列表放在箭头(`->`)的左边。闭包体包含了需要执行的代码块，放在箭头的右边。

#### 闭包的例子

##### 无参数闭包：

```Groovy
def hello = { println "Hello, World!" }
hello()
```

这个例子创建了一个简单的闭包，它打印"Hello, World!"，然后通过调用`hello()`来执行闭包。

##### 带参数的闭包：

```Groovy
def greet = { name ->
    println "Hello, ${name}!"
}
greet("Groovy")
```

这个例子中的闭包接受一个参数`name`，然后打印出带有这个名字的问候语。调用`greet("Groovy")`时，输出将是"Hello, Groovy!"。

##### 返回值的闭包：

```Groovy
def square = { int x ->
    return x * x
}
println square(4)  // 输出 16
```

这个闭包接受一个整数参数`x`，返回它的平方值。调用`square(4)`将返回16。

##### 闭包作为参数：

Groovy允许将闭包作为方法的参数，这在编写高阶函数时非常有用。

```Groovy
def operate(a, b, operation) {
    operation(a, b)
}

def result = operate(5, 3, { x, y -> x + y })
println result  // 输出 8
```

这个例子中，`operate`方法接受两个数字和一个闭包作为参数。闭包定义了对这两个数字进行的操作（在这个例子中是相加）。调用`operate(5, 3, { x, y -> x + y })`时，结果是8。

#### 使用闭包的好处

- **灵活性**：闭包可以轻松传递代码块作为参数，使得编写高阶函数和控制结构变得简单。
- **简洁的语法**：闭包提供了一种简洁的方式来定义匿名函数，这使得代码更加易于阅读和维护。
- **功能强大**：闭包可以捕获周围作用域的变量，这意味着它们可以用来编写功能丰富的回调函数和事件处理器。

Groovy的闭包是该语言提供的一种强大的特性，它极大地增强了Groovy的表达能力和灵活性。

### DSL与类

DSL（Domain-Specific Language）语言是为了解决特定问题领域而设计的计算机语言。它们提供了一种更高级的抽象，让开发者能够以声明的方式专注于"配置"，而非传统的命令式编程语法。Groovy的DSL是一个强大的示例，它通过闭包和动态类型的特性，提供了一种灵活而简洁的方式来定义DSL。

以下教程将通过一个Android构建脚本的例子，展示如何利用Groovy的闭包实现DSL。

考虑以下Android构建配置DSL：

```Groovy
{
    compileSdkVersion 27
    defaultConfig {
        versionName "1.0"
    }
}
```

这段代码定义了一个Android项目的编译SDK版本和默认配置。现在，我们将探索如何用Groovy代码等价替换这段DSL，以便深入理解DSL的工作原理。

#### 步骤1: 定义闭包

首先，定义一个闭包`myAndroid`，它包含了DSL中的配置：

```Groovy
def myAndroid = {
    compileSdkVersion 27
    defaultConfig {
        versionName "1.0"
    }
}
```

#### 步骤2: 创建模型类

接下来，定义两个Groovy类`Android`和`DefaultConfig`，这两个类将用来处理DSL中的配置数据。

```Groovy
class DefaultConfig {
    private String versionName

    def versionName(String versionName) {
        this.versionName = versionName
    }

    @Override
    String toString() {
        return "DefaultConfig{versionName='$versionName'}"
    }
}

class Android {
    private int compileSdkVersion
    private DefaultConfig defaultConfig = new DefaultConfig()

    def compileSdkVersion(int compileSdkVersion) {
        this.compileSdkVersion = compileSdkVersion
    }

    def defaultConfig(Closure closure) {
        closure.delegate = defaultConfig
        closure.call()
    }

    @Override
    String toString() {
        return "Android{compileSdkVersion=$compileSdkVersion, defaultConfig=$defaultConfig}"
    }
}
```

#### 步骤3: 关联闭包与对象

创建`Android`类的实例，并将`myAndroid`闭包与这个实例关联起来。这样，闭包内的代码就可以操作`Android`实例了。

```Groovy
Android a = new Android()
myAndroid.delegate = a
myAndroid.call()

println("myAndroid = $a")
```

通过执行上述步骤，`myAndroid`闭包内的配置会应用到`Android`实例`a`上，最终打印出该实例的状态，显示编译SDK版本和默认配置的版本名。

通过这个例子，我们展示了如何使用Groovy的闭包和委托特性来实现一个简单的DSL。DSL让开发者能够以声明的方式明确地表达配置，而无需担心底层的实现细节。这种方法提高了代码的可读性和可维护性，使得配置工作变得更加直观和简洁。

### Android构建过程解析

Android应用的构建是一个复杂的过程，涉及代码编译、资源处理、打包、签名等多个步骤。Gradle作为Android官方推荐的自动化构建工具，其构建生命周期主要分为三个阶段：初始化阶段、配置阶段和执行阶段。理解这三个阶段对于优化构建过程和解决构建问题非常有帮助。本教程将帮助新手开发者理解Android构建的基本流程。

#### 1. 初始化阶段

在初始化阶段，Gradle确定哪些项目参与构建，并为每个项目创建一个`Project`实例。如果你的项目包括多个模块（例如，应用模块和库模块），Gradle会为每个模块创建一个`Project`实例。

##### 关键任务：

- 确定参与构建的项目和模块。
- 为每个项目和模块创建`Project`实例。

##### 示例：

假设你有一个包含app模块和library模块的Android项目，Gradle会分别为这两个模块创建`Project`实例。

#### 2. 配置阶段

配置阶段发生在初始化阶段之后。在这个阶段，Gradle会评估并执行所有项目的`build.gradle`文件，配置每个项目所需的任务和依赖。

##### 关键任务：

- 解析并应用`build.gradle`文件中的配置。
- 根据依赖关系图配置项目的依赖。
- 创建和配置构建所需的任务。

##### 示例：

在app模块的`build.gradle`文件中，你可能会配置编译SDK版本、依赖库、签名配置等。Gradle会解析这些配置，并准备执行构建所需的任务，例如编译代码、处理资源和打包APK。

#### 3. 执行阶段

执行阶段是Gradle构建生命周期的最后一个阶段。在这个阶段，Gradle根据配置阶段生成的任务图执行具体的构建任务。

##### 关键任务：

- 按照依赖和顺序执行构建任务。
- 编译代码、处理资源、打包APK等。
- 执行测试、生成报告等附加任务。

##### 示例：

如果你执行`./gradlew assembleDebug`命令来构建一个调试版本的APK，Gradle会执行一系列任务，包括编译Java代码、处理资源和布局文件、打包APK并签名。

### root project/Project/Task

在Gradle构建生命周期中，确实有三个非常重要的角色：Root Project、Project和Task。它们在不同阶段扮演关键角色，共同确保构建过程的顺利进行。以下是对每个角色的详细介绍及其用法示例。

#### 1. Root Project（根项目）

在Gradle的世界里，每一个构建都涉及至少一个项目。这个项目可以是一个单独的项目，也可以是多个项目的集合。在包含多个项目的情况下，所有项目共享一个根项目，即Root Project。根项目主要用于进行全局配置，这些配置会应用到所有的子项目中。

在根目录下的`build.gradle`文件中，你可以配置全局依赖管理，比如：

```Groovy
// 根项目的build.gradle文件
allprojects {
    repositories {
        google()
        jcenter()
    }
}
```

这段配置确保了所有项目（根项目和子项目）都会从Google和jCenter仓库中解析依赖。

在构建过程中的初始化阶段，我们就能直接使用root project了

#### 2. Project（项目）

在Gradle中，Project代表构建过程中的一个项目或模块。每个Project都可以独立定义自己的构建脚本（`build.gradle`文件），其中可以包含用来构建和配置该项目的Task、依赖等信息。

在模块的`build.gradle`文件中，你可以配置该项目的特定依赖、插件等：

```Groovy
// 模块的build.gradle文件
apply plugin: 'com.android.application'

android {
    compileSdkVersion 30
    // 其他配置...
}

dependencies {
    implementation 'androidx.core:core-ktx:1.3.2'
    // 其他依赖...
}
```

在构建过程中的第二个阶段 配置阶段，我们可以在每个模块中使用project，如可以在app模块的build.gradle中

```Groovy
project.projectDir
```

来获取当前project的路径

#### 3. Task（任务）

Task是构建过程中的最基本单位，代表了构建过程中的一个单独步骤，比如编译代码、打包资源等。Task可以依赖其他Task，形成一个依赖图，Gradle会根据这个依赖图来决定任务的执行顺序。

定义一个简单的Task来打印一条消息：

```Groovy
// 在任意的build.gradle文件中
task hello {
    doLast {
        println 'Hello, Gradle!'
    }
}
```

Task依赖的例子

你可以让一个Task依赖另一个Task，确保在执行前后有一定的顺序：

```Groovy
task taskA {
    doLast {
        println 'Running Task A'
    }
}

task taskB {
    dependsOn taskA
    doLast {
        println 'Running Task B'
    }
}
```

在这个例子中，`taskB`依赖于`taskA`，这意味着在执行`taskB`之前，Gradle会先执行`taskA`。