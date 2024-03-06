# Replugin插件化

关键问题：

Replugin的原理是什么？

1. 从宿主如何启动插件的四大组件，以activity为例进行说明 -》 **Replugin 全面解析（1） 和 Replugin 全面解析（2）**
2. 从插件如何启动activity？不管是宿主的，还是插件的 -》**Replugin 全面解析（3）**

## **Replugin 全面解析（1）**

https://www.jianshu.com/p/5994c2db1557

###### **唯一Hook点：****`RepluginClassLoader`**

在应用启动的时候，Replugin使用`RepluginClassLoader`将系统的`PathClassLoader`替换掉，并且只篡改了`loadClass`方法的行为，用于加载插件的类，后面我们会详细讲解。每一个插件都会有一个PluginDexClassLoader，`RepluginClassLoader`会调用插件的`PluginDexClassLoader`来加载插件中的类与资源。

```Java
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    Class<?> c = null;
    c = PMF.loadClass(className, resolve);   //主力这里就是Hook点，先使用插件的
    if (c != null) {                         //PluginDexClassLoader加载
        return c;
    }
    //只有在插件没有找到相应的类，才使用系统原来的PathClassLoader加载宿主中的类
    try {
        c = mOrig.loadClass(className);
        return c;
    } catch (Throwable e) {
    }

    return super.loadClass(className, resolve);
}
```

###### **坑位**

坑位是Replugin中设计非常巧妙的一个概念，它的功能是与`RepluginClassLoader`配合才能实现的。所谓坑位就是预先在Host的Manifest中注册的一些组件（Activity, Service, Content Provider，唯独没有Broadcast Receiver)，叫做坑位。这些坑位组件的代码都是由gradle插件在编译时生成的，他们实际上并不会被用到。在启动插件的组件时，会用这些坑位去替代要启动的组件，并且会建立一个坑位与真实组件之间的对应关系（用`ActivityState`表示)，然后在加载类的时候`RepluginClassLoader` 会根据前文提到的被篡改过的行为偷偷使用插件的`PluginDexClassLoader`加载要启动的真实组件类，骗过了系统，这就是唯一hook点的作用。

## **Replugin 全面解析（2）**

https://www.jianshu.com/p/74a70dd6adc9

Activity作为四大组件中最重要的组件，在Replugin中对它的支持的架构设计也是最复杂的，所以本篇分析我们就来看看Activity的启动流程。

以下这张图简要的画出类Activity启动的过程，当然简化了一些流程：

- `Pmbase`根据`Intent`找到对应的插件
- 分配坑位`Activity`，与插件中的`Activity`建立一对一的关系并保存在`PluginContainer`中
- 让系统启动坑位`Activity`，因为它是在Manifest中注册过的
- Android系统会尝试使用`RepluginClassLoader`加载坑位`Activity`的`Class`对象
- `RepluginClassLoader` 通过建立的对应关系找到插件`Activity`，并使用`PluginDexClassLoader` 加载插件`Activity` 的Class对象并返回
- Android系统就使用这个插件中的`Activity`的Class对象来运行生命周期函数

Android系统就是这样被欺骗了！

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDUzNWIxODM1MjA4YWU0MmU3NjMyZDcxMDljOTg1MmZfallQZWtGN09jRjljQkhTQVM1OFFzOEdMU1FBZlRyUTRfVG9rZW46WVRGU2JrcUxpb3ZLU0J4WHZOMGN0bXhXbmRkXzE3MDk3MjAxNTk6MTcwOTcyMzc1OV9WNA)

## **Replugin 全面解析（3）**

https://www.jianshu.com/p/8465585b3507

通过这两篇文章，我们总结一下一个插件`Activity`启动的流程大致是：

- 加载目标`Activity`信息
- 开始查找`Activity`信息 —> 找到对应的`Pugin`信息 —> 解压APK文件 —> 加载Dex文件内容 —> 创建`Application`对象 —> 创建`Entry`对象初始化Plugin环境
- 寻找坑位
- 查找坑位 —> 将目标`Activity`与找到的坑位以`ActivityState`的形式缓存在`PluginContainer`s中  —> 启动坑位`Activity` —>  系统调用`RepluginClassLoader` —> 调用`PluginDexClassLoader`加载坑位`Activity`类 —> 通过`PluginContainers`找到坑位对应的目标`Activity`类 —> 系统调用`PluginActivity`的`attachBaseContext`函数 —> 创建`PluginContext`对象并替换 —> 目标`Activity`的正常启动流程（`onCreate`，`onStart`，`onResume`.....)
- 

如果你已经读完了这两篇分析文章，但还没有将Replugin的里面如此多的类的关系理清楚，那么下面这张图也许可以帮到你。不过这张图目前还不完整，只针对目前已经讲到的部分，后面会逐渐补全。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MWJmZjFmYzJmYWQzY2I2NDgyYTRkNGE5YTg3OTAwODVfSm51SEtlUzZEbHIxUDNONVp0REt0QW80RTdlOWFheXRfVG9rZW46QXNQR2JsanBKb3FidGR4QnNYRmNpYWFmbldUXzE3MDk3MjAxNTk6MTcwOTcyMzc1OV9WNA)

绿色的部分是`replugin-plugin-lib`，蓝色和红色部分都是是`replugin-host-lib`包中的类，但是蓝色部分是运行在UI进程中，而红色部分是运行在Persistent进程中。

绿色部分和蓝色部分之间的调用都是通过反射来实现的，所以用类虚线箭头，同样蓝色部分和红色部分是通过Binder进程间通信机制来调用的，也用虚线箭头表示。

## **Replugin 全面解析（4）**

service的实现原理

## **Replugin 全面解析（5）**

`BroadcaseReceiver`和`ContentProvider`的实现原理