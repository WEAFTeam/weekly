---
title: Android 混淆
description: Android 混淆
tags:
  - ANDROID
author:
  - 
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/android.jpg'
category: ANDROID
date: '2018-08-11 02:04:44'
---

#### 一：开启混淆
Android studio中开启混淆很简单，找到build.gradle文件，设置minifyEnabled=true。如下：

         buildTypes {
            release {
                minifyEnabled true
                shrinkResources true   
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
 shrinkResources设置为true可以在开启混淆后去掉无用的资源文件，减小应用的体积
 

#### 二：配置混淆文件    
找到proguard-rules.pro文件，就可以开始我们的混淆规则了。
一些简单的规则需要我们了解下

    # 代表行注释符
    - 表示一条规则的开始
    keep 保留 ：
    dont 不要 : dontwarn：表示不要提示警告
    ignore 忽略，例如ignorewarning：表示忽略警告
    # 不优化
    -dontoptimize
    # 代码循环优化次数，0-7，默认为5
    -optimizationpasses 5
    # 不做预校验
    -dontpreverify

首先需要区分下 * 和 **；

    -keep class xxxx.info.**
    -keep class xxxx.info.*
    
前者表示本包以及子包下的类名都保持，而后者表示本包不混淆，子包下的类名会被混淆。
当然这两者都是会混淆具体的方法名和变量的，所以你如果想要都保持，不被混淆处理的话，需要写成下面这种：

    -keep class xxxx.info.* {*;}

另外我们还可以保留类中的某些部分不被混淆，如：

    -keep class xxxx.info.One {
         public <methods>;
    }

或许你觉得类名也不需要保留，那就不能使用keep了，这里还有几种别的，如

    -keepclassmembers 不保留包名 防止成员被移除或者被重命名
    -keepclasseswithmembers 保留类名和成员名
    
#####  1：基本规则

一般情况下我们需要保存四大组件，自定义view不被混淆，因为  这些子类都有可能被外部调用。

    -keep public class * extends android.app.Activity
    -keep public class * extends android.app.Application
    -keep public class * extends android.support.multidex.MultiDexApplication
    -keep public class * extends android.app.Service
    -keep public class * extends android.content.BroadcastReceiver
    -keep public class * extends android.content.ContentProvider
    -keep public class * extends android.app.backup.BackupAgentHelper
    -keep public class * extends android.preference.Preference
    -keep public class * extends android.view.View
    -keep class android.support.** {*;}

##### 2：反射
反射用到的类一般需要保留，否则会出现问题。
实体类不被混淆

     -keep class xxxx.info.Bean.** { *; }

##### 3：枚举不能被混淆

    -keepclassmembers enum * {
        public static **[] values();
        public static ** valueOf(java.lang.String);
    }

##### 4：继承的保留

    -keep public class * extends android.support.v4.**
    -keep public class * extends android.support.v7.**
    -keep public class * extends android.support.annotation.**
    -keep class * implements android.os.Parcelable {
         public static final android.os.Parcelable$Creator *;
    }
    
##### 5：jni  方法不可混淆

    -keepclasseswithmembernames class * {
            native <methods>;
   }       

##### 6：  资源文件不被混淆

    -keep class **.R$* {
        *;
    }
    -keepclassmembers class **.R$* {
         public static <fields>;
    }

##### 7：webview的一些处理

    -keepclassmembers class fqcn.of.javascript.interface.for.Webview { 
         public *; 
    }
     -keepclassmembers class * extends android.webkit.WebViewClient {
      public void *(android.webkit.WebView, java.lang.String, android.graphics.Bitmap);
      public boolean *(android.webkit.WebView, java.lang.String);
    }
    -keepclassmembers class * extends android.webkit.WebViewClient {
     public void *(android.webkit.WebView, jav.lang.String);
    }
在app中与HTML5的JavaScript的交互进行特殊处理 我们需要确保这些js要调用的原生方法不能够被混淆，于是我们需要做如下处理：
     
    -keepclassmembers class com.ljd.example.JSInterface {
     <methods>;
     }   
##### 8：其他的一些操作
删除代码中Log相关的代码

    -assumenosideeffects class android.util.Log {
        public static boolean isLoggable(java.lang.String, int);
        public static int v(...); public static int i(...);
        public static int w(...); public static int d(...);
         public static int e(...);
    }
保留测试相关的代码

     -dontnote junit.framework.**
     -dontnote junit.runner.**
     -dontwarn android.test.**
     -dontwarn android.support.test.**
     -dontwarn org.junit.**

 
 另外还有一些第三方的我这里就不贴出来了，接入时文档都会给出混淆策略。
 
 
     
     
    

    
    
    

    
    
    
    
    