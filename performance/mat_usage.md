# JVM 内存分析工具 MAT 的深度讲解与实践——入门篇

https://juejin.cn/post/6908665391136899079

JVM 内存分析往往由团队较资深的同学来做，本系列通过3篇文章，深度解析并帮助读者全面深度掌握 MAT 的使用方法。即使没有 JVM 内存分析的实践经验，也能快速成为内存分析高手！

**本系列共三篇文章如下，** 本文是第一篇入门篇：

- **《JVM 内存分析工具 MAT 的深度讲解与实践——入门篇》** 介绍 MAT 产品功能、基础概念、与其他工具对比、Quick Start 指南。
- **《JVM 内存分析工具 MAT 的深度讲解与实践——进阶篇》** 展开并详细介绍 MAT 的核心功能，并在具体实战场景下讲解帮大家加深体会。
- **《JVM 内存分析工具 MAT 的深度讲解与实践——高阶篇》** 总结复杂内存问题的系统性分析方法，并通过一个综合案例提升大家的实战能力。

1. **MAT 工具简介**

MAT（全名：Memory Analyzer Tool），是一款快速便捷且功能强大丰富的 JVM 堆内存离线分析工具。其通过展现 JVM 异常时所记录的运行时堆转储快照（Heap dump）状态（正常运行时也可以做堆转储分析），帮助定位内存泄漏问题或优化大内存消耗逻辑。

## **1.1 MAT 使用场景及主要解决问题**

场景一：内存溢出，JVM堆区或方法区放不下存活及待申请的对象。如：高峰期系统出现 OOM（Out of Memory）异常，需定位内存瓶颈点来指导优化。

场景二：内存泄漏，不会再使用的对象无法被垃圾回收器回收。如：系统运行一段时间后出现 Full GC，甚至周期性 OOM 后需人工重启解决。

场景三：内存占用高。如：系统频繁 GC ，需定位影响服务实时性、稳定性、吞吐能力的原因。

## **1.2 基础概念**

### **1.2.1 Heap Dump**

Heap Dump 是 Java 进程堆内存在一个时间点的快照，支持 HPROF 及 DTFJ 格式，前者由 Oracle 系列 JVM 生成，后者是 IBM 系列 JVM 生成。其内容主要包含以下几类：

- 所有对象的实例信息：对象所属类名、基础类型和引用类型的属性等。
- 所有类信息：类加载器、类名、继承关系、静态属性等。
- GC Root：GC Root 代表通过可达性分析来判定 JVM 对象是否存活的起始集合。JVM 采用追踪式垃圾回收（Tracing GC）模式，**从所有 GC Roots 出发通过引用关系可以关联的对象**就是存活的（且不可回收），其余的不可达的对象（Unreachable object：如果无法从 GC Root 找到一条引用路径能到达某对象，则该对象为Unreachable object）可以回收。
- 线程栈及局部变量：快照生成时刻的所有线程的线程栈帧，以及每个线程栈的局部变量。

### **1.2.2 Shallow Heap**

Shallow Heap 代表一个**对象结构自身**所占用的内存大小，不包括其属性引用对象所占的内存。如 java.util.ArrayList 对象的 Shallow Heap 包含8字节的对象头、8字节的对象数组属性 elementData 引用 、 4字节的 size 属性、4字节的 modCount 属性（从 AbstractList 继承及对象头占用内存大小），有的对象可能需要加对齐填充但 ArrayList 自身已对齐不需补充，注意不包含 elementData 具体数据占用的内存大小。

### **1.2.3 Retained Set**

一个对象的 Retained Set，指的是该对象被 GC 回收后，所有能被回收的对象集合（如下图所示，G的 Retain Set 只有 G 并不包含 H，原因是虽然 H 也被 G 引用，但由于 H 也被 F 引用 ，G 被垃圾回收时无法释放 H）；另外，当该对象无法被 GC 回收，则其 Retained set 也必然无法被 GC 回收。

### **1.2.4 Retained Heap**

Retained Heap 是一个对象被 GC 回收后，可释放的内存大小，等于释放对象的 Retained Heap 中所有对象的 Shallow Heap 的和（如下图所示，E 的 Retain Heap 就是 G 与 E 的 Shallow Heap 总和，同理不包含 H）。

### **1.2.5 Dominator tree**

如果所有指向对象 Y 的路径都经过对象 X，则 X 支配（dominate） Y（如下图中，C、D 均支配 F，但 G 并不支配 H）。Dominator tree 是根据对象引用及支配关系生成的整体树状图，支配树清晰描述了对象间的依赖关系，下图左的 Dominator tree 如下图右下方支配树示意图所示。支配关系还有如下关系：

- Dominator tree 中任一节点的子树就是被该节点支配的节点集合，也就是其 Retain Set。
- 如果 X 直接支配 Y，则 X 的所有支配节点均支配 Y。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NTZkODY4NDUyZDA5YjFlZTdmN2FiYWNhZmNlZjIyZDRfa3J5MVJ3ZGlNczVZbjNEaFU1VktwZzdNWjVEYmowN3ZfVG9rZW46R1pacGJBVUxTb2ZXZnN4OGN5UmNUZlVwbkxiXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

### **1.2.6 OQL**

OQL 是类似于 SQL 的 MAT 专用统一查询语言，可以根据复杂的查询条件对 dump 文件中的类或者对象等数据进行查询筛选。

### **1.2.7 references**

outgoing references、incoming references 可以直击对象间依赖关系，MAT 也提供了链式快速操作。

- outgoing references：对象引用的外部对象（注意不包含对象的基本类型属性。基本属性内容可在 inspector 查看）。
- incoming references：直接引用了当前对象的对象，每个对象的 incoming references 可能有 0 到多个。

1. **MAT 功能概述及对比**

## **2.1 MAT 功能概述**

*注：MAT 的产品能力非常丰富，本文简要总结产品特性帮大家了解全貌，在下一篇文章《JVM 内存分析实战进阶篇——核心功能及应用场景》中，会详细展开介绍各项核心功能的场景、案例、最佳实践等。*

MAT 的工作原理是对 dump 文件建立多种索引，并基于索引来实现 [1]内存分布、[2]对象间依赖（如实体对象引用关系、线程引用关系、ClassLoader引用关系等）、[3]对象状态（内存占用量、字段属性值等）、[4]条件检索（OQL、正则匹配查询等）这四大核心功能，并通过可视化展现辅助 Developer 精细化了解 JVM 堆内存全貌。

### **2.1.1 内存分布**

- 全局概览信息：堆内存大小、对象个数、类的个数、类加载器的个数、GC root 个数、线程概况等全局统计信息。
- Dominator tree：按对象的 Retain Heap 排序，也支持按多个维度聚类统计，最常用的功能之一。
- Histogram：罗列每个类实例的内存占比，包括自身内存占用量（Shallow Heap）及支配对象的内存占用量（Retain Heap），支持按 package、class loader、super class、class 聚类统计，最常用的功能之一。
- Leak Suspects：直击引用链条上占用内存较多的可疑对象，可解决一些基础问题，但复杂的问题往往帮助有限。
- Top Consumers：展现哪些类、哪些 class loader、哪些 package 占用最高比例的内存。

### **2.1.2 对象间依赖**

- References：提供对象的外部引用关系、被引用关系。通过任一对象的直接引用及间接引用详情（主要是属性值及内存占用），进而提供完善的依赖链路详情。
- Dominator tree：支持按对象的 Retain Heap 排序，并提供详细的支配关系，结合 references 可以实现大对象快速关联分析；
- Thread overview：展现转储 dump 文件时线程栈帧等详细状态，也提供各线程的 Retain Heap 等关联内存信息。
- Path To GC Roots：提供任一对象到GC Root的链路详情，帮助了解不能被 GC 回收的原因。

### **2.1.3 对象状态**

- 最核心的是通过 inspector 面板提供对象的属性信息、类继承关系信息等数据，协助分析内存占用高与业务逻辑的关系。
- 集合状态的检测，如：通过 ArrayList 或数组的填充率定位空集合空数组造成的内存浪费、通过 HashMap 冲突率判定 hash 策略是否合理等。

### **2.1.4 按条件检索对象**

- OQL：提供一种类似于SQL的对象（类）级别统一结构化查询语言。如：查找 size＝0 且未使用过的 ArrayList：  select * from java.util.ArrayList where size=0 and modCount=0；查找所有的String的length属性的：  select s.length from instanceof String s。
- 内存分布及对象间依赖的众多功能，均支持按字符串检索、按正则检索等操作。
- 按虚拟内存地址寻址，根据对象的十六进制地址查找对象。

此外，为了便于记忆与回顾，整理了如下脑图： 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=OWM1MDdmOTMzNzg0ZGU4YzBjZjZkZTE5MGFlMWU0MzZfQUl3ZVJ1ZnVsdVM3NktGR2hKWHNIeFJNdjdKNHd3ZFdfVG9rZW46WnZsZWJvQWlEbzI1Tnd4SEhuTGNSR053bmNkXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

作者：Q的博客

链接：https://juejin.cn/post/6908665391136899079

来源：稀土掘金

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

# JVM 内存分析工具 MAT 的深度讲解与实践——进阶篇

链接：https://juejin.cn/post/6911624328472133646

1. **前言**

熟练掌握 MAT 是 Java 高手的必备能力，但实践时大家往往需面对众多功能，眼花缭乱不知如何下手，小编也没有找到一篇完善的教学素材，所以整理本文帮大家系统掌握 MAT 分析工具。

本文详细讲解 MAT 众多内存分析工具功能，这些功能组合使用异常强大，熟练使用几乎可以解决所有的堆内存离线分析的问题。我们将功能划分为4类：内存分布详情、对象间依赖、对象状态详情、按条件检索。每大类有多个功能点，本文会逐一讲解各功能的场景及用法。此外，添加了原创或引用案例加强理解和掌握。

