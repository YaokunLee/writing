# 自定义路由框架详解（二）- 标记页面、收集页面

## 为什么需要路由框架？

在Android开发中，路由框架的引入主要是为了解决应用内页面跳转、组件间通信和模块化开发中的复杂性和灵活性问题。路由框架为开发者提供了一种更简洁、高效和解耦的方式来处理这些问题。以下是路由框架的几个主要优点：

1. 解耦: 在传统的Android开发中，页面间的跳转通常需要显式地声明Intent和目标Activity。这种方式使得组件之间紧密耦合，难以管理和维护。路由框架通过统一的路由表和动态绑定，允许开发者用URL-like的字符串来指代目标页面，减少了组件间的直接依赖。
2. 简化页面跳转: 使用路由框架，开发者可以轻松实现页面跳转和参数传递，只需一行代码即可完成跳转逻辑，无需手动处理Intent和Bundle，使代码更加简洁。
3. 模块化和组件化支持: 随着项目规模的扩大，模块化和组件化开发变得越来越重要。路由框架支持跨模块的页面跳转和通信，有助于实现真正的模块化开发，每个模块可以独立开发、编译、测试，最终通过路由框架动态集成。
4. 动态功能加载: 高级路由框架支持动态加载模块或功能，这对于需要按需加载资源或实现热更新功能的应用尤其重要。
5. 统一拦截和预处理: 路由框架通常提供拦截器（Interceptor）功能，允许开发者在跳转流程中插入自定义逻辑，如登录检查、权限验证、数据预加载等，增强了应用的安全性和用户体验。
6. 支持跨平台跳转: 一些路由框架设计之初就考虑到了跨平台的需求，支持从Web页面跳转到原生页面，或者反过来，这对于混合开发（Hybrid Development）尤其有用。

总之，路由框架通过提供一套中心化、灵活的机制来处理应用内的页面跳转和组件间通信，极大地提升了开发效率，降低了维护成本，并有助于实现高质量的应用架构。

## 整体方案

自定义路由框架整体上可以分为以下5步：

1. 标记页面
2. 收集页面
3. 生成文档
4. 注册映射
5. 打开页面

来详细解释一下上面五步

1. 标记页面，要求我们提供一个注解，使用这个注解标记就代表这个类将会被注册，后续会通过注解中的相关信息跳转到这个类。我们将提供一个Destination 的注解，来供使用者标记
2. 收集页面   通过注解处理器将通过Destination 注册的activity信息收集起来
3. 生成文档   在第二步中，多项目的页面被分散在不同的项目中，需要有一个文档，涵盖了所有子项目注册的页面，这样方便程序员查找activity对应的Router路径
4. 注册映射  内存中需要记录所有注册了的activity的信息，这样当业务方通过Router跳转时，才能获取到最终要跳转到的类
5. 打开页面  Router可以携带参数信息，这一步需要解析参数信息，并根据第四步中的所有的注册信息跳转到对应的activity

## 标记页面

自定义了Destination注解

```Java
@Target({ElementType.TYPE})

@Retention(RetentionPolicy.CLASS)
public @interface Destination {

    String url();

    String description();
}
```

使用方法如下：

```Java
@Destination(
        url = "router://page-home",
        description = "主页"
)
public class MainActivity extends Activity {
    ...
}
```

## 收集页面

我们希望在这一步达成的目标是，对于每个应用插件的模块，都生成一个类似下面的java文件，该文件中记录了所有在这个模块中注册了的类

```Java
package com.imooc.router.mapping;

import java.util.HashMap;
import java.util.Map;

public class RouterMapping_1708842731648 {

    public static Map<String, String> get() {

        Map<String, String> mapping = new HashMap<>();

        mapping.put("router://page-home", "com.imooc.router.demo.MainActivity");
        mapping.put("router://page-kotlin", "com.imooc.router.demo.KtMainActivity");
        mapping.put("router://imooc/profile", "com.imooc.router.demo.ProfileActivity");
        return mapping;
    }
}
```

下面是我们自定义的注解处理器的详细代码

