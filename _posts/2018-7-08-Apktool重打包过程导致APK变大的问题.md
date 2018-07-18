---
layout:     post
title:      Apktool重打包过程导致APK变大的问题
date:       2018-07-08
author:     Gavin
header-img: img/post-bg-coffee.jpg
catalog: true
tags:
    - apktool
    - 反编译
    - yaml
---



### 1.背景

上周通过脚本重新打包导致apk包变大，经调研和分析发现：在apktool 2.0.3之后为了快速解压和打包，加入了反编译文件回编译`不压缩`机制。该配置文件位于apktool.yml文件中。

![1-1 反编译产物](https://upload-images.jianshu.io/upload_images/1689923-15f69ca69efbeb1e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![1-2 apktool.yml详情](https://upload-images.jianshu.io/upload_images/1689923-545bd2ad2644fcb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

几乎所有的二进制和非二进制文件打包时都未采用压缩，导致生成的apk变大（即使未签名依然比原apk大），最后在回编过程进行分析。

在反编译文件中查看yml文件，该文件记录了apk反编译信息（见`apktool`源码`brut/androlib/meta/MetaInfo`）
`compressionType`：记录arsc的是否压缩
`doNotCompress`：记录不压缩的文件类型和路径


### 2.解法
由于最新版本的apktool回编采用了不压缩的策略，导致我们的重编译apk变大。
可以想到的有两种方法：
1.通过apktool高版本来释放1.apk（framworks apk见上次分享），低版本处理反编译和打包的问题。
2.通过脚本将apktool.yml文件进行修改和定制，可以使得我们的release包更小。

### 3.YAML格式
我们需要对yaml格式的文件进行解析，有一个支持yaml格式的著名开源库`snakeyaml`（用java实现）。
我采用jar的形式对这一部分代码（解析YAML和写入文件）进行实现。通过shell脚本插入该任务，从而达到了自动化过程。
`snakeyaml`的使用参考：
snakeyaml项目地址：https://bitbucket.org/asomov/snakeyaml
snakeyaml java使用：https://www.jianshu.com/p/d8136c913e52

### 4.CI上打包结果
通过对反编译脚本的优化使得回编译签名包和zipalign包显著减小。

| 原始apk        | 反编译签名apk           | 反编译zipalign-apk  |
| ------------- |-------------| ----- |
| 23.90M     | 22.48M| 22.48M |

![4-1 CI上打包产物](https://upload-images.jianshu.io/upload_images/1689923-257f9985b17ae0ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 5.脚本实现

```bash
#定义out文件的路径
rebApkFile=$cache_dir/rebuildApp.apk
echo "rebuildTools> 修改yml打包的配置文件"
#定义jar的路径，用来提取和修改apktool.yml文件
yamlFileCovert_jar=./yamlFileCovert.jar
#执行yamlFileCovert_jar
java -jar $yamlFileCovert_jar ${apkDecodeDir}/apktool.yml ${apkDecodeDir}/apktool.yml
echo "rebuildTools> 开始回编译apk并替换目标文件..."

apktool b -o ${rebApkFile} ${apkDecodeDir}
echo "rebuildTools> 开始签名新apk..."

...

```

java部分：

```java
public class YamlFileParse {

    public static void main(String[] args) {
        String fileName;
        String outFile;
        if (args != null && args.length >= 2) {
            fileName = args[0];
            outFile = args[1];
        } else {
            System.out.print("请输入yml路径!\n" +
                    "java -jar yamlFileCovert.jar [in] [out] ");

            return;
        }
        try {
            MetaInfo metaInfo = loadYml(fileName);
            saveYml(outFile, metaInfo);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void saveYml(String outFile, MetaInfo ret) throws IOException {
        FileOutputStream fos = new FileOutputStream(new File(outFile));
        OutputStreamWriter outputStreamWriter = new OutputStreamWriter(fos, StandardCharsets.UTF_8);
        Writer writer = new BufferedWriter(outputStreamWriter);
        save(ret, writer);
        outputStreamWriter.close();
    }

    private static MetaInfo loadYml(String fileName) throws IOException {
        FileInputStream yamlInput = new FileInputStream(new File(fileName));
        MetaInfo ret = new Yaml().loadAs(yamlInput, MetaInfo.class);
        yamlInput.close();
        ret.doNotCompress = null;
        return ret;
    }

    private static void save(MetaInfo info, Writer output) {
        DumperOptions options = new DumperOptions();
        options.setDefaultFlowStyle(DumperOptions.FlowStyle.BLOCK);
        new Yaml().dump(info, output);
    }
}

```

java的源码部分包括：`yaml`格式文件的读写和`MetaInfo`（用于apktool.yml序列化指定，原因见apktool.yml首行）类。
![5-1 java工程的结构](https://upload-images.jianshu.io/upload_images/1689923-af2d872841d354af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以下是apktool.yml文件，由于文件中包含了MetaInfo类，所以必须指定yml序列化需要的类（这一部分拷贝了apktool源码中部分MetaInfo部分）。

```yml
!!brut.androlib.meta.MetaInfo
apkFileName: Rong360-release.apk
compressionType: false
doNotCompress: null
isFrameworkApk: false
packageInfo: {forcedPackageId: '127', renameManifestPackage: null}
sdkInfo: {minSdkVersion: '14', targetSdkVersion: '21'}
sharedLibrary: false
sparseResources: false
usesFramework:
  ids: [1]
  tag: null
version: 2.3.3
versionInfo: {versionCode: '316', versionName: 3.1.6}
```

运行`java -jar $yamlFileCovert_jar ${apkDecodeDir}/apktool.yml `对yml进行修改后，回编apk，从而产物的apk大小达到了预期。