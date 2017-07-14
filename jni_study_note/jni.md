#JNI学习心得	


[oracle官方文档请点这里](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html)

##1.简介

JNI全称是Java Native Interface，用于实现java与其他语言的粘合（C, C++, assembly等）。

使用JNI的情景：

+ java标准库不支持应用所需的独立于平台的特性
+ 已有其他语言编写的库，希望java通过JNI调用这个库
+ 想用底层语言例如汇编实现小部分对时效要求严格的代码

##2.整体设计

###1.JNI Interface Functions and Pointers
本地代码通过JNI functions访问java虚拟机。JNI functions通过*interface pointer*起作用。Interface pointer是指针的指针，指向一个指针数组，每个数组对象都是一个interface function。

！[fig.2-1 interface pointer](./pic/1.gif)

###2.加载动态链接库
	package pkg
	class Cls{
		native double f(int i, String s);
		static{
			System.loadLibrary("pkg_Cls");
		}
	}

在*unix系统中，系统将加载库名称转化为libpkg_Cls.so，win32系统中将加载库名称转化为pkg_Cls.dll。

除了通过native关键字声明本地方法关联的方法名，还可以用```RegisterNatives()```来注册与类相关的本地方法。```RegisterNatives()```方法对于静态链接的方法格外有用。

###3.解析本地方法名称
动态链接器通过名称来解析入口，本地方法名称通过以下组件连接组成：

+	前缀 Java_
+	完全限定类名
+	下划线（“_”）分隔
+	方法名称
+	对于重载的本地方法，两道下划线（“__”）后接着参数签名

VM检查本地库中包含方法名称时，首先检查 short name， 即不包含参数签名的部分，然后再查找 long name，即函数名以及函数签名。

###4.本地方法参数
JNI方法的第一个参数是JNI interface pointer，参数类型为*JNIEnv*。

方法的第二个参数取决于本地方法是*static*还是*nonstatic*，nonstatic本地方法的第二个参数是实例对象的引用（**jobject**），static本地方法的第二个参数是Java class的引用（**jclass**）。

剩余参数与常规Java方法参数一致。

e.g.:
	
package pkg;  

class Cls { 

     native double f(int i, String s); 

   		  ... 

	} 

通过C实现f：
	
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

通过C++实现f：

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

###5.引用Java对象
原始类型，integers,characters等，通过*复制（copy）*在Java和本地代码间传递；
任意的Java实例，则通过*引用（reference)*传递。

###6.全局引用vs局部引用
JNI将实例引用分为两类：全局引用（global reference）和局部引用（local reference）。局部引用仅在本地方法调用期间有效，本地方法返回后，局部引用将被自动释放；全局引用在被显示释放之前一直是有效的。

实例以*局部引用*的形式传递到本地方法中。所有JNI functions返回的Java实例都是局部引用。JNI允许通过局部引用创建全局引用，且接受Java实例的JNI functions都能够接收全局引用和局部引用。本地方法可以返回局部引用或者全局引用作为结果。

大多数情况下，开发者应该依赖VM释放局部引用，有些时候需要开发者手动释放局部引用：

+	本地方法接触到较大的java实例，因而需要创建此实例的局部引用，因为局部引用的存在使得实例无法被GC，即使大实例在剩下的计算中不再被使用。
+	本地方法中创建大量的局部引用。因为VM需要部分空间记录保存局部引用，但创建太多局部引用可能导致系统OOM。例如，本地方法需要遍历对象数组，获取每个对象的局部引用，对其进行操作。每次便利后，不再需要数组元素的局部引用。

JNI允许手动删除局部引用。为了保证能手动释放局部引用，除了作为返回对象的结果，JNI functions不允许创建额外的局部引用。

局部引用（Local references）仅在被创建的线程当中有效，故本地代码中不允许将局部引用从一个线程传递给另一个线程。

###7.实现局部引用












