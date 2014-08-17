---
layout: post
title:  "Building Xposed Modules using Gradle"
date:   2014-08-15 03:22:13
categories: android gradle xposed jar java build sdk
---

This is a going to be a very quick blog post describing how to build an Xposed module with gradle without modifying the Android SDK. When building for Xposed, you want to build against the Xposed API jar, but include the .jar in your APK/classes.dex.  It it not very obvious how to do this. There are a few posts [floating](http://forum.xda-developers.com/showpost.php?p=41904291&postcount=1570) around out [there](http://forum.xda-developers.com/showpost.php?p=41338415&postcount=1321) that achieve this by means of creating a custom Android SDK target that includes the XposedBridgeApi. This seemed very unclean to me as it is not portable; requiring developers to modify their toolchain.

As a solution to our problem, gradle has directives to do exactly what we want. Here are the instructions:

  - Create a new Android project that uses gradle either via Android Studio or the [cli](http://stackoverflow.com/questions/20801042/how-to-create-android-project-with-gradle-command-line).
  - Follow the [developer tutorial](https://github.com/rovo89/XposedBridge/wiki/Development-tutorial) by rovo89, to create the base structure for Xposed.
  - Add your [```XposedBridgeApi.jar```](http://forum.xda-developers.com/xposed/xposed-api-changelog-developer-news-t2714067) to ```<Project>/app/libs/```

Now, we need to tell gradle to build against the .jar, but not include in it in the APK. In your ```<Project>/app/build.gradle```, add the following to the bottom of the file:

```groovy
dependencies {
    provided fileTree(dir: 'libs', include: ['*.jar'])
}
```

An example project using this technique can be found at [https://github.com/Fuzion24/JustTrustMe](https://github.com/Fuzion24/JustTrustMe).