```Java
@AutoService(Processor.class)
public class DestinationProcessor extends AbstractProcessor {

    private static final String TAG = "DestinationProcessor";

    /**
     *
     * @param set
     * @param roundEnvironment
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> set,
                           RoundEnvironment roundEnvironment) {

        //1. 有什么作用？为什么要写这个if？
        if (roundEnvironment.processingOver()) {
            return false;
        }

        System.out.println(TAG + " >>> process start ...");

        String rootDir = processingEnv.getOptions().get("root_project_dir");

        // 2. 过滤当前project获取到的被Destination注解的Element
        Set<Element> allDestinationElements = (Set<Element>)
                roundEnvironment.getElementsAnnotatedWith(Destination.class);

        System.out.println(TAG + " >>> all Destination elements count = "
                + allDestinationElements.size());


        if (allDestinationElements.size() < 1) {
            return false;
        }


        String className = "RouterMapping_" + System.currentTimeMillis();

        StringBuilder builder = new StringBuilder();
        // 3. 构建一个java类，这个java类中有一个get方法，通过get方法可以获取到该
        // project中的所有注册信息
        builder.append("package com.imooc.router.mapping;\n\n");
        builder.append("import java.util.HashMap;\n");
        builder.append("import java.util.Map;\n\n");
        builder.append("public class ").append(className).append(" {\n\n");
        builder.append("    public static Map<String, String> get() {\n\n");
        builder.append("        Map<String, String> mapping = new HashMap<>();\n\n");

        final JsonArray destinationJsonArray = new JsonArray();

        
        for (Element element : allDestinationElements) {
            // 4. 对于每个注册的类，把它放入到一个json文件中，这个json文件记录类每个注册类的
            // url、description、realPath
            final TypeElement typeElement = (TypeElement) element;


            final Destination destination =
                    typeElement.getAnnotation(Destination.class);

            if (destination == null) {
                continue;
            }

            final String url = destination.url();
            final String description = destination.description();
            final String realPath = typeElement.getQualifiedName().toString();

            System.out.println(TAG + " >>> url = " + url);
            System.out.println(TAG + " >>> description = " + description);
            System.out.println(TAG + " >>> realPath = " + realPath);

            builder.append("        ")
                    .append("mapping.put(")
                    .append("\"" + url + "\"")
                    .append(", ")
                    .append("\"" + realPath + "\"")
                    .append(");\n");

            JsonObject item = new JsonObject();
            item.addProperty("url", url);
            item.addProperty("description", description);
            item.addProperty("realPath", realPath);

            destinationJsonArray.add(item);
        }
        
```

看到上面代码的注释 1 - 4，我们一步一步解释process方法中的代码：

1. 同一个project会调用process方法两遍（不确定是不是所有project都会调用两遍，但app 模块是调用了两遍），这个if 可以在第二遍调用process方法的时候，及时推出，否则可能会导致最终生成的文件中有两遍内容。
2. 过滤当前project 获取到的被 Destination 注解的Element
3. 构建最开始说的RouterMapping_xxx.java类，这个java类中有一个get方法，通过get方法可以获取到该project中的所有注册信息
4. 对于每个注册的类，把它放入到一个json文件中，这个json文件记录类每个注册类的 url、description、realPath

上面的代码为了方便阅读只显示了一半，下面是另一半

```Java
        // 5. 接着第3步，构建java类结束
        builder.append("        return mapping;\n");
        builder.append("    }\n");
        builder.append("}\n");

        String mappingFullClassName = "com.imooc.router.mapping." + className;

        System.out.println(TAG + " >>> mappingFullClassName = "
                + mappingFullClassName);

        System.out.println(TAG + " >>> class content = \n" + builder);


        try {
            // 6.写入java类
            JavaFileObject source = processingEnv.getFiler()
                    .createSourceFile(mappingFullClassName);
            Writer writer = source.openWriter();
            writer.write(builder.toString());
            writer.flush();
            writer.close();
        } catch (Exception ex) {
            throw new RuntimeException("Error while create file", ex);
        }


        File rootDirFile = new File(rootDir);
        if (!rootDirFile.exists()) {
            throw new RuntimeException("root_project_dir not exist!");
        }


        File routerFileDir = new File(rootDirFile, "router_mapping");
        if (!routerFileDir.exists()) {
            routerFileDir.mkdir();
        }

        File mappingFile = new File(routerFileDir,
                "mapping_" + System.currentTimeMillis() + ".json");


        try {
            // 7. 写入json文件
            BufferedWriter out = new BufferedWriter(new FileWriter(mappingFile));
            String jsonStr = destinationJsonArray.toString();
            out.write(jsonStr);
            out.flush();
            out.close();
        } catch (Throwable throwable) {
            throw new RuntimeException("Error while writing json", throwable);
        }

        System.out.println(TAG + " >>> process finish.");

        return false;
    }

    /**
     *
     * @return
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(
                Destination.class.getCanonicalName()
        );
    }
}
```

1. 接着第3步，构建java类结束
2. 写入java文件
3. 写入json文件