注：在该系列开篇文章《[JVM 内存分析工具 MAT 的深度讲解与实践——入门篇](https://juejin.cn/post/6908665391136899079)中介绍了 MAT 的使用场景及安装方法，不熟悉 MAT 的读者建议先阅读上文并安装，本文案例很容易在本地实践。另外，上文中产品介绍部分顺序也对应本文功能讲解的组织，如下图： 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MGM0NDA4Y2IzYTcxN2Y3YTdhNDU4NTdiMTM2OWM0MzVfeXBUa21oeW1kd2ZsMXpWZjhROEZOUjB2MDc4SThQTHdfVG9rZW46TFdmb2JVN29sb1ZRUEJ4UGxTdGNMZFJJbkZlXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

 为减少对眼花缭乱的菜单的迷茫，可以通过下图先整体熟悉下各功能使用入口，后续都会讲到。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NzU5OTJmNDNiOWVmOGQ4YjBmYThhOTRkMTRmMWU2MWZfR0FKcHR4V1hmSWl4ZWFBeXFPZ3ROdnBSQ0hTNTQxU1lfVG9rZW46SFFUdGJRNUhCbzdoOXF4bFpYVGNFbGsxbmdlXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. **内存分布详解及实战**

## **2.1 全局信息概览**

**功能**：展现堆内存大小、对象数量、class 数量、class loader 数量、GC Root 数量、环境变量、线程概况等全局统计信息。

**使用入口**：MAT 主界面 → Heap Dump Overview。

**举例**：下面是对象数量、class loader 数量、GC Root 数量，可以看出 class loader 存在异常。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=OWFmZjg1YzgwZTI5NDdlODkzYjUxOTFkMGMxZWExMjlfZlhCSFpBMWgzWVdEdWNXejhTekxKclVkZmR2b0tsdkFfVG9rZW46VGpoS2JmOWRzb0xseHJ4enRONGNpSlhHbm1lXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

**举例**：下图是线程概况，可以查看每个线程名、线程的 Retained Heap、daemon 属性等。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=M2FiMjc3MGU5MDQ5Nzg3NWNiZDg4YTg5ZGJlY2JjY2FfV0V6QTdBU1pScTJCRWlQam5nS29yc01RenFwOVM0OEVfVG9rZW46WTFGT2I3TUJjb29UakR4elJNV2NJcW5sbm1ZXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

**使用场景** 全局概览呈现全局统计信息，重点查看整体是否有异常数据，所以有效信息有限，下面几种场景有一定帮助：

- 方法区溢出时（Java 8后不使用方法区，对应堆溢出），查看 class 数量异常多，可以考虑是否为动态代理类异常载入过多或类被反复重复加载。
- 方法区溢出时，查看 class loader 数量过多，可以考虑是否为自定义 class loader 被异常循环使用。
- GC Root 过多，可以查看 GC Root 分布，理论上这种情况极少会遇到，笔者只在 JNI 使用一个存在 BUG 的库时遇到过。
- 线程数过多，一般是频繁创建线程但无法执行结束，从概览可以了解异常表象，具体原因可以参考本文线程分析部分内容，此处不展开。

## **2.2 Dominator tree**

***注：笔者使用频率的 Top1，是高效分析 Dump 必看的功能。***

**功能**

- 展现对象的支配关系图，并给出对象支配内存的大小（支配内存等同于 Retained Heap，即其被 GC 回收可释放的内存大小）
- 支持排序、支持按 package、class loader、super class、class 聚类统计

**使用入口**：全局支配树： MAT 主界面 → Dominator tree。

**举例：** 下图中通过查看 Dominator tree，了解到内存主要是由 ThreadAndListHolder-thread 及 main 两个线程支配（后面第2.6节会给出整体案例）。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTQ5OTJiNDdjNDUxMWY5OWM1OTU1MDY0NWZhYTZlYmVfTEo0WERCWU1hTHNOTFBudENxVGxWMW1DV2E2bUFxSWJfVG9rZW46S3Q5WGIwNGtVb29taER4eHFPd2NDR0w5bldkXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

**使用场景**

- 开始 Dump 分析时，首先应使用 Dominator tree 了解各支配树起点对象所支配内存的大小，进而了解哪几个起点对象是 GC 无法释放大内存的原因。
- 当个别对象支配树的 Retained Heap 很大存在明显倾斜时，可以重点分析占比高的对象支配关系，展开子树进一步定位到问题根因，如下图中可看出最终是 SameContentWrapperContainer 对象持有的 ArrayList 过大。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NjlhZTMwODkwNDczYTZkMzkwMzk3NjY5YjNlN2FmNzFfd0dxaTdBR1N5aWY5V3lwdU42cXo0aWZxUnNiTWtHdFlfVG9rZW46WElyN2JxSGdkb0I4VlJ4Q3RCdWNYTHhzbjdnXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

- 在 Dominator tree 中展开树状图，可以查看支配关系路径（与 outgoing reference 的区别是：如果 X 支配 Y，则 X 释放后 Y必然可释放；如果仅仅是 X 引用 Y，可能仍有其他对象引用 Y，X 释放后 Y 仍不能释放，所以 Dominator tree 去除了 incoming reference 中大量的冗余信息）。
- 有些情况下可能并没有支配起点对象的 Retained Heap 占用很大内存（比如 class X 有100个对象，每个对象的 Retained Heap 是10M，则 class X 所有对象实际支配的内存是 1G，但可能 Dominator tree 的前20个都是其他class 的对象），这时可以按 class、package、class loader 做聚合，进而定位目标。
  - 下图中各 GC Roots 所支配的内存均不大，这时需要聚合定位爆发点。 
  - ![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NDNlN2ViZTdiMzgzMTJjZjMxZjlhODEyNTFlM2UxZjFfRFI3YTlpT2lNUmlDaXpsU29EaEpWU1NieUMyZzl6aVZfVG9rZW46T1dnMGJkNXRmb0s2ODJ4RmZQeWN0aUdibldoXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)
  - 在 Dominator tree 展现后按 class 聚合，如下图： 
  - ![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NmE5OGZhMDQ2ZDM2YWFmNWZlNThjNDI0M2FmZTE3OWZfbEpieFlkMzdMQjQzVkdLWWJndExnUFd4ZzdqSVg2TFlfVG9rZW46TzdNWWJJbjk4bzE2N0p4Tmh3amNtYmhibnJjXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)
  - 可以定位到是 SomeEntry 对象支配内存较多，然后结合代码进一步分析具体原因。 
  - ![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MWE0YWVkNDRhNzRlOThkZWNlNmQzOTQ0MjgwZTU1YmVfb0pXOTJ5RWowekhxZjZNSzlZMk14eXZURVg5MFA1R09fVG9rZW46RThtYmJFRGdNbzJCNG94ME43UGNnb1lDbktjXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)
- 在一些操作后定位到异常持有 Retained Heap 对象后（如从代码看对象应该被回收），可以获取对象的直接支配者，操作方式如下。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=YTBiMDczOTBkMDNlNjk2ODc4ZjljMGM3NjhjZDVlOTJfMWJBQ0tkcjNVQlBsQzI0T2NLWVZUa0hybENMVWlaY3RfVG9rZW46TzY5OGJKR292b3pGZTJ4WFNQY2NjVUdkbnFiXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

## **2.3 Histogram 直方图**

***注：笔者使用频率 Top2***

**功能**

- 罗列每个类实例的数量、类实例累计内存占比，包括自身内存占用量（Shallow Heap）及支配对象的内存占用量（Retain Heap）。
- 支持按对象数量、Retained Heap、Shallow Heap（默认排序）等指标排序；支持按正则过滤；支持按 package、class loader、super class、class 聚类统计，

**使用入口**：MAT 主界面 → Histogram；注意 Histogram 默认不展现 Retained Heap，可以使用计算器图标计算，如下图所示。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2M5YzA1ZTZjYzFhYTQyM2MwNzRhNjBiMzdkNTVkN2FfU3lZRnBxbGNoeENnUW12cm9Vb0hxdHpyS1VURmw1QnVfVG9rZW46TmhQUmJWWDF6b2ZSTWt4UjN2T2NyME5lbmRlXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

**使用场景**

- 有些情况 Dominator tree 无法展现出热点对象（上文提到 Dominator tree 支配内存排名前20的占比均不高，或者按 class 聚合也无明显热点对象，此时 Dominator tree 很难做关联分析判断哪类对象占比高），这时可以使用 Histogram 查看所有对象所属类的分布，快速定位占据 Retained Heap 大头的类。
- 使用技巧 
  - Integer，String 和 Object[] 一般不直接导致内存问题。为更好的组织视图，可以通过 class loader 或 package 分组进一步聚焦，如下图。 
  - ![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MmJlNTI5NjVjNDExMjRiOWE1MjQ5MzE3MmU0Yjk2MWZfQkpUVnVkM1lYaUFoY04ySXNXQXp2Q1U0cGZEYnhQbWtfVG9rZW46WlFISmJxamdyb3RaNUN4MjhaY2NNR1ZWbkVmXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)
  - Histogram 支持使用正则表达式来过滤。例如，我们可以只展示那些匹配com.q.*的类。 
  - ![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=YzkzY2QzZTQzN2FkZjA0NWE3NDY5NGY2Y2ZmYjYzNWFfMEJ2TEtONkNxTGlsSkIyYTh1RlhLREpwVkdtSWtjbk1fVG9rZW46TjR3UmJOZDdXb2tybWF4ejdWNWNGZ0R2bmJkXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)
  - 可以在 Histogram 的某个类继续使用 outgoing reference 查看对象分布，进而定位哪些对象是大头 
  - ![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NWU2ZDUyMTlkZmM4Y2U2YjE1MzFmNjJhMjkwOThhNTlfS3BVOFM3Q2lzVjdtTTU3aUpET29uZjY3WmpXZ1YwNm5fVG9rZW46Q1htdmJ0anZKb1V3NjB4a0lhSmN6MGdZbjFHXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

## **2.4 Leak Suspects**

**功能**：具备自动检测内存泄漏功能，罗列可能存在内存泄漏的问题点。

**使用入口**：一般当存在明显的内存泄漏时，分析完Dump文件后就会展现，也可以如下图在 MAT 主页 → Leak Suspects。

**使用场景**：需要查看引用链条上占用内存较多的可疑对象。这个功能可解决一些基础问题，但复杂的问题往往帮助有限。

**举例**

- 下图中 Leak Suspects 视图展现了两个线程支配了绝大部分内存。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmJmNmMzOGQwYjAzMjYwZjNjMzZjNWY3YTg3NzQxYTFfakY5NkR1OTQ4Z0RoOHVQZnBheWdUQ2h4VlVNRmxlNWlfVG9rZW46TU80aWJqbGNCb0lZdXN4dGdNaGNGbXE5bmdlXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

- 下图是点击上图中 Keywords 中 "Details" ,获取实例到 GC Root 的最短路径、dominator 路径的细信息。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=Yjg4YWVmOWYwNjMxNzFiMzgwNzY2ODc1ZjY2M2Y0NGRfSmRsYUE2Q1lCd2d1OFlZZmhsaTBOMzIzZGczbUI4akxfVG9rZW46RFFCVWJOZm9vb2E1cm94VzNjT2NRMWpPbjZnXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

## **2.5 Top Consumers**

**功能**：最大对象报告，可以展现哪些类、哪些 class loader、哪些 package 占用最高比例的内存，其功能 Histogram 及 Dominator tree 也都支持。

**使用场景**：应用程序发生内存泄漏时，查看哪些泄漏的对象通常在 Dump 快照中会占很大的比重。因此，对简单的问题具有较高的价值。

## **2.6 综合案例一**

**使用工具项**：Heap dump overview、Dominator tree、Histogram、Class Loader Explorer（见3.4节）、incoming references（见3.1节）

**程序代码**

```Java
package com.q.mat;

import java.util.*;
import org.objectweb.asm.*;

public class ClassLoaderOOMOps extends ClassLoader implements Opcodes {

    public static void main(final String args[]) throws Exception {
        new ThreadAndListHolder(); // ThreadAndListHolder 类中会加载大对象

        List<ClassLoader> classLoaders = new ArrayList<ClassLoader>();
        final String className = "ClassLoaderOOMExample";
        final byte[] code = geneDynamicClassBytes(className);

        // 循环创建自定义 class loader，并加载 ClassLoaderOOMExample
        while (true) {
            ClassLoaderOOMOps loader = new ClassLoaderOOMOps();
            Class<?> exampleClass = loader.defineClass(className, code, 0, code.length); //将二进制流加载到内存中
            classLoaders.add(loader);
            // exampleClass.getMethods()[0].invoke(null, new Object[]{null});  // 执行自动加载类的方法，通过反射调用main
        }
    }

    private static byte[] geneDynamicClassBytes(String className) throws Exception {
        ClassWriter cw = new ClassWriter(0);
        cw.visit(V1_1, ACC_PUBLIC, className, null, "java/lang/Object", null);

        //生成默认构造方法
        MethodVisitor mw = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);

        //生成构造方法的字节码指令
        mw.visitVarInsn(ALOAD, 0);
        mw.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V");
        mw.visitInsn(RETURN);
        mw.visitMaxs(1, 1);
        mw.visitEnd();

        //生成main方法
        mw = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "main", "([Ljava/lang/String;)V", null, null);
        //生成main方法中的字节码指令
        mw.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");

        mw.visitLdcInsn("Hello world!");
        mw.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V");
        mw.visitInsn(RETURN);
        mw.visitMaxs(2, 2);
        mw.visitEnd();  //字节码生成完成

        return cw.toByteArray();  // 获取生成的class文件对应的二进制流

    }
}
package com.q.mat;

import java.util.*;
import org.objectweb.asm.*;

public class ThreadAndListHolder extends ClassLoader implements Opcodes {
    private static Thread innerThread1;
    private static Thread innerThread2;
    private static final SameContentWrapperContainerProxy sameContentWrapperContainerProxy = new SameContentWrapperContainerProxy();

    static {
        // 启用两个线程作为 GC Roots
        innerThread1 = new Thread(new Runnable() {
            public void run() {
                SameContentWrapperContainerProxy proxy = sameContentWrapperContainerProxy;
                try {
                    Thread.sleep(60 * 60 * 1000);
                } catch (Exception e) {
                    System.exit(1);
                }
            }
        });
        innerThread1.setName("ThreadAndListHolder-thread-1");
        innerThread1.start();

        innerThread2 = new Thread(new Runnable() {
            public void run() {
                SameContentWrapperContainerProxy proxy = proxy = sameContentWrapperContainerProxy;
                try {
                    Thread.sleep(60 * 60 * 1000);
                } catch (Exception e) {
                    System.exit(1);
                }
            }
        });
        innerThread2.setName("ThreadAndListHolder-thread-2");
        innerThread2.start();
    }
}

class IntArrayListWrapper {
    private ArrayList<Integer> list;
    private String name;

    public IntArrayListWrapper(ArrayList<Integer> list, String name) {
        this.list = list;
        this.name = name;
    }
}

class SameContentWrapperContainer {
    // 2个Wrapper内部指向同一个 ArrayList，方便学习 Dominator tree
    IntArrayListWrapper intArrayListWrapper1;
    IntArrayListWrapper intArrayListWrapper2;

    public void init() {
        // 线程直接支配 arrayList，两个 IntArrayListWrapper 均不支配 arrayList，只能线程运行完回收
        ArrayList<Integer> arrayList = generateSeqIntList(10 * 1000 * 1000, 0);
        intArrayListWrapper1 = new IntArrayListWrapper(arrayList, "IntArrayListWrapper-1");
        intArrayListWrapper2 = new IntArrayListWrapper(arrayList, "IntArrayListWrapper-2");
    }

    private static ArrayList<Integer> generateSeqIntList(int size, int startValue) {
        ArrayList<Integer> list = new ArrayList<Integer>(size);
        for (int i = startValue; i < startValue + size; i++) {
            list.add(i);
        }
        return list;
    }
}

class SameContentWrapperContainerProxy {
    SameContentWrapperContainer sameContentWrapperContainer;

    public SameContentWrapperContainerProxy() {
        SameContentWrapperContainer container = new SameContentWrapperContainer();
        container.init();
        sameContentWrapperContainer = container;
    }
}
启动参数：-Xmx512m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/gjd/Desktop/dump/heapdump.hprof -XX:-UseCompressedClassPointers -XX:-UseCompressedOops
```

**引用关系图**

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ODZiZWI0MmJhMzY0YzY0MjNmM2UzZTUxMjkxNGZhYjRfa2Vic0hUV011Z05BS0JTS1hSaG9FYkZNQUVXemVwSmxfVG9rZW46RmVKY2I1ZEYyb09HcjV4emN3NWNZNE1ybjRmXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

**分析过程**

1. 首先进入 Dominator tree，可以看出是 SameContentWrapperContainerProxy 对象与 main 线程两者持有99%内存不能释放导致 OOM。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MWMxYjNkYmEzNTRjNGRlOTM0YWE1MThkMDExMDJhMzhfMG5RUkFFNWg0dUJjQ0p1N09FRkkxZWpkUnptVlBHSGtfVG9rZW46RXNRWmJpS0ZJb3Z0dXp4enV3ZWNhR1VLbllkXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MGI5NGVkMTgwZDViNzA3ZTc4YWJjOWJlNmY1MjNhMDFfczBaS3FON1VQVjQ1em52amV0WWg1SGlNQ3RrZTVLaTdfVG9rZW46UUhWOGJFSnNzb1o3UjR4Y2x3eWNBdTQxbnFjXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 先来看方向一，在 Heap Dump Overview 中可以快速定位到 Number of class loaders 数达50万以上，这种基本属于异常情况，如下图所示。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MDc5ZDA2OTc2MzRiOTMxY2EwYjg0YjM0ZDJmNDkyZDdfb0hFajVrM3BXMVozOXpiNVl1UEZnWkk4OXRvY0tmdXpfVG9rZW46TjExUWJDNW9yb29CS3p4VDRvdmNqSVVJbllMXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 使用 Class Loader Explorer 分析工具，此时会展现类加载详情，可以看到有524061个 class loader。我们的案例中仅有ClassLoaderOOMOps 这样的自定义类加载器，所以很快可以定位到问题。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=N2RkYzMwYjkzNjg5NzRiZmU2MzhiN2Y4MDk3YzQzNmFfcGNEME5mbGxlZXhSZXZsTVI0VE5ncjBnakNNY01jNWRfVG9rZW46SHBFQWJSOVVwb1Nlc1B4MDdGWGNpOWl5bjliXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDcwYjJkOGNmMzViN2YyNDJlOWQ1OTMzOTBiYzMzMmJfZ2hPUHVpVkdqeDFxaURzeWp5bWNPWk5zaFBTSldUMzNfVG9rZW46R0xWQWJPck5Xb2xNMW94UWU3N2N5azRmbnJEXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 如果类加载器较多，不能确定是哪个引发问题，则可以将所有的 class loader对象按类做聚类，如下图所示。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NTVmM2U4YTUyNGFlNWI3NWE0ZWE0ZWNhODc4YTE4MTdfSEQ5TmZsbTd4OWkwRzFYNUVLdWV2aE9zZ2lVOGkwVlFfVG9rZW46WnJsM2JadWpYb2tCSjJ4QmxkY2N0TldkbkNjXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. Histogram 会根据 class 聚合，并展现对象数量级其 Shallow Heap 及 Retained Heap（如Retained Heap项目为空，可以点击下图中计算机的图标并计算 Retained Heap），可以看到 ClassLoaderOOMOps 有524044个对象，其 Retain Heap 占据了370M以上（上述代码是100M左右）。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MDVmNDcyZjE0MDI3NzdhYWRlY2QyMzM0YjNlNDdhNDBfNmp1RUtseFZNeDM2NXlLTFBVMVh3UDlXUzltdDZaZVNfVG9rZW46STZjeWJlbGQwb2h5TUJ4VjNNWmN6aEtBbm5jXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 使用 incoming references，可以找到创建的代码位置。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MTJjYjFiYzE4OGJhNjFmY2YzMmY3OWIzODYwMzNjMTVfWGxPM1p0eWpMaWdXcFNqb0swOFRuRE5ybmRQU0QwSzdfVG9rZW46RHVCV2J2S0tHb3FNbHl4N0IxaWN2eENPbk5iXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 再来看方向二，同样在占据319M内存的 Obejct 数组采用 incoming references 查看引用路径，也很容易定位到具体代码位置。并且从下图中我们看出，Dominator tree 的起点并不一定是 GC根，且通过 Dominator tree 可能无法获取到最开始的创建路径，但 incoming references 是可以的。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MDQyZjQzZDJkYjc2YWQzYzU1OWU1ZGQ3MDBhMjAxNGRfZ2g2b0FuQmtlTVpNQjZKa0FLZGhlWE9wNXFQcGcwVVVfVG9rZW46Tlo3bGJHcmN6b1Fhamt4eGtLeGNlemM0bjRjXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. **对象间依赖详解及实战**

## **3.1 References**

***注：笔者使用频率 Top2***

**功能**：在对象引用图中查看某个特定对象的所有引用关系（提供对象对其他对象或基本类型的引用关系，以及被外部其他对象的引用关系）。通过任一对象的直接引用及间接引用详情（主要是属性值及内存占用），提供完善的依赖链路详情。

**使用入口**：目标域右键 → List objects → with outgoing references/with incoming references.

**使用场景**

- outgoing reference：查看对象所引用的对象，并支持链式传递操作。如查看一个大对象持有哪些内容，当一个复杂对象的 Retained Heap 较大时，通过 outgoing reference 可以查看由哪个属性引发。下图中 A 支配 F，且 F 占据大量内存，但优化时 F 的直接支配对象 A 无法修改。可通过 outgoing reference 看关系链上 D、B、E、C，并结合业务逻辑优化中间环节，这依托 dominator tree 是做不到的。
- incoming reference：查看对象被哪些对象引用，并支持链式传递操作。如查看一个大对象都被哪些对象引用，下图中 K 占内存大，所以 J 的 Retained Heap 较大，目标是从 GC Roots 摘除 J 引用，但在 Dominator tree 上 J 是树根，无法获取其被引用路径，可通过 incoming reference 查看关系链上的 H、X、Y ，并结合业务逻辑将 J 从 GC Root 链摘除。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MDcxZDMyMTFhY2IzNzI5MTE3N2RkZjdhNDI3MmZjZjlfM0dGYlljbENsV21hOUVVSjg3RkoxSEVmY2tSbGlJTTdfVG9rZW46UTdVYWJpTFlEb0EyY1d4aHEybmNrYWlWbkxmXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

## **3.2 Thread overview**

**功能**：展现转储 dump 文件时线程执行栈、线程栈引用的对象等详细状态，也提供各线程的 Retained Heap 等关联内存信息。

**使用入口**：MAT 主页 → Thread overview

**使用场景**

- 查看不同线程持有的内存占比，定位高内存消耗线程（开发技巧：不要直接使用 Thread 或 Executor 默认线程名避免全部混合在一起，使用线程尽量自命名方便识别，如下图中 ThreadAndListHolder-thread 是自定义线程名，可以很容易定位到具体代码）
- 查看线程的执行栈及变量，结合业务代码了解线程阻塞在什么地方，以及无法继续运行释放内存，如下图中 ThreadAndListHolder-thread 阻塞在 sleep 方法。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=Yjg0YjFhNzQwOTc2N2JiMjg4ODUwNWJmYTFhOTljNWJfOHdHYXpzeUR6Rkd1VWJ2ajc2NTdrRUx6MFVSZjFSeVhfVG9rZW46WUNOMmJBNFE0b3FEbFB4d3gzYmNxQUxzblVoXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

## **3.3 Path To GC Roots**

**功能**：提供任一对象到 GC Root 的路径详情。

**使用入口**：目标域右键 → Path To GC Roots

**使用场景**：有时你确信已经处理了大的对象集合但依然无法回收，该功能能快速定位异常对象不能被 GC 回收的原因，直击异常对象到 GC Root 的引用路径。比 incoming reference 的优势是屏蔽掉很多不需关注的引用关系，比 Dominator tree 的优势是可以得到更全面的信息。

*小技巧：在排查内存泄漏时，建议选择 exclude all phantom/weak/soft etc.references 排除虚引用/弱引用/软引用等的引用链，因为被虚引用/弱引用/软引用的对象可以直接被 GC 给回收，聚焦在对象否还存在 Strong 引用链即可。*

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=M2E2MWZkNTU1NGYzMDk5MzU4NjFjMmY1MTU4ZWQ5MDdfSVppWG9kQmVzWExLem81bk5xUGNsYmR0SDFCQkZPajJfVG9rZW46REh6MGJFYTZub2NxbDJ4YTRETGNlN0RjblZlXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

## **3.4 class loader 分析**

**功能**

- 查看堆中所有 class loader 的使用情况（入口：MAT 主页菜单蓝色桶图标 → Java Basics → Class Loader Explorer）。
- 查看堆中被不同class loader 重复加载的类（入口：MAT 主页菜单蓝色桶图标 → Java Basics → Duplicated Classes）。

**使用场景**

- 当从 Heap dump overview 了解到系统中 class loader 过多，导致占用内存异常时进入更细致的分析定位根因时使用。
- 解决 NoClassDefFoundError 问题或检测 jar 包是否被重复加载

具体使用方法在 2.6 及 3.5 两节的案例中有介绍。

## **3.5 综合案例二**

**使用工具项**：class loader（重复类检测）、inspector、正则检索。

**异常现象** ：运行时报 NoClassDefFoundError，在 classpath 中有两个不同版本的同名类。

**分析过程**

1. 进入 MAT 已加载的重复类检测功能，方式如下图。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGE0NGFmMDBhMzQyYmY2ZWNmYjQ3MWMzY2NiYzU0OTNfRWpxYXh6Z3czY2N0aUNnMjBkRWxKYUtCWm5ySnhBRW9fVG9rZW46VGh2c2JUNUNnb2t5RXd4OXhlS2Nyd1dNbk5jXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 可以看到所有重复的类，以及相关的类加载器，如下图。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDI5M2Y3ZjdkYmYzNzJjNzQ3OWI1NWNkYmQzOTVhNDRfcGRreDJrUHFmVkJscXI4b0hJb0dJSVBxRFE0aXY5d3lfVG9rZW46RnFwemI0ZXdwbzF6bEt4d3VQN2M3cDVtbk1jXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 根据类名，在`<Regex>`框中输入类名可以过滤无效信息。
2. 选中目标类，通过Inspector视图，可以看到被加载的类具体是在哪个jar包里。（本例中重复的类是被 URLClassloader 加载的，右键点击 “_context” 属性，最后点击 “Go Into”，在弹出的窗口中的属性 “_war” 值是被加载类的具体包位置）

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmYwYzFhOWUyNjhlYTc1ZTYxMGNhMTYzZDQ0MjE5OTZfd24xRGJ6WEtQenF0VXRzVkRrVU5BT1hCbUFWVVEzVjdfVG9rZW46REZUTWI1WWpnb2puUnp4MG5oZGMwNTVXbjllXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=OTYzYjRmMTYwMWUzZjE2YTY0ODc3ZGY4M2FjZGVjNmZfNGZhQ1ZWdTNuT09BTUZhZVFGM2luQ0ppR1QzN2N5VXBfVG9rZW46V2c4cmIxVGU3b2JzMkJ4cFNwYWM5TVF5bmxlXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. **对象状态详解及实战**

## **4.1 inspector**

**功能**：MAT 通过 inspector 面板展现对象的详情信息，如静态属性值及实例属性值、内存地址、类继承关系、package、class loader、GC Roots 等详情数据。

**使用场景**

- 当内存使用量与业务逻辑有较强关联的场景，通过 inspector 可以通过查看对象具体属性值。比如：社交场景中某个用户对象的好友列表异常，其 List 长度达到几亿，通过  inspector 面板获取到异常用户 ID，进而从业务视角继续排查属于哪个用户，本例可能有系统账号，与所有用户是好友。
- 集合等类型的使用会较多，如查看 ArrayList 的 size 属性也就了解其大小。

**举例**：下图中左边的 Inspector 窗口展现了地址 0x125754cf8 的 ArrayList 实例详情，包括 modCount 等并不会在 outgoing references 展现的基本属性。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=Mjk3NzFlYmMyMDU2M2I2MDFmYzA4MDkxMTRhM2Q0NjVfNW5jcDBHSEh3WElmT1JWTFR4MDVYeWZZNWE0b2tPUzRfVG9rZW46WlRrMmJicXpjb21kN3l4c2Z0ZGNqb1FwbkljXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

## **4.2 集合状态**

**功能**：帮助更直观的了解系统的内存使用情况，查找浪费的内存空间。

**使用入口**：MAT 主页 → Java Collections → 填充率/Hash冲突等功能。

**使用场景**

- 通过对 ArrayList 或数组等集合类对象按填充率聚类，定位稀疏或空集合类对象造成的内存浪费。
- 通过 HashMap 冲突率判定 hash 策略是否合理。

具体使用方法在 4.3 节案例详细介绍。

## **4.3 综合案例三**

**使用工具项**：Dominator tree、Histogram、集合 ratio。

**异常现象** ：程序 OOM，且 Dominator tree 无大对象，通过 Histogram 了解到多个 ArrayList 占据大量内存，期望通过减少 ArrayList 优化程序。

**程序代码**

```Java
package com.q.mat;

import java.util.ArrayList;
import java.util.List;

public class ListRatioDemo {

    public static void main(String[] args) {
        for(int i=0;i<10000;i++){
            Thread thread = new Thread(new Runnable() {
                public void run() {
                    HolderContainer holderContainer1 = new HolderContainer();
                    try {
                        Thread.sleep(1000 * 1000 * 60);
                    } catch (Exception e) {
                        System.exit(1);
                    }
                }
            });
            thread.setName("inner-thread-" + i);
            thread.start();
        }

    }
}

class HolderContainer {
    ListHolder listHolder1 = new ListHolder().init();
    ListHolder listHolder2 = new ListHolder().init();
}

class ListHolder {
    static final int LIST_SIZE = 100 * 1000;
    List<String> list1 = new ArrayList(LIST_SIZE); // 5%填充
    List<String> list2 = new ArrayList(LIST_SIZE); // 5%填充
    List<String> list3 = new ArrayList(LIST_SIZE); // 15%填充
    List<String> list4 = new ArrayList(LIST_SIZE); // 30%填充

    public ListHolder init() {
        for (int i = 0; i < LIST_SIZE; i++) {
            if (i < 0.05 * LIST_SIZE) {
                list1.add("" + i);
                list2.add("" + i);
            }
            if (i < 0.15 * LIST_SIZE) {
                list3.add("" + i);
            }
            if (i < 0.3 * LIST_SIZE) {
                list4.add("" + i);
            }
        }
        return this;
    }
}
```

**分析过程**

1. 使用 Dominator tree 查看并无高占比起点。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=YWJlNTM0MmIxMWYzZDZhZWE3NDc0MzYwMzkxNTBjYjNfbHVBbXNkZ0ZNNlNIQmljdVhCZ3hkSm5nd1FROVJEaW5fVG9rZW46U2J3OWJuRXljb1B5cjZ4MWJLRGNUSkFybmdkXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 使用 Histogram 定位到 ListHolder 及 ArrayList 占比过高，经过业务分析很多 List 填充率很低浪费内存。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjMwYWU3NTdkNDc3MGIxN2FlZDE2NWQ0MGRhYzgxMTNfZnB6aTBocHhNZTdmZGs2OU4yd2VuM0dBWTJUZ1hJcEFfVG9rZW46SmF5VGJKRTFHb2VWN0h4bjBQM2NJSkF6bjBjXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 查看 ArrayList 的填充率，MAT 首页 → Java Collections → Collection Fill Ratio。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=N2YwNGIwYWYzODFkN2I4ZGUyNDIzZTMyMjQ0NDhjZDJfcFc0TE1DMkxaNjgxMjVrSU5WYjRvb1ZrNDEwclVqRU9fVG9rZW46VlQ3TmJpS2Vqb3J0Qnp4bVVEU2NOSm5FbmlmXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 查看类型填写 java.util.ArrayList。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=OThkOGFmMDUwOWRiZmIyY2NmOTc4NmFhZTgxY2RmZWNfSzFxRUdtQVpjQXRWc2VzNGpDelJVemlxekZrUHA2U1ZfVG9rZW46TFl2MGJkY2tjbzRvT3R4UzRyRGNLSmo2bm1iXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)

1. 从结果可以看出绝大部分 ArrayList 初始申请长度过大。 

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=N2JkMzg3YjNjOTlmMzQ0NjRlOWRhZjY1YTk2Y2FjZGZfQXZ1Z1NGU2RJRUpoQXBHOVpwMGR1bEM1d1R3RXVmQmxfVG9rZW46UGtUaWI1N3Vpb3l4NGh4TGxyTWNHTm51bkZlXzE3MDk3MTk5NzI6MTcwOTcyMzU3Ml9WNA)