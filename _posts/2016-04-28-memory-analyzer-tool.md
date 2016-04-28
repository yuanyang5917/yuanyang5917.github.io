---
layout: post
title: Android内存分析之MAT
date: 2016-04-28 00:00:00 +0800
cover: false
tags: android
subclass: 'post tag-fiction'
categories: 'casper'
cover: 'assets/images/cover6.jpg'
navigation: True
logo: 'assets/images/ghost.png'

---

　　面试中经常会问到内存优化，我们在开发过程中也多少会遇到OOM的问题，根据大牛们的博客，记录下我的学习思路

一、为何会OOM？
==

### 1. 一直以来Andorid手机的内存都比iPhone(iPhone6RAM1G)大的多，Android却经常出现OOM，这是为何？

　 这个是因为Android系统对dalvik的vm heapsize 作了硬性限制，当java进程申请的java空间超过阀值时，就会抛出OOM异常(这个阀值可以是48M、24M、16M等，视机型而定)，可以通过adb shell getprop ｜ grep dalvik.vm.heapgrowthlimit查看此值。(我的一加2 Android6.0.1已经达到了256M)

　 也就是说，程序发生OOM并不表示RAM不足，而是因为程序申请的java heap对象超过了dalvik.vm.heapgrowthlimit。也就是说，在RAM充足的情况下，也可能发生OOM

　 这样的设计似乎有些不合理，但是Google为什么这样做呢？这样设计的目的是为了让Android系统能同时让比较多的进程常驻内存，这样程序启动时就不用每次都重新加载到内存，能够给用户更快的响应。迫使每个应用程序使用较小的内存，移动设备非常有限的RAM就能使比较多的app常驻其中。

### 2. 大型游戏如何在较小的heapsize上运行？

- 创建子进程

     创建一个新的进程，那么我们就可以把一些对象分配到新进程的heap上了，从而达到一个应用程序使用更多的内存的目的，当然，创建子进程会增加系统开销，而且并不是所有应用程序都适合这样做，视需求而定。
 
     创建子进程的方法：使用android:process标签

- 使用jni在native heap上申请空间（推荐使用）

     nativeheap的增长并不受dalvik vm heapsize的限制，从图6可以看出这一点，它的native heap size已经远远超过了dalvik heap size的限制。
 
     只要RAM有剩余空间，程序员可以一直在native heap上申请空间，当然如果 RAM快耗尽，memory killer会杀进程释放RAM。大家使用一些软件时，有时候会闪退，就可能是软件在native层申请了比较多的内存导致的。比如，我(余龙飞)就碰到过UC web在浏览内容比较多的网页时闪退，原因就是其native heap增长到比较大的值，占用了大量的RAM，被memory killer杀掉了。(Fresco使用的就是这种方式)

- 使用显存（操作系统预留RAM的一部分作为显存）

     使用OpenGL textures等API，texture memory不受dalvik vm heapsize限制，这个我没有实践过。再比如Android中GraphicBufferAllocator申请的内存就是显存。

### 3. Android内存究竟如何?(native heap、java heap)

- 进程的地址空间

     在32位操作系统中(Native Process)，进程的地址空间为0到4GB：
 
     ![图一](http://img.blog.csdn.net/20130513154042627)

     1. kernel space(内河空间)：这些地址用户代码不能读也不能写
     2. Memory Mapping Segment(内存映射)：段文件映射（包括动态库）和匿名映射。
     3. Stack：（进栈和出栈）由操作系统控制，其中主要存储函数地址、函数参数、局部变量等等，所以Stack空间不需要很大，一般为几MB大小。
     4. Heap：空间的使用由程序员控制，程序员可以使用malloc、new、free、delete等函数调用来操作这片地址空间。Heap为程序完成各种复杂任务提供内存空间，所以空间比较大，一般为几百MB到几GB。正是因为Heap空间由程序员管理，所以容易出现使用不当导致严重问题。


- Android中的进程

     1. native进程：采用C/C++实现，不包含dalvik实例的linux进程，/system/bin/目录下面的程序文件运行后都是以native进程形式存在的。如下图/system/bin/surfaceflinger、/system/bin/rild、procrank等就是native进程。

     2. java进程：实例化了dalvik虚拟机实例的linux进程，进程的入口main函数为java函数。dalvik虚拟机实例的宿主进程是fork()系统调用创建的linux进程，所以每一个android上的java进程实际上就是一个linux进程，只是进程中多了一个dalvik虚拟机实例。因此，java进程的内存分配比native进程复杂。如图3，Android系统中的应用程序基本都是java进程，如桌面、电话、联系人、状态栏等等。
　![这里写图片描述](http://img.blog.csdn.net/20130513155151825)
　


- Android中进程的堆内存
 
 	    第一张图和下面这张图分别介绍了native process和java process的结构，这个是我们需要深刻理解的，进程空间中的heap空间是我们需要重点关注的。heap空间完全由程序员控制，我们使用的malloc、C++ new和java new所申请的空间都是heap空间， C/C++申请的内存空间在native heap中，而java申请的内存空间则在dalvik heap中。

 ![这里写图片描述](http://img.blog.csdn.net/20130513155252901)

        注：Java中的code segment，data segment，heap，stack
        stack(栈)：对象引用都是在栈里的，相当于C/C++的指针
        heap(堆)：new出来的对象实例才是在堆里
        data segment：一般存放常量和静态常量
        code segment：方法，函数什么的都是放在code segment

- Bitmap分配在native heap还是dalvik heap上？

　**3.0后是分配在dalvik heap上，和3.x之前是分配在native heap**


- 4. 以上主要来自：现任支付宝大神余龙飞著作——**[Android进程的内存管理分析](http://blog.csdn.net/gemmem/article/details/8920039)**


二、内存分析之MAT
==

### 1. 谷歌提供了几种内存检测工具：

- Memory Monitor:内存监视器 
![这里写图片描述](http://img.blog.csdn.net/20160428014853422)
- Heap Viewer:堆查看器
![这里写图片描述](http://img.blog.csdn.net/20160428015014728)
![这里写图片描述](http://img.blog.csdn.net/20160428015028213)
- Allocation Tracker:分配追踪器
![这里写图片描述](http://img.blog.csdn.net/20160428014732467)
- Investigating Your RAM Usage：调查您的RAM使用
		
		通过Log输出的GC命令来判断：
		
		GC_CONCURRENT：heap快满了
		GC_FOR_MALLOC：因为你的应用程序试图分配内存时，你已经充分引起GC堆，所以系统必须停止你的应用和回收内存。
		GC_HPROF_DUMP_HEAP：GC发生时，你要创建的请求HPROF文件来分析你的堆。
		GC_EXPLICIT：一个明确的GC，当收到调用gc()时出现，应该尽量避免手动调用，而是相信GC会自动清理
		GC_EXTERNAL_ALLOC：只会在API10以及以下才会出现
		

		GC的原因：
		Concurrent：不会暂停应用线程，在后台运行，不会影响内存分配
		Alloc：GC是因为你的应用程序试图分配内存时，你heapwas已满。在这种情况下，垃圾收集发生在分配线程。
		Explicit：手动调用gc()，我们应该避免手动调用，我们要相信GC，手动调用会影响线程分配以及没必要的cpu周期，还可能导致其他线程的抢占。
		NativeAlloc：native内存的回收，主要来自人为造成的native内存压力，例如：Bitmap、渲染脚本分配的对象
		CollectorTransition：.....由于用到的太少，后面的就不再详述

### 2. 触发内存泄漏

- 多次切换屏幕的横纵，在Activity不同状态旋转屏幕，然后再返回。旋转设备往往会导致应用程序的Activity、Context、或View对象泄漏，因为系统中重新创建活动，如果程序中其他地方拥有这些对象的引用，系统无法回收。
- 多个应用之间切换，在Activity不同的状态下切换（导航至主屏幕，然后返回到您的应用程序）。

### 3. 怎样的内存是健康的？

- `内存使用率低，使用率稳定(波动小)`
- 没有正在使用的对象，要`能够被GC回收`(避免内存泄漏)
- 不再使用的内存对象、或着大型内存，`使用结束(虚引用)马上回收`(`finalize()方法进行清理`，通过 java.lang.ref.PhantomReference实现)

		我们进行内存分析具体分析什么呢？
		1. 大型对象
		2. 不使用的未能被释放的对象(内存泄漏)
		
		而谷歌目前提供的内存分析工具只能从宏观上进行内存分析，无法针对某个对象进行分析
		
		这里我们这里需要使用强大的第三方内存分析工具MAT(Memory Analyzer Tool)针对具体内存进行分析

### 4. MAT基础知识

- MAT简介
	- MAT(Memory Analyzer Tool)，一个基于Eclipse的内存分析工具，是一个快速、功能丰富的JAVA heap分析工具，它可以帮助我们查找内存泄漏和减少内存消耗。使用内存分析工具从众多的对象中进行分析，快速的计算出在内存中对象的占用大小，看看是谁阻止了垃圾收集器的回收工作，并可以通过报表直观的查看到可能造成这种结果的对象。 
	- 当然MAT也有独立的不依赖Eclipse的版本，只不过这个版本在调试Android内存的时候，需要将DDMS生成的文件进行转换，才可以在独立版本的MAT上打开。不过Android SDK中已经提供了这个Tools，所以使用起来也是很方便的。

- MAT下载地址

	- 独立版本下载地址： https://eclipse.org/mat/downloads.php
		这种方式有个麻烦的地方就是DDMS导出的文件，需要进行转换才可以在MAT中打开。
		
	- Eclipse插件地址：http://download.eclipse.org/mat/1.5/update-site/

- MAT中重要概念介绍
	`要看懂MAT的列表信息，Shallow heap、Retained Heap、GC Root 这几个概念一定要弄懂。`
	-	 Shallow heap
	`Shallow size就是对象本身占用内存的大小，不包含其引用的对象。`
	1. 常规对象`（非数组）的Shallow size有其成员变量的数量和类型决定。`
	
	2. `数组的shallow size有数组元素的类型（对象类型、基本类型）和数组长度决定 `
	
			因为不像c++的对象本身可以存放大量内存，java的对象成员都是些引用。真正的内存都在堆上，看起来是一堆原生的byte[],char[], int[]，所以我们如果只看对象本身的内存，那么数量都很小。所以我们看到Histogram图是以Shallow size进行排序的，排在第一位第二位的是byte，char 。

	- Retained Heap
	
		`Retained Heap的概念：如果一个对象被释放掉，那么该对象引用的所有对象（包括被递归释放的）占用的heap也会被释放。`
		
		如果一个对象的某个成员new了一大块int数组，那这个int数组也可以计算到这个对象中。`与shallow heap比较，Retained heap可以更精确的反映一个对象实际占用的大小`（因为如果该对象释放，retained heap都可以被释放）。

		注意：A和B都引用到同一内存，A释放时，该内存不会被释放。所以这块内存不会被计算到A或者B的Retained Heap中。故`Retained Heap并不总是那么有效。 `
		
		这一点并不重要，因为MAT引入了Dominator Tree－－对象引用树，计算Retained Heap就会非常方便，显示也非常方便。对应到MAT UI上，在dominator tree这个view中，显示了每个对象的shallow heap和retained heap。然后可以以该节点为树根，一步步的细化看看retained heap到底是用在什么地方了。

	- GC Root
	
		`GC发现通过任何reference chain(引用链)无法访问某个对象的时候，该对象即被回收。`

		![这里写图片描述](http://img.blog.csdn.net/20131025114957718?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2VtbWVt/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

		![这里写图片描述](http://img.blog.csdn.net/20151127104034762)

		名词GC Roots正是分析这一过程的起点，例如JVM自己确保了对象的可到达性(那么JVM就是GC Roots)，所以GC Roots就是这样在内存中保持对象可到达性的，一旦不可到达，即被回收。

		通常GC Roots是一个在current thread(当前线程)的call stack(调用栈)上的对象（例如方法参数和局部变量），或者是线程自身或者是system class loader(系统类加载器)加载的类以及native code(本地代码)保留的活动对象。所以`GC Roots是分析对象为何还存活于内存中的利器。`


- MAT界面功能介绍
	- 打开经过转换的hprof文件：
	![这里写图片描述](http://img.blog.csdn.net/20151127103729946)
	可不选
	- Actions区域，几种分析方法：
		- Histogram：列出内存中的对象，对象的个数以及大小
		![这里写图片描述](http://img.blog.csdn.net/20151127103905066)

		- Dominator Tree：列出最大的对象以及其依赖存活的Object （大小是以Retained Heap为标准排序的） 
		![这里写图片描述](http://img.blog.csdn.net/20151127103921964)

		> 点开每个对象，`检查内部的超大对象`
		
		- Top Consumers ： 通过图形列出最大的object 
		![这里写图片描述](http://img.blog.csdn.net/20151127103941622)

		> 一般Histogram和 Dominator Tree是最常用的。

- MAT分析对象的引用
	- Path to GC Root
	在Histogram或者Domiantor Tree的某一个条目上，右键可以查看其GC Root Path：
	![这里写图片描述](http://img.blog.csdn.net/20151127104314761)
	
	- 点击Path To GC Roots –> with all references 
	
		![这里写图片描述](http://img.blog.csdn.net/20151127104222628)

		> 通过这个图`查看(内存泄漏)`该内存还被谁所引用，为何还不能释放
		
- MAT基础介绍来自[Gracker](http://www.jianshu.com/p/d8e247b1e7b2#)

三、内存问题总览
==

- 内存泄漏
	- 非静态内部类的静态实例容易造成内存泄漏
		非静态内部类的存活需要依赖外部类
		
	- Activity使用静态成员(静态成员引用Drawable、Bitmap等大内存对象)
	
	- 使用handler时的内存问题
	因为Handler的非即时性，导致部分代码不能及时释放
	可以使用Badoo开发的第三方的 [WeakHandler](https://github.com/badoo/android-weak-handler)
	
	- 注册某个对象后未反注册
	注册广播接收器、注册观察者等等

	- 集合中对象没清理造成的内存泄露

	- 资源对象没关闭造成的内存泄露
	资源性对象比如(Cursor，File文件等)往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们，以便它们的缓冲及时回收内存。

- 一些不良代码成内存压力
	- 循环以及递归
	- 数据随意申请大小
	- 如果没有用到不要定义全局变量
	
- Bitmap使用不当
	- 及时的销毁
	
	虽然，系统能够确认Bitmap分配的内存最终会被销毁，但是由于它占用的内存过多，所以很可能会超过Java堆的限制。因此，在用完Bitmap时，要`及时的recycle掉`。recycle并不能确定立即就会将Bitmap释放掉，但是会给虚拟机一个暗示：“该图片可以释放了”。

	- 设置一定的采样率(二次采样)
	有时候，我们要显示的区域很小，没有必要将整个图片都加载出来，而只需要记载一个缩小过的图片，这时候可以设置一定的采样率，那么就可以大大减小占用的内存。

	- 巧妙的运用软引用（SoftRefrence）
	
		有些时候，我们使用Bitmap后没有保留对它的引用，因此就无法调用Recycle函数。这时候巧妙的运用软引用，可以使Bitmap在内存快不足时得到有效的释放
		目前但凡是个图片加载框架都会使用SoftRefrence

- 构造Adapter时，没有使用缓存的 convertView

- 频繁的方法中创建对象
	不要在经常调用的方法中创建对象，尤其是忌讳在循环中创建对象。可以适当的使用 hashtable ， vector 创建一组对象容器，然后从容器中去取那些对象，而不用每次 new 之后又丢弃。

四、MAT内存分析
==

- 使用Dominator Tree -> Path To GC Roots –> with all references 
	上面MAT基础里面已经讲

- 根据某种类型的对象个数来分析内存泄漏。
	Actions -> Histogram
	![这里写图片描述](http://img.blog.csdn.net/20131025120850406)

	上图展示了内存中各种类型的对象个数和Shallow heap，我们看到byte[]占用Shallow heap最多，那是因为Honeycomb之后Bitmap Pixel Data的内存分配在Dalvik heap中。右键选中byte[]数组，选择List Objects -> with incomingreferences，可以看到byte[]具体的对象列表：

	![这里写图片描述](http://img.blog.csdn.net/20131025123513640)

	![这里写图片描述](http://img.blog.csdn.net/20131025123529984)

	我们发现第二个byte[]的Retained heap较大，内存泄漏的可能性较大，因此右键选中这行，Path To GC Roots -> exclude weak references，同样可以看到上文所提到的情况，我们的Bitmap对象被leak所引用到，这里存在着内存泄漏。

	![这里写图片描述](http://img.blog.csdn.net/20131025121635500)

> 1. 讲的是对象及其应用的内存大小
> 2. 讲的是大型对象被谁所引用


内存分析工具还有一个比较流行的内存泄漏检测库：[LeakCannary](http://www.liaohuqiu.net/cn/posts/leak-canary/)