# 背景

在 app 的开发过程中，我们经常需要根据不同的场景打不通环境的包

在提测的时候需要打测试包，上线验收前需要打生产包

而app不同于网站，可以通过网站的域名一眼就能区分出这是测试站还是正式站

app打的包在给测试和产品验收时，经常不能直接区分出这个包是连接的测试服还是生产服

这就会多出很多无意义的沟通成本，所以就有了本篇文章

# 需求

我需要根据我的环境配置不同的app名，最好能够直接就能从蒲公英的下载地址上就能区分出
这个app是测试包还是正式包。即使不是从蒲公英下载的安装包也能通过安装包的文件名，安装后的app名直接区分出这是测试包还是正式包。

通过安装包的文件名识别测试包和正式包的需求主要来自apk的分发

1. 担心自己打好包了传给产品时自己传错
2. 担心产品上传各应用商店时传错或者出了问题互相甩锅

所以需要动态配置app名，还需要给生成的apk文件名上加上环境、版本、时间戳

# 需要的前置知识

- 原生如何修改配置app名
    1. iOS 了解编译前动态配置app名（xcconfig、info.plist、shell）
    2. Android 同理，需要了解gradle脚本（配置app名、获取版本信息）
- 如何自动为打包好的apk文件名加上环境、版本、时间信息

- flutter 如何将配置信息传递给原生

## 修改 iOS App名

前提 flutter 版本需要高于 3.7 ，原因后面再说

以 flutter 3.7.12 为例子新建 flutter 工程

新建好的工程使用 Xcode 打开，会发现以下三个xcconfig配置文件

![xcconfig](https://pica.zhimg.com/80/v2-ec39d64478234ad1aacf157e31333868_1440w.png)

编辑 Debug.xcconfig、Release.xcconfig


```shell
# Debug 下加入
APP_NAME=测试名
# Release 下加入
APP_NAME=正式名
```

配置info.plist

![配置info.plist](https://pica.zhimg.com/80/v2-818b5030f8f778ea9157e6d37a10e6da_1440w.png)

打开 info.plis 并找到 Bundle display name 修改其值为 $(APP_NAME)

通过修改编译配置就可以运行得到结果了，Debug 时 app 显示测试名、Release 显示正式名

![修改编译配置](https://pic1.zhimg.com/80/v2-5d3d3d902634bf0071bfb09325d1cab5_1440w.gif)

修改 iOS app 名先简单介绍到这里

## 修改 Android app 名

1. 找到 app 下的 build.gradle 定义变量app名

```groovy
def appName = '测试名'
```

2. 设置 app 名

```groovy
android {
    defaultConfig {
        resValue("string", "appName", appName)
    }
}
```

![完整gradle代码](https://pic1.zhimg.com/80/v2-816ee3a80594cc39ddc5706360841732_1440w.png)

3. 找到 AndroidMainfest.xml 使用 app 名 android:label="@string/appName"

```xml
<application android:label="@string/appName"/>
```
![](https://pic1.zhimg.com/80/v2-5e264e9992f9bf2413afe10788efe2f7_1440w.png)


## flutter 配置编译变量 --dart-define 命令

flutter 可以通过 --dart-define 命令配置添加编译时变量，可以同时用于flutter与原生端。

有了这个命令我们就可以实现动态设置 app 名的功能，就可以将我们的需求给串起来了。

通过命令我们就可以在编译前配置app名，编译过程中为app设置名称

### 在 Android 配置

1. --dart-define 命令 配置 app 名

```shell
flutter build apk --dart-define APP_NAME=测试名
```

2. Android 获取配置的 app 名，修改 app 下的 build.gradle

```
def dartEnvironmentVariables = [
        APP_NAME: project.hasProperty('APP_NAME') ? project.property('APP_NAME') : '默认app名',
]

if (project.hasProperty('dart-defines')) {
    dartEnvironmentVariables = dartEnvironmentVariables + project.property('dart-defines')
            .split(',')
            .collectEntries { entry ->
                def pair = new String(entry.decodeBase64(), 'UTF-8').split('=')
                [(pair.first()): pair.last()]
            }
}

def appName = dartEnvironmentVariables.APP_NAME
```
![完整代码](https://picx.zhimg.com/80/v2-82e852c33e625cba82575047943ba42c_1440w.png)

### 在 iOS 中配置

1. 新增编译脚本

![](https://pic1.zhimg.com/80/v2-3466286ef284c5bb78dd022f0c43aac6_1440w.png)


```shell
IFS=',' read -r -a define_items <<< "$DART_DEFINES" 
#从环境变量 DART_DEFINES 中取值，并以 , 分割，放入 define_items 数组中

has_app_name=0; #默认没有app名
temp_array=();
#echo ${define_items[@]} 打印数组中所有值
for index in "${!define_items[@]}"
do
    # base64 解码后赋值给value
    value=$(echo "${define_items[$index]}" | base64 -D);
    app_name="APP_NAME"
    # 判断是否包含APP_NAME
    if [[ $value == *$app_name* ]]
    then
        has_app_name=1;
        temp_array[$index]=$value;
    fi    
done

# 没有app名设置默认app名
if [ $has_app_name == 0 ]
then
    temp_array["APP_NAME"]="APP_NAME=默认app名";
fi    
echo "----> ${temp_array[@]}"
# 将 APP_NAME 写入 DartDefines.xcconfig
printf "%s\n" "${temp_array[@]}" > ${PROJECT_DIR}/Flutter/DartDefines.xcconfig
```

2. **先运行一次生成 DartDefines.xcconfig 并引用至项目**

![](https://pic1.zhimg.com/80/v2-8a0386e8c06d3cd7da61ec4c9a28669b_1440w.gif)


## 在 android studio 配置好参数运行一下看看


```
--dart-define APP_NAME=测试名2
```

![](https://pic1.zhimg.com/80/v2-6c278a5ebf9bdf1b9fe5749e078b065d_1440w.gif)

# Android 安装包文件名带上版本号信息

app 下的 build.gradle 新增以下代码


```groovy
android {
    applicationVariants.configureEach { variant ->
        variant.outputs.configureEach { output ->
            if (variant.buildType.name == "release") {
                def createTime = new Date().format('yyyy_MM_dd_HH_mm_ss')
                def newApkName = "${dartEnvironmentVariables.APP_NAME}_app_v${defaultConfig.versionName}_${createTime}.apk"
                outputFileName = new File(newApkName)
            }
        }
    }
}

```

使用以下命令打包看看输出的apk是否带上版本信息了
```
flutter build apk --dart-define APP_NAME=测试名字
```
apk 路径 项目名/build/app/outputs/apk


