# 自定义路由框架详解（五） - 打开页面



## 打开页面

在自定义路由框架中，最后一步是实现页面的跳转。这个过程涉及到根据URL解析目标页面的类名，并通过反射机制启动对应的Activity。以下是实现这一功能的教程，包括关键代码说明：

### 初始化路由映射表

首先，需要在应用启动时（例如在`Application`的`onCreate`方法或者是特定的初始化逻辑中）调用`Router.init()`方法来初始化路由映射表。这个方法会加载编译期间生成的映射表类，并将其内容填充到运行时的映射表中。

```Kotlin
Router.init()
```

### 跳转到指定页面

使用`Router.go(context, url)`方法来根据URL跳转到指定的Activity。方法内部逻辑如下：

1. **解析URL**：解析传入的URL，获取其scheme、host和path部分，用于后续的匹配查找。

```Kotlin
val uri = Uri.parse(url)
val scheme = uri.scheme
val host = uri.host
val path = uri.path
```

1. **查找目标Activity类名**：遍历映射表，找到与URL匹配的条目，获取其对应的Activity类名。

```Kotlin
var targetActivityClass = ""
mapping.onEach {
    val ruri = Uri.parse(it.key)
    if (ruri.scheme == scheme && ruri.host == host && ruri.path == path) {
        targetActivityClass = it.value
    }
}
```

1. **解析URL参数**：解析URL中的查询参数（query），并将参数封装到`Bundle`中。

```Kotlin
val bundle = Bundle()
uri.query?.split("&")?.forEach { arg ->
    val (key, value) = arg.split("=")
    bundle.putString(key, value)
}
```

1. **启动Activity**：使用反射找到目标Activity的`Class`对象，并通过`Intent`启动它，同时将参数`Bundle`传递过去。

```Kotlin
try {
    val activityClass = Class.forName(targetActivityClass)
    val intent = Intent(context, activityClass).apply {
        putExtras(bundle)
    }
    context.startActivity(intent)
} catch (e: Throwable) {
    Log.e(TAG, "go: error while start activity: $targetActivityClass, e = $e")
}
```

### 教程总结

通过上述步骤，你可以实现一个简单的路由跳转功能，该功能允许应用根据URL跳转到相应的Activity，并传递URL中的参数。这种方式大大简化了Activity之间的跳转和参数传递，使得页面跳转逻辑更加清晰和灵活。在实际开发中，你可以根据需要扩展这个基础功能，比如添加跳转动画、处理特殊的URL格式、增加权限检查等。