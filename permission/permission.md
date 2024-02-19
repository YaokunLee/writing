# 轮子哥权限库解析

本文飞书文档链接：https://is0frj68ok.feishu.cn/wiki/AymJwIeLQijnz3kDb8scNJvOnbh?from=from_copylink 建议飞书打开，有些飞书支持的语法，markdown不支持似乎

代码链接： [GitHub - getActivity/XXPermissions: Android 权限请求框架，已适配 Android 14](https://github.com/getActivity/XXPermissions)

轮子哥号称可以一句话解决权限申请，那么是怎么做到的，这其中有什么设计模式值得我们学习？

接下来让我们来认真分析一下整体的实现，并看看我们从中可以学到哪些东西。





## 框架用法

该框架只需要如下代码，就可以完成权限申请

```Java
XXPermissions.with(this)
    .permission(Permission.CAMERA)
    .interceptor(new PermissionInterceptor())
    .request(new OnPermissionCallback() {

        @Override
        public void onGranted(@NonNull List<String> permissions, boolean allGranted) {
            if (!allGranted) {
                return;
            }
            toast(String.format(getString(R.string.demo_obtain_permission_success_hint),
                PermissionNameConvert.getPermissionNames(MainActivity.this, permissions)));
        }
    });
```

## 整体架构

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=M2RmYjAxMWNjNGE0OTNlODg2MTQwOGI3N2EwMmJmZjBfbG9DVFJ2SldyMUN6WEFQOUpJU2xiOVBzYmo5VDdSQnZfVG9rZW46UVNZcWJHdVRsbzliY2Z4MzEzeGNpZk1ybmJmXzE3MDgzMTMwOTg6MTcwODMxNjY5OF9WNA)

向外暴露的方法为XXPermission, 就是通过调用XXPermission来实现一句话申请权限的。

看到一句话申请权限的用法

```Java
XXPermissions.with(this)
    .permission(Permission.CAMERA)
    .interceptor(new PermissionInterceptor())
    .request(new OnPermissionCallback() {

        @Override
        public void onGranted(@NonNull List<String> permissions, boolean allGranted) {
            if (!allGranted) {
                return;
            }
            toast(String.format(getString(R.string.demo_obtain_permission_success_hint),
                PermissionNameConvert.getPermissionNames(MainActivity.this, permissions)));
        }
    });
```

第三行，这里也可以自定自己的OnPermissionInterceptor，这样可以自定义如何向用户询问获取权限，整个框架提供了已实现的 PermissionInterceptor（通过用户无感知的 Fragment 来作为询问权限的载体）你也可以自己实现OnPermissionInterceptor，修改成用 activity来作为载体。

同时OnPermissionInterceptor 还能自定义用户同意权限、拒绝权限后的回调。这里实际上跟OnPermissionCallback 的功能有重叠，其实可以去掉

## 如何实现一句话权限申请的

有两个关键问题需要解决

1. 必须通过 Activity 或者 Fragment 来申请权限，所以在申请权限时，必须配合 Activity/Fragment 或者自己新生成来进行权限申请，怎么解决？
2. 申请权限必须等用户点击同意或者反对之后，才能执行对应的回调，这整个过程如果用安卓自带的权限申请来做，整个过程是异步的（从代码角度看起来也是异步的），轮子哥的权限库让异步过程同步化（也是kotlin协程的思想核心），怎么做到的

### 问题1：怎么隐藏Fragment来申请权限的？

怎么实现的？request方法进入，最终调用到

```Java
interceptor.launchPermissionRequest(activity, permissions, callback);


default void launchPermissionRequest(@NonNull Activity activity, @NonNull List<String> allPermissions,
                                     @Nullable OnPermissionCallback callback) {
    PermissionFragment.launch(activity, allPermissions, this, callback);
}
```

\>>>> 插入

Q：在activity中执行下面的代码，结果是怎么样的？

```Java
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();

Fragment fragment = new Fragment();

fragmentTransaction.add(fragment, "myFragment");
fragmentTransaction.commit();
```

A: 执行完以上代码之后，界面上并不会多任何东西，但如果将上面的代码换成如下：

```Java
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();

MyFragment fragment = new MyFragment();

//fragmentTransaction.add(fragment, "myFragment");
fragmentTransaction.add(R.id.container_id, fragment, "myFragment"); // 
fragmentTransaction.commit();
```

那么将会将MyFragment onCreateView 方法中返回的 View 给展示在 R.id.container_id 中

利用以上方法，就可以在Fragment中申请权限，界面不会多任何内容

```Java
PermissionFragment.launch(activity, allPermissions, this, callback);
```

这行代码实际上就是完成了这个过程

### 问题2：异步过程如何同步化的？

看到最上面的架构图，PermissionFragment 持有了

```Java
XXPermissions.with(this)
    .permission(Permission.CAMERA)
    .interceptor(new PermissionInterceptor())
    .request(new OnPermissionCallback() { ...  });
```

这里传进去的 interceptor 和 callback，在PermissionFragment中重写了onRequestPermissionsResult

```Java
@Override
public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
    // Github issue 地址：https://github.com/getActivity/XXPermissions/issues/236
    if (permissions == null || permissions.length == 0 || grantResults == null || grantResults.length == 0) {
        return;
    }

    Bundle arguments = getArguments();
    Activity activity = getActivity();
    if (activity == null || arguments == null || mInterceptor == null ||
            requestCode != arguments.getInt(REQUEST_CODE)) {
        return;
    }

    OnPermissionCallback callback = mCallBack;
    mCallBack = null;

    OnPermissionInterceptor interceptor = mInterceptor;
    mInterceptor = null;

    // 优化权限回调结果
    PermissionUtils.optimizePermissionResults(activity, permissions, grantResults);

    // 将数组转换成 ArrayList
    List<String> allPermissions = PermissionUtils.asArrayList(permissions);

    // 释放对这个请求码的占用
    REQUEST_CODE_ARRAY.remove((Integer) requestCode);
    // 将 Fragment 从 Activity 移除
    detachByActivity(activity);

    // 获取已授予的权限
    List<String> grantedPermissions = PermissionApi.getGrantedPermissions(allPermissions, grantResults);

    // 如果请求成功的权限集合大小和请求的数组一样大时证明权限已经全部授予
    if (grantedPermissions.size() == allPermissions.size()) {
        // 代表申请的所有的权限都授予了
        interceptor.grantedPermissionRequest(activity, allPermissions, grantedPermissions, true, callback);
        // 权限申请结束
        interceptor.finishPermissionRequest(activity, allPermissions, false, callback);
        return;
    }

    // 获取被拒绝的权限
    List<String> deniedPermissions = PermissionApi.getDeniedPermissions(allPermissions, grantResults);

    // 代表申请的权限中有不同意授予的，如果有某个权限被永久拒绝就返回 true 给开发人员，让开发者引导用户去设置界面开启权限
    interceptor.deniedPermissionRequest(activity, allPermissions, deniedPermissions,
            PermissionApi.isDoNotAskAgainPermissions(activity, deniedPermissions), callback);

    // 证明还有一部分权限被成功授予，回调成功接口
    if (!grantedPermissions.isEmpty()) {
        interceptor.grantedPermissionRequest(activity, allPermissions, grantedPermissions, false, callback);
    }

    // 权限申请结束
    interceptor.finishPermissionRequest(activity, allPermissions, false, callback);
}
```

上面我黄色标注的地方，就是调用interceptor 里的回调的地方。

interceptor最终也是调用callback中的对应的回调