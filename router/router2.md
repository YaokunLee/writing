# 自定义路由框架详解（二）- 标记页面、收集页面



自定义路由框架整体上可以分为以下5步：

1. 标记页面
2. 收集页面
3. 生成文档
4. 注册映射
5. 打开页面

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