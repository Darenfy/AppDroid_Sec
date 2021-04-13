# AppDroid_Sec
Something To Do Android Application Security Research

## 基础知识
- [Android Init 语法解析](https://github.com/Darenfy/AppDroid_Sec/blob/main/Android%20Init%20%E8%AF%AD%E6%B3%95%E8%A7%A3%E6%9E%90.md)

### 文件格式相关
- DEX/ODEX
  - [Dalvik 操作码](https://github.com/Darenfy/AppDroid_Sec/blob/main/Dalvik%E6%93%8D%E4%BD%9C%E7%A0%81.pdf)
  - [DEX 文件格式](https://github.com/Darenfy/AppDroid_Sec/blob/main/dex.jpg) -> 出自虫神
  - [ODEX 文件格式](https://github.com/Darenfy/AppDroid_Sec/blob/main/odex.jpg) -> 出自虫神

- [AXML](https://github.com/Darenfy/AppDroid_Sec/blob/main/axml.png) -> 出自虫神  
  - AXML 文件格式的修改: [AmBinaryEditor](https://github.com/ele7enxxh/AmBinaryEditor)、[AndroidManifestFix](https://github.com/zylc369/AndroidManifestFix)
- [ARSC](https://github.com/Darenfy/AppDroid_Sec/blob/main/arsc.png) -> 出自虫神

- [OAT](https://github.com/Darenfy/AppDroid_Sec/blob/main/oat.png) -> 出自虫神

- ELF  
  - [ARM 汇编指令集支持情况](https://github.com/Darenfy/AppDroid_Sec/blob/main/ins_set.png)

- 文件格式分析工具  
  - 010 Editor  
  - [gnu binutils](https://www.gnu.org/software/binutils/)

不知名网友的统概图（待完善）
- [Android Dalvik虚拟机](https://github.com/Darenfy/AppDroid_Sec/blob/main/Android%20Dalvik%E8%99%9A%E6%8B%9F%E6%9C%BA.png)

- [Android文件格式](https://github.com/Darenfy/AppDroid_Sec/blob/main/Android%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F.png)



## 工具包

- [Android-Crack-tool for Mac](https://github.com/Jermic/Android-Crack-Tool)  
支持常规的 ``APK`` 编译与反编译功能

- [GDA](https://github.com/Darenfy/AppDroid_Sec/blob/main/GDA.md)  
国内研发的 ``APK`` 分析工具 【注：目前体验感不是太好】

- [Androguard](https://github.com/androguard/androguard)  
用来处理 ``APK`` 的 ``Python`` 工具包，很多自动化分析项目都内嵌

- [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF)  
跨平台开源自动化动态分析框架



#### 分析工具
- ``IDA Pro``  
- [Hopper](https://www.hopperapp.com/)  
- 动态调试器  
  - GDB  
  - LLDB  
- 编译与反编译
  - [JEB](https://github.com/Darenfy/jeb_gather)
  - [Smali/BakSmali](https://github.com/JesusFreke/smali)
  - [ApkTool](https://ibotpeaches.github.io/Apktool/)
- 反编译查看
  - [JD-GUI](https://github.com/java-decompiler/jd-gui/releases)
  - [JADX](https://github.com/skylot/jadx)
  - [ByteCode Viewer](https://github.com/Konloch/bytecode-viewer) -> 本人基本没用过

### 辅助工具
- [dex-finder](https://github.com/LeadroyaL/dex-finder)  
快速寻找一个类所在 dex 的小工具

---
## 系统源码

- ### 查看各个版本的 Android 源码
  - http://aosp.opersys.com/#  
  - https://cs.android.com/

- ### Android 系统启动流程 8.1.0_r1 版本
  根据 r0ysue 知识星球的内容画的流程图  
  https://github.com/Darenfy/AppDroid_Sec/blob/main/init.png

  参考链接：https://wx.zsxq.com/dweb2/index/search/%E5%90%AF%E5%8A%A8/alltopics


- ### [Zygote 和 System 进程的启动过程](https://github.com/Darenfy/AppDroid_Sec/blob/main/Zygote%20%E5%92%8C%20System%20%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.md)

- ### [Android应用程序进程启动过程](https://github.com/Darenfy/AppDroid_Sec/blob/main/Android%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.md)

---
## HOOK 与注入  
HOOK 类型：  
- Dalvik Hook  
- ART Hook  
- So Hook  
  - LD_PRELOAD Hook  
  - GOT Hook  
  - Inline Hook  

Hook 注入框架：  
- [Xposed](https://repo.xposed.info/)    
- [Frida](https://frida.re/)  

动态注入：  
- DEX 注入  
- SO 注入

注：  
1. 当在进行 Hook 时，如果遇到错误，可以使用 ``cat /proc/pid/maps`` 查看所要 hook 的代码是否已经加载
