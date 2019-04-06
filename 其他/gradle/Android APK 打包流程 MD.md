| Markdown版本笔记 | 我的GitHub首页 | 我的博客 | 我的微信 | 我的邮箱 |  
| :------------: | :------------: | :------------: | :------------: | :------------: |  
| [MyAndroidBlogs][Markdown] | [baiqiantao][GitHub] | [baiqiantao][博客] | bqt20094 | baiqiantao@sina.com |  
  
[Markdown]:https://github.com/baiqiantao/MyAndroidBlogs  
[GitHub]:https://github.com/baiqiantao  
[博客]:http://www.cnblogs.com/baiqiantao/  
  
Android APK 打包流程 MD  
***  
目录  
===  

- [APK 的打包流程](#APK-的打包流程)
	- [整体流程](#整体流程)
	- [资源的编译和打包](#资源的编译和打包)
		- [资源ID](#资源ID)
		- [资源索引](#资源索引)
- [概况](#概况)
- [具体打包过程](#具体打包过程)
	- [aapt阶段](#aapt阶段)
	- [aidl阶段](#aidl阶段)
	- [Java Compiler阶段](#Java-Compiler阶段)
	- [dex阶段](#dex阶段)
	- [apkbuilder阶段](#apkbuilder阶段)
	- [Jarsigner阶段](#Jarsigner阶段)
	- [zipalign阶段](#zipalign阶段)
  
# APK 的打包流程  
[参考](https://github.com/guoxiaoxing/android-open-source-project-analysis)  
  
Android的包文件APK分为两个部分：`代码和资源`，所以打包方面也分为资源打包和代码打包两个方面，这篇文章就来分析资源和代码的编译打包原理。  
  
Android打包流程详图：  
  
<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/vm/apk_package_flow_detail.png"/>  
  
## 整体流程  
  
APK整体的的打包流程如下图所示：  
  
<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/vm/apk_package_flow.png"/>  
  
具体说来：  
- 通过`AAPT`工具进行资源文件（包括AndroidManifest.xml、布局文件、各种xml资源等）的打包，生成R.java文件。  
- 通过`AIDL`工具处理AIDL文件，生成相应的Java文件。  
- 通过`Javac`工具编译项目源码，生成Class文件。  
- 通过`DX`工具将所有的Class文件转换成DEX文件，该过程主要完成Java字节码转换成Dalvik字节码，压缩常量池以及清除冗余信息等工作。  
- 通过`ApkBuilder`工具将资源文件、DEX文件打包生成APK文件。  
- 利用`KeyStore`对生成的APK文件进行签名。  
- 如果是正式版的APK，还会利用`ZipAlign`工具进行对齐处理，对齐的过程就是将APK文件中所有的资源文件举例文件的起始距离都偏移4字节的整数倍，这样通过内存映射访问APK文件的速度会更快。  
  
上述流程都是Android Studio在编译时调用各种`编译命令`自动完成的，具体说来，如下所示：  
  
1、创建Android工程。  
> android  create project \  
    -n packageTest2 \  
    -a MainActivity \  
    -k com.package.test2 \  
    -t android-23 \  
    -p ./PackageTest2  
      
2、编译R文件  
> aapt  package \  
   -f \  
   -J ./gen \  
   -M ./AndroidManifest.xml \  
   -S ./res/ \  
   -I /Users/RadAsm/Library/AndroidSDK/sdk/platforms/android-23/android.jar  
   
3、编译源代码文件  
> javac -source 1.6 \  
   -target 1.6 \  
   -cp /Users/RadAsm/Library/AndroidSDK/sdk/platforms/android-23/android.jar \  
   ./src/com/packtest/test1/MainActivity.java ./src/com/packtest/test1/R.java \  
   -d ./gen/classes  
  
4、编译DEX文件  
>  dx --dex \  
    --verbose \  
    --output ./gen/dex/packtest1.dex   
    ./gen/classes/  
  
5、生成APK文件  
> aapt package   
     -f \  
     -J ./gen \  
     -M ./AndroidManifest.xml \  
     -S ./res/ \  
     -I /Users/RadAsm/Library/AndroidSDK/sdk/platforms/android-23/android.jar \  
     -F ./output/res.apk  
    
6、APK文件对齐  
> zipalign -v -p 4 packagetest_unsigned.apk packagetest_aligned_unsigned.apk  
  
7、APK签名  
> apksigner sign --ks my-release-key.jks my-app.apk  
  
以上便是APK打包的整个流程，我们再来总结一下：  
- 除了assets和res/raw资源被原装不动地打包进APK之外，其它的资源都会被编译或者处理；  
- 除了assets资源之外，其它的资源都会被赋予一个资源ID；  
- 打包工具负责编译和打包资源，编译完成之后，会生成一个resources.arsc文件和一个R.java，前者保存的是一个资源索引表，后者定义了各个资源ID常量。  
- 应用程序配置文件AndroidManifest.xml同样会被编译成二进制的XML文件，然后再打包到APK里面去。  
- 应用程序在运行时通过AssetManager来访问资源，或通过资源ID来访问，或通过文件名来访问。  
  
理解了整体的流程，我们再来看看具体的细节。  
  
## 资源的编译和打包  
在分析资源的编译和打包之前，我们先来了解一下Android程序包里有哪些资源。  
  
我们知道Android应用程序的设计也是`代码与资源相分离`的，Android的资源文件可以分为两大类：  
  
> assets：assets资源放在主工程assets目录下，它里面保存一些原始的文件，可以以任何方式来进行组织，这些文件最终会原封不动的被打包进APK文件中。  
  
获取asset资源也十分简单，如下所示：  
```java  
InputStream is = getAssets.open("fileName");  
```  
  
> res：res资源放在主工程的res目录下，这类资源一般都会在编译阶段生成一个资源ID供我们使用。  
  
res资源包含了我们开发中使用的各种资源，具体说来：  
- animator  
- anim  
- color  
- drawable  
- layout  
- menu  
- raw  
- values  
- xml  
  
这些资源的含义大家应该都很熟悉，这里就不再赘述。  
  
上述9种类型的资源文件，除了`raw`类型资源，以及Bitmap文件的`drawable`类型资源之外，其它的资源文件均为文本格式的`XML`文件，它们在打包的过程中，会被编译成二进制格式的XML文件。这些二进制格式的XML文件分别有一个字符串资源池，用来保存文件中引用到的每一个字符串，包括XML元素标签、属性名称、属性值，以及其它的一切文本值所使用到的字符串。这样原来在文本格式的XML文件中的每一个放置字符串的地方在二进制格式的XML文件中都被替换成一个索引到字符串资源池的整数值，这写整数值统一保存在  
`R.java`类中，R.java会和其他源文件一起编译到APK中去。  
  
前面我们提到xml编写的Android资源文件都会编译成二进制格式的xml文件，资源的打包都是由`AAPT`工具来完成的，资源打包主要有以下流程：  
- 解析`AndroidManifest.xml`，获得应用程序的包名称，创建资源表。  
- 添加被引用资源包，被添加的资源会以一种资源ID的方式定义在`R.java`中。  
- 资源打包工具创建一个`AaptAssets`对象，收集当前需要编译的资源文件，收集到的资源保存在AaptAssets对象对象中。  
- 将上一步AaptAssets对象保存的资源，添加到资源表`ResourceTable`中去，用于最终生成资源描述文件`resources.arsc`。  
- 编译`values`类资源，这类资源包括数组、颜色、尺寸、字符串等值。  
- 给bag、style、array这类资源分配资源ID。  
- 编译xml资源文件，编译的流程分为四步：① 解析xml文件 ② 赋予属性名称资源ID ③ 解析属性值 ④ 将xml文件从文本格式转换为二进制格式。  
- 生成资源索引表`resources.arsc`。  
  
### 资源ID  
每个Android项目里都有有一个R.java文件，如下所示：  
```java  
public final class R {  
     //...  
     public static final class anim {  
        public static final int abc_fade_in=0x7f010000;  
     }  
     public static final class attr {  
         public static final int actionBarDivider=0x7f020000;  
     }  
     public static final class string {  
          public static final int actionBarDivider=0x7f020000;  
     }  
     //...  
}  
```  
  
每个资源项后的整数就是资源ID，资源ID是一个4字节的无符整数，如下所示：  
- 最高字节是Package ID表示命名空间，标明资源的来源，Android系统自己定义了两个Package ID，系统资源命名空间：0x01 和 应用资源命名空间：0x7f。  
- 次字节是Type ID，表示资源的类型，例如：anim、color、string等。  
- 最低两个字节是Entry ID，表示资源在其所属资源类型中所出现的次序。  
  
### 资源索引  
上面提到，最终生成的是资源索引表`resources.arsc`，Android正是利用这个索引表根据资源ID进行资源的查找，为不同语言、不同地区、不同设备提供相对应的最佳资源。查找是通过Resources和AssetManger来完成的，这个我们下面会讲。  
  
resources.arsc 是一个编译后的二进制文件，在Android Stduio里打开以后是这样的，如下所示：  
<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/resource/resources_arsc_file.png"/>  
  
可以看到resources.arsc里存放了各类资源的索引参数和配置信息。  
  
resources.arsc的文件格式如下所示：  
<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/resource/resources_arsc_structure.png"/>  
  
注：整个文件都是有一系列chuck（块）构成的，chuck是整个文件的划分单位，每个模块都是一个chuck，chuck最前面是一个ResChunk_header的结构体，用来描述整个chunk的信息，更多关于索引表格式的细节，可以查阅源码：  
  
👉 [ResourceTypes.h](https://android.googlesource.com/platform/frameworks/base/+/56a2301/include/androidfw/ResourceTypes.h)  
  
resources.arsc 索引表从上至下文件格式依次为：  
- 文件头：数据结构用ResTable_header来描述，用来描述整个文件的信息，包括文件头大小，文件大小，资源包Package的数量等信息。  
- 全局字符串池：存放所有的字符串，所以资源复用这些字符串，字符串里存放的是资源文件的路径名和资源值等信息。全局字符串池分为资源类型（type）字符串池和  
- 资源包：会有多个（例如：系统资源包、应用资源包）。  
  
资源包也被划分为以下几个部分：  
- 包头：描述资源包相关信息。  
- 资源类型字符串池：存放资源的类型。  
- 资源名称字符串池：存放资源的名称。  
- 配置列表：存放语音、位置等手机配置信息，用来作为查找资源的标准。  
  
从这里可以看到resources.arsc索引表存在很多常量池，常量池的使用目的也很明显，就是提供资源的复用率，减少resources.arsc索引表的体积，提高索引效率。  
  
  
# 概况  
[参考](https://www.jianshu.com/p/7c288a17cda8)  
  
Android APK是如何来的呢？  
  
怀着这个问题去查资料，发现了下边这张图。  
![](https://upload-images.jianshu.io/upload_images/1441907-ea4c00318441fe9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/747)  
  
解压一个普通的apk文件后，解压出来的文件包括：  
- classes.dex：.dex文件  
- resources.arsc：resources resources文件  
- AndroidManifest.xml：AndroidManifest.xml文件  
- res：uncompiled resources  
- META-INF：签名文件夹  
    - `MANIFEST.MF`文件：版本号以及每一个文件的哈希值(BASE64)，包括资源文件。这个是对每个文件的整体进行SHA1(hash)。  
    - `CERT.SF`：这个是对每个文件的头3行进行SHA1 hash。  
    - `CERT.RSA`：这个文件保存了签名和公钥证书。  
  
# 具体打包过程  
  
## aapt阶段  
使用aapt来打包res资源文件，生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）  
- res目录，有9种子目录  
- R.java文件。里面拥有很多个静态内部类，比如layout，string等。每当有这种资源添加时，就在R.java文件中添加一条静态内部类里的静态常量类成员，且所有成员都是int类型。  
- resources.arsc文件。这个文件记录了所有的应用程序资源目录的信息，包括每一个资源名称、类型、值、ID以及所配置的维度信息。我们可以将这个文件想象成是一个资源索引表，这个资源索引表在给定资源ID和设备配置信息的情况下，能够在应用程序的资源目录中快速地找到最匹配的资源。  
  
## aidl阶段  
AIDL，Android接口定义语言，Android提供的IPC的一种独特实现。  
这个阶段处理.aidl文件，生成对应的Java接口文件。  
  
## Java Compiler阶段  
通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件。  
  
## dex阶段  
通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex。  
  
## apkbuilder阶段  
将`classes.dex`、`resources.arsc`、`res`文件夹(res/raw资源被原装不动地打包进APK之外，其它的资源都会被编译或者处理)、Other Resources(`assets`文件夹)、`AndroidManifest.xml`打包成apk文件。  
  
## Jarsigner阶段  
对apk进行签名，可以进行Debug和Release 签名。  
  
## zipalign阶段  
release mode 下使用 aipalign 进行align，即对签名后的apk进行对齐处理。  
  
Zipalign是一个android平台上整理APK文件的工具，它对apk中未压缩的数据进行4字节对齐，对齐后就可以使用`mmap`函数读取文件，可以像读取内存一样对普通文件进行操作。如果没有4字节对齐，就必须显式的读取，这样比较缓慢并且会耗费额外的内存。  
  
在 Android SDK 中包含一个名为 `zipalign` 的工具，它能够对打包后的 app 进行优化。 其位于 SDK 的 `\build-tools\23.0.2\zipalign.exe` 目录下  
  
2019-2-18  
