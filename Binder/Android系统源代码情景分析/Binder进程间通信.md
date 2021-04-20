## 简介
Android 系统基于 Linux 内核开发，但是没有采用内核提供的传统进程间通信机制，而是开发了新的进程间通信机制——Binder。

Binder 在进程间传输数据时，只需要执行一次复制操作，不仅提高效率，还节省内存空间。

Binder 采用 CS 通信方式，提供服务进程称为 Server 进程，访问服务的进程称为 Client 进程。

每一个 Service 进程和 Client 进程都维护一个 Binder 线程池来处理进程间通信请求。

Service 进程和 Client 进程的通信要依靠运行在内核空间的 Binder 驱动程序来进行。

## Binder 驱动程序
#### Binder 驱动程序中有两种类型的数据结构，一种在内部使用，一种内外通用
##### 内部使用
- binder_work -> 待处理工作项

- binder_node -> Binder 实体对象，包括进程状态，引用状态，组件地址，事务状态，配置条件

  - 每一个 Service 组件在 Binder 驱动程序中都对应有一个 Binder 实体对象

  - Binder驱动程序将一个事务保存在一个线程的一个 todo 队列中，表示由该线程处理该事务，事务与 Binder 实体对象相关联，即与该 Binder 实体对象相应的 Service 组件在指定的线程中处理事务。

- binder_ref_death -> Service 组件死亡接收通知
  - Service 组件死亡、注册死亡接收通知 -> Binder 驱动程序向 Client 进程发送死亡通知
  - Client 进程注销死亡接收通知 -> Binder 驱动程序根据 Service 组件状态发送不同的 binder_ref_death 结构
- binder_ref -> Binder 引用对象描述，包括所引用的 Binder 实体对象
  - 每一个 Client 组件在 Binder 驱动程序中都对应有一个 Binder 引用对象

> 注： 一个 Binder 引用对象的句柄值在进程范围内是唯一的，因此，在两个不同的进程中，同一个句柄值可能代表的是两个不同的目标 Service 组件

- binder_buffer -> 内核缓冲区，进程间传输数据
  - 每一个使用 Binder 进程间通信机制的进程在 Binder 驱动程序中都有一个内核缓冲区列表

- binder_proc -> 正在使用 Binder 进程间通信机制的进程
  - 内核缓冲区有两个地址，内核空间和用户空间，均为虚拟地址，对应物理页面保存在成员变量 pages 中，按照按需分配原则为内核缓冲区分配物理页面

- binder_thread -> Binder 线程池中的线程

- binder_transaction -> 进程间通信过程（事务）
  - 目标线程在处理事务时，线程优先级不能低于目标 Service 组件所要求的线程优先级，也不能低于源线程的优先级

##### 内外通用 -> 一系列 IO 控制命令与应用程序进程通信
- binder_write_read -> 进程间通信所传输的数据
  - write_buffer & read_buffer 数据格式由通信协议代码及其通信数据组成，分别为命令协议代码（BinderDriverCommandProtocol） & 返回协议代码（BinderDriverReturnProtocol）
  - 数据格式 -> code + content of code
- binder_ptr_cookie -> Binder 实体对象或者一个 Service 组件死亡接收通知
- binder_transaction_data -> 进程间通信所传输的数据
  - transaction_flags -> 通信标志要求
  - 数据缓冲区内存布局 -> ptr.buffer + data + offset
- flat_binder_object -> 描述 Binder 实体对象、Binder 引用对象或者文件描述符

#### Binder 设备的初始化过程
- 创建目录 /proc/binder/proc
- 创建文件 state,stats,transactions,transaction_log,failed_tramsaction_log
- 创建 Binder 设备 misc_register

#### Binder 设备文件打开过程
- 创建 binder_proc 结构体 proc
- 初始化 proc
- 创建以进程 ID 为名称的只读文件

#### Binder 设备文件的内存映射过程
- 将虚拟设备映射到进程的地址空间的目的是为了给进程分配内核缓冲区，以便传输进程间通信数据。
- Binder 驱动程序最多可以为进程分配 4M 内核缓冲区传输进程间通信数据
- Binder 驱动程序为进程分配的内存缓冲区在用户空间只可以读，不可以写，不可以复制，禁止设置可能会执行写操作标识位

#### Binder 内核缓冲区管理
##### 分配
BC_TRANSACTION or BC_REPLY
binder_alloc_buf
- 判断内核缓冲区大小是否合理
- 判断内核缓冲区是否用于异步事务
- 使用最佳适配算法检查有无最合适内核缓冲区可用
- 为内核缓冲区分配物理页面
- 修改内核缓冲区列表和内核缓冲区红黑树
- 内核缓冲区初始化
##### 释放
在 Binder 驱动程序的返回协议 BR_TRANSACTION or BR_REPLY 后，进程使用命令协议 BC_FREE_BUFFER 释放相应内核缓冲区
binder_free_buf -> 删除内核缓冲区
- 保存要释放的内核缓冲区大小
- 判断是否用于异步事务
- 释放用来保存数据的地址空间所占用的物理页面

binder_delete_free_buffer -> 删除 binder_buffer 结构体
- 判断 binder_buffer 与 prev & next binder_buffer 所在虚拟地址页面的关系
- 释放对应的物理页面

##### 查询
binder_buffer_lookup
- 分步计算 binder_buffer 结构体的用户空间地址,内核空间地址
- 在已分配内核缓冲区红黑树中循环查找与内核空间地址相应的节点

## Binder 进程间通信库
模版参数 INTERFACE 是一个由进程自定义的 Service 组件接口，模版类 BnInterface 和 BpInterface 都需要实现该接口

BBinder 类为 Binder 本地对象提供了抽象的进程间通信接口，拥有两个重要的成员函数 transact & ontransact
- transact 负责处理 Binder 代理对象向 Binder 本地对象发出的进程间通信请求
- onTransact 负责分发与业务相关的进程间通信请求

RefBase -> IBinder -> BBinder -> BnInterface
Binder 本地对象通过引用计数技术维护生命周期

RefBase -> BpRefBase -> BpInterface
Binder 代理对象亦可以通过强指针和弱指针维护生命周期

BpRefBase 类为 Binder 代理对象提供了抽象的进程间通信接口，有重要成员变量 mRemote，指向 BpBinder 对象，可通过成员函数 remote 获取

BpBinder 类实现了 BpRefBase 类的进程间通信接口，成员变量 mHandle 是整数，表示 Client 组件的句柄值，通过成员函数 handle 获取

BpBinder.transact(mHandle, data) -> Binder Driver -> Binder 引用对象 -> Binder 实体对象 -> Service 组件

BBinder 与 BpBinder 均通过 IPCThreadState 类和 Binder 驱动程序交互

Service 组件实现原理
![](截屏2020-09-28%20下午4.35.24.png)

Client 组件实现原理
![](截屏2020-09-28%20下午4.35.46.png)

## Binder 进程间通信应用实例
asInterface() -> 将 IBinder 对象转换成 IFregService 接口

**手动敲一遍代码 ！！！**

## Binder 对象引用计数技术
Binder 实体对象、引用对象、本地对象、代理对象的交互过程
![](截屏2020-09-28%20下午5.17.15.png)

依赖关系：
代理对象 -> 引用对象 -> 实体对象 -> 本地对象

必须采用一种技术措施保证不能销毁一个还被其他对象依赖着的对象

#### 本地对象的生命周期
Service 进程中的其他对象可以通过智能指针引用这些 Binder 本地对象
但 Binder 驱动程序中的 Binder 实体对象运行在内核空间，不能通过智能指针引用运行在用户空间的 Binder 本地对象

Binder 驱动程序通过 BR_INCREFS、BR_ACQUIRE、BR_DECREFS、BR_RELEASE 协议引用运行在 Service 进程中的 Binder 本地对象。

#### 实体对象的生命周期
binder_inc_node() -> 增加 Binder 实体对象的引用计数
binder_dec_node() -> 减少 Binder 实体对象的引用计数
依据参数：strong、internal

一个引用列表 refs
三个引用计数 -> 内外相对于 Server/Client 进程而言
- internal_strong_refs  -> 外部强引用
- local_strong_refs     -> 内部强引用
- local_weak_refs       -> 内部弱引用

一个 Binder 实体对象的外部强引用计数 internal_strong_refs 与它所对应的 Binder 本地对象的引用计数是多对一的关系，这样可以减少 Binder 驱动程序与 Service 进程之间的交互，即减少它们之间的协议来往

#### 引用对象的生命周期
Binder 引用对象运行在内核空间，Binder 代理对象运行在用户空间

通过 BC_ACQUIRE、BC_INCREFS、BC_RELEASE、BC_DECREFS 协议修改 Binder 引用对象的强引用计数和弱引用计数

binder_thread_write()
- binder_inc_ref()
- binder_dec_ref()
- binder_delete_ref()
依据参数 strong

一个 Binder 引用对象的强引用计数与它所引用的 Binder 实体对象的外部强引用计数是多对一的关系，这样可以减少修改 Binder 实体对象的引用计数的操作

#### 代理对象的生命周期
可以理解为与 Binder 本地对象类似
Client 进程中的其他对象可以通过智能指针引用代理对象
Binder 引用对象是运行在内核空间中的，Binder 代理对象不能通过智能指针引用它们

Client 进程通过 BC_ACQUIRE、BC_INCREFS、BC_RELEASE、BC_DECREFS 协议引用 Binder 驱动程序中的 Binder 引用对象

一个 Binder 代理对象可能会被用户空间的其他对象引用，当其他对象通过强指针来引用这些 Binder 代理对象时，Client 进程就需要申请 Binder 驱动程序增加相应的 Binder 引用对象的强指针计数

一个 Binder 代理对象的引用计数与它所引用的 Binder 引用对象的引用计数是多对一的关系，这样可以减少 Client 进程和 Binder 驱动程序之间的交互，既减少协议往来

## Binder 对象死亡通知机制
#### 注册
代理对象 linkToDeath()
- 是否已经接收到死亡通知
- 是否存在于讣告列表 mObituaries 中
- requestDeathNotification() 首次注册死亡通知接收者
- flushCommands() 立刻使线程进入驱动程序中
驱动 binder_thread_write()
- 取出代理对象的句柄值和地址值并获取引用对象 ref
- 是否已注册过死亡接收通知
- 注册并检测引用对象所引用的本地对象是否已经死亡

#### 发送
Binder 驱动程序将设备文件 /dev/binder 的释放操作设置为 binder_release
只要在 binder_release 中检查进程退出时是否有 Binder 本地对象在里面运行，就可以判断是不是死亡的 Binder 本地对象

如果一个进程的 delivered_death 队列不为空，说明 Binder 驱动程序正在向它发送死亡接收通知

#### 注销
代理对象 unlinkToDeath()
- 是否已经接收过死亡通知，代理对象主动注销
- 检查是否存在于讣告列表 mObituaries 中
- clearDeathNotification()
- flushCommands()
与注册操作类似
若在 Binder 驱动程序将 BINDER_WORK_DEAD_BINDER 工作项发送给目标 Client 进程处理之前，目标 Client 进程刚好又请求 Binder 驱动程序注销与该工作项对应的死亡接收通知，那么 Binder 驱动程序会将发送与注销操作合并成 BINDER_WORK_DEAD_BINDER_AND_CLEAR 工作项交给 Client 进程来处理。

## Service Manager 的启动过程
Binder 进程间通信机制的核心组件之一
- Binder 进程间通信机制上下文管理者
- 系统中 Service 组件管理者
- Client 组件获取 Service 代理对象

Service_manager.c
- binder_open 打开设备文件并映射
  - open()
  - mmap()
- binder_become_context_manager 注册为上下文管理者
  - ioctl()
  - binder_ioctl()
- binder_loop 循环等待和处理 Client 进程通信请求

特殊组件 -> 对应的 Binder 本地对象是虚拟对象，地址值等于 0，并且被引用的引用对象的句柄值也等于 0。

> Binder 驱动程序为 Binder 线程创建 binder_thread 结构体时，会将状态设置为 BINDER_LOOPER_STATE_NEED_RETURN，表示线程完成当前操作后，需要马上返回用户空间，不能去处理进程间的通信请求

Binder 驱动程序允许同一个进程重复使用 IO 控制命令 BINDER_SET_CONTEXT_MGR 注册 Binder 进程间通信机制的上下文管理者

是否需要请求当前线程所属的进程 proc 增加一个新的 Binder 线程来处理进程间通信请求。
- 进程 proc 的空闲线程数 ready_threads 等于 0
- Binder 驱动程序当前不是正在请求进程 proc 增加一个新的 Binder 线程，即它的成员变量 requested_threads 的值等于 0
- Binder 驱动程序请求进程 proc 增加的 Binder 线程数 requested_threads_started 小雨预设的最大数 max_threads
- 当前线程是一个已经注册了的 Binder 线程，即它的状态为 BINDER_LOOPER_STATE_REGISTERED or BINDER_LOOPER_STATE_ENTERED

当一个进程使用 IO 控制命令 BINDER_WRITE_READ 与 Binder 驱动程序交互时
- binder_write_read 输入缓冲区长度 > 0    -> binder_thread_write()
- binder_write_read 输出缓冲区长度 > 0    -> binder_thread_read()

## Service Manager 代理对象的获取过程
defaultServiceManager()
-> gDefaultServiceManager = interface_cast<IServiceManager>(ProcessState::self()->getContextObject(NULL))

## Service 组件的启动过程

#### 注册
defaultServiceManager()->addService()

Client 进程和 Server 进程的一次进程间通信过程分五步
- Client 将数据封装成 Parcel 对象
- Client 发送 BC_TRANSACTION 命令协议 -> 驱动程序发送 BR_TRANSACTION_COMPLETE 返回协议 -> Client 处理后在驱动中等待 Server 进程返回通信结果
- 驱动程序发送 BR_TRANSACTION 返回协议
- Server 处理后发送 BC_REPLY 命令协议 -> 驱动程序处理后发送 BR_TRANSACTION_COMPLETE
- 驱动程序发送 BR_REPLY

> Binder 驱动程序发送 BR_TRANSACTION_COMPLETE 返回协议响应 Client 进程和 Server 进程的原因是为了让 Client 进程和 Server 进程在执行进程间通信的过程中，有机会返回到用户空间去做其他事情，从而增加它们的并发处理能力

Parcel 对象 data 的成员函数 writeStrongBinder 将要注册的 Service 组件封装成一个 flat_binder_object 结构体，然后传递给 Binder 驱动程序

命令协议缓冲区 mOut 内存布局
![](截屏2020-09-29%20下午4.16.49.png)

remote()->transact(code, data, reply, flags)
- data.write -> parcel data 
- writeTransactionData -> struct binder_transaction_data 
- talkWithDriver -> struct binder_write_read 
-> Binder Driver

binder_transaction() -> 处理 BC_TRANSACTION & BC_REPLY 协议
- 根据参数找到 Binder 实体对象
- 找到最优目标线程 target_thread 接收 BR_TRANSACTION 返回协议
- 为 BR_TRANSACTION & BR_TRANSACTION_COMPLETE 协议分配结构体并封装成工作项加入 todo 队列
- 循环依次处理进程间通信数据中的 Binder 对象

binder_thread_read()
- 检查并取出工作项
- 将 BR_TRANSACTION_COMPLETE 返回协议写入用户空间提供的缓冲区中

binder_parse()
- binder_object -> flat_binder_object
- binder_txn -> binder_transaction_data
- binder_io -> (Class) Parcel 

BC_FREE_BUFFER -> binder_thread_write()
- 得到内核缓冲区的用户空间地址并查找对应 buffer
- transaction & async_transaction 是否为 0
- binder_transaction_buffer_release() -> 减少引用计数
- binder_free_buf() -> 释放内核缓冲区 buffer

binder_transaction_buffer_release()
- 判断内核缓冲区 buffer 是否是分配给一个 Binder 实体对象使用的
- 遍历释放内核缓冲区中的 Binder 对象，并减少所对应的 Binder 实体对象或引用对象的引用计数

BC_REPLY -> binder_transaction()
- 通过 binder_transaction 结构体找到请求与线程 thread 进行进程间通信的线程
- 传递 todo 队列和 wait 队列
- 创建结构体，封装工作项

当一个 Server 线程将一个进程间通信结果返回给一个 Client 线程时，不需要在进程间通信结果数据中指定一个目标 Binder 实体对象

> Binder 驱动程序向目标线程发送一个  BR_REPLY 返回协议后，就可以将协议相关的 binder_transaction 结构体释放，但是分配给 binder_transaction 结构体的内核缓冲区 buffer 还未被释放，因为 Binder 驱动程序还需要通过它将进程间通信结果传递给线程 thread，当线程 thread 处理完成 Binder 驱动程序发送的 BR_REPLY 返回协议后，它就会使用 BC_FREE_BUFFER 释放内核缓冲区 buffer

#### 启动 Binder 线程池
ProcessState::startThreadPool()
-> spawnPooledThread()
-> PoolThread.run()
-> PoolThread.threadloop()
-> joinThreadPool(mIsMain)

Binder 线程的生命周期
- 注册到 Binder 线程池
- 无限循环不断等待和处理进程间通信请求
- 退出 Binder 线程池

## Service 代理对象的获取过程
defaultServiceManager()
-> getService()
-> asInterface()

getService() 实现的是一个标准的 Binder 进程间通信过程，可以划分为五个步骤：
- FregClient 封装要获取的 Service 组件数据在 Parcel 中，传递给 Binder 驱动程序
- FregClient BC_TRANSACTION -> Binder 驱动程序 BR_TRANSATION_COMPLETE
- Binder 驱动程序 BR_TRANSACTION 请求 Service Manager 进程执行 CHECK_SERVICE_TRANSACTION 操作
- Service Manager BC_REPLY -> Binder 驱动程序创建 Binder 引用对象 -> Binder 驱动程序 BR_TRANSACTION_COMPLETE
- Binder 驱动程序 BR_REPLY 引用对象句柄值

CHECK_SERVICE_TRANSACTION
bio_get_string16() -> do_find_service() -> find_svc() -> bio_put_ref()
si->ptr -> 引用对象 -> 实体对象 -> 创建引用对象

BINDER_TYPE_HANDLE | BINDER_TYPE_WEAK_HANDLE
- binder_get_ref()
- binder_get_ref_for_node()
- binder_inc_ref()

## Binder 进程间通信机制的 Java 接口
#### Service Manager 的 Java 代理对象的获取过程
Java 代理对象类型：ServiceManagerProxy

创建顺序:
Binder 代理对象 -> Java 服务代理对象 -> Java 代理对象

Binder 代理对象通过 ObjectManager 类的 mObjects 成员变量来与 Java 服务代理对象关联起来

Binder 本地对象 -> Java 服务 -> Java Service Manager 服务

#### Java 服务接口的定义与解析
Android 应用程序中，通过 Android 接口描述语言（AIDL）定义 Java 服务接口

编译后，生成文件定义一个 Java 接口 IFregService、一个 Java 抽象类 IFregService.Stub 和一个 Java 类 IFregService.Stub.Proxy

IFregService.Stub 类描述 Java 服务
IFregService.Stub.Proxy 类描述 Java 服务代理对象

#### Java 服务的启动过程
Android 系统进程 System 和 Android 应用程序进程在启动时会在内部启动一个 Binder 线程池，运行在里面的 Java 服务在启动时，将自己注册到 Service Manager 中就可以

本质：不是将 Java 服务注册到 Service Manager 中，而是将其对应的类型为 JavaBBinder 的 Binder 本地对象注册到 Service Manager 中
#### Java 服务代理对象的获取过程
```
fregService = IFregService.Stub.asInterface(
          ServiceManager.getService("freg"));
```

#### Java 服务的调用过程
```
client: 
fregService.getVal() -> mRemote.transact()

service: 
JavaBBinder.onTransact() -> execTransact() -> IFregService.Stub.onTransact()
```