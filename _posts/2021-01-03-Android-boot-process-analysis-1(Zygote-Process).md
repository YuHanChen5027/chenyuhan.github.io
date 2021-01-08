---
layout: post
title:  "Android开机启动流程分析一(Zygote进程)"
author: "陈宇瀚"
date:   2021-01-03 17:13:02 +0800
categories: article
tags:
  - Android 
  - Zygote
  - 开机启动流程
---
&ensp;&ensp; 该解析从**system/core/init/init.cpp**初始化一系列文件系统，解析**system/core/rootdir/init.rc**文件启动**Zygote**进程分析至**Zygote**进程启动完成。

&ensp;&ensp; 当我们驱动层启动完毕时首先会启动**init进程**，也就是**用户进程**。在**system/core/init/init.cpp**中会对**init.rc**文件进行解析，之后会孵化出一些进程和一些重要服务，最后会创建一个**Zygote**进程。这个实现过程的源码流程如下：

&ensp;&ensp; *以下源码基于rk3399_industry Android7.1.2*
## init.rc
**system/core/init/init.cpp**
```
    ....
    //构造出解析文件用的parser对象
    Parser& parser = Parser::GetInstance();
    //为一些类型的关键字，创建特定的parser
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    //开始解析init.rc文件
    //init.rc文件是在init进程启动后执行的启动脚本，文件中记录着init进程需执行的操作。在Android系统中，使用init.rc和init.{ hardware }.rc两个文件。
    //init.rc文件在Android系统运行过程中用于通用的环境设置与进程相关的定义
    //init.{hardware}.rc（例如，高通有init.qcom.rc，MTK有init.mediatek.rc）用于定义Android在不同平台下的特定进程和环境设置等。
    //此处加载的init.rc文件是system/core/rootdir/init.rc，此处记载放在下方解析
    parser.ParseConfig("/init.rc");
    
    ....
```
**system/core/rootdir/init.rc**
```
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc
....
```
**init.rc**中**zygote**相关的启动配置文件根据**ro.zygote**而定，**ro.zygote**的值可分为**zygote32、zygote64、zygote32_64、zygote64_32**四种，分别代表
- init.zygote32.rc：32位的**zygote**进程，对应的执行程序是**app_process** 
- init.zygote64.rc：64位的**zygote**进程，对应的执行程序是**app_process64**
- init.zygote32_64.rc：启动两个**zygote**进程 (**zygote**和 **zygote_secondary**)，32位的**app_process32** 为主、64位的**app_process64**为辅。
- init.zygote64_32.rc：启动两个**zygote**进程 (**zygote**和 **zygote_secondary**)，64位的**app_process64**为主、32位的**app_process32**为辅。
**ro.zygote**的属性值定义是在 **build/target/product/core_64_bit.mk**
而**core_64_bit.mk**在**device/rockchip/rk3399/product.mk**内调用(根据芯片平台不同，对应的文件便不同，这里以RK3399为例子)，代码如下
**device/rockchip/rk3399/product.mk**
```
PRODUCT_RUNTIMES := runtime_libart_default

$(call inherit-product, device/rockchip/rk3399/device.mk)

$(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk)
```
其中$(SRC_TARGET_DIR)为**build/target**。
**build/target/product/core_64_bit.mk**
```
ifeq ($(strip $(TARGET_BOARD_PLATFORM_PRODUCT)), box)
  PRODUCT_COPY_FILES += system/core/rootdir/init.zygote64_32.box.rc:root/init.zygote64_32.rc
else
  PRODUCT_COPY_FILES += system/core/rootdir/init.zygote64_32.rc:root/init.zygote64_32.rc
endif

# 设置zygote属性以选择64位的主脚本，32位的从脚本，必须在core_minimal.mk中的脚本之前解析这一行
PRODUCT_DEFAULT_PROPERTY_OVERRIDES += ro.zygote=zygote64_32

TARGET_SUPPORTS_32_BIT_APPS := true
TARGET_SUPPORTS_64_BIT_APPS := true
```
&ensp;&ensp; 由于我的源码系统种使用的是**init.zygote64_32.rc**文件，所以分析这个文件：
**system/core/rootdir/init.zygote64_32.rc**
```
#创建一个名为zygote的进程，对应的可执行程序路径为/system/bin/app_process64
# -Xzygote：jvm使用的参数；在AndroidRuntime.cpp类的startVm()函数中调用JNI_CreateJavaVM()时被使用。
# /system/bin：代表虚拟机程序所在目录
# --zygote  指明以ZygoteInit类作为虚拟机执行的入口，如果没有–zygote参数，则需要明确指定需要执行的类名。
# --start-system-server仅在指定–zygote参数时才有效，意思是告知ZygoteInit启动完毕后孵化出第一个进程SystemServe
# --socket-name=zygote:指定zygote所用到的socket名为zygote
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    #  指定zygote服务使用到的名为zygote的socket，类型为stream，读写权限为660，用户为root，用户组为system。
    socket zygote stream 660 root system
    # onrestart 服务重启时执行的命令。
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    # writepid <file…> :fork 时将子进程的 pid 写入给定文件。
    writepid /dev/cpuset/foreground/tasks

#与上面类似
service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary
    class main
    socket zygote_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks
    
#rc文件结构可以简单为
#service <name> <pathname> [ <argument> ]*
#<option>
#<option>
#service 服务名称 服务对应的命令的路径 命令的参数
#选项
#选项
```
## app_process的执行
&ensp;&ensp; 接下来分析**zygote64**的可执行程序**/system/bin/app_process64**文件，**app_process64**由**frameworks/base/cmds/app_process**生成，**app_process64**的执行入口是目录下的**app_main.cpp**中的**main**函数，就从这里开始分析**Zygote**的启动流程：
**rameworks/base/cmds/app_process/app_main.cpp**
```
#if defined(__LP64__)
static const char ABI_LIST_PROPERTY[] = "ro.product.cpu.abilist64";
static const char ZYGOTE_NICE_NAME[] = "zygote64";
#else
static const char ABI_LIST_PROPERTY[] = "ro.product.cpu.abilist32";
static const char ZYGOTE_NICE_NAME[] = "zygote";
#endif

//传入的参数根据上方分析得知有
//{-Xzygote , /system/bin , --zygote , --start-system-server , --socket-name=zygote}
// argc = 5
// argv[] = "-Xzygote /system/bin --zygote --start-system-server --socket-name=zygote";
int main(int argc, char* const argv[])
{
    //下方有AppRuntime和 computeArgBlockSize的代码
    //computeArgBlockSize方法计算并且返回了传入参数的字节数
    //之后会创建有AppRuntime对象runtime
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // 进程命令行参数
    // 忽略 argv[0] -Xzygote
    argc--;
    argv++;

    // '--'开头或者第一个字符非'-'参数开头的所有内容都将进入vm。
    // 虚拟机参数后的第一个参数是“父目录”，即/system/bin，目前还未使用。
    // 在父目录之后，需要一个或多个内部参数:
    // --zygote : zygote 模式启动
    // --start-system-server : 启动系统服务
    // --application : 开始应用(独立，非Zygote)模式。
    // --nice-name : 进程名
    // 对于非Zygote模式开始，这些参数后面将跟着启动类名。所有剩余的参数都传递给这个类的main方法。
    // 对于zygote模式开始，所有剩余参数都传递给zygote main函数。
    // 注意，我们必须复制参数字符串值，因为当我们将nice名称应用到argv0时，我们将重写整个参数块。
    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        //这里过滤后 这部分只有'-'开头的参数
        runtime.addOption(strdup(argv[i]));
    }

    // 解析运行时参数。在第一次未被识别的选项时停止。
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // 跳过未使用的“父目录”参数。
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {//启动zygot模式存在--zygote
            zygote = true;
            niceName = ZYGOTE_NICE_NAME; //将niceName设置为"zygote"或者"zygote64"
        } else if (strcmp(arg, "--start-system-server") == 0) {//启动zygot模式存在--start-system-server
            startSystemServer = true; 
        } else if (strcmp(arg, "--application") == 0) {//启动zygote模式不存在--application
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {//{//启动zygot模式不存在--nice-name
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {//启动zygot模式时不成立
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

  Vector<String8> args;//该参数接下来要传入AndroidRuntime.start函数
    if (!className.isEmpty()) { //启动zygote时未进入
        // 我们没有处于zygote模式，我们需要传递给RuntimeInit的唯一参数是应用程序参数。
        // 其余的参数传递给启动类的main()。在使用进程名覆盖它们之前，对它们进行复制。
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        //处于zygot模式
        maybeCreateDalvikCache();//创建/data/dalvik-cache路径,下方会放源码

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }
        
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        //将所有剩余参数传递给zygote main()方法。
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }   
    //启动zygote模式，此时的args的值为
    //{"start-system-server","--abi-list=ro.product.cpu.abilist64","-Xzygote","/system/bin","--zygote","--start-system-server"}

    if (!niceName.isEmpty()) { //启动zygot模式时不成立
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }
    if (zygote) { //启动zygot模式时成立，调用
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}

//这里计算并且返回了传入参数的字节数，这样的目的可以确保传入的四个参数是在一个连续的内存中。
static size_t computeArgBlockSize(int argc, char* const argv[]) {
    uintptr_t start = reinterpret_cast<uintptr_t>(argv[0]);
    uintptr_t end = reinterpret_cast<uintptr_t>(argv[argc - 1]);
    end += strlen(argv[argc - 1]) + 1;
    return (end - start);
}

//创建了/data/dalvik-cache目录，并修改该路径的用户、用户组和权限信息
static void maybeCreateDalvikCache() {
#if defined(__aarch64__)
    static const char kInstructionSet[] = "arm64";
#elif defined(__x86_64__)
    static const char kInstructionSet[] = "x86_64";
#elif defined(__arm__)
    static const char kInstructionSet[] = "arm";
#elif defined(__i386__)
    static const char kInstructionSet[] = "x86";
#elif defined (__mips__) && !defined(__LP64__)
    static const char kInstructionSet[] = "mips";
#elif defined (__mips__) && defined(__LP64__)
    static const char kInstructionSet[] = "mips64";
#else
#error "Unknown instruction set"
#endif
    const char* androidRoot = getenv("ANDROID_DATA");//这里的androidRoot为"data
    LOG_ALWAYS_FATAL_IF(androidRoot == NULL, "ANDROID_DATA environment variable unset");

    char dalvikCacheDir[PATH_MAX];
    const int numChars = snprintf(dalvikCacheDir, PATH_MAX,
            "%s/dalvik-cache/%s", androidRoot, kInstructionSet);
    LOG_ALWAYS_FATAL_IF((numChars >= PATH_MAX || numChars < 0),
            "Error constructing dalvik cache : %s", strerror(errno));

    //0711 只有属主有读、写、执行权限；而属组用户和其他用户只有执行权限。 
    int result = mkdir(dalvikCacheDir, 0711); //创建/data/dalvik-cache目录
    LOG_ALWAYS_FATAL_IF((result < 0 && errno != EEXIST),
            "Error creating cache dir %s : %s", dalvikCacheDir, strerror(errno));
    
    //更改/data/dalvik-cache目录的用户与用户组为root用户、root用户组
    result = chown(dalvikCacheDir, AID_ROOT, AID_ROOT);
    LOG_ALWAYS_FATAL_IF((result < 0), "Error changing dalvik-cache ownership : %s", strerror(errno));

    result = chmod(dalvikCacheDir, 0711);//更改/data/dalvik-cache路径的权限
    LOG_ALWAYS_FATAL_IF((result < 0),
            "Error changing dalvik-cache permissions : %s", strerror(errno));
}

class AppRuntime : public AndroidRuntime
{
public:
    AppRuntime(char* argBlockStart, const size_t argBlockLength)
        : AndroidRuntime(argBlockStart, argBlockLength)
        , mClass(NULL)
    {
    }
    ....
}
/** frameworks/base/core/jni/AndroidRuntime.cpp **/
//静态全局变量gCurRuntime指向AndroidRuntime对象本身,之后我们若需要获取AndroidRuntime对象，获取gCurRuntime就可以了。
static AndroidRuntime* gCurRuntime = NULL;
//创建AndroidRuntime对象
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength)
{   
    SkGraphics::Init();
    // 预先分配足够的空间来容纳相当数量的选项。
    mOptions.setCapacity(20);
    
    assert(gCurRuntime == NULL);        // one per process
    gCurRuntime = this;
}
```
&ensp;&ensp; 由上面源码分析可得。由于rc内启动**zygote**时带有参数**--zygote**，所以最后会执行到**runtime.start("com.android.internal.os.ZygoteInit", args, true);**函数，接下来分析这个函数；

&ensp;&ensp; **runtime**为**AppRuntime**，而**AppRuntime**继承自**AndroidRuntime**，**AppRuntime**其内部并没有**start**方法，所以这里其实调用的是**AndroidRuntime**的**start**方法，源码如下：
**frameworks/base/core/jni/AndroidRuntime.cpp**
```
/*
 * 启动 Android runtime.这部分涉及到启动虚拟机并在名为“className”的类中调用其
 * “static void main(String[] args)”方法。
 * 向main函数传递两个参数，类名和指定的选项字符串。
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    //此时的options为{"start-system-server","--abi-list=ro.product.cpu.abilist64","-Xzygote",
    //"/system/bin","--zygote","--socket-name=zygote",--start-system-server"}
    static const String8 startSystemServer("start-system-server");

    ....
    const char* rootDir = getenv("ANDROID_ROOT");//rootDir为"/system"目录
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }
    
    /* 启动虚拟机 */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
    /*
     *注册安卓jni函数。
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * 创建一个带有参数的数组来保存类名和选项字符串参数
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    //String数组
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    //即ZygoteInit类名字符串
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    //将类名传入String数组的第0个位置
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    //循环添加输入的选项
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * 启动虚拟机。该线程成为VM的主线程，并且在VM退出之前不会返回。
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");//获取ZygoteInit类的main函数
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            //调用ZygoteInit.main函数
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
            
#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    //释放slashClassName
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}


```
根据源码得出**AndroidRuntime.start**内可分为如下几步：
- 设置了**rootDir**的目录为**/system**；
- 通过**startVm(&mJavaVM, &env, zygote)**启动了虚拟机；
- 调用**startReg(env)**注册**jni**函数；
- 创建字符串数组来保存类和设置选项
- 将生成的字符串数组传入并调用**ZygoteInit.main**方法

### AndroidRuntime.startVm:启动虚拟机
&ensp;&ensp; **startVm**内部定义了虚拟机的一系列参数，下面简单介绍一些虚拟机的参数，大部分省略掉了，太多了。
```
/*
 * 启动Dalvik虚拟机。
 *
 * 传入的各种参数(大多数由系统属性决定)。还有一些由“mOptions” Vector更新。
 *
 * 成功返回0
 */
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
    ....

    bool checkJni = false;
     //JNI检测功能，用于native层调用jni函数时进行常规检测，比较弱字符串格式是否符合要求，资源是否正确释放。
     //该功能一般用于早期系统调试或手机Eng版，对于User版往往不会开启，引用该功能比较消耗系统CPU资源，降低系统性能。
    property_get("dalvik.vm.checkjni", propBuf, "");
    if (strcmp(propBuf, "true") == 0) {
        checkJni = true;
    } else if (strcmp(propBuf, "false") != 0) {
        /* 判断kernel层是否开启 */
        property_get("pa.kernel.android.checkjni", propBuf, "");
        if (propBuf[0] == '1') {
            checkJni = true;
        }
    }
    ALOGD("CheckJNI is %s\n", checkJni ? "ON" : "OFF");
    if (checkJni) {
        /* 开启JNI检测功能 */
        addOption("-Xcheck:jni");
    }
    //dalvik虚拟机启动模式
    property_get("dalvik.vm.execution-mode", propBuf, "");
    if (strcmp(propBuf, "int:portable") == 0) {
        executionMode = kEMIntPortable;
    } else if (strcmp(propBuf, "int:fast") == 0) {
        executionMode = kEMIntFast;
    } else if (strcmp(propBuf, "int:jit") == 0) {
        executionMode = kEMJitCompiler;
    }
    //虚拟机产生的trace文件，主要用于分析系统问题，路径默认为/data/anr/traces.txt
    parseRuntimeOption("dalvik.vm.stack-trace-file", stackTraceFileBuf, "-Xstacktracefile:");
    
    ....

    /*
     * 堆的默认启动和最大大小。根据不同的设备可以做相应的调整
     */
    //dalvik.vm.heapstartsize:应用程序启动后为其分配的初始大小
    parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
    //dalvik.vm.heapsize:单个虚拟机可分配的最大内存
    parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
    //dalvik.vm.heapgrowthlimit:每个应用程序最大可分配内存
    parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitOptsBuf, "-XX:HeapGrowthLimit=");
    //dalvik.vm.heapminfree：堆最小空闲内存
    parseRuntimeOption("dalvik.vm.heapminfree", heapminfreeOptsBuf, "-XX:HeapMinFree=");
    //dalvik.vm.heapmaxfree：堆最大空闲内存
    parseRuntimeOption("dalvik.vm.heapmaxfree", heapmaxfreeOptsBuf, "-XX:HeapMaxFree=");
    //dalvik.vm.heaptargetutilization：堆利用率
    parseRuntimeOption("dalvik.vm.heaptargetutilization",
                       heaptargetutilizationOptsBuf,
                       "-XX:HeapTargetUtilization=");
    //heapminfree、heapmaxfree、heaptargetutilization这三个值的设置则对垃圾回收（GC）的某些性能有影响。

    /*
     * JIT编译器相关配置。JIT：即时编译技术
     */
    ....

    //确保有一个预加载的类文件preloaded-classes(由WritePreloadedClassFile.java生成)。
    //在ZygoteInit类中会预加载工作将其中的classes提前加载到内存，以提高系统性能
    if (!hasFile("/system/etc/preloaded-classes")) {
        ALOGE("Missing preloaded-classes file, /system/etc/preloaded-classes not found: %s\n",
              strerror(errno));
        return -1;
    }
    ....

    //创建虚拟机
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }

    return 0;
}
```
### AndroidRuntime.startReg:注册jni函数
```
/*
 * 向VM注册android jni函数。
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME("RegisterAndroidNatives");
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
    env->PushLocalFrame(200);
    
    //调用register_jni_procs进行jni接口注册,gRegJNI就是通过REG_JNI宏定义的jni函数数组
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);

    return 0;
}

static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
#ifndef NDEBUG
            ALOGD("----------!!! %s failed to load\n", array[i].mName);
#endif
            return -1;
        }
    }
    return 0;
}
//REG_JNI宏定义
#define REG_JNI(name)      { name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
    };
    
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_android_os_SystemClock),
    REG_JNI(register_android_util_EventLog),
    REG_JNI(register_android_util_Log),
    REG_JNI(register_android_util_MemoryIntArray),
    REG_JNI(register_android_util_PathParser),
    REG_JNI(register_android_app_admin_SecurityLog),
    REG_JNI(register_android_content_AssetManager),
    REG_JNI(register_android_content_StringBlock),
    REG_JNI(register_android_content_XmlBlock),
    REG_JNI(register_android_text_AndroidCharacter),
    REG_JNI(register_android_text_StaticLayout),
    REG_JNI(register_android_text_AndroidBidi),
    REG_JNI(register_android_view_InputDevice),
    ....
    }
```
&ensp;&ensp; 到此，虚拟机已经启动，jni函数也已经注册，这里只是简单分析，毕竟完整的分析就有点偏离启动流程了。之后进入**ZygoteInit.main**类的**main**函数；
## ZygoteInit.main
**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java**
```
  private static final String ABI_LIST_ARG = "--abi-list=";

  private static final String SOCKET_NAME_ARG = "--socket-name=";
  
//此时的argv[]为
//{"start-system-server","--abi-list=ro.product.cpu.abilist64","-Xzygote","/system/bin","--zygote","--socket-name=zygote","--start-system-server"}
 public static void main(String argv[]) {
        ....
        try {
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "ZygoteInit");
            //DDMS:DalvikDebugMonitorServer,Dalvik调试监控服务器
            RuntimeInit.enableDdms();//开启DDMS监听，监听DDMS消息
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();//启动性能统计

            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    //获得adb类型，此时为ro.product.cpu.abilist64
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                     //socketName = zygote
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }
            //没有获取到abi
            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }
            //创建名为zygote的的Server端socket
            registerZygoteSocket(socketName);
            preload();// 预加载类和资源
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            SamplingProfilerIntegration.writeZygoteSnapshot();

            //初始化gc
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PostZygoteInitGC");
            gcAndFinalize();
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            // 关闭trace以防止fork出的进程trace错误
            Trace.setTracingEnabled(false);

            // Zygote进程卸载根存储空间，不太明白，希望大神赐教
            Zygote.nativeUnmountStorageOnInit();

            // 停止无多线程模式
            ZygoteHooks.stopZygoteNoThreadCreation();
            if (startSystemServer) {
                startSystemServer(abiList, socketName);//启动system_server
            }

            //进入循环等待
            //等待Activity管理服务ActivityManagerService请求Zygote进程创建新的应用程序进程
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (Throwable ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```
### registerZygoteSocket
&ensp;&ensp; **registerZygoteSocket**:创建名为**zygote**的**Server**端**socket**：用来等待**ActivityManagerService**请求**Zygote**进程创建新的应用程序进程。
```
 private static LocalServerSocket sServerSocket;
 private static final String ANDROID_SOCKET_PREFIX = "ANDROID_SOCKET_";
 private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {   
                //env = ANDROID_SOCKET_zygote
                String env = System.getenv(fullSocketName);
                //将env转成文件描述符
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {   
                //根据fileDesc创建一个Server端socket
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                //保存至ZygoteInit的静态成员变量sServerSocket
                sServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```
&ensp;&ensp; 创建完成后，会进行一些预加载类和资源，如果有需要会调用**startSystemServer**启动**System**进程，然后调用**runSelectLoop**进行循环等待。
### runSelectLoop
&ensp;&ensp; **runSelectLoop**:进入循环等待,等待**Activity**管理服务**ActivityManagerService**(以下简称**AMS**)，请求**Zygote**进程创建新的应用程序进程。
```
    /**
     * 运行Zygote进程的循环等待，当新的连接请求发生时生成新的应用程序进程。
     */
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        //peers是与Zygote进程的Server Socket(mServerSocket)连接的连接列表
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        //将mServerSocket的文件描述符放入fds
        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);
        //无限循环等待AMS请求Zygote进程创建新的应用程序进程
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    //AMS与Zygote进程的mServerSocket建立连接
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    //将这个连接添加到sdocket连接列表peer中
                    peers.add(newPeer);
                    //将该连接的socket文件描述符添加到fds中
                    fds.add(newPeer.getFileDesciptor());
                    //添加完成后就可以接受AMS发送过来的创建应用程序进程请求
                } else {
                     //处理AMS与Zygote发送的请求
                     //peers.get(i)就是之前建立起来的连接(ZygoteConnection)
                     //调用其runOnce()来创建一个新的应用程序进程
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        //处理完成了这个连接，将对应的连接和描述符删除
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```
&ensp;&ensp; **runSelectLoop**发生在启动**System进程**完成之后，根据代码得到**Zygote**进程会进入一个无限循环，不断等待**AMS**的连接和请求创建新的应用程序进程。针对如何通过**ZygoteConnection.runOnce()**来创建一个新的进程过程比较复杂，之后另开一篇文章分析；

整个过程可以用流程图简单概括：
![image](http://note.youdao.com/yws/res/9192/C4DDB06C09A043F7AE1608B1DC369979)


