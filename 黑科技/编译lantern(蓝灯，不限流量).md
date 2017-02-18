>官方下载版的蓝灯限制高速流量，但是通过源码编译的蓝灯貌似不限流量。通过这篇文章记录下在mac上编译lantern的过程。

#####一.为什么要自己编译
受到这篇博客《[lantern编译过程](http://blog.lanyus.com/archives/290.html)》的启发，文中说到自己编译的蓝灯不限流量。我在尝试编译了mac版的蓝灯后，的确发现并没有流量的显示，也没有购买专业版的链接，并且看youtube高清视频都很快很流畅，感觉是真的没有流量限制。

并且那篇博客作者在文中介绍了用官方提供的Dockerfile构建的带有编译环境的docker容器来编译linux和windows版lantern，非常方便，而且不污染自己的系统环境，如果需要linux和windows版本的可以参考。但是我自己开发环境是mac，编译mac版lantern必须要mac环境，所以没办法用docker，只能在自己mac上安装编译环境后编译，顺便编译了安卓版。
#####二.安装编译环境
1. 首先，安利一下 homebrew，mac上安装各种开发工具的利器（类似于apt-get，yum）。
2. [lantern的github地址](https://github.com/getlantern/lantern)，git clone到本地:
```
git clone --depth=1 https://github.com/getlantern/lantern.git
```
3. 安装go、node、gulp、appdmg、svgexport (后两个是编译mac才需要的包，仅mac上可装)
```
brew install go
brew install node
npm i gulp-cli -g
npm install -g appdmg
npm install -g svgexport
```

#####三.编译mac版lantern
进入到lantern目录，设置版本号（我设为9.9.9，避免出现需要更新的提示），编译。
```
cd lantern
export VERSION=9.9.9
make darwin
```
tip:如果遇到MaxIdleTime和EnforceMaxIdleTime的报错，我看了go的源码，好像net/http包下Transport类，并没有MaxIdleTime这个变量和EnforceMaxIdleTime这个方法，但是有个变量叫IdleConnTimeout应该是MaxIdleTime的意思吧，所以我把报错的地方的MaxIdleTime该为IdleConnTimeout，然后调用EnforceMaxIdleTime()这个方法的语句注释掉，然后编译就没问题了，目录下应该出现lantern.app了，双击打开就好了。
#####四.编译安卓版
1. 自行安装java，然后安装gradle(我遇到[这个问题](http://blog.csdn.net/rodulf/article/details/52593187)，后来降到2.14.1，所以建议安装3.0以下版本，最好2.2-2.8的，一脸心酸泪)。
```
brew install homebrew/versions/gradle214
```
2. 安装Android SDK 和NDK以及一些jar包依赖啥的
下载[SDK tools](https://developer.android.com/studio/index.html)，我只下了命令行工具。新建目录android-sdk(该目录对应后面环境变量里的ANDROID_HOME)，将SDK tools压缩包解压到该目录下，命名为tools，运行/android-sdk/tools/android打开Android SDK Manager，安装Android Platform-tools、Android Build-tools、Android6.0（API 23）、Extras/Android Support Repository、Extras/Google Repository。
下载[NDK](https://developer.android.com/ndk/downloads/index.html)，解压命名为android-ndk(该目录对应后面环境变量里的NDK_HOME)
3. 设置环境变量
```
export ANDROID_HOME=/Users/mhq/projects/lantern/android-sdk #改为前面提到的android-sdk的路径
export PATH=$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools:$ANDROID_HOME/build-tools:$PATH
export NDK_HOME=/Users/mhq/projects/lantern/android-ndk #改为前面提到的android-ndk的路径
export PATH=$NDK_HOME:$PATH
```
4. 编译安卓版lantern
```
make android-debug
```
正常的话，可以在`src/github.com/getlantern/lantern-mobile/app/build/outputs/apk`下找到lantern-debug.apk，放到手机上安装即可。
5. 我遇到过的问题
 1. “Gradle version 2.2 is required.” 
好像gradle>2.8都会报这个错，修改MobileSDK/build.gradle文件，在buildscript {下加一行：
```
	System.properties['com.android.build.gradle.overrideVersionCheck'] = 'true'
```
 2.  “failed to find Build Tools revision 23.0.3”
因为lantern默认用这个版本的build tools，如果你不是这个版本的，请去${ANDROID_HOME}/build-tools下查看自己有的版本，例如版本是25.0.2的，然后修改以下目录中的build.gradle文件，改为 `buildToolsVersion "25.0.2"`
```
MobileSDK/sdk  
LanternMobileTestbed/app
PubSub/sdk
src/github.com/getlantern/lantern-mobile
```
