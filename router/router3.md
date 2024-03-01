# 自定义路由框架详解（三）- 生成文档



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