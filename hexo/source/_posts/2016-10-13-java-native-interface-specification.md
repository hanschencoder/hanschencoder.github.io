---
layout: post
title: JNI教程与技术手册
tags: [jni,java,java native interface specification,ndk]
categories: 
- Android
---

# 概述

对于JNI，有些童鞋在没有接触过的时候，可能会觉得比较复杂，但是其实当你真正去了解、去使用的时候，就会发现JNI的使用还是比较简单的，JNI本身提供了一系列的API让我们可以在native方法中操作java。JNI的使用无非也就是使用这些接口和java交互。这几天在学习JNI接口的时候，发现网上搜索的JNI的中文虽然不少，但是很多都是零零碎碎的小例子，有一些官方文档的翻译，但要么是不全面，要么是资料比较旧了，干脆自己根据[java native interface specification][1]整理了一份技术资料。当然，很多时候看中文资料是词不达意的，如果文中有疑问的地方欢迎指出，或者翻阅原文

首先，JNI是一个本地编程接口。它允许运行在Java虚拟机的Java代码与用其他语言（如C,C++和汇编）编写的库交互
JNI最大的好处是JNI不受Java虚拟机实现方式的限制，因此，Java虚拟机厂商添加JNI的支持并不会影响虚拟机其它功能模块。native代码只需要编写一遍，就可以在所有支持JNI的虚拟机上工作。

通过JNI，你可以在native代码中：

 - 创建、检查或者更新java对象
 - 调用java方法
 - 捕捉和抛出异常
 - 加载class和获取class信息
 - 运行时类型检查

<!-- more -->

# JNI接口函数和指针
本地代码通过JNI函数（接口指针，接口指针是指针的指针）来访问java VM。
![2019-9-2-11-13-8.png](http://image.hanschen.site/master/2019-9-2-11-13-8.png)

JNI接口指针只在当前线程有效，因此在native方法中**不要跨线程传递接口指针参数**。native方法接收JNI接口指针参数，VM确保在同一个线程中调用native方法的时候，传递同一个接口指针给接口指针给native方法。然而，native方法可能在Java中的不同线程中调用，所以native方法接收到的接口指针可能是不一样的。
 
# 编译、加载和链接本地方法

## 编译
Java VM是多线程的 ，所以native libraries应该用多线程编译器来进行编译和链接。例如使用Sun Studio compiler编译器的时候，要为c++代码添加`-mt`标记；使用 GNU gcc compiler的时候，需添加`-D_REENTRANT` 或`-D_POSIX_C_SOURCE`

## 加载
native库通过`System.loadLibrary`方法进行加载。如：
```java
package pkg;  

class Cls { 

     native double f(int i, String s); 
     
     static { 
         System.loadLibrary("pkg_Cls"); 
     } 
} 
```
系统会对library名会进行转换，在不同平台上有不同的转换方式，例如，Solaris系统转换pkg_Cls为libpkg_Cls.so，而Win32系统转换pkg_Cls为pkg_Cls.dll

## 链接
如果系统不支持动态链接，那么所有本地方法需要预链接到虚拟机，这种情况下，VM已经完成`System.loadLibrary`了。程序员也可以调用JNI函数RegisterNatives()来注册该类关联的本地方法

# Native Method命名解析
一个本地方法名有以下几个组成部分：

 1. 前缀`Java_`
 2. 完整类名（类名中的`.`用`_`代替）
 3. 下划线`_`
 4. 方法名（方法名中的特殊字符需要转义）
 5. 参数签名（非必须，有重载方法的时候才需要），如果有重载的本地方法，需要再添加两个下划线`__`，然后再添加方法签名（由java字段描述符描述，用`_`代替描述符中的包名分割`/`符，签名中的特殊字符需要转义）

## Unicode 转义字符

| 转义符 | 说明   |
| :------ | :----- |
| _0XXXX | 一个Unicode字符XXXX。注意小写是用来表示非ascii Unicode字符, 如:_0abcd与_0ABCD不相同 |
| _1 | 字符`_` | 
| _2 | 参数签名中的字符`;` | 
| _3 | 参数签名中的字符`[` | 

<span id="java_signatures"></span>
## java字段描述符

| Java 类型   | 符号   |
| :--------   | :----- |
| Boolean | Z |
| Byte | B |
| Char | C |
| Short | S |
| Int | I |
| Long | J |
| Float | F |
| Double | D |
| Void | V |
| 数组 | [ , 如：`int[]`->  `[I`,  `int[][]`->  `[[I`, `Thread[]`->  `[Ljava/lang/Thread;` |
| objects | 以"L"开头，以";"结尾，中间是用"/" 隔开的包及类名。比如：`Ljava/lang/String;`如果是嵌套类，则用`$`来表示嵌套。例如 "`(Ljava/lang/String;Landroid/os/FileUtils$FileStatus;)Z`" |

## 举例
```java
package pkg;  

class Cls { 
    //对应本地方法名：Java_pkg_Cls_f__ILjava_lang_String_2
    native double f(int i, String s); 

     ... 

} 
```

# Native方法参数

 - JNI接口指针是native方法的第一个参数，JNI接口指针的类型是`JNIEnv`。
 - 第二个参数取决于native method是否静态方法，如果是非静态方法，那么第二个参数是对对象的引用，如果是静态方法，则第二个参数是对它的class类的引用
 - 剩下的参数跟Java方法参数一一对应

```java
package pkg;  

class Cls { 

     native double f(int i, String s); 

     ... 

}
```
```c
//注意，c和c++在使用JNI接口的时候有点不一致，请仔细观察通过env调用接口的调用方式
//C版本
jdouble Java_pkg_Cls_f__ILjava_lang_String_2 (
     JNIEnv *env,        /* interface pointer */
     jobject obj,        /* "this" pointer */
     jint i,             /* argument #1 */
     jstring s)          /* argument #2 */
{
     /* Obtain a C-copy of the Java string */
     const char *str = (*env)->GetStringUTFChars(env, s, 0);

     /* process the string */
     ...

     /* Now we are done with str */
     (*env)->ReleaseStringUTFChars(env, s, str);

     return ...
}

//C++版本
extern "C" /* specify the C calling convention */  
jdouble Java_pkg_Cls_f__ILjava_lang_String_2 ( 

     JNIEnv *env,        /* interface pointer */ 

     jobject obj,        /* "this" pointer */ 

     jint i,             /* argument #1 */ 

     jstring s)          /* argument #2 */ 

{ 

     const char *str = env->GetStringUTFChars(s, 0); 

     ... 

     env->ReleaseStringUTFChars(s, str); 

     return ... 

} 
```

# java对象引用

 - 基本类型（如整型，字符等）在Java和native之间是采用值传递
 - Java对象采用的是引用传递

虚拟机必须保持已传递给native的对象的引用，以使这些对象不被垃圾回收器回收。native code也必须有一种方法通知虚拟机它不再需要某个对象，并且垃圾收集器必须能够将其回收
 
# 全局引用、局部引用和弱全局引用
JNI中使用的引用可以划分为三类：`全局引用`、`局部引用`和`弱全局引用`。局部引用在方法调用的时候有效，在方法调用结束之后会自动释放。全局引用会一直可用，直到显式地对其进行释放。弱全局引用跟全局引用的区别是弱全局引用持有的java对象可以被VM进行回收，所以才使用弱全局引用前，我们需要对其进行检测，看它对应的对象是否被回收了。

对象是作为局部引用传递给native方法的，所有通过`JNIEnv`方法（也就是JNI提供的API）返回的java对象都是局部引用。JNI允许程序从局部引用创建一个全局引用。`JNIEnv`的方法既可以接收全局引用也可以接收局部引用。一个native方法既可以返回局部引用也可以返回全局引用。

大多数情况下，在方法调用结束之后，我们依赖VM去帮我们释放所有局部引用，但是以下几种情况下，我们应该显式地释放局部引用：

 - 方法中创建了一个比较大的java对象的，并持有其局部引用，使用完之后，如果接下来都不再需要使用了，如果仍然不对它进行释放的话，在方法结束之前，这个对象都不会进行释放，这样会对资源造成浪费
 - JNI会将创建的局部引用都存储在一个局部引用表中，如果这个表超过了最大容量限制，就会造成局部引用表溢出，使程序崩溃。比如在一个循环中创建局部引用，最好在每一轮循环中释放局部引用，否则随着循环次数增加，很可能就内存溢出了

局部引用仅仅在其创建的线程内有效，**native代码不能跨线程传递局部引用**。

# 访问字段和方法

JNI允许native代码访问对象的成员以及调用它的方法，通过两个步骤即可实现访问，比如，我们需要调用`cls`中的`f`方法：
```java
jmethodID mid = env->GetMethodID(cls, "f", "(ILjava/lang/String;)D");//mid可以重复使用
jdouble result = env->CallDoubleMethod(obj, mid, 10, str);
```

但是需要注意的是，字段ID或方法ID并不能防止VM卸载该类。当类被卸载后，方法ID和字段ID将变成不可用的。因此，我们需要确保：
 - 持有class的引用，让它不被卸载，或者
 - 重新获取方法id或者字段id

# 程序错误检测
JNI不对空指针或非法参数类型等错误进行检测，因为：

 - 检查所有可能的错误会降低方法执行的性能
 - 在很多时候，没有足够的运行时信息去进行检测

程序员不得传递一个非法指针或者错误的类型给JNI函数，否则可能会导致系统异常货虚拟机崩溃

# Java异常
JNI允许本地方法抛出处理任何异常，也可以处理Java中抛出的异常，剩下没有处理的异常会继续给VM处理
 
# 异常和错误码
大多数情况下，JNI提供的方法通过返回错误码或者抛出java异常来处理错误，因此，程序中可以：

 - 检查JNI函数返回值
 - 调用`ExceptionOccurred()`方法，获取方法中抛出的异常

**有两种情况下，程序需要优先检测java异常而不是先检测返回码**

 - 通过JNI调用Java方法的时候，需要`ExceptionOccurred()`检测是否在Java方法中抛出了异常
 - 一些访问数组的方法，它不返回错误码，但是会抛出`ArrayIndexOutOfBoundsException` 或者 `ArrayStoreException`异常

# 异常处理
有两种方法可以在本地方法中处理异常

 - 检测到异常的时候立即返回，异常将会在调用该本地代码的地方抛出
 - 在本地方法中调用`ExceptionClear()`清除异常，处理接下来的逻辑

异常抛出的时，本地方法需清除异常后，才能继续调用其他JNI接口方法，有异常发生后，只有以下方法才能被安全调用：
```java
  ExceptionOccurred()
  ExceptionDescribe()
  ExceptionClear()
  ExceptionCheck()
  ReleaseStringChars()
  ReleaseStringUTFChars()
  ReleaseStringCritical()
  Release<Type>ArrayElements()
  ReleasePrimitiveArrayCritical()
  DeleteLocalRef()
  DeleteGlobalRef()
  DeleteWeakGlobalRef()
  MonitorExit()
  PushLocalFrame()
  PopLocalFrame()
```

# 原始类型
| Java 类型   | native类型   | 描述  |
| :--------   | :----- | :----- |
| boolean | jboolean | unsigned 8 bits |
| byte | jbyte | signed 8 bits |
| char | jchar | unsigned 16 bits |
| short | jshort | signed 16 bits |
| int | jint | signed 32 bits |
| long | jlong | signed 64 bits |
| float | jfloat | 32 bits |
| double | jdouble | 64 bits |
| void | void | N/A |

JNI中还定义了以下两个宏定义方便使用：
```c
#define JNI_FALSE  0 
#define JNI_TRUE   1 
```

# 引用类型
JNI为不同的java对象提供了不同的引用类型，JNI引用类型如下：
![2019-9-2-11-13-39.png](http://image.hanschen.site/master/2019-9-2-11-13-39.png)
在c里面，所有JNI引用类型其实都是`jobject`

# 字段和方法ID
在C中，字段和方法ID是一个指向结构体的指针
```c
struct _jfieldID;                       /* opaque structure */
typedef struct _jfieldID* jfieldID;     /* field IDs */

struct _jmethodID;                      /* opaque structure */
typedef struct _jmethodID* jmethodID;   /* method IDs */
```

# 值类型
值类型`jvalue `是一个联合体结构，定义如下：
```c
typedef union jvalue {
    jboolean    z;
    jbyte       b;
    jchar       c;
    jshort      s;
    jint        i;
    jlong       j;
    jfloat      f;
    jdouble     d;
    jobject     l;
} jvalue;
```

# 签名类型描述
请参考：[java字段描述符](#java_signatures)
如：
```java
//方法签名为：(ILjava/lang/String;[I)J 
long f (int n, String s, int[] arr);
```

# UTF-8字符
JNI的 UTF-8与标准的 UTF-8格式有两个区别：

 - 空字符`(char)0`使用双字节格式编码，而不是单字节编码，所以Java虚拟机的UTF-8字符串不可能有嵌入的空值。
 - 只使用单字节、双字节和三字节编码格式，不支持标准的四字节编码，用`two-times-three-byte`格式代替  

# JNI接口函数
在下面的说明中，`必须`说明JNI函数必须接受一个非空对象，你必须保证传入的参数不为空，JNI函数不需要再对它进行空指针判断

## 返回码说明
```c
#define JNI_OK          (0)         /* no error */
#define JNI_ERR         (-1)        /* generic error */
#define JNI_EDETACHED   (-2)        /* thread detached from the VM */
#define JNI_EVERSION    (-3)        /* JNI version error */

#define JNI_COMMIT      1           /* copy content, do not free buffer */
#define JNI_ABORT       2           /* free buffer w/o copying back */
```

## 接口详细
```c
    /**
     * 返回本地方法接口的版本
     *
     * @param env JNI接口指针
     *
     * @return 高16位返回主版本号，低16位返回次版本号,如在JDK/JRE 1.6中，返回0x00010006。也有可能返回 JNI_EDETACHED 和 JNI_EVERSION 错误码
     */
    jint (*GetVersion)(JNIEnv *);


    /**
     * 从二进制的.class的数据缓冲区中加载类
     *
     * @param env JNI接口指针
     * @param name UTF8编码的需要加载的类的名字
     * @param loader 类加载器
     * @param buf 包含.class字节码的数组
     * @param bufLen 长度
     *
     * @return class对象或NULL
     *
     * @throws ClassFormatError 不是有效的class数据
     * @throws ClassCircularityError 类或接口是自身的父类或自身继承了该接口
     * @throws OutOfMemoryError 内存不足
     * @throws SecurityException 如果该类是属于java包的
     */
    jclass (*DefineClass)(JNIEnv *, const char *, jobject, const jbyte *, jsize);


    /**
     * 用于加载本地定义的类
     *
     * @param env JNI接口指针
     * @param name 完整的包名("/"代替".") 或 数组类型字段描述("["开头，紧跟签名描述)，如"java/lang/String" for java.lang.String, "[Ljava/lang/Object;" for java.lang.Object[]
     *
     * @return class对象或NULL
     *
     * @throws ClassFormatError 不是有效的class数据
     * @throws ClassCircularityError 类或接口是自身的父类或自身继承了该接口
     * @throws OutOfMemoryError 内存不足
     * @throws NoClassDefFoundError 找不到name对应的class类
     */
    jclass (*FindClass)(JNIEnv *, const char *);


    /**
     * 从java.lang.reflect.Method 或 java.lang.reflect.Constructor 获取method ID
     *
     * @param env JNI接口指针
     * @param method java.lang.reflect.Method 或 java.lang.reflect.Constructor对象
     *
     * @return 方法ID
     */
    jmethodID (*FromReflectedMethod)(JNIEnv *, jobject);


    /**
     * 从java.lang.reflect.Field获取field ID
     *
     * @param env JNI接口指针
     * @param field java.lang.reflect.Field对象
     *
     * @return field ID
     */
    jfieldID (*FromReflectedField)(JNIEnv *, jobject);


    /**
     * 从method ID获取 java.lang.reflect.Method 或 java.lang.reflect.Constructor 对象
     *
     * @param env JNI接口指针
     * @param cls 该方法的类对象
     * @param methodID 方法ID
     * @param isStatic 是否静态方法
     *
     * @return java.lang.reflect.Method 或 java.lang.reflect.Constructor 对象
     *
     * @throws OutOfMemoryError 内存不足
     */
    jobject (*ToReflectedMethod)(JNIEnv *, jclass, jmethodID, jboolean);


    /**
     * 如果clazz不是class对象或接口，则返回该class的超类
     *
     * @param env JNI接口指针
     * @param clazz class对象
     *
     * @return 返回输入类的父类 或 NULL
     */
    jclass (*GetSuperclass)(JNIEnv *, jclass);


    /**
     * class1是否可以安全地转换为class2，以下三种情况会返回TRUE
     * 1. 当class1和class2是同一个java class的引用
     * 2. class1是class2的子类
     * 3. class2是class1的某个接口
     *
     * @param env JNI接口指针
     * @param clazz1 class1
     * @param clazz2 class2
     *
     * @return JNI_TRUE or JNI_FALSE
     */
    jboolean (*IsAssignableFrom)(JNIEnv *, jclass, jclass);


    /**
     * 根据 field ID 获取 java.lang.reflect.Field 对象
     *
     * @param env JNI接口指针
     * @param cls 该方法的类对象
     * @param fieldID 字段ID
     * @param isStatic 是否静态变量
     *
     * @return java.lang.reflect.Field 对象
     *
     * @throws OutOfMemoryError 内存不足
     */
    jobject (*ToReflectedField)(JNIEnv *, jclass, jfieldID, jboolean);


    /**
     * 抛出异常
     *
     * @param env JNI接口指针
     * @param obj java.lang.Throwable 对象
     *
     * @return 0：成功， 负数：失败
     *
     * @throws Throwable
     */
    jint (*Throw)(JNIEnv *, jthrowable);


    /**
     * 根据clazz和message构造一个异常对象，并将它抛出
     *
     * @param env JNI接口指针
     * @param clazz java.lang.Throwable的子类
     * @param message 错误信息
     *
     * @return 0：成功， 负数：失败
     *
     * @throws Throwable
     */
    jint (*ThrowNew)(JNIEnv *, jclass, const char *);


    /**
     * 判断是否有异常抛出，在调用ExceptionClear()或java代码处理了exception之前，都可以用这个方法判断是否有异常
     *
     * @param env JNI接口指针
     *
     * @return 异常对象 or NULL
     */
    jthrowable (*ExceptionOccurred)(JNIEnv *);


    /**
     * 打印异常信息
     *
     * @param env JNI接口指针
     */
    void (*ExceptionDescribe)(JNIEnv *);


    /**
     * 清除所有已抛出的异常
     *
     * @param env JNI接口指针
     */
    void (*ExceptionClear)(JNIEnv *);


    /**
     * 抛出致命错误并且不希望虚拟机进行恢复。无返回值
     *
     * @param env JNI接口指针
     * @param msg 错误信息
     */
    void (*FatalError)(JNIEnv *, const char *);


    /**
     * 创建一个新的本地引用帧
     *
     * @param env JNI接口指针
     * @param capacity 容量
     *
     * @return 0：成功，负数：失败
     *
     * @throws OutOfMemoryError
     */
    jint (*PushLocalFrame)(JNIEnv *, jint);


    /**
     * 弹出当前本地引用帧，释放所有本地引用
     *
     * @param env JNI接口指针
     * @param result
     *
     * @return
     */
    jobject (*PopLocalFrame)(JNIEnv *, jobject);


    /**
     * 为传入的obj创建全局引用，obj可以是全局引用也可以是局部引用。全局引用需要调用DeleteGlobalRef来释放
     *
     * @param env JNI接口指针
     * @param obj 全局或局部引用
     *
     * @return 全局引用 or NULL(内存不足)
     */
    jobject (*NewGlobalRef)(JNIEnv *, jobject);


    /**
     * 释放全局引用
     *
     * @param env JNI接口指针
     * @param globalRef 全局引用
     */
    void (*DeleteGlobalRef)(JNIEnv *, jobject);


    /**
     * 释放局部引用
     *
     * @param env JNI接口指针
     * @param localRef 局部引用
     */
    void (*DeleteLocalRef)(JNIEnv *, jobject);


    /**
     * 判断两个引用是否同一java对象的引用
     *
     * @param env JNI接口指针
     * @param ref1 引用1
     * @param ref2 引用2
     *
     * @return JNI_TRUE:两个引用指向同一个java对象
     */
    jboolean (*IsSameObject)(JNIEnv *, jobject, jobject);


    /**
     * 为传入的ref创建局部引用，ref可以是全局引用也可以是局部引用
     *
     * @param env JNI接口指针
     * @param ref 全局或局部引用
     *
     * @return 局部引用 or NULL
     */
    jobject (*NewLocalRef)(JNIEnv *, jobject);


    /**
     * 确保当前线程可以创建capacity个局部引用。在进入本地方法时，VM确保可以可以创建最少16个局部引用
     *
     * @param env JNI接口指针
     * @param capacity 局部引用个数
     *
     * @return 0：成功，负数：失败
     *
     * @throws OutOfMemoryError 内存不足
     */
    jint (*EnsureLocalCapacity)(JNIEnv *, jint);


    /**
     * 创建一个新的java对象（不会调用对象的构造方法）
     *
     * @param env JNI接口指针
     * @param clazz 非数组class对象
     *
     * @return java对象
     *
     * @throws InstantiationException clazz是一个接口或抽象类
     * @throws OutOfMemoryError 内存不足
     */
    jobject (*AllocObject)(JNIEnv *, jclass);


    /**
     * 构造一个新的java对象，method ID指用以生成该类的构造方法，method ID必须是通过GetMethodID()获得
     *
     * @param env JNI接口指针
     * @param clazz 非数组class对象
     * @param ... 传递给构造方法的参数
     *
     * @return java对象 or NULL(对象构造失败)
     *
     * @throws InstantiationException clazz是一个接口或抽象类
     * @throws OutOfMemoryError 内存不足
     */
    jobject (*NewObject)(JNIEnv *, jclass, jmethodID, ...);


    /**
     * 构造一个新的java对象，method ID指用以生成该类的构造方法，method ID必须是通过GetMethodID()获得
     *
     * @param env JNI接口指针
     * @param clazz 非数组class对象
     * @param args va_list结构，里面有传递给构造方法的参数
     *
     * @return java对象 or NULL(对象构造失败)
     *
     * @throws InstantiationException clazz是一个接口或抽象类
     * @throws OutOfMemoryError 内存不足
     */
    jobject (*NewObjectV)(JNIEnv *, jclass, jmethodID, va_list);


    /**
     * 构造一个新的java对象，method ID指用以生成该类的构造方法，method ID必须是通过GetMethodID()获得
     *
     * @param env JNI接口指针
     * @param clazz 非数组class对象
     * @param args 参数数组，里面是传递给构造方法的参数
     *
     * @return java对象 or NULL(对象构造失败)
     *
     * @throws InstantiationException clazz是一个接口或抽象类
     * @throws OutOfMemoryError 内存不足
     */
    jobject (*NewObjectA)(JNIEnv *, jclass, jmethodID, jvalue *);


    /**
     * 返回对象对应的class对象
     *
     * @param env JNI接口指针
     * @param obj 非空java对象
     *
     * @return class对象
     */
    jclass (*GetObjectClass)(JNIEnv *, jobject);


    /**
     * 判断obj是否clazz的实例对象
     *
     * @param env JNI接口指针
     * @param obj java对象
     * @param clazz class对象
     *
     * @return
     */
    jboolean (*IsInstanceOf)(JNIEnv *, jobject, jclass);


    /**
     * 返回非静态方法的method ID
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param name 方法名
     * @param sig 方法签名
     *
     * @return 方法ID or NULL
     *
     * @throws NoSuchMethodError 找不到对应的方法
     * @throws ExceptionInInitializerError class初始化失败
     * @throws OutOfMemoryError 内存不足
     */
    jmethodID (*GetMethodID)(JNIEnv *, jclass, const char *, const char *);


    /**
     * Call<type>Method(JNIEnv *env, jobject obj, jmethodID methodID, ...);调用参数放到可变参数中
     * Call<type>MethodA(JNIEnv *env, jobject obj, jmethodID methodID, const jvalue *args);调用参数放入jvalue数组
     * Call<type>MethodV(JNIEnv *env, jobject obj, jmethodID methodID, va_list args);调用参数放入va_list结构中
     *
     * 以上三组调用接口都是根据 method ID调用java实例方法（非静态方法）的接口，其中method ID是通过GetMethodID()获取的
     * 当这些方法用于调用java对象的私有方法或构造函数时，method ID必须从obj的真实类获取，而不应从其某个父类获取
     * <type>是方法的返回类型，三类接口间唯一的区别是methodID参数之后调用参数的不同
     *
     *
     * @param env JNI接口指针
     * @param obj java对象
     * @param methodID 方法ID
     * @param args 调用参数
     *
     * @return java方法返回结果
     *
     * @throws java方法中可能抛出的异常
     */
    jobject (*CallObjectMethod)(JNIEnv *, jobject, jmethodID, ...);
    jobject (*CallObjectMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jobject (*CallObjectMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jboolean (*CallBooleanMethod)(JNIEnv *, jobject, jmethodID, ...);
    jboolean (*CallBooleanMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jboolean (*CallBooleanMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jbyte (*CallByteMethod)(JNIEnv *, jobject, jmethodID, ...);
    jbyte (*CallByteMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jbyte (*CallByteMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jchar (*CallCharMethod)(JNIEnv *, jobject, jmethodID, ...);
    jchar (*CallCharMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jchar (*CallCharMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jshort (*CallShortMethod)(JNIEnv *, jobject, jmethodID, ...);
    jshort (*CallShortMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jshort (*CallShortMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jint (*CallIntMethod)(JNIEnv *, jobject, jmethodID, ...);
    jint (*CallIntMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jint (*CallIntMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jlong (*CallLongMethod)(JNIEnv *, jobject, jmethodID, ...);
    jlong (*CallLongMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jlong (*CallLongMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jfloat (*CallFloatMethod)(JNIEnv *, jobject, jmethodID, ...);
    jfloat (*CallFloatMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jfloat (*CallFloatMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    jdouble (*CallDoubleMethod)(JNIEnv *, jobject, jmethodID, ...);
    jdouble (*CallDoubleMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    jdouble (*CallDoubleMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);
    void (*CallVoidMethod)(JNIEnv *, jobject, jmethodID, ...);
    void (*CallVoidMethodV)(JNIEnv *, jobject, jmethodID, va_list);
    void (*CallVoidMethodA)(JNIEnv *, jobject, jmethodID, jvalue *);


    /**
     * CallNonvirtual<type>Method(JNIEnv *env, jobject obj, jclass clazz, jmethodID methodID, ...);调用参数放到可变参数中
     * CallNonvirtual<type>MethodA(JNIEnv *env, jobject obj, jclass clazz, jmethodID methodID, const jvalue *args);调用参数放入jvalue数组
     * CallNonvirtual<type>MethodV(JNIEnv *env, jobject obj, jclass clazz, jmethodID methodID, va_list args);调用参数放入va_list结构中
     *
     * 以上三组调用接口都是根据 method ID 和 class 调用java实例方法（非静态方法）的接口，其中method ID是基于clazz通过GetMethodID()获取的
     * <type>是方法的返回类型，三类接口间唯一的区别是methodID参数之后调用参数的不同
     * 注意，和Call<type>Method不同，如果子类重写了父类的方法，Call<type>Method调用的是子类的方法，如果想调用父类的方法，
     * 则需要用CallNonvirtual<type>Method，这个方法可以传入父类的class和父类的method id，从而达到调用父类方法的效果
     *
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param obj java对象
     * @param methodID 方法ID
     * @param args 调用参数
     *
     * @return java方法返回结果
     *
     * @throws java方法中可能抛出的异常
     */
    jobject (*CallNonvirtualObjectMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jobject (*CallNonvirtualObjectMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jobject (*CallNonvirtualObjectMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jboolean (*CallNonvirtualBooleanMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jboolean (*CallNonvirtualBooleanMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jboolean (*CallNonvirtualBooleanMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jbyte (*CallNonvirtualByteMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jbyte (*CallNonvirtualByteMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jbyte (*CallNonvirtualByteMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jchar (*CallNonvirtualCharMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jchar (*CallNonvirtualCharMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jchar (*CallNonvirtualCharMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jshort (*CallNonvirtualShortMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jshort (*CallNonvirtualShortMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jshort (*CallNonvirtualShortMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jint (*CallNonvirtualIntMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jint (*CallNonvirtualIntMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jint (*CallNonvirtualIntMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jlong (*CallNonvirtualLongMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jlong (*CallNonvirtualLongMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jlong (*CallNonvirtualLongMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jfloat (*CallNonvirtualFloatMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jfloat (*CallNonvirtualFloatMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jfloat (*CallNonvirtualFloatMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    jdouble (*CallNonvirtualDoubleMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    jdouble (*CallNonvirtualDoubleMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    jdouble (*CallNonvirtualDoubleMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);
    void (*CallNonvirtualVoidMethod)(JNIEnv *, jobject, jclass, jmethodID, ...);
    void (*CallNonvirtualVoidMethodV)(JNIEnv *, jobject, jclass, jmethodID, va_list);
    void (*CallNonvirtualVoidMethodA)(JNIEnv *, jobject, jclass, jmethodID, jvalue *);


    /**
     * 根据class对象获取非静态成员变量的field ID
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param name 变量名
     * @param sig 变量签名
     *
     * @return field ID or NULL
     *
     * @throws NoSuchFieldError 找不到对应的变量ID
     * @throws ExceptionInInitializerError class初始化失败
     * @throws OutOfMemoryError 内存不足
     */
    jfieldID (*GetFieldID)(JNIEnv *, jclass, const char *, const char *);


    /**
     * 根据field id取出对象中相应的变量值，field Id通过GetFieldID()获取
     *
     * @param env JNI接口指针
     * @param obj java对象
     * @param fieldID 有效的field id
     *
     * @return 相应的变量值
     */
    jobject (*GetObjectField)(JNIEnv *, jobject, jfieldID);
    jboolean (*GetBooleanField)(JNIEnv *, jobject, jfieldID);
    jbyte (*GetByteField)(JNIEnv *, jobject, jfieldID);
    jchar (*GetCharField)(JNIEnv *, jobject, jfieldID);
    jshort (*GetShortField)(JNIEnv *, jobject, jfieldID);
    jint (*GetIntField)(JNIEnv *, jobject, jfieldID);
    jlong (*GetLongField)(JNIEnv *, jobject, jfieldID);
    jfloat (*GetFloatField)(JNIEnv *, jobject, jfieldID);
    jdouble (*GetDoubleField)(JNIEnv *, jobject, jfieldID);


    /**
     * 根据field id为相应的变量设置新的值，field Id通过GetFieldID()获取
     *
     * @param env JNI接口指针
     * @param obj java对象
     * @param fieldID 有效的field id
     * @param value 要设置的值
     */
    void (*SetObjectField)(JNIEnv *, jobject, jfieldID, jobject);
    void (*SetBooleanField)(JNIEnv *, jobject, jfieldID, jboolean);
    void (*SetByteField)(JNIEnv *, jobject, jfieldID, jbyte);
    void (*SetCharField)(JNIEnv *, jobject, jfieldID, jchar);
    void (*SetShortField)(JNIEnv *, jobject, jfieldID, jshort);
    void (*SetIntField)(JNIEnv *, jobject, jfieldID, jint);
    void (*SetLongField)(JNIEnv *, jobject, jfieldID, jlong);
    void (*SetFloatField)(JNIEnv *, jobject, jfieldID, jfloat);
    void (*SetDoubleField)(JNIEnv *, jobject, jfieldID, jdouble);


    /**
     * 返回静态方法的method ID
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param name 方法名
     * @param sig 方法签名
     *
     * @return 方法ID or NULL
     *
     * @throws NoSuchMethodError 找不到对应的方法
     * @throws ExceptionInInitializerError class初始化失败
     * @throws OutOfMemoryError 内存不足
     */
    jmethodID (*GetStaticMethodID)(JNIEnv *, jclass, const char *, const char *);


    /**
     * CallStatic<type>Method(JNIEnv *env, jclass clazz, jmethodID methodID, ...);调用参数放到可变参数中
     * CallStatic<type>MethodA(JNIEnv *env, jclass clazz, jmethodID methodID, const jvalue *args);调用参数放入jvalue数组
     * CallStatic<type>MethodV(JNIEnv *env, jclass clazz, jmethodID methodID, va_list args);调用参数放入va_list结构中
     *
     * 以上三组调用接口都是根据 method ID调用java静态方法的接口，其中method ID是通过GetStaticMethodID()获取的
     * method ID必须从clazz的真实类获取，而不应从其某个父类获取
     * <type>是方法的返回类型，三类接口间唯一的区别是methodID参数之后调用参数的不同
     *
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param methodID 方法ID
     * @param args 调用参数
     *
     * @return java方法返回结果
     *
     * @throws java方法中可能抛出的异常
     */
    jobject (*CallStaticObjectMethod)(JNIEnv *, jclass, jmethodID, ...);
    jobject (*CallStaticObjectMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jobject (*CallStaticObjectMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jboolean (*CallStaticBooleanMethod)(JNIEnv *, jclass, jmethodID, ...);
    jboolean (*CallStaticBooleanMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jboolean (*CallStaticBooleanMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jbyte (*CallStaticByteMethod)(JNIEnv *, jclass, jmethodID, ...);
    jbyte (*CallStaticByteMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jbyte (*CallStaticByteMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jchar (*CallStaticCharMethod)(JNIEnv *, jclass, jmethodID, ...);
    jchar (*CallStaticCharMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jchar (*CallStaticCharMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jshort (*CallStaticShortMethod)(JNIEnv *, jclass, jmethodID, ...);
    jshort (*CallStaticShortMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jshort (*CallStaticShortMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jint (*CallStaticIntMethod)(JNIEnv *, jclass, jmethodID, ...);
    jint (*CallStaticIntMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jint (*CallStaticIntMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jlong (*CallStaticLongMethod)(JNIEnv *, jclass, jmethodID, ...);
    jlong (*CallStaticLongMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jlong (*CallStaticLongMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jfloat (*CallStaticFloatMethod)(JNIEnv *, jclass, jmethodID, ...);
    jfloat (*CallStaticFloatMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jfloat (*CallStaticFloatMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    jdouble (*CallStaticDoubleMethod)(JNIEnv *, jclass, jmethodID, ...);
    jdouble (*CallStaticDoubleMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    jdouble (*CallStaticDoubleMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);
    void (*CallStaticVoidMethod)(JNIEnv *, jclass, jmethodID, ...);
    void (*CallStaticVoidMethodV)(JNIEnv *, jclass, jmethodID, va_list);
    void (*CallStaticVoidMethodA)(JNIEnv *, jclass, jmethodID, jvalue *);


    /**
     * 根据class对象获取静态成员变量的field ID
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param name 变量名
     * @param sig 变量签名
     *
     * @return field ID or NULL
     *
     * @throws NoSuchFieldError 找不到对应的变量ID
     * @throws ExceptionInInitializerError class初始化失败
     * @throws OutOfMemoryError 内存不足
     */
    jfieldID (*GetStaticFieldID)(JNIEnv *, jclass, const char *, const char *);


    /**
     * 根据field id取出对象中相应的变量值，field Id通过GetStaticFieldID()获取
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param fieldID 有效的field id
     *
     * @return 相应的静态变量值
     */
    jobject (*GetStaticObjectField)(JNIEnv *, jclass, jfieldID);
    jboolean (*GetStaticBooleanField)(JNIEnv *, jclass, jfieldID);
    jbyte (*GetStaticByteField)(JNIEnv *, jclass, jfieldID);
    jchar (*GetStaticCharField)(JNIEnv *, jclass, jfieldID);
    jshort (*GetStaticShortField)(JNIEnv *, jclass, jfieldID);
    jint (*GetStaticIntField)(JNIEnv *, jclass, jfieldID);
    jlong (*GetStaticLongField)(JNIEnv *, jclass, jfieldID);
    jfloat (*GetStaticFloatField)(JNIEnv *, jclass, jfieldID);
    jdouble (*GetStaticDoubleField)(JNIEnv *, jclass, jfieldID);


    /**
     * 根据field id为相应的静态变量设置新的值，field Id通过GetStaticFieldID()获取
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param fieldID 有效的field id
     * @param value 要设置的值
     */
    void (*SetStaticObjectField)(JNIEnv *, jclass, jfieldID, jobject);
    void (*SetStaticBooleanField)(JNIEnv *, jclass, jfieldID, jboolean);
    void (*SetStaticByteField)(JNIEnv *, jclass, jfieldID, jbyte);
    void (*SetStaticCharField)(JNIEnv *, jclass, jfieldID, jchar);
    void (*SetStaticShortField)(JNIEnv *, jclass, jfieldID, jshort);
    void (*SetStaticIntField)(JNIEnv *, jclass, jfieldID, jint);
    void (*SetStaticLongField)(JNIEnv *, jclass, jfieldID, jlong);
    void (*SetStaticFloatField)(JNIEnv *, jclass, jfieldID, jfloat);
    void (*SetStaticDoubleField)(JNIEnv *, jclass, jfieldID, jdouble);


    /**
     * 创建一个新的java.lang.String对象
     *
     * @param env JNI接口指针
     * @param unicodeChars 指向Unicode字符串的指针
     * @param len Unicode字符串的长度
     *
     * @return String对象 or NULL
     *
     * @throws OutOfMemoryError 内存不足
     */
    jstring (*NewString)(JNIEnv *, const jchar *, jsize);


    /**
     * 返回java.lang.String的长度（Unicode字符数）
     *
     * @param env JNI接口指针
     * @param string String对象
     *
     * @return 长度
     */
    jsize (*GetStringLength)(JNIEnv *, jstring);


    /**
     * 返回指向Unicode字符数组的指针
     * 该指针在调用ReleaseStringchars()前一直有效
     * 如果isCopy非空，则在复制完成后将*isCopy设为JNI_TRUE。否则设为JNI_FALSE
     *
     * @param env JNI接口指针
     * @param string String对象
     * @param isCopy 指向boolean的指针
     *
     * @return 指向字符串的指针 or NULL
     */
    const jchar *(*GetStringChars)(JNIEnv *, jstring, jboolean *);


    /**
     * 通知VM无需再访问chars
     * chars是一个指针，通过GetStringChars()
     *
     * @param env JNI接口指针
     * @param string String对象
     * @param chars 指向字符串的指针
     */
    void (*ReleaseStringChars)(JNIEnv *, jstring, const jchar *);


    /**
     * 根据UTF-8编码的字符数组创建一个新的java.lang.String对象
     *
     * @param env JNI接口指针
     * @param bytes 指向UTF-8字符串的指针
     *
     * @return String对象 or NULL
     *
     * @throws OutOfMemoryError 内存不足
     */
    jstring (*NewStringUTF)(JNIEnv *, const char *);


    /**
     * 返回字符串以UTF-8为编码的字节数
     *
     * @param env JNI接口指针
     * @param string String对象
     *
     * @return 字符串的UTF-8字节数
     */
    jsize (*GetStringUTFLength)(JNIEnv *, jstring);


    /**
     * 返回指向UTF-8编码字符数组的指针
     * 该指针在调用ReleaseStringUTFChars()前一直有效
     * 如果isCopy非空，则在复制完成后将*isCopy设为JNI_TRUE。否则设为JNI_FALSE
     *
     * @param env JNI接口指针
     * @param string String对象
     * @param isCopy 指向boolean的指针
     *
     * @return 指向字符串的指针 or NULL
     */
    const char *(*GetStringUTFChars)(JNIEnv *, jstring, jboolean *);


    /**
     * 通知VM无需再访问utf
     * utf是一个指针，通过GetStringUTFChars()
     *
     * @param env JNI接口指针
     * @param string String对象
     * @param utf 指向字符串的指针
     */
    void (*ReleaseStringUTFChars)(JNIEnv *, jstring, const char *);


    /**
     * 获取数组元素个数
     *
     * @param env JNI接口指针
     * @param array java数组对象
     *
     * @return 数组长度
     */
    jsize (*GetArrayLength)(JNIEnv *, jarray);


    /**
     * 创建新的elementClass类型数组，所有元素初始值均设为initialElement
     *
     * @param env JNI接口指针
     * @param length 数组大小
     * @param elementClass 数组类型
     * @param initialElement 初始值
     *
     * @return 数组对象 or NULL
     */
    jobjectArray (*NewObjectArray)(JNIEnv *, jsize, jclass, jobject);


    /**
     * 获取对象数组中指定index的值
     *
     * @param env JNI接口指针
     * @param array java数组
     * @param index 索引
     *
     * @return 索引对象的对象
     *
     * @throws ArrayIndexOutOfBoundsException
     */
    jobject (*GetObjectArrayElement)(JNIEnv *, jobjectArray, jsize);


    /**
     * 设置对象数组中指定index的值
     *
     * @param env JNI接口指针
     * @param array java数组
     * @param index 索引
     * @param value 新的值
     *
     * @throws ArrayIndexOutOfBoundsException
     */
    void (*SetObjectArrayElement)(JNIEnv *, jobjectArray, jsize, jobject);


    /**
     * ArrayType New<PrimitiveType>Array(JNIEnv *env, jsize length);
     * 创建基本类型数组对象
     *
     * @param env JNI接口指针
     * @param length 数组大小
     *
     * @return 数组对象 or NULL
     */
    jbooleanArray (*NewBooleanArray)(JNIEnv *, jsize);
    jbyteArray (*NewByteArray)(JNIEnv *, jsize);
    jcharArray (*NewCharArray)(JNIEnv *, jsize);
    jshortArray (*NewShortArray)(JNIEnv *, jsize);
    jintArray (*NewIntArray)(JNIEnv *, jsize);
    jlongArray (*NewLongArray)(JNIEnv *, jsize);
    jfloatArray (*NewFloatArray)(JNIEnv *, jsize);
    jdoubleArray (*NewDoubleArray)(JNIEnv *, jsize);


    /**
     * NativeType *Get<PrimitiveType>ArrayElements(JNIEnv *env, ArrayType array, jboolean *isCopy);
     * 返回基本类型数组中的数据，通过返回的指针可以访问这些数据，若虚拟机支持pinning，则指针指向原始数组，否则指向原始数组的拷贝
     * 返回的指针在Release<PrimitiveType>ArrayElements()调用前一直有效
     * 数组用使用结束后，调用Release<PrimitiveType>ArrayElements，并在调用参数中决定是否把修改提交给java
     *
     * @param env JNI接口指针
     * @param array java数组
     * @param isCopy 指向boolean的指针，若不为NULL，则执行了复制设为JNI_TRUE，否则设为JNI_FALSE
     *
     * @return 指向数组元素的指针 or NULL
     */
    jboolean *(*GetBooleanArrayElements)(JNIEnv *, jbooleanArray, jboolean *);
    jbyte *(*GetByteArrayElements)(JNIEnv *, jbyteArray, jboolean *);
    jchar *(*GetCharArrayElements)(JNIEnv *, jcharArray, jboolean *);
    jshort *(*GetShortArrayElements)(JNIEnv *, jshortArray, jboolean *);
    jint *(*GetIntArrayElements)(JNIEnv *, jintArray, jboolean *);
    jlong *(*GetLongArrayElements)(JNIEnv *, jlongArray, jboolean *);
    jfloat *(*GetFloatArrayElements)(JNIEnv *, jfloatArray, jboolean *);
    jdouble *(*GetDoubleArrayElements)(JNIEnv *, jdoubleArray, jboolean *);


    /**
     * Release<PrimitiveType>ArrayElements
     * 通知VM不再需要访问这些数组，根据mode参数的不同，将决定是否把数组的修改复制到源数组
     *
     * @param env JNI接口指针
     * @param array java数组对象
     * @param elems 指向数组元素的指针
     * @param mode 释放模式，0：把数据复制回源数组并释放elems缓冲区，JNI_COMMIT：把数据复制回源数组但不释放elems缓冲区，JNI_ABORT：不把数据复制回源数组，释放elems缓冲区
     */
    void (*ReleaseBooleanArrayElements)(JNIEnv *, jbooleanArray, jboolean *, jint);
    void (*ReleaseByteArrayElements)(JNIEnv *, jbyteArray, jbyte *, jint);
    void (*ReleaseCharArrayElements)(JNIEnv *, jcharArray, jchar *, jint);
    void (*ReleaseShortArrayElements)(JNIEnv *, jshortArray, jshort *, jint);
    void (*ReleaseIntArrayElements)(JNIEnv *, jintArray, jint *, jint);
    void (*ReleaseLongArrayElements)(JNIEnv *, jlongArray, jlong *, jint);
    void (*ReleaseFloatArrayElements)(JNIEnv *, jfloatArray, jfloat *, jint);
    void (*ReleaseDoubleArrayElements)(JNIEnv *, jdoubleArray, jdouble *, jint);


    /**
     * void Get<PrimitiveType>ArrayRegion(JNIEnv *env, ArrayType array, jsize start, jsize len, NativeType *buf);
     * 把基本类型数组拷贝到buf中
     *
     * @param env JNI接口指针
     * @param array java数组
     * @param start 开始index
     * @param len 拷贝长度
     * @param buf 目标地址
     *
     * @throws ArrayIndexOutOfBoundsException
     */
    void (*GetBooleanArrayRegion)(JNIEnv *, jbooleanArray, jsize, jsize, jboolean *);
    void (*GetByteArrayRegion)(JNIEnv *, jbyteArray, jsize, jsize, jbyte *);
    void (*GetCharArrayRegion)(JNIEnv *, jcharArray, jsize, jsize, jchar *);
    void (*GetShortArrayRegion)(JNIEnv *, jshortArray, jsize, jsize, jshort *);
    void (*GetIntArrayRegion)(JNIEnv *, jintArray, jsize, jsize, jint *);
    void (*GetLongArrayRegion)(JNIEnv *, jlongArray, jsize, jsize, jlong *);
    void (*GetFloatArrayRegion)(JNIEnv *, jfloatArray, jsize, jsize, jfloat *);
    void (*GetDoubleArrayRegion)(JNIEnv *, jdoubleArray, jsize, jsize, jdouble *);


    /**
     * void Set<PrimitiveType>ArrayRegion(JNIEnv *env, ArrayType array, jsize start, jsize len, const NativeType *buf);
     * 把buf中的内容拷贝回数组中
     *
     * @param env JNI接口指针
     * @param array java数组
     * @param start 开始index
     * @param len 拷贝长度
     * @param buf 源数据
     *
     * @throws ArrayIndexOutOfBoundsException
     */
    void (*SetBooleanArrayRegion)(JNIEnv *, jbooleanArray, jsize, jsize, const jboolean *);
    void (*SetByteArrayRegion)(JNIEnv *, jbyteArray, jsize, jsize, const jbyte *);
    void (*SetCharArrayRegion)(JNIEnv *, jcharArray, jsize, jsize, const jchar *);
    void (*SetShortArrayRegion)(JNIEnv *, jshortArray, jsize, jsize, const jshort *);
    void (*SetIntArrayRegion)(JNIEnv *, jintArray, jsize, jsize, const jint *);
    void (*SetLongArrayRegion)(JNIEnv *, jlongArray, jsize, jsize, const jlong *);
    void (*SetFloatArrayRegion)(JNIEnv *, jfloatArray, jsize, jsize, const jfloat *);
    void (*SetDoubleArrayRegion)(JNIEnv *, jdoubleArray, jsize, jsize, const jdouble *);


    /**
     * 为clazz类注册本地方法
     *
     * @param env JNI接口指针
     * @param clazz class对象
     * @param methods clazz类中的本地方法，指向方法数组
     * @param nMethods 本地方法个数
     *
     * @return 0：成功， 负数：失败
     *
     * @throws NoSuchMethodError
     */
    jint (*RegisterNatives)(JNIEnv *, jclass, const JNINativeMethod *, jint);


    /**
     * 取消clazz类本地方法的注册
     *
     * @param env JNI接口指针
     * @param clazz class对象
     *
     * @return 0：成功， 负数：失败
     */
    jint (*UnregisterNatives)(JNIEnv *, jclass);


    /**
     * 进入与obj所引用的Java对象相关联的监控，obj 必须为非空
     *
     * @param env JNI接口指针
     * @param obj java对象 或 class对象
     *
     * @return 0：成功， 负数：失败
     */
    jint (*MonitorEnter)(JNIEnv *, jobject);


    /**
     * 退出与obj所引用的Java对象相关联的监控，obj 必须为非空
     * 当前线程必须是与obj所引用的Java对象相关联的监控程序的所有者
     * 监控程序次数的计数器减 1。如果计数器的值变为 0，则释放当前线程的监控程序
     *
     * @param env JNI接口指针
     * @param obj java对象 或 class对象
     *
     * @return 0：成功， 负数：失败
     */
    jint (*MonitorExit)(JNIEnv *, jobject);


    /**
     * 获取当前线程关联的Java VM接口
     *
     * @param env JNI接口指针
     * @param vm java VM接口指针
     *
     * @return 0：成功， 负数：失败
     */
    jint (*GetJavaVM)(JNIEnv *, JavaVM **);


    /**
     * 从start index开始，拷贝len个Unicode字符到buf
     *
     * @param env JNI接口指针
     * @param str string对象
     * @param start 开始index
     * @param len 拷贝长度
     * @param buf 目标地址
     *
     * @throws StringIndexOutOfBoundsException
     */
    void (*GetStringRegion)(JNIEnv *, jstring, jsize, jsize, jchar *);


    /**
     * 从start index开始，取出len个Unicode字符转换为UTF-8编码后拷贝到buf
     *
     * @param env JNI接口指针
     * @param str string对象
     * @param start 开始index
     * @param len 拷贝长度
     * @param buf 目标地址
     *
     * @throws StringIndexOutOfBoundsException
     */
    void (*GetStringUTFRegion)(JNIEnv *, jstring, jsize, jsize, char *);


    /**
     * 与Get/Release<primitivetype>ArrayElements方法非常相似，在这个方法中VM尽量返回指向原始数组的指针
     *
     * @since JDK/JRE 1.2
     *
     * @param env JNI接口指针
     * @param array java数组
     * @param isCopy 指向boolean的指针，若不为NULL，则执行了复制设为JNI_TRUE，否则设为JNI_FALSE
     *
     * @return 指向数组元素的指针 or NULL
     */
    void *(*GetPrimitiveArrayCritical)(JNIEnv *, jarray, jboolean *);
    void (*ReleasePrimitiveArrayCritical)(JNIEnv *, jarray, void *, jint);


    /**
     * 与Get/ReleaseStringChars方法非常相似，在这个方法中VM尽量返回指向原始字符串的指针
     *
     * @since JDK/JRE 1.2
     *
     * @param env JNI接口指针
     * @param string String对象
     * @param isCopy 指向boolean的指针
     *
     * @return 指向字符串的指针 or NULL
     */
    const jchar *(*GetStringCritical)(JNIEnv *, jstring, jboolean *);
    void (*ReleaseStringCritical)(JNIEnv *, jstring, const jchar *);


    /**
     * 为传入的obj创建弱全局引用
     * 弱全局引用不会阻止VM释放所引用的对象，程序中可以通过使用IsSameObject比较弱全局引用和NULL来确认所引用的对象是否被释放
     *
     * @param env JNI接口指针
     * @param obj 全局或局部引用
     *
     * @return 弱全局引用 or NULL
     */
    jweak (*NewWeakGlobalRef)(JNIEnv *, jobject);


    /**
     * 删除弱全局引用
     *
     * @param env JNI接口指针
     * @param obj 弱全局引用
     */
    void (*DeleteWeakGlobalRef)(JNIEnv *, jweak);


    /**
     * 判断是否有未处理异常
     *
     * @param env JNI接口指针
     *
     * @return JNI_TRUE表示有未处理异常，否则为JNI_FALSE
     */
    jboolean (*ExceptionCheck)(JNIEnv *);


    /**
     * 创建并返回java.nio.ByteBuffer对象，该对象引用以address为开始地址，大小为capacity的内存块
     *
     * @since JDK/JRE 1.4
     *
     * @param env JNI接口指针
     * @param address 开始地址
     * @param capacity 内存大小
     *
     * @return Jjava.nio.ByteBuffer or NULL
     *
     * @throws OutOfMemoryError
     */
    jobject (*NewDirectByteBuffer)(JNIEnv *, void *, jlong);


    /**
     * 根据java.nio.ByteBuffer对象，获取相应的内存数据并返回开始地址
     *
     * @since JDK/JRE 1.4
     *
     * @param env JNI接口指针
     * @param buf java.nio.ByteBuffer对象
     *
     * @return 数据的开始地址 or NULL
     */
    void *(*GetDirectBufferAddress)(JNIEnv *, jobject);


    /**
     * 根据java.nio.ByteBuffer对象，获取相应的内存数据的大小
     *
     * @since JDK/JRE 1.4
     *
     * @param env JNI接口指针
     * @param buf java.nio.ByteBuffer对象
     *
     * @return 数据大小 or -1
     */
    jlong (*GetDirectBufferCapacity)(JNIEnv *, jobject);


    /**
     * 获取java对象的引用类型，可能的返回值有：
     * JNIInvalidRefType
     * JNILocalRefType：局部引用
     * JNIGlobalRefType：全局引用
     * JNIWeakGlobalRefType ：全局弱若引用
     *
     * @since JDK/JRE 1.6
     *
     * @param env JNI接口指针
     * @param obj java对象的引用
     *
     * @return 引用类型
     */
    jobjectRefType (*GetObjectRefType)(JNIEnv *, jobject);
    
```


  
   


  [1]: http://docs.oracle.com/javase/7/docs/technotes/guides/jni/
  [2]: http://static.zybuluo.com/shensky711/7m11gpnzhtfdqnjmzpvcwa7y/image_1aup7dbls1pnm1l2d1j8dajc1umh9.png
  [3]: http://static.zybuluo.com/shensky711/xdjuj8ypjwh6v9858yy36okl/image_1aurdi9ud1jtgtngelh1fm6st7g.png