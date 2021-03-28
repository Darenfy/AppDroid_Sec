## Android Init 语言
该语言主要包含五种主要的语句类：
- Actions 操作
- Commands 命令
- Services 服务
- Options 选项
- Imports 导入

**大概特征**
1. 使用 ``#`` 作为注释
2. 面向行，即每一行都是一个语句
3. 语句标记均由空格分隔
4. 可以使用语法 ``${property.name}`` 拓展系统属性
5. 操作和服务隐式声明一个新部分，所有属于该部分的命令和选项都属于最近声明的部分
6. 服务具有唯一的名称

### Init.rc 文件
``init.rc`` 文件是最主要的 ``.rc`` 文件，由 ``init`` 可执行文件在执行开始时加载，负责系统的初始设置

系统中相应目录的对应目的：
- ``/system/etc/init`` 用于核心系统项，例如 ``SurfaceFlinger``, ``MediaService`` 和 ``logd``

- ``/vendor/etc/init`` 用于 ``SoC`` 供应商项，比如核心 ``SoC`` 功能所需的操作或守护程序

- ``/odm/etc/init`` 用于设备制造商项，比如运动传感器或其他外围功能所需要的操作或守护程序

### Actions
Actions 是命令序列，拥有触发器。触发器用来确定何时操作被触发。

当某个与操作匹配的事件触发时，然后操作将被添加到要执行的队列的尾部

每一个队列中的操作都按照顺序出队，操作中的每一个命令按顺序执行。

Init处理活动中命令执行之间的其他活动，包括：设备创建和销毁、属性设置、进程重启

Actions 形式
```bash
on <trigger> [&& <trigger>]*
    <command>
    <command>
    <command>
```

### Services
服务是 init 启动的程序，当退出时可能会重新启动

Services 形式
```bash
service <name> <pathname> [ <argument> ]*
    <option>
    <option>
    ...
```

### Options
Options 是服务的编辑器，影响 init 运行服务的方式和时间

常规 Options
```bash
class <name> [ <name>\* ] -> 指定服务的类名

group <groupname> [ <groupname>\* ] -> 在执行服务前，请改为 groupname

priority <priority> -> 调整服务进程的优先级

user <username> -> 在执行服务前，更改为 username

oneshot -> 服务退出时不用重新启动

onrestart -> 当服务重启时执行命令

socket <name> <type> <perm> [ <user> [ <group> [ <seclabel> ] ] ] -> 创建一个名为 /dev/socket/name 的 Unix 套接字，并将其 fd 传递给启动的进程

writepid <file> [ <file>\* ] -> 当 fork 时将子进程的 pid 写入指定文件
```

### Triggers
Trigger 是用于匹配特定类型事件并引起操作发生的字符串

可以被分成 事件触发器 和 属性触发器

事件触发器：通过 ``trigger`` 命令或者 init 可执行文件中 ``QueueEventTrigger()`` 函数触发的字符串

属性触发器：当命名属性的值变成给定新值或者更改为任意新值时触发的字符串

一个操作可以有多个属性触发器，但是只能有一个事件触发器

### Commands
常规 Command
```bash
enable <servicename> -> 将禁用服务转换为启用服务

exec_start <service> -> 启用给定的服务，并停止其他初始化命令的处理，直到返回

mkdir <path> [<mode>] [<owner>] [<group>] [encryption=<action>] [key=<key>] -> 根据 path 创建目录

mount_all [ <fstab> ] [--<option>] -> 使用给定的 fs_mgr 格式 fstab 调用 fs_mgr_mount_all

mount <type> <device> <dir> [ <flag>\* ] [<options>] -> 尝试挂载命名设备到目录 dir_flag_s

restart <service> -> 停止并重新启动正在运行的服务

setprop <name> <value> -> 设置系统属性 name 为 value

start <service> -> 启动服务

stop <service> -> 停止服务

trigger <event> -> 触发一个事件，通常是将操作排到另一个操作之后
```

### Imports
``import <path>``
解析一个 init 配置文件，拓展到当前配置。如果 path 是目录，则将目录中的每个文件都解析为配置文件，但是并不会解析嵌套目录

### Early Init Boot Sequence
初始化引导顺序分为三个阶段：
- 第一阶段 Init
负责设置用来加载系统其余部分的最小要求配置，具体来说包括：
  - 挂载 ``/dev``、``/proc``
  - 挂载 ‘early mount’ 分区（该分区需要引入所有包含系统代码的分区，例如系统和供应商）
  - 将 ``system.img`` 挂载到具有 ``ramdisk`` 的设备根目录

- Selinux 设置
使用 ‘selinux_setup‘ 参数执行 ``/system/bin/init``，可以选择将 ``SELinux`` 编译并加载到系统中

- 第二阶段 Init
使用 ‘second_stage‘ 参数执行 ``/system/bin/init``，init 主要阶段将运行，通过 ``init.rc`` 脚本继续启动过程



---
#### 参考资料
https://android.googlesource.com/platform/system/core/+/master/init/README.md

https://www.jianshu.com/p/21584df4387b
