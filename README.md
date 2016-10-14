#APP自动修复简介
![AppAutoFix](https://ss0.bdstatic.com/94oJfD_bAAcT8t7mm9GUKT-xh_/timg?image&quality=100&size=b4000_4000&sec=1476165692&di=355cc71c610b5bb86146a0370ea62bac&src=http://pic.baike.soso.com/p/20140321/20140321103905-1683108126.jpg)

##热修复 Hot Fix
* 什么是热修复？
* 热修复在APP中的实现

###关于iOS实现

####iOS前置小知识
* 关于iOS编程语言Objective-C运行时特性简介

```Objective-C
//OC重的方法调用 = 向对象发消息

[receiver message];
// 底层运行时会被编译器转化为：
objc_msgSend(receiver, selector)
// 如果其还有参数比如：
[receiver message:(id)arg...];
// 底层运行时会被编译器转化为：
objc_msgSend(receiver, selector, arg1, arg2, ...)
```
#####关于 receiver

objc_msgSend第一个参数类型为id，大家对它都不陌生，它是一个指向类实例的指针：
typedef struc objc_object *id;

那objc_object又是啥呢：

struct objc_object { Class isa;};

#####关于 SEL

objc_msgSend函数第二个参数类型为SEL，它是selector在Objc中的表示类型（Swift中是Selector类）。selector是方法选择器，可以理解为区分方法的ID，而这个ID的数据结构是SEL：

typedef struct objc_selector *SEL;

其实它就是个映射到方法的C字符串，你可以用Objc编译器命令@selector()或者 Runtime 系统的sel_registerName函数来获得一个SEL类型的方法选择器。

#####关于isa查找

![img](http://upload-images.jianshu.io/upload_images/1330553-81f64a11ad20c764.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240&_=5489086)

#####Method Swizzling 原理
每个类都有一个方法列表，存放着selector的名字和方法实现的映射关系。IMP有点类似函数指针，指向具体的Method实现。
![img](http://img.blog.csdn.net/20130718230259187?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWl5YWFpeHVleGk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们可以利用 method_exchangeImplementations 来交换2个方法中的IMP，
我们可以利用 class_replaceMethod 来修改类，
我们可以利用 method_setImplementation 来直接设置某个方法的IMP，
归根结底，都是偷换了selector的IMP，如下图所示：
![img](http://img.blog.csdn.net/20130718230430859?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWl5YWFpeHVleGk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

####热修复在iOS中实现思路
#####实现思路
假如我们有一个结构体
```json
{
  "className": "AClass",
  "exchangeMethod": {
    "name": "sayHello",
    "impl": "function() {
      NSLog(@"say Yep");
    }"
  }
}
```
#####插件：JSPatch
> [iOS JSPatch](https://github.com/bang590/JSPatch)

###关于Android实现

####Android前置小知识
#####Android安装包Apk介绍

解压 apk 后，一般的可看到的目录结构如下：

| 文件或目录       | 作用           |
| ---------------|--------------|
| META-INF/      | 也就是一个 manifest ，从 Java jar 文件引入的描述包信息的目录 |
| res/           | 存放资源文件的目录      |
| libs/          | 如果存在的话，存放的是 ndk 编出来的 so 库    |
| AndroidManifest.xml          | 程序全局配置文件    |
| libs/          | 最终生成的 dalvik 字节码    |
| resources.ars          | 编译后的二进制资源文件    |

Android 应用程序的生成过程，输入就是我们在 eclipse 或源码中监理的工程及其下面的源文件。输出就是处理后的 apk 文件。整个过程可以如下图所示：

![img](http://hi.csdn.net/attachment/201105/24/0_1306213778XZI2.gif)

#####Android dex分包方案 - 65535问题

```Java
UNEXPECTED TOP-LEVEL EXCEPTION:  
java.lang.IllegalArgumentException: method ID not in [0, 0xffff]: 65536  
at com.android.dx.merge.DexMerger$6.updateIndex(DexMerger.java:501)  
at com.android.dx.merge.DexMerger$IdMerger.mergeSorted(DexMerger.java:282)  
at com.android.dx.merge.DexMerger.mergeMethodIds(DexMerger.java:490)  
at com.android.dx.merge.DexMerger.mergeDexes(DexMerger.java:167)  
at com.android.dx.merge.DexMerger.merge(DexMerger.java:188)  
at com.android.dx.command.dexer.Main.mergeLibraryDexBuffers(Main.java:439)  
at com.android.dx.command.dexer.Main.runMonoDex(Main.java:287)  
at com.android.dx.command.dexer.Main.run(Main.java:230)  
at com.android.dx.command.dexer.Main.main(Main.java:199)  
at com.android.dx.command.Main.main(Main.java:103)
```

 在Android系统中，一个App的所有代码都在一个Dex文件里面。Dex是一个类似Jar的存储了多有Java编译字节码的归档文件。因为Android系统使用Dalvik虚拟机，所以需要把使用Java Compiler编译之后的class文件转换成Dalvik能够执行的class文件。这里需要强调的是，Dex和Jar一样是一个归档文件，里面仍然是Java代码对应的字节码文件。当Android系统启动一个应用的时候，有一步是对Dex进行优化，这个过程有一个专门的工具来处理，叫DexOpt。DexOpt的执行过程是在第一次加载Dex文件的时候执行的。这个过程会生成一个ODEX文件，即Optimised Dex。执行ODex的效率会比直接执行Dex文件的效率要高很多。但是在早期的Android系统中，DexOpt有一个问题。DexOpt会把每一个类的方法id检索起来，存在一个链表结构里面。但是这个链表的长度是用一个short类型来保存的，导致了方法id的数目不能够超过65536个。当一个项目足够大的时候，显然这个方法数的上限是不够的。尽管在新版本的Android系统中，DexOpt修复了这个问题，但是我们仍然需要对低版本的Android系统做兼容.


####热修复在Android中实现思路

DexClassLoader介绍
DexClassLoader是一个类加载器，可以用来从.jar和.apk文件中加载class。可以用来加载执行没用和应用程序一起安装的那部分代码。

所以我们的补丁可以是一个dex文件，把dex文件当作资源文件下发，app动态加载该dex，即可替换原先有bug的dex。

相关问题
> * 简单的概括一下，就是把多个dex文件塞入到app的classloader之中，但是android dex拆包方案中的类是没有重复的，如果classes.dex和classes1.dex中有重复的类，当用到这个重复的类的时候，系统会选择哪个类进行加载呢？

> * 当AClass中用到了BClass，但是A和B不在同一个类中，该如何处理?(CLASS_ISPREVERIFIED)


##总结
###优点
> iOS
> * 框架轻便小巧。
> * 可以进行简单的热修复，实时的修复一些简单的 bug。
> * 可以远程访问，也可以存本地。
> * 基本的类型都已经支持,包括 GCD，block，协议，动画等。

> Android
> * ？


###缺点

> iOS
> * 没有 IDE支持，开发效率低。要求使用者熟悉 JavaScript 和 objective -c 两种语言。需要一定的语言功底，在进行 JS 脚本的编写时，由于要把 oc 中对象的属性等原样书写，只能用 OC 的思维编写代码。容易造成拼写错误，没有提示，只能在运行程序时候调试。
> * 在网络不稳定的情况下，有可能不能很好加载脚本，是否会引起未知
> * 代码碎片化

> Android
> * ？

##相关资料
> [安卓App热补丁动态修复技术介绍](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=1&srcid=1106Imu9ZgwybID13e7y2nEi#wechat_redirect)
>
> [iOS JSPatch](JSPatch)
>
> [Android DroidFix](DroidFix)
>
> [Android Nuwa](https://github.com/jasonross/Nuwa)
>
> [Android AndFix](https://github.com/alibaba/AndFix)
>
> [ Android执行文件apk的组成结构](http://blog.csdn.net/itachi85/article/details/6460158)
>
> [ Android 解决65535的限制 使用android-support-multidex解决Dex超出方法数的限制问题,让你的应用不再爆棚](http://blog.csdn.net/x_i_a_o_h_a_i/article/details/46544341)
>
> [ Android dex分包方案](http://blog.csdn.net/vurtne_ye/article/details/39666381)
>
> [dex 分包变形记](https://my.oschina.net/bugly/blog/536448)
>
> [Android 热补丁动态修复框架小结](http://blog.csdn.net/lmj623565791/article/details/49883661)
