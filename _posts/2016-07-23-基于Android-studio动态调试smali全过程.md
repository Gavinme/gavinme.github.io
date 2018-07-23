---
layout:     post
title:      基于Android studio动态调试smali全过程
date:       2018-07-23
author:     Gavin
header-img: img/post-bg-coffee.jpg
catalog: true
tags:
    - android
    - smali
    - 反编译
---
### 1 工具和环境
1. Android studio 用于集成idea插件和导入smali源码
2. idea插件
	- [插件下载](https://link.jianshu.com/?t=https://bitbucket.org/JesusFreke/smali/downloads/
)
	- 或者在studio中搜索Smalidea进行插件下载（要翻墙）
![Smalidea.png](https://upload-images.jianshu.io/upload_images/1689923-0181c17f552c11f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Android studio中通过idea插件来调试smali和在idea中调试很类似，下面就来讲解调试的步骤。
### 2 smali反编译和导入
调试的步骤大概如下：
1. 通过apktool工具反编译目标Apk获取smali文件，修改xml中`android:debuggable="true"`。
2. 导入smali文件至Android studio
3. 在相应位置打好断点后，启动调试进程。
开始Apk动态调试调试吧！

#### 2.1 获取smali文件
通过apktool获取反编译之后的smali文件非常简单。
```shell
apktool d *.apk
```
通过上面的apktool命令获取反编译的smali文件。
![image.png](https://upload-images.jianshu.io/upload_images/1689923-be4b7ca5f800026f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根目录demo-release默认为apk的名字，也可以通过-o指定路径名。如果遇到某些apk（如支付宝、微信等）采取了防止apktool破解的策略，我们也可以通过修改[apktool](https://github.com/iBotPeaches/Apktool)码来修复漏洞（因为是开源代码），但是这不是本文的重点。
#### 2.2 导入smali文件夹
通过Android studio 提供的Import Project(Eclipse ADT, Gradle)功能来导入文件夹。注意：smali作文根目录导入。
![image.png](https://upload-images.jianshu.io/upload_images/1689923-eb1848ac6dfea92d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

导入后，工程目录如下：
![image.png](https://upload-images.jianshu.io/upload_images/1689923-44db02b40595fb61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以在as中查看和修改smali代码。



#### 2.3 调试smali

下面的调试方法可以在进程起来就能附加调试进程。
##### 2.3.1 配置Android Studio调试环境
成功导入smali文件夹后配置远程调试的选项。选择 Run -->Edit Configurations，增加一个Remote调试的调试选项，端口选择:8700(未占用端口均可)。
![image.png](https://upload-images.jianshu.io/upload_images/1689923-3b24a5e29a9faac0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 2.3.2 进入等待调试
在需要的地方打好断点，通过一下命令行启动进程调试等待模式：
- 命令行启动调试模式，`adb shell am start -D -n packagename/ MainActivity`
packagename为进程名，MainActivity为首页Activity
启动调试app，通过`adb shell dumpsys activity top | grep  --color=always ACTIVITY`在终端获取包名和页面信息。

- 执行`adb shell ps | grep packagename `获取进程pid
- 执行`adb forward tcp:8700 jdwp:pid` 建立端口转发

也可以通过shell脚本，直接进入等待调试：

```
#!/bin/sh
if [[ $# == 0 ]]; then
	echo "./debugTool  [package] [activity] "
	exit 1
fi
package=$1
activity=$2
adb shell am start -D -n ${package}/${activity}
varId=`adb shell ps | grep package| awk '{print $2}'`
adb forward tcp:8800 jdwp:$varId
```

进入等待调试后，在Android studio中执行`Run` -> `Debug`启动刚才创建的远程调试器，进入动态调试了。通过断点可以查看内存的信息。

如下，查看smali中寄存器v3的对象：![image.png](https://upload-images.jianshu.io/upload_images/1689923-b6852b65b38448ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3生成Smali代码和Apk插桩
#### 3.1 生成Smali代码
Smali是Android系统中Dalvik虚拟机指令语言，Smali代码一般为dex反编译之后得到，所以我们可以通过下面方法得到。
1. 编写java代码`*.java`
2. 编译jvm字节码`javac *.class`
3. 字节码转成dex文件，class转成dex文件
通过android-sdk\build-tools\xx.xx.xx\lib下提供dx工具
` java -jar dx.jar --dex --output=*.dex *.class`
4. dex转成smali文件，通过android-sdk\platform-tools\baksmali.jar可得到Smali
`java -jar baksmali.jar *.dex`
5. 此时会生成一个out目录，里面包含了*.smali

同时，我们也可以通过Android studio插件java2smali来一键生成。
#### 3.2 Apk插桩
插桩：表示在反编译Apk之后回编译re.apk，并且在关键代码中插入我们需要的log信息，通过logcat来进行查看，插入日志的过程我们称之为插桩。

由于Smali代码需要行号寄存器等信息的描述，直接插入代码显得比较困难。一般采用静态方法的方式来进行插入。具体请查看[Github](https://github.com/CarryGanLove/igloggerForSmali)，
插桩完成后，需要重新打包Apk。

mac或linux环境下可以通过下面的shell脚本进行快速的打包和安装。

```
#!/bin/sh
if [[ $# == 0 ]]; then
	echo "./jarsignTool [path] "
	exit 1
fi

path=${1}
apktool b ${path}
jksUrl=https://raw.githubusercontent.com/CarryGanLove/igloggerForSmali/826bd04eaa18352dd36a861c8f2d7b4a9c373d4a/rong_debug.jks
apk_file=${path}/dist/${path##*/}.apk
echo "apk_file: ${apk_file}"

keystorePath="./rong_debug.jks"
if [ ! -f "${keystorePath}" ]; then 
	curl -o rong_debug.jks $jksUrl
fi

if [ ! -f "${keystorePath}" ];then 
	echo "download jsk error!" 
	exit 1
fi

apk_out_file=${apk_file/.apk}-re-signed.apk
echo "apk_out_file: ${apk_out_file}"

jarsigner -verbose -keystore ${keystorePath} -signedjar ${apk_out_file} ${apk_file}  -sigalg SHA1withRSA -digestalg SHA1 -storepass rong123 -keypass rong123  rong_debug

adb install -r ${apk_out_file}
```

安装至手机后可以开始运行Apk查看日志了。另外，告诉大家一个小技巧，无论是APP还是SDK一般都会封装一个log工具类，该类或许通过debug开关控制日志输出，或许通过proguard将日志调用给删除。如果是后者我们可以通过插桩的方法将日志继续抛出，如图。

![image.png](https://upload-images.jianshu.io/upload_images/1689923-4b18872a5f6a17b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
反编译得到d类为log的封装（工具类一般只包含静态方法），由于混淆的原因，`android.util.Log`类方法调用已被清除，我们插入了实现代码。通过上面的方法，将java代码转化成smali语句。替换反编译smali相应的文件。
![image.png](https://upload-images.jianshu.io/upload_images/1689923-09e99f302e63c9ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
执行打包脚本后，得到新Apk，运行可以查看所有开发者输出日志，通过日志配合查看反编译代码几乎可以完全了解所有java层逻辑。喜欢请点赞！



