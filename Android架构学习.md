## 概览
安卓架构图  
![](https://source.android.com/devices/images/ape_fwk_all.png)

#### HAL hardware abstraction layer
HAL定义一个标准接口以供硬件供应商实现
必须为产品所提供的特定硬件实现相应的HAL（和驱动程序）
HAL实现通常内置在共享库模块（.so）中

HAL接口包含两个组件：模块和设备

**HAL类型**
搭载android 8.0或更高版本必须支持HIDL语言编写的HAL，可以是绑定式也可以是直通式
Android R 也支持 AIDL 编写的HAL。所有的 AIDL HDL 均为绑定式
推出时即搭载8.0必须只支持绑定式，升级到8.0可以使用直通式HAL
- 绑定式 HAL：以HIDL和AIDL表示的HAL android 框架和HAL之间通过Binder进程间通信调用进行通信
- 直通式 HAL：HIDL封装的传统HAL或旧版HAL

Android提供的以下HIDL接口一律在绑定模式下使用：
android.frameworks.*    android.system.*    android.hidl.*
(no android.hidl.memory@1.0)

#### HIDL 一般信息
HIDL 允许指定类型和方法调用（会汇集到接口和软件包中）
HIDL 旨在用于进程间通信 (IPC)
HIDL 可指定数据结构和方法签名 -> 整理归类到接口 -> 汇集到软件包

**使用Binder IPC**
Android 8 开始，Android 框架和 HAL 使用 Binder互相通信，极大增加Binder流量

- 多个Binder域
    每个Binder上下文都有设备节点和上下文（服务）管理器，只能通过上下文管理器所属的设备节点进行访问，通过上下文传递Binder节点，只能由另一个进程从相同的上下文访问上下文管理器
- 分散-集中
    Android 8 将副本数量减少到1，Binder驱动程序立即将数据复制到目标进程中。
- 精细锁定
- 实时优先级继承
- 用户空间变更

Android 8 之后，拥有三个 IPC 域：
IPC域	  ｜      说明
/dev/binder	 ｜   框架/应用进程之间的 IPC，使用 AIDL 接口
/dev/hwbinder	｜ 框架/供应商进程之间的 IPC，使用 HIDL 接口    供应商进程之间的 IPC，使用 HIDL 接口
/dev/vndbinder	｜ 供应商/供应商进程之间的 IPC，使用 AIDL 接口

#### HIDL C++
Android O 重新设计了Android操作系统的架构，以在独立于设备的安卓平台与特定于设备和供应商的代码之间定义清晰的接口。
Android已经以HAL接口的形式（在 hardware/libhardware 中定义为C头文件）定义了许多此类接口。
HIDL将HAL接口替换为带版本编号的稳定接口

HIDL接口的C++实现 hidl-gen编译器基于 HIDL .hal 文件自动生成的文件，及如何打包，如何将这些文件和C++代码集成

**客户端和服务器实现**
客户端实现是指通过在接口上调用方法使用该接口的代码
服务器实现是指可接受来自客户端的调用并返回结果

旧版HAL发展历程  
![](https://source.android.com/devices/architecture/images/treble_cpp_legacy_hal_progression.png)

HIDL 接口软件包位于 hardware/interfaces 或 vendor/ 目录下
顶层 hardware/interfaces 直接映射到 android.hardware 软件包命名空间

hidl-gen 将 .hal 文件编译成一组 .h .cpp文件
🌰：
```c++
package android.hardware.samples@1.0;
interface IFoo {
    struct Foo {
       int64_t someValue;
       handle  myHandle;
    };

    someMethod() generates (vec<uint32_t>);
    anotherMethod(Foo foo) generates (int32_t ret);
};
```
IFoo.hal 文件应该在 hardware/interfaces/samples/1.0 下
会在 samples 软件包中创建一个 IFoo interface  
![](https://source.android.com/devices/architecture/images/treble_cpp_compiler_generated_files.png)

HIDL 软件包中定义的每个接口在其软件包的命名空间内都有自己的自动生成的C++类。
服务器实现接口，客户端在接口上调用方法
接口可以由服务器按名称注册，也可以作为参数传递到以HIDL定义的方法




#### AIDL
Android Interface Descriptor Language 是一款可供用户用来抽象化IPC的工具

参考资料：
https://source.android.com/devices/architecture
