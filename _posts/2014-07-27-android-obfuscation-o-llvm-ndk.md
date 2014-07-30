---
layout: post
title:  "Obfuscating Android Applications using O-LLVM and the NDK"
date:   2014-07-27 01:18:48
categories: android obfuscation ndk llvm o-llvm
---

Obfuscation is a technique employed to hide the intent of an application.  The techniques used to obscure the intent of an application can vary widely.  The most effective techniques can increase the effort of reverse engineering and hinder cracking, and theft of intellectual property. Many applications exist that allow you to obfuscate your Android applications. A few are as follows:

 - [Proguard](http://proguard.sourceforge.net/)
 - [DashO](https://www.preemptive.com/products/dasho)
 - [Dexguard](https://www.saikoa.com/dexguard)
 - [DexProtector](http://dexprotector.com/)
 - [ApkProtect](http://www.apkprotect.com/)
 - [Shield4j](http://shield4j.com/main/sidecc63dd15b9656e1c877087e024689646fc27b38)
 - [Stringer](https://jfxstore.com/stringer/)
 - [Allitori](http://www.allatori.com/features/android-obfuscation.html)


The level at which these obfuscators operates varies.  The Android compilation process is roughly Java Code(.java) -> Java Bytecode(.class) -> Dalvik Bytecode(classes.dex).  Some obfuscators operate directly on the source files transforming them before compilation, some operate on the Java bytecode, and otheres operate on the Dalvik bytecode.

The Android NDK allows programmers to sidestep the VM to squeeze performance or interact more directly with the kernel and hardware.  Google's description of the NDK is as follows: "The NDK is a toolset that allows you to implement parts of your app using native-code languages such as C and C++. For certain types of apps, this can be helpful so you can reuse existing code libraries written in these languages, but most apps do not need the Android NDK."

Android applications that have components that target native machine code as opposed to the DalvikVM do not have as many obfuscation options.  One notable exception is the [Obfuscator-LLVM](https://github.com/obfuscator-llvm/obfuscator/wiki) project.  This is a particularly interesting project because it targets the LLVM.  This makes the tool extremely portable as it can apply to as many languages as LLVM has front-ends for and work on as many CPU archtectures as there are LLVM back-ends.  0vercl0k released a [paper](http://download.tuxfamily.org/overclokblog/Obfuscation%20of%20steel%3a%20meet%20my%20Kryptonite/0vercl0k_Obfuscation_of_steel_meet_kryptonite.pdf) before the release of o-llvm explaining that details some of the advantages of working with LLVM as well as a simple code transform.

I had set up and used O-LLVM + NDK [some time ago](https://gist.github.com/Fuzion24/9698c95fd7458c16077d).  I decided to write a blog post about it after I saw [TowelRoot](https://towelroot.com/) using O-LLVM.   Towelroot is a Android privledge escalation that exploits a Linux kernel bug in the futext syscall. A writeup about the exploit can be found [here](http://tinyhack.com/2014/07/07/exploiting-the-futex-bug-and-uncovering-towelroot/). O-LLVM was presumably used here so that others could not copy and use the exploit for malicious purposes or to repack it and sell it under a different name.

The instructions, as follows, should help you get up and running to allow you to obfuscate your native bits as seen on TowelRoot.


## Using NDK O-LLVM binary overlay

I've compiled the obfustor as a binary overlay on top the NDK for both OSX and Linux. You can also perform this step manually by building from source (see the last section). The obfuscator binary overlays are here:

 - [OSX-NDK-Obfuscator.tar.bz2](https://github.com/Fuzion24/AndroidObfuscation-NDK/blob/master/prebuilt/android-ndk32-r10-darwin-x86_64-obfuscator.tar.bz2?raw=true)
 - [Linux-NDK-Obfuscator.tar.bz2](https://github.com/Fuzion24/AndroidObfuscation-NDK/blob/master/prebuilt/android-ndk64-r10-linux-x86_64-obfuscator.tar.bz2?raw=true)


Download the appropriate binary overlay, natigate to your NDK directory and extract using ```tar xvf <Obfuscator.tar.bz2>``` where <Obfuscator.tar.bz2> is the appropiate path and filename to the file you just downloaded

## Setting up an O-LLVM NDK project
Now, we want to configure our Android NDK project to use our obfuscator. The layout of our project will look like:

```
➜  AndroidObfuscation-NDK git:(master) tree .
.
├── jni
│   ├── Android.mk
│   ├── Application.mk
│   └── obfuscationTest.c
```

In your Application.mk of your project,

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

APP_ABI := armeabi

NDK_TOOLCHAIN_VERSION := clang3.4-obfuscator

include $(BUILD_EXECUTABLE)
```

Information about the various transforms can be found here: [Obfuscator Wiki](https://github.com/obfuscator-llvm/obfuscator/wiki/Installation#how-to-use-it).  The way to pass those flags through to the obfuscator is via the ```LOCAL_CFLAGS``` directive.  The obfuscator flags need to be prefixed with -mllvm so that clang will pass them through.

An example Android.mk would look like:

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := obfuscated
LOCAL_SRC_FILES := obfuscationTest.c

LOCAL_LDLIBS := -static

LOCAL_CFLAGS := -mllvm -sub -mllvm -fla -mllvm -bcf

include $(BUILD_EXECUTABLE)
```

Now, we can build the project:

```
➜  AndroidObfuscation-NDK git:(master) ndk-build
[armeabi] Compile thumb  : obfuscated <= obfuscationTest.c
[armeabi] Executable     : obfuscated
[armeabi] Install        : obfuscated => libs/armeabi/obfuscated
```

An example project can be found here using the above setup and structure: [https://github.com/Fuzion24/AndroidObfuscation-NDK](https://github.com/Fuzion24/AndroidObfuscation-NDK).

The binary overlay I prebuilt above also contains the experimental string obfuscation techniques built by [yag00](https://github.com/yag00).  You can enable the string obfuscation by passing the "-mllvm -xse" flag through the LOCAL_CFLAGS directive.

```
➜  AndroidObfuscation-NDK git:(master) cat jni/obfuscationTest.c
#include <stdio.h>

int main(void){
  printf("Hello, world\n");
  return 0;
}
```

Before using the string obfuscation pass:

```
➜  AndroidObfuscation-NDK git:(master) strings libs/armeabi/obfuscated | grep Hello
Hello, world
```

After using the string obfuscation pass:

```
➜  AndroidObfuscation-NDK git:(master) strings libs/armeabi/obfuscated | grep Hello
```



## (Optional) Building O-LLVM from source for the NDK
```bash
git clone -b llvm-3.4 https://github.com/obfuscator-llvm/obfuscator.git
cd obfuscator
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE:String=Release ../obfuscator/
make -j5
```

Full instructions for building o-llvm are [here](https://github.com/obfuscator-llvm/obfuscator/wiki/Installation), but the above instructions should suffice.


```bash
cp -r $NDK_PATH/toolchains/arm-linux-androideabi-clang3.4 $NDK_PATH/toolchains/arm-linux-androideabi-clang3.4-obfuscator
```


Open ```$NDK_PATH/toolchains/arm-linux-androideabi-clang3.4-obfuscator/setup.mk``` and change

```
TARGET_CC := $(LLVM_TOOLCHAIN_PREFIX)clang$(HOST_EXEEXT)
TARGET_CXX := $(LLVM_TOOLCHAIN_PREFIX)clang++$(HOST_EXEEXT)
```

to (making sure to change <PATH_TO_OBFUSCATOR_REPO> to the path where you cloned o-llvm):

```
LLVM_TOOLCHAIN_PATH := <PATH_TO_OBFUSCATOR_REPO>/build/bin/
TARGET_CC := $(LLVM_TOOLCHAIN_PATH)clang$(HOST_EXEEXT)
TARGET_CXX := $(LLVM_TOOLCHAIN_PATH)clang++$(HOST_EXEEXT)
```

