## 自定义路由框架详解（二）

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

## 生成文档

这一步需要把所有projects 生成的RouterMapping_xxxx.java类汇总起来，生成一个记录了所有注册的类的文档。

我们在buildSrc插件中，实现了这一步。

```Java
import com.android.build.api.transform.Transform
import com.android.build.gradle.AppExtension
import com.android.build.gradle.AppPlugin
import org.gradle.api.Plugin
import org.gradle.api.Project
import groovy.json.JsonSlurper
import org.gradle.api.artifacts.transform.TransformAction

class RouterPlugin implements Plugin<Project> {

    // 实现apply方法，注入插件的逻辑
    void apply(Project project) {

        // 1. 注册 Transform
        if (project.plugins.hasPlugin(AppPlugin)) {
            AppExtension appExtension = project.extensions.getByType(AppExtension)
            Transform transform = new RouterMappingTransform()
            appExtension.registerTransform(transform)
        }

        // 2. 自动帮助用户传递路径参数到注解处理器中
        if (project.extensions.findByName("kapt") != null) {
            project.extensions.findByName("kapt").arguments {
                arg("root_project_dir", project.rootProject.projectDir.absolutePath)
            }
        }
        // 3. 实现旧的构建产物的自动清理
        project.clean.doFirst {
            // 删除 上一次构建生成的 router_mapping目录
            File routerMappingDir =
                    new File(project.rootProject.projectDir, "router_mapping")

            if (routerMappingDir.exists()) {
                routerMappingDir.deleteDir()
            }
        }

        if (!project.plugins.hasPlugin(AppPlugin)) {
            return
        }
        println("I am from RouterPlugin, apply from ${project.name}")
        project.getExtensions().create("router", RouterExtension)

        project.afterEvaluate {

            RouterExtension extension = project["router"]

            println("用户设置的WIKI路径为 : ${extension.wikiDir}")
            
            // 4. 在javac任务 (compileDebugJavaWithJavac) 后，汇总生成文档
            // 这里为什么要过滤以compile开头，以JavaWithJavac结尾的task呢？
            project.tasks.findAll { task ->
                task.name.startsWith('compile') &&
                        task.name.endsWith('JavaWithJavac')
            }.each { task ->
                
                task.doLast {
                    File routerMappingDir =
                            new File(project.rootProject.projectDir,
                                    "router_mapping")

                    if (!routerMappingDir.exists()) {
                        return
                    }
                    File[] allChildFiles = routerMappingDir.listFiles()

                    if (allChildFiles.length < 1) {
                        return
                    }

                    StringBuilder markdownBuilder = new StringBuilder()
                    markdownBuilder.append("# 页面文档\n\n")
                    
                    allChildFiles.each { child ->

                        if (child.name.endsWith(".json")) {

                            JsonSlurper jsonSlurper = new JsonSlurper()
                            def content = jsonSlurper.parse(child)
                            
                            content.each { innerContent ->
                                def url = innerContent['url']
                                def description = innerContent['description']
                                def realPath = innerContent['realPath']
                                markdownBuilder.append("## $description \n")
                                markdownBuilder.append("- url: $url \n")
                                markdownBuilder.append("- realPath: $realPath \n\n")
                            }
                        }
                    }

                    File wikiFileDir = new File(extension.wikiDir)
                    if (!wikiFileDir.exists()) {
                        wikiFileDir.mkdir()
                    }

                    File wikiFile = new File(wikiFileDir, "页面文档.md")
                    if (wikiFile.exists()) {
                        wikiFile.delete()
                    }
                    
                    wikiFile.write(markdownBuilder.toString())

                }
            }
        }
    }
}
```

上面代码注释处为关键处，下面是它们的解释

1. 注册 Transform：
   1.  这部分代码检查项目是否已应用了`AppPlugin`。如果是，它会获取项目的`AppExtension`实例，并注册一个自定义的`Transform`，名为`RouterMappingTransform`。`Transform`是Android Gradle插件提供的一个API，允许开发者在编译时修改字节码，`RouterMappingTransform`可能用于拦截和处理编译过程中的类文件，例如用于修改、添加注解等操作。
2. 自动传递路径参数到注解处理器中：
   1.  这段代码检查项目是否配置了`kapt`（Kotlin注解处理工具）。如果配置了，它会自动将项目根目录的绝对路径作为参数`root_project_dir`传递给注解处理器。这样，注解处理器在运行时就能使用这个路径信息，例如用于确定生成文件的位置。
3. 实现旧的构建产物的自动清理：
   1.  在项目的`clean`任务执行前，这段代码会自动删除`router_mapping`目录。这个目录可能包含了上一次构建过程中生成的一些文件或文档，通过删除这个目录，可以确保每次构建都是从干净的状态开始，避免因旧文件残留导致的潜在问题。
4. 在`javac`任务后汇总生成文档：
   1.  这部分代码为什么要过滤以`compile`开头，以`JavaWithJavac`结尾的任务？
   2.  因为这样可以精确地定位到Java编译任务。在Android项目中，通常有多个变体（比如debug和release），每个变体都会有自己的编译Java代码的任务（如`compileDebugJavaWithJavac`、`compileReleaseJavaWithJavac`等）。这段代码监听这些任务的完成，然后执行自定义的逻辑：它读取`router_mapping`目录下的所有JSON文件，从中提取URL、描述和实际路径等信息，将这些信息汇总成Markdown格式的文档，并保存在用户配置的`wikiDir`目录下。这样做的目的是自动化文档生成过程，为项目中的页面或功能提供一个易于阅读的参考文档，便于开发者和其他利益相关者理解和使用这些功能。

## 注册映射

### 第一步：实现RouterMappingTransform

1. 定义Transform：`RouterMappingTransform`类需要继承自`Transform`类，并重写相关方法，例如`getName`、`getInputTypes`、`getScopes`以及`isIncremental`。这些方法定义了Transform的基本属性和行为。
2. 处理输入：在`transform`方法中，遍历所有的输入文件（包括目录输入和JAR输入），并对它们进行处理。这里的处理可能包括复制文件、修改字节码或生成新的字节码文件。
3. 生成映射类：使用`RouterMappingByteCodeBuilder`来生成映射类的字节码。这个类将包含所有路由的映射信息，以便在运行时进行查询。

在`RouterMappingTransform`类中，关键的代码如下：

```Groovy
@Override
String getName() {
    return "RouterMappingTransform"
}

@Override
Set<QualifiedContent.ContentType> getInputTypes() {
    return TransformManager.CONTENT_CLASS
}

@Override
Set<? super QualifiedContent.Scope> getScopes() {
    return TransformManager.SCOPE_FULL_PROJECT
}

@Override
boolean isIncremental() {
    return false
}

@Override
void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
    // 初始化映射收集器
    RouterMappingCollector collector = new RouterMappingCollector()

    transformInvocation.inputs.each {
        // 处理文件夹和JAR输入
        it.directoryInputs.each { directoryInput ->
            // ...收集和复制逻辑
            collector.collect(directoryInput.file)
        }
        it.jarInputs.each { jarInput ->
            // ...收集和复制逻辑
            collector.collectFromJarFile(jarInput.file)
        }
    }

    // ...生成映射类字节码并写入JAR
}
```

### 第二步：创建RouterMappingByteCodeBuilder

1. 生成字节码：`RouterMappingByteCodeBuilder`利用ASM库来动态生成字节码。这里，你需要定义映射类的结构，包括构造方法和`get`方法。`get`方法将返回一个包含所有映射信息的`Map`。
2. 添加映射信息：在`get`方法的实现中，动态地将所有收集到的映射信息添加到`Map`中并返回。

`RouterMappingByteCodeBuilder`利用ASM库生成映射类的字节码，关键代码如下：

```Groovy
static byte[] get(Set<String> allMappingNames) {
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS)
    // 定义类、构造方法和get方法
    // get方法中，创建HashMap并填入映射信息
    allMappingNames.each {
        // 向Map中添加映射信息
    }
    return cw.toByteArray()
}
```

### 第三步：收集映射信息

1. 定义RouterMappingCollector：这个类负责从项目的编译输入中收集所有的路由映射信息。它会检查所有的类文件，寻找符合特定命名规则的类，并收集这些类的名称。
2. 处理类文件和JAR：`RouterMappingCollector`需要能够处理单个类文件和包含在JAR文件中的类文件。对于每个找到的映射类，将其名称添加到一个集合中，以便后续生成映射类时使用。

`RouterMappingCollector`类用于从编译输入中收集映射信息，关键代码如下：

```Groovy
void collect(File classFile) {
    // 判断是否为映射类文件，并收集
}

void collectFromJarFile(File jarFile) {
    // 从JAR文件中收集映射类信息
}
```

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