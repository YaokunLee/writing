### 安卓内存优化

飞书文档链接，建议飞书打开： [安卓内存优化](https://is0frj68ok.feishu.cn/wiki/HJN5wbVJZiY4AnkTjomcCuemnse?from=from_copylink)

## Android 内存管理机制

Java 内存分配

- 方法区📚：存放类信息、常量、静态变量等数据，是所有线程共享的区域。
- 虚拟机栈📉：每个Java方法在执行的时候都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接等信息。每个线程都有自己的虚拟机栈。
- 本地方法栈🌐：与虚拟机栈相似，但是服务于Native方法。
- 堆：GC主要作用的区域，内存泄漏发生在这个区域💧。堆是Java内存管理中最大的一块，是所有线程共享的区域，主要用于存放对象实例和数组。
- 程序计数器🔢：当前线程所执行的字节码的行号指示器。每个线程都有自己的程序计数器，是线程私有的。

## java内存回收算法

### 标记清除算法 

1. 标记出所有需要回收的对象
2. 统一回收所有被标记的对象

灰色区域：可被回收

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=OGMwYWI1ZTc5MGVjYjVhYWMyNDgyZGM3OTA5MzMxYTVfTG1hRXBaV0R2RE5NbWRkSnNsVWJEd21yRGtmZERjUzdfVG9rZW46V2hVTWJMTDRsbzBjZXB4NVI0SGMyYlhBbkVlXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

效率不高，需要扫两遍内存，产生大量不连续的内存碎片

### 复制算法

1. 将内存划分为大小相等的两块
2. 一块内存用完之后复制到另一块
3. 清理另一块

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MGFiODc1Mzk1MDE2YTMzZTZmZjlkODczYmM4MzNlMTBfbGl5cXY2aTJacTFCY01sTXdnUk85QXNDVXA2MDlqVmZfVG9rZW46VDRSYWJOekNWbzFzeDR4amlRcWNFVVdmbnBlXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

实现简单，运行高效（相对于前一个方法）

浪费一半空间

### 标记-整理算法

1. 标记过程与标记清除算法一样
2. 存活对象往一端进行移动
3. 清理其余内存

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2NiYmRmNzYwYzVlNDQ5NDJjZjYyYTNjYmUxNGM0ZmRfYnpNaTRLcE16ZFc5RThudkJoZks5akpKVTNXMmQ1MW5fVG9rZW46WjhlQ2I3WXA1b2lFU094VVp5NmMyRWtwblVjXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

### 分代收集算法

结合多种收集算法优势

新生代对象存活率低，复制

老年代存活率高，标记整理

分代收集算法🔄是Java垃圾收集（GC）中的一种策略，它基于这样一个观察：不同对象的生命周期各不相同。因此，将堆内存分成几个不同的区域（代），根据每个对象的年龄将它们分配到不同的代中，以此来提高垃圾收集的效率。

- 新生代（Young Generation）🌱：存放新创建的对象。大部分对象在这里被迅速回收，只有少数存活的对象会被移动到老年代。新生代通常采用复制算法（Copying），因为新生代中的大多数对象都是“朝生夕灭”的。
- 老年代（Old Generation）🌳：存放经过多次垃圾收集仍然存活的对象。这些对象通常存活时间较长，或者是大小较大的对象。老年代的垃圾收集频率较低，通常采用标记-清除（Mark-Sweep）或标记-整理（Mark-Compact）算法。
- 永久代/元空间（PermGen/Metaspace，Java 8以后使用元空间替代永久代）🏛️：用于存放静态文件如Java类、方法等。这部分区域的回收主要针对常量池的回收和类型的卸载。

分代收集算法的核心思想是：通过区分对象的生命周期，采用最适合的垃圾收集算法，来提升垃圾收集的效率和减少收集时的停顿时间✨。这种方法能有效地减少全堆扫描的次数，加快垃圾收集的速度。

## Android 内存管理机制

内存弹性分配，分配值与最大值受设备影响

OOM场景：内存真正不足，可用内存不足

Dalvik 与 Art （android 5之后使用的都是Art虚拟机）区别

1. Dalvik仅固定一种回收算法
2. Art回收算法可运行期选择回收算法
3. Art具备内存整理能力

Low memory killer

- 进程分类
- 回收收益

### **内存抖动**

> 含义：短时间内有大量对象创建销毁，它伴随着频繁的GC。

1. 查看：可以使用android studio自带的profile工具检测。
2. 现象：在profile中的内存图像就像是心电图一样，忽上忽下，如下图所示：

1. 常见场景：循环使用字符串拼接，比如我们项目的日志打印等
2. 预防内存抖动方法：

- 避免在循环中创建对象，能复用的尽量复用。
- 避免在频繁调用的方法中创建对象，如自定义view中的onDraw（）等方法中创建画笔。
- 获取对象尽量从对象池中获取，如Handler获取Message对象应使用obtain（）方法获取了。

### **内存泄漏**

> 程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费。 长生命周期对象持有短生命周期对象强引用，从而导致短生命周期对象无法被回收！

1. 查看：使用profile工具检测内存情况，重复执行进入然后退出一个activity，看activity实例是否还存在。如果activity实例还存在，很可能就出现了内存泄漏。
2. 现象：反复进入A，然后退出A ，执行三次，可以看到A 的实例存在两个。如下图，VideoPlayerActivity：

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NjVjM2MzMGZiNDg2ODZiNjA2NDlmNTA2NzU4ZTE1NjZfb2JxRUVlWFhzM2VtakxvWUtTUGlYdkV6REZUOGluUmNfVG9rZW46WU5uc2JuSWt1b1NrdUR4TTdNbmNkNWx0bnFiXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

这就说明我们的activity并没有被销毁，至少目前是这样的。至于究竟会不会内存泄漏，就需要接下来使用另一款工具配合使用了。

1. 如何判断内存泄漏:

- 使用可达性分析法

通过一系列称为“GC Roots”的对象作为起始点，从这些节点向下搜索，搜索所有的引用链，当一个对象到GC Roots没有任何引用链（即GC Roots到对象不可达）时，则证明此对象是不可用的。也就会被回收。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MDRjNDIyYjJlNTQxZWRhMTY0ZWFmMjQ3ZDkyN2VjMDVfN056ZGF6aWVBbjJRMFQzclNmTjUzb3M1dUk1VzhkZjNfVG9rZW46Rm9ROGJKSHoxb0xsdGF4VmMwSmNVWEltblRjXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

1. 何为GC Roots 对象，一般静态变量就是gc root对象，可以理解成生命周期很长的对象。
2. 如何预防内存泄漏：

- 使用 软引用、弱引用间接的持有对象的引用。
- 软引用：

定义一些还有用但并非必须的对象。对于软引用关联的对象，GC不会直接回收，而是在系统将要内存溢出之前才会触发GC将这些对象进行回收。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=YjM1ZjI3MDQzNGY2Zjc2MzA5ZWE3ODk3MjEyOTlkNjRfbnB1dG42Q0ZuS3FucU55VHl5VUZnRG1oU1phT2hHdHdfVG9rZW46SFRqa2JCUkY3b3l4M2x4SWJ1RmNVN2Q1bjFnXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

- 弱引用 ：

同样定义非必须对象。被弱引用关联的对象在GC执行时会被直接回收。

1. 造成内存泄漏的常见场景：

- 使用集合时，例如add一个监听器，我们必须要手动remove掉。
- 使用静态成员变量/单利对象时，如果持有短生命周期对象的引用（Activity）将导致短生命周期对象无法被释放。
- 进行文件io操作时，没有close（）。最好写在finally{ }里面；
- android 系统bug、第三方类库造成的内存泄漏。

### **内存溢出**

> 内存溢出(Out Of Memory，简称OOM)是指应用系统中存在无法回收的内存或使用的内存过多，最终使得程序运行要用到的内存大于能提供的最大内存。此时程序就运行不了，系统会提示内存溢出，有时候会自动关闭软件，重启电脑或者软件后释放掉一部分内存又可以正常运行该软件

- 频繁的出现内存抖动或者大量内存泄漏很有可能就会导致内存溢出（OOM）。

## 内存泄漏检测实战

下面这个类，持有一个Bitmap 

```Java
package com.material.components.mine.testmemory.memory;

import android.app.Activity;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.os.Bundle;
import android.widget.ImageView;


import androidx.appcompat.app.AppCompatActivity;

import com.material.components.R;


/**
 * 模拟内存泄露的Activity
 */
public class MemoryLeakActivity extends AppCompatActivity implements CallBack{

    Bitmap bitmap;

    @Override
    protected void onCreate( Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_memoryleak);
        ImageView imageView = findViewById(R.id.iv_memoryleak);
        bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.splash);
        imageView.setImageBitmap(bitmap);

        CallBackManager.addCallBack(this);
    }



    @Override
    protected void onDestroy() {
        super.onDestroy();
//        CallBackManager.removeCallBack(this);
    }

    @Override
    public void dpOperate() {
        // do sth
    }
}
```

对应的CallBackManager 中,ArrayList<CallBack> 是static，故而是gc root，所以只要没有removeCallBack, MemoryLeakActivity 都是可达的，垃圾回收不会将其回收，所以会内存泄漏

```Java
package com.material.components.mine.testmemory.memory;

import java.util.ArrayList;

public class CallBackManager {

    public static ArrayList<CallBack> sCallBacks = new ArrayList<>();

    public static void addCallBack(CallBack callBack) {
        sCallBacks.add(callBack);
    }

    public static void removeCallBack(CallBack callBack) {
        sCallBacks.remove(callBack);
    }
}
```

在android studio自带的工具中，我们也可以发现已经内存泄漏了

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=Njk5ZjI0MDM5YmE5MzQ3YzE4N2IyNzFkMGJiMTQ1MDhfeG5LV2pqRTZOUkZvZ3RtSm10YnQxQTllZnlnR3hrZFNfVG9rZW46QWZiVGJLTjlpb0g2bW94bXpJRGNxaEJibnJkXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

我们使用更高级的mat来检测内存泄漏

**如下图，可以看到，一共有7个MemoryLeakActivity在内存中，这显然是不正常的**

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=M2U1Y2UxNjk5NmU1MjJhMWE5YmFhNGYzZmNlMjA0NzRfOXV2ZWJMZGFEMDJtWEFmUzFIcHRHMHFCTnlWMzFURFZfVG9rZW46VU1WTWJiMkFzb3plMHN4RTdBM2NUUUk1bnBlXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

右键，查看path to GC root，选择with all references, 可以得到如下结果：

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=OTlkMjQzYWFkZjVjMjIyNmU2YTg5Mzc5Y2Y3OTg5ZmNfbjVWNDBaMGhOOUpDaTZ2aVFtRGNFUkNGY2c1dHpEcWRfVG9rZW46R1BWc2JsREJSb1VvV2p4b2ltQWNuU3E1bjhjXzE3MDg0ODk1ODA6MTcwODQ5MzE4MF9WNA)

可见是最终是因为 sCallBacks 是gc root，所以不会被垃圾回收。