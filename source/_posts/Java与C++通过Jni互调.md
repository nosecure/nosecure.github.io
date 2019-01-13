---
title: Java与C++通过Jni互调
data: 2018-10-25
categories: program
tags:
    - Java
    - C++
    - Jni
comments: true
---


### 方法详解
1. jbyte * GetByteArrayElements(jbyteArray array, jboolean *isCopy);
用于获取传递进去的jbyteArray的数据，返回值为 jbyte *, 对 jbyte * 进行修改，不会影响原数据值。
2. void ReleaseByteArrayElements(jbyteArray array, jbyte *elems, jint mode);
用于把GetByteArrayElements得到数据后做的修改同步到原数据。
3. jmethodID GetMethodID(jclass clazz, const char *name, const char *sig);
clazz是jclass，name是方法名称，sig是签名。
签名形式：()V，括号里面是参数的签名，后面是返回值签名。
例：
``` java
要回调的java程序：
public byte[] onTestMixed(byte[] bytes, boolean isBoolean, String s) {
    logger.info("This is onTestMixed callback function. Message: [bytes={}, isBoolean={}, s={}]", bytes, isBoolean, s);
    return bytes;
}
回调示例：
jclass javaClass = (*env).FindClass("com/jnitest/jni/JniTest");
if (javaClass == 0) {
    return false;
}

jmethodID javaCallbackId = (*env).GetMethodID(javaClass, "onTestMixed", "([BZLjava/lang/String;)[B");
if (javaCallbackId == NULL) {
    return false;
}

jbyteArray arg1 = env->NewByteArray(3);
signed char chs[3] = {1, 2, 3};
jbyte* jbs = chs;
env->SetByteArrayRegion(arg1, 0, 3, jbs);

jboolean arg2 = true;
jstring arg3 = (*env).NewStringUTF("3");

(*env).CallVoidMethod(jobj, javaCallbackId, arg1, arg2, arg3);
```
其他完整示例见：[]()

### 注意事项
C++回调java函数，只能回调该Jni文件的函数，寻找该类有两种方法：
对应的C++方法：
``` cpp
JNIEXPORT void JNICALL Java_com_jnitest_jni_JniTest_testVoid
  (JNIEnv *env, jobject jobj) {
      // 下方的方法一和二
      // 调用env中的方法有两种形式，env->和(*env).，示例见方法一和二
  }
```
1. 
```
//直接用GetObjectClass找到Class, 也就是JniTest.class.
jclass javaClass = env->GetObjectClass(jobj);
```
2. 
```
//直接用findClass找到Class, 也就是JniTest.class.
jclass javaClass = (*env).FindClass("com/jnitest/jni/JniTest");
```
3. 
java类型与签名表
|java类型|对应的签名|C++对应类型|
|-|-|-|
|void|V|void|
|boolean|Z|jboolean|
|char|C|jchar|
|byte|B|jbyte|
|short|S|jshort|
|int|I|jint|
|long|J|jlong|
|float|F|jfloat|
|double|D|jdouble|
|string|Ljava/lang/String;|jstring|
|byte[]|[B|jbyteArray|

### 遇到的问题
1. Jni和C++类型相互转换问题，即C++对应的类型与C++原生类型(或由原生类型定义出来的其他类型)相互转换问题：
jstring <-> char *
jbyteArray <-> BYTE *

2. 由多线程引起的问题
问题：在java第一次调用C++库时，把
JNIEnv *env; 和 jobject jobj;
保存为全局静态变量。
C++库中有线程在接收数据，之后通过保存的env和jobj回调java方法，接着就会报出异常、结束程序、生成core以及其他错误文件。
原因：
JNI文档上说,JNI接口的指针JNIEnv*不能在c++的线程间共享。
解决办法：
JNI接口指针不可为多个线程共用，但是java虚拟机的JavaVM指针是整个jvm公用的。
声明JavaVM *和jobject为全局变量：
```
static JavaVM *gs_jvm = NULL;   // Returns “0” on success; returns a negative value on failure.
static jobject gs_jobj = NULL;  // 不可以直接保存obj到全局变量，应该调用该函数
```
在第一次调用C++库时，保存gs_jvm和gs_jobj：
```
int retJvm = env->GetJavaVM(&gs_jvm);
gs_jobj = env->NewGlobalRef(jobj);
```
jobject指针也不能在多个线程中共享，就是说 不能直接在保存一个线程中的jobject指针到全局变量中 然后在另外一个线程中使用它。所以需要用上面的方法来保存。
调用方式：
```
在方法中调用：
JNIEnv *env;
gs_jvm->AttachCurrentThread((void **) &env, NULL):
jclass javaClass = env->GetObjectClass(gs_jobj);
```

### 参考资料
http://www.cocos.com/docs/native/v2/sdk-integration/android-jni/zh.html
https://blog.csdn.net/popop123/article/details/1511180
[JNI 文档下载](http://files.cnblogs.com/luxiaofeng54/JNI_Docs.rar)