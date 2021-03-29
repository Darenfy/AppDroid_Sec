本文主要是介绍应用启动的主要流程

源码版本 ``8.1.0_r1``

**简易流程图**  
![](https://github.com/Darenfy/AppDroid_Sec/blob/main/zygote%26system.png)

## Zygote 进程启动
在 8.1.0 版本，zygote 分成四个部分：
- init.zygote32.rc
- init.zygote32_64.rc
- init.zygote64.rc
- init.zygote64_32.rc

启动脚本为：
```bash
service zygote /system/bin/app_process{"", 32, 64} -Xzygote /system/bin --zygote --start-system-server
  socket zygote stream 660 root system
  ...

service zygote_secondary /system/bin/app_process{64, 32} -Xzygote /system/bin --zygote --socket-name=zygote_secondary
  socket zygote_secondary stream 660 root system
  ...
```
该脚本说明在 Zygote 进程是以服务的形式启动的,启动过程中，会创建名称为 "zygote" 的 Socket，启动完成后，会立刻将 System 进程启动起来

在 init.rc 中会引入这些文件，并在 late-init 过程中启动 zygote
```cpp
import /init.${ro.zygote}.rc

# Mount filesystems and start core system services.
on late-init
    # Now we can start zygote for devices with file based encryption
    trigger zygote-start

# It is recommended to put unnecessary data/ initialization from post-fs-data
# to start-zygote in device's init.rc to unblock zygote start.
on zygote-start && property:ro.crypto.state=unencrypted
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=unsupported
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary
```

#### init -> service
```cpp
#include "service.h"

void handle_control_message(const std::string& msg, const std::string& name) {
    Service* svc = ServiceManager::GetInstance().FindServiceByName(name);
    if (svc == nullptr) {
        LOG(ERROR) << "no such service '" << name << "'";
        return;
    }

    if (msg == "start") {
        svc->Start();
    } else if (msg == "stop") {
        svc->Stop();
    } else if (msg == "restart") {
        svc->Restart();
    } else {
        LOG(ERROR) << "unknown control msg '" << msg << "'";
    }
}
```
可以看到 init 会根据拿到 ServiceManager 的实例，根据 msg 去进行不同的操作

**我们进入 service.cpp 查看**
**system/core/init/service.cpp**
```cpp
bool Service::Start() {
    ...
    LOG(INFO) << "starting service '" << name_ << "'...";

    pid_t pid = -1;
    if (namespace_flags_) {
        pid = clone(nullptr, nullptr, namespace_flags_ | SIGCHLD, nullptr);
    } else {
        pid = fork();
    }

    if (pid == 0) {
        for (const auto& ei : envvars_) {
            add_environment(ei.name.c_str(), ei.value.c_str());
        }

        std::for_each(descriptors_.begin(), descriptors_.end(),
                      std::bind(&DescriptorInfo::CreateAndPublish, std::placeholders::_1, scon));
        ...
        // As requested, set our gid, supplemental gids, uid, context, and
        // priority. Aborts on failure.
        SetProcessAttributes();

        if (!ExpandArgsAndExecve(args_)) {
            PLOG(ERROR) << "cannot execve('" << args_[0] << "')";
        }
```
根据上下文字符串创建文件描述符，将信息绑定，然后拓展参数并执行
**system/core/init/descriptors.cpp**
```cpp
void DescriptorInfo::CreateAndPublish(const std::string& globalContext) const {
  // Create
  const std::string& contextStr = context_.empty() ? globalContext : context_;
  int fd = Create(contextStr);
  if (fd < 0) return;

  // Publish
  std::string publishedName = key() + name_;
  std::for_each(publishedName.begin(), publishedName.end(),
                [] (char& c) { c = isalnum(c) ? c : '_'; });

  std::string val = std::to_string(fd);
  add_environment(publishedName.c_str(), val.c_str());

  // make sure we don't close on exec
  fcntl(fd, F_SETFD, 0);
}
```
**system/core/init/service.cpp**
```cpp
static bool ExpandArgsAndExecve(const std::vector<std::string>& args) {
    std::vector<std::string> expanded_args;
    std::vector<char*> c_strings;

    expanded_args.resize(args.size());
    c_strings.push_back(const_cast<char*>(args[0].data()));
    for (std::size_t i = 1; i < args.size(); ++i) {
        if (!expand_props(args[i], &expanded_args[i])) {
            LOG(FATAL) << args[0] << ": cannot expand '" << args[i] << "'";
        }
        c_strings.push_back(expanded_args[i].data());
    }
    c_strings.push_back(nullptr);

    return execve(c_strings[0], c_strings.data(), (char**)ENV) == 0;
}
```

#### zygote 进程的启动过程  
首先同样是创建一个 AppRuntime 对象 runtime。
```cpp
AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
```
随后检查启动参数 arg 是够是否包含 ``--zygote`` 选项，如果包含，则说明应用程序 ```app_process`` 是在 Zygote 进程中启动的

然后会进行操作确定当前进程名称为 ``zygote``
```cpp
// Parse runtime arguments.  Stop at first unrecognized option.
bool zygote = false;
bool startSystemServer = false;
String8 niceName;
++i;  // Skip unused "parent dir" argument.
while (i < argc) {
    const char* arg = argv[i++];
    if (strcmp(arg, "--zygote") == 0) {
        zygote = true;
        niceName = ZYGOTE_NICE_NAME;
    } else if (strcmp(arg, "--start-system-server") == 0) {
        startSystemServer = true;
```

紧接着开始执行 AppRuntime 类的成员函数 start() 
```cpp
 if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
```
开始分析 ``AppRuntime::start()`` 函数，首先调用成员函数``startVm()``
```cpp
if (startVm(&mJavaVM, &env, zygote) != 0) {
    return;
}
```
随后开始判断并注册一系列 ``JNI`` 方法
```cpp
if (startReg(env) < 0) {
    ALOGE("Unable to register all android natives\n");
    return;
}
```
最后调用 ``ZygoteInit`` 类的静态成员函数 ``main``
来进一步启动 ``Zygote`` 进程
```cpp
env->CallStaticVoidMethod(startClass, startMeth, strArray);
```
继续分析静态成员函数的实现，首先创建一个 ``ZygoteServer`` 实例，然后创建一个 ``Server`` 端的 ``Socket``，这个 ``Socket`` 是用来等待 ``AcitivityManagerService`` 请求 ``Zygote`` 进程创建新的应用进程的,如果 ``startSystemServer`` 为 ``true``，则调用函数 ``forkSystemServer()`` 启动系统服务进程，如果请求创建的不是 ``Zygote`` 进程，则执行 ``run()`` 进行创建新的进程，随后执行 ``runSelectLoop()`` 等待 ``ActivityManagerService``  请求创建新的应用程序进程
```java
ZygoteServer zygoteServer = new ZygoteServer();

zygoteServer.registerServerSocket(socketName);


if (startSystemServer) {
    Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
    // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
    // child (system_server) process.
    if (r != null) {
        r.run();
        return;
    }
}

// The select loop returns early in the child process after a fork and
// loops forever in the zygote.
caller = zygoteServer.runSelectLoop(abiList);

```
分别分析 ``ZygoteInit`` 类的静态成员函数
首先获取 ``zygote`` 环境变量的值，并转换为文件描述符，随后根据文件描述符创建 ``Server`` 端的 ``Socket`` 
**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java**
```java
void registerServerSocket(String socketName) {
    if (mServerSocket == null) {
        int fileDesc;
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
        try {
            String env = System.getenv(fullSocketName);
            fileDesc = Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException(fullSocketName + " unset or invalid", ex);
        }

        try {
            FileDescriptor fd = new FileDescriptor();
            fd.setInt$(fileDesc);
            mServerSocket = new LocalServerSocket(fd);
        } catch (IOException ex) {
            throw new RuntimeException(
                    "Error binding to local socket '" + fileDesc + "'", ex);
        }
    }
}
```
首先创建一个字符串数组，保存 ``System`` 进程的启动参数，接着调用 ``Zygote`` 类的静态成员函数 ``forkSystemServer`` 创建子进程。可以看出，``System`` 进程的用户 ID 和用户组 ID 均被设置为 1000，而且还具有 1001-1010、1018··· 权限
Zygote 类的 ``forkSystemServer`` 函数通过 jni 调用，最终使用 ``fork`` 函数创建子进程，随后使用 ``handleSystemServerProcess`` 处理 ``System`` 进程
**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java**
```java
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
     String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
    try {
        pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        }
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            zygoteServer.closeServerSocket();
            return handleSystemServerProcess(parsedArgs);
        }
```
**frameworks/base/core/java/com/android/internal/os/Zygote.java**
```java
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    VM_HOOKS.preFork();
    // Resets nice priority for zygote process.
    resetNicePriority();
    int pid = nativeForkSystemServer(
            uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
    // Enable tracing as soon as we enter the system_server.
    if (pid == 0) {
        Trace.setTracingEnabled(true, debugFlags);
    }
    VM_HOOKS.postForkCommon();
    return pid;
}

native private static int nativeForkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities);
```
**frameworks/base/core/jni/com_android_internal_os_Zygote.cpp**
```cpp
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                      NULL, NULL, NULL);
    return pid;
}

// Utility routine to fork zygote and specialize the child process.
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jintArray fdsToIgnore,
                                     jstring instructionSet, jstring dataDir) {
    pid_t pid = fork();
    ...

    return pid;
```
调用 ``runSelectLoop`` 等待 ``ActivityManagerService`` 请求创建新进程。首先创建文件描述符数组，然后将 Zygote Socket 文件描述符添加到列表中，此后开始循环等待请求并处理
**frameworks/base/core/java/com/android/internal/os/ZygoteServer.java**
```java
Runnable runSelectLoop(String abiList) {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

    fds.add(mServerSocket.getFileDescriptor());
    peers.add(null);

    while (true) {
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        for (int i = 0; i < pollFds.length; ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
        }
        for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }

                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    try {
                        ZygoteConnection connection = peers.get(i);
                        final Runnable command = connection.processOneCommand(this);
```
## System 进程启动
``System`` 进程是在 ``ZygoteInit`` 类中的静态成员函数 ``handleSystemServerProcess``中开始启动的，逐步跟进，首先使用 ``RuntimeInit.commonInit`` 设置时区、代理等基础信息，随后使用``nativeZygoteInit`` 函数在 ``System`` 进程中创建 ``Binder`` 代理池
**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java**
```java
private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
    if (parsedArgs.invokeWith != null) {
        ...
    } else {
        ClassLoader cl = null;

        return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }



public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
    if (RuntimeInit.DEBUG) {
        Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
    RuntimeInit.redirectLogStreams();

    RuntimeInit.commonInit();
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}
```
通过 ``RuntimeInit.applicationInit`` 函数找到 ``System Server`` 的函数主入口点，并进行调用
**frameworks/base/core/java/com/android/internal/os/RuntimeInit.java**
```java
protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
        ClassLoader classLoader) {
    // If the application calls System.exit(), terminate the process
    // immediately without running any shutdown hooks.  It is not possible to
    // shutdown an Android application gracefully.  Among other things, the
    // Android runtime shutdown hooks close the Binder driver, which can cause
    // leftover running threads to crash before the process actually exits.
    nativeSetExitWithoutCleanup(true);

    // We want to be fairly aggressive about heap utilization, to avoid
    // holding on to a lot of memory that isn't needed.
    VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

    final Arguments args = new Arguments(argv);

    // The end of of the RuntimeInit event (see #zygoteInit).
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    // Remaining arguments are passed to the start class's static main
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```
``System Server`` 首先会判断系统是否引导完成，然后对时区语言等配置进行检查，然后进行系统服务启动的引导，成功后启动各种服务程序，最后进行无限循环等待
**frameworks/base/services/java/com/android/server/SystemServer.java**
```java
public static void main(String[] args) {
    new SystemServer().run();
}

public SystemServer() {
    // Check for factory test mode.
    mFactoryTestMode = FactoryTest.getMode();
    // Remember if it's runtime restart(when sys.boot_completed is already set) or reboot
    mRuntimeRestart = "1".equals(SystemProperties.get("sys.boot_completed"));
}

private void run() {
    try {
        traceBeginAndSlog("InitBeforeStartServices");
        ...
     // Start services.
    try {
        traceBeginAndSlog("StartServices");
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
        SystemServerInitThreadPool.shutdown();
        ...

    // Loop forever.
    Looper.loop();
```

在启动的核心服务中，我们暂时主要关注 ``ActivityManagerService``,``ContentService``,``PackageManagerService``,``WindowManagerService``
**ActivityManagerService**
```java
## frameworks/base/services/java/com/android/server/SystemServer.java
## 启动 ActivityManagerService 并获取实例
mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();

## frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        ...
        startService(service);
        return service;
    }

public void startService(@NonNull final SystemService service) {
        // Register it.
        mServices.add(service);

## frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
## 实例创建后会创建各种线程处理函数
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context);
    }

    @Override
    public void onStart() {
        mService.start();
    }

    @Override
    public void onCleanupUser(int userId) {
        mService.mBatteryStatsService.onCleanupUser(userId);
    }

    public ActivityManagerService getService() {
        return mService;
    }
}

public ActivityManagerService(Context systemContext) {
    ...
    mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    mHandlerThread.start();
    mHandler = new MainHandler(mHandlerThread.getLooper());
    mUiHandler = mInjector.getUiHandler(this);
    ...
```
**PackageManagerService**
```java
## frameworks/base/services/java/com/android/server/SystemServer.java
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();

## frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();

    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    m.enableSystemUserPackages();
    ServiceManager.addService("package", m);
    final PackageManagerNative pmn = m.new PackageManagerNative();
    ServiceManager.addService("package_native", pmn);
    return m;
}
```
**ContentService**
```java
## frameworks/base/services/java/com/android/server/SystemServer.java
private static final String CONTENT_SERVICE_CLASS =
        "com.android.server.content.ContentService$Lifecycle";
...
traceBeginAndSlog("StartContentService");
mSystemServiceManager.startService(CONTENT_SERVICE_CLASS);

## frameworks/base/services/core/java/com/android/server/content/ContentService.java
public static class Lifecycle extends SystemService {
    private ContentService mService;

    public Lifecycle(Context context) {
        super(context);
    }

    @Override
    public void onStart() {
        final boolean factoryTest = (FactoryTest
                .getMode() == FactoryTest.FACTORY_TEST_LOW_LEVEL);
        mService = new ContentService(getContext(), factoryTest);
        publishBinderService(ContentResolver.CONTENT_SERVICE_NAME, mService);
    }

## frameworks/base/services/core/java/com/android/server/SystemService.java
/**
 * Publish the service so it is accessible to other services and apps.
 */
protected final void publishBinderService(String name, IBinder service) {
    publishBinderService(name, service, false);
}

/**
 * Publish the service so it is accessible to other services and apps.
 */
protected final void publishBinderService(String name, IBinder service,
        boolean allowIsolated) {
    ServiceManager.addService(name, service, allowIsolated);
}
```
**WindowManagerService**
```java
## frameworks/base/services/java/com/android/server/SystemServer.java
wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore, new PhoneWindowManager());
ServiceManager.addService(Context.WINDOW_SERVICE, wm);
ServiceManager.addService(Context.INPUT_SERVICE, inputManager);

## frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public static WindowManagerService main(final Context context, final InputManagerService im,
        final boolean haveInputMethods, final boolean showBootMsgs, final boolean onlyCore,
        WindowManagerPolicy policy) {
    DisplayThread.getHandler().runWithScissors(() ->
            sInstance = new WindowManagerService(context, im, haveInputMethods, showBootMsgs,
                    onlyCore, policy), 0);
    return sInstance;
}
```
### 参考资料
[Android 源码在线查看](https://cs.android.com/)  
Android 软件安全权威指南  
Android 系统源代码情景分析  
r0ysue、feicong大佬的星球  
https://blog.csdn.net/nanyou519/article/details/104953960  
https://mp.weixin.qq.com/s/krFFzBlJFVKhirgDhF55Rw  
https://mp.weixin.qq.com/s/AwksinkWU099bLap01sZeA  
