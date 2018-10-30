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

### 遇到的问题
Jni和C++类型相互转换问题


### Jni资料
http://www.cocos.com/docs/native/v2/sdk-integration/android-jni/zh.html