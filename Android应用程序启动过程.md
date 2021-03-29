
### 应用程序进程创建过程
当 ``ActivityManagerService`` 需要创建一个新的应用程序进程来启动一个应用程序组件时，需要向 ``Zygote`` 发送创建请求  
**frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java**
```java
private final void startProcessLocked(ProcessRecord app,
            String hostingType, String hostingNameStr) {
    startProcessLocked(app, hostingType, hostingNameStr, null /* abiOverride */,
                null /* entryPoint */, null /* entryPointArgs */);
}

private final void startProcessLocked(ProcessRecord app, String hostingType,
        String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    try {
        int uid = app.uid;
        int[] gids = null;
        int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
        ...
        // 获取 uid, gid
        boolean isActivityProcess = (entryPoint == null);
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        checkTime(startTime, "startProcess: asking zygote to start proc");
        ProcessStartResult startResult;
        // 判断是否为 Webview 并创建进程
        if (hostingType.equals("webview_service")) {
            startResult = startWebView(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, null, entryPointArgs);
        } else {
            startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, invokeWith, entryPointArgs);
        }
    
```
**frameworks/base/core/java/android/os/Process.java**
```java
public static final ProcessStartResult start(final String processClass,
                                final String niceName,
                                int uid, int gid, int[] gids,
                                int debugFlags, int mountExternal,
                                int targetSdkVersion,
                                String seInfo,
                                String abi,
                                String instructionSet,
                                String appDataDir,
                                String invokeWith,
                                String[] zygoteArgs) {
    return zygoteProcess.start(processClass, niceName, uid, gid, gids,
                debugFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
}
```
**frameworks/base/core/java/android/os/ZygoteProcess.java**
```java
public final Process.ProcessStartResult start(final String processClass,
                                                final String niceName,
                                                int uid, int gid, int[] gids,
                                                int debugFlags, int mountExternal,
                                                int targetSdkVersion,
                                                String seInfo,
                                                String abi,
                                                String instructionSet,
                                                String appDataDir,
                                                String invokeWith,
                                                String[] zygoteArgs) {
    try {
        return startViaZygote(processClass, niceName, uid, gid, gids,
                debugFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
    } catch (ZygoteStartFailedEx ex) {
        Log.e(LOG_TAG,
                "Starting VM process through Zygote failed");
        throw new RuntimeException(
                "Starting VM process through Zygote failed", ex);
    }
}

//通过 Zygote 启动
private Process.ProcessStartResult startViaZygote(final String processClass,
                                                    final String niceName,
                                                    final int uid, final int gid,
                                                    final int[] gids,
                                                    int debugFlags, int mountExternal,
                                                    int targetSdkVersion,
                                                    String seInfo,
                                                    String abi,
                                                    String instructionSet,
                                                    String appDataDir,
                                                    String invokeWith,
                                                    String[] extraArgs)
                                                    throws ZygoteStartFailedEx {
    ArrayList<String> argsForZygote = new ArrayList<String>();
    ...
    //添加参数
    synchronized(mLock) {
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
    }

// 输入参数，返回带有 pid 值的结果
private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, ArrayList<String> args)
        throws ZygoteStartFailedEx {
    ...
    // 参数设置
    final DataInputStream inputStream = zygoteState.inputStream;
    Process.ProcessStartResult result = new Process.ProcessStartResult();
    result.pid = inputStream.readInt();
    result.usingWrapper = inputStream.readBoolean();
    return result;
}
```
当 ``ZygoteServer`` 接收到请求时  
**frameworks/base/core/java/com/android/internal/os/ZygoteServer.java**
```java
Runnable runSelectLoop(String abiList) {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

    // 首先会添加文件描述符列表
    fds.add(mServerSocket.getFileDescriptor());
    peers.add(null);

    //随后循环处理链接信息
    while (true) {
        ...
        for(){
            if (i == 0) {
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                try {
                    ZygoteConnection connection = peers.get(i);
                    final Runnable command = connection.processOneCommand(this);
                }
        }
```
**frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java**
```java
Runnable processOneCommand(ZygoteServer zygoteServer) {
    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;

    try {
        //读取参数
        args = readArgumentList();
        descriptors = mSocket.getAncillaryFileDescriptors();
    } 
    //各种预处理
    ...

    pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
            parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
            parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.instructionSet,
            parsedArgs.appDataDir);

    try {
        if (pid == 0) {
            // in child
            zygoteServer.setForkChild();

            zygoteServer.closeServerSocket();
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;

            return handleChildProc(parsedArgs, descriptors, childPipeFd);
        }
    } 

private Runnable handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors,
        FileDescriptor pipeFd) {
    if (parsedArgs.invokeWith != null) {
        ...
    } else {
        return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs,
                    null /* classLoader */);
    }
```
剩下的处理部分请参照 ``Zygote & System`` 启动过程文中 ``System`` 启动部分

### Binder 线程池启动过程
在一个新的应用程序进程创建完成之后，会调用 ``nativeZygoteInit`` 启动线程池  
**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java**
```java
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

    private static final native void nativeZygoteInit();
```
**frameworks/base/core/jni/AndroidRuntime.cpp**
```cpp
int register_com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env)
{
    const JNINativeMethod methods[] = {
        { "nativeZygoteInit", "()V",
            (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
    };
    return jniRegisterNativeMethods(env, "com/android/internal/os/ZygoteInit",
        methods, NELEM(methods));
}

static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}

// gCurRuntime 在构造函数中初始化
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength)
{
    SkGraphics::Init();

    // Pre-allocate enough space to hold a fair number of options.
    mOptions.setCapacity(20);

    assert(gCurRuntime == NULL);        // one per process
    gCurRuntime = this;
}
```
**frameworks/base/cmds/app_process/app_main.cpp**
```cpp
class AppRuntime : public AndroidRuntime
{
    virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
```
**frameworks/native/libs/binder/ProcessState.cpp**
```cpp
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}

void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}

```

### 消息循环的创建机制
当应用进程创建完成后，会调用 ``findStaticMain`` 函数将新创建的应用程序函数入口点加入，加入过程中，会创建消息循环  
**frameworks/base/core/java/com/android/internal/os/RuntimeInit.java**
```java
private static Runnable findStaticMain(String className, String[] argv,
        ClassLoader classLoader) {
    Class<?> cl;
    cl = Class.forName(className, true, classLoader);
    m = cl.getMethod("main", new Class[] { String[].class });


    return new MethodAndArgsCaller(m, argv);


static class MethodAndArgsCaller implements Runnable {
    /** method to call */
    private final Method mMethod;

    /** argument array */
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }

    public void run() {
        try {
            mMethod.invoke(null, new Object[] { mArgs });
        }
    }
}
```
``mMethod`` 调用了 ``ActivityThread`` 类的静态成员函数 ``main``
**frameworks/base/core/java/android/app/ActivityThread.java**
```java
public static void main(String[] args) {
    ...
    //在当前进程中创建一个消息循环
    Looper.prepareMainLooper();

    //创建一个 Activity Thread 实例
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    ...
    //进入到前面创建的消息循环中
    Looper.loop();
```
在 ``Android`` 系统中，每当一个应用程序进程启动后，都会自动进入到消息循环中，这样应用程序组件就可以方便的使用消息处理机制实现自己的业务逻辑
