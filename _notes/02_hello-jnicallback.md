[SourceCode](https://github.com/a284628487/android-ndk/tree/master/hello-jniCallback)

## hello-jnicallback.c

```C
#include <string.h>
#include <inttypes.h>
#include <pthread.h>
#include <jni.h>
#include <android/log.h>
#include <assert.h>


// Android log function wrappers
static const char *kTAG = "hello-jniCallback";
#define LOGI(...) \
  ((void)__android_log_print(ANDROID_LOG_INFO, kTAG, __VA_ARGS__))
#define LOGW(...) \
  ((void)__android_log_print(ANDROID_LOG_WARN, kTAG, __VA_ARGS__))
#define LOGE(...) \
  ((void)__android_log_print(ANDROID_LOG_ERROR, kTAG, __VA_ARGS__))

// processing callback to handler class
typedef struct tick_context {
    JavaVM *javaVM;
    jclass jniHelperClz;
    jobject jniHelperObj;
    jclass mainActivityClz;
    jobject mainActivityObj;
    pthread_mutex_t lock;
    int done;
} TickContext;
TickContext g_ctx;

/*
 *  调用 java 静态方法 JniHelper::getBuildVersion()
 *  调用 java 对象实例方法 JniHelper::getRuntimeMemorySize()
 */
void queryRuntimeInfo(JNIEnv *env, jobject instance) {
    // 获取静态方法getBuildVersion的methodID
    jmethodID versionFunc = (*env)->GetStaticMethodID(
            env, g_ctx.jniHelperClz,
            "getBuildVersion", "()Ljava/lang/String;");
    if (!versionFunc) {
        LOGE("Failed to retrieve getBuildVersion() methodID @ line %d",
             __LINE__);
        return;
    }
    jstring buildVersion = (*env)->CallStaticObjectMethod(env,
                                                          g_ctx.jniHelperClz, versionFunc);
    // 从 jstring 中获取 char
    const char *version = (*env)->GetStringUTFChars(env, buildVersion, NULL);
    if (!version) {
        LOGE("Unable to get version string @ line %d", __LINE__);
        return;
    }
    LOGE("Android Version - %s", version);
    (*env)->ReleaseStringUTFChars(env, buildVersion, version);

    // 删除LocalRef，to avoid leaking
    (*env)->DeleteLocalRef(env, buildVersion);

    // 获取实例方法ID
    jmethodID memFunc = (*env)->GetMethodID(env, g_ctx.jniHelperClz,
                                            "getRuntimeMemorySize", "()J");
    if (!memFunc) {
        LOGE("Failed to retrieve getRuntimeMemorySize() methodID @ line %d",
             __LINE__);
        return;
    }
    jlong result = (*env)->CallLongMethod(env, instance, memFunc);
    LOGE("Runtime free memory size: %" PRId64, result);
    (void) result;  // silence the compiler warning
}

/*
 * 通过System.loadLibrary("")调用时，JNI_OnLoad方法被调用一次，用于初始化操作:
 *      缓存JavaVM对象
 *      查找Java类对象JniHandler，并创建它的一个实例
 *      Make global reference since we are using them from a native thread
 * Note:
 *      所有的申请创建的资源不会被应用自动被释放，JNI_OnUnload()方法永远不会被调用.
 *     All resources allocated here are never released by application
 *     we rely on system to free all global refs when it goes away;
 *     the pairing function JNI_OnUnload() never gets called at all.
 */
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    // 重置g_ctx内存区域
    memset(&g_ctx, 0, sizeof(g_ctx));
    // 保存JavaVM对象
    g_ctx.javaVM = vm;
    // 创建JNIEnv对象
    if ((*vm)->GetEnv(vm, (void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR; // JNI version not supported.
    }

    const char *jniHandler = "com/example/hellojnicallback/JniHandler";
    jclass clz = (*env)->FindClass(env, jniHandler);
    // 查找Java类对象
    g_ctx.jniHelperClz = (*env)->NewGlobalRef(env, clz);
    // 查找Constructor方法ID
    jmethodID jniHelperCtor = (*env)->GetMethodID(env, g_ctx.jniHelperClz, "<init>", "()V");
    // 通过构造方法id创建类实例对象
    jobject handler = (*env)->NewObject(env, g_ctx.jniHelperClz, jniHelperCtor);
    g_ctx.jniHelperObj = (*env)->NewGlobalRef(env, handler);
    // 调用JniHandler的方法
    queryRuntimeInfo(env, g_ctx.jniHelperObj);
    //
    g_ctx.done = 0;
    g_ctx.mainActivityObj = NULL;
    return JNI_VERSION_1_6;
}

/*
 * 调用Java对象的私有方法 JniHandler::updateStatus(String msg)
 */
void sendJavaMsg(JNIEnv *env, jobject instance,
                 jmethodID func, const char *msg) {
    jstring javaMsg = (*env)->NewStringUTF(env, msg);
    (*env)->CallVoidMethod(env, instance, func, javaMsg);
    (*env)->DeleteLocalRef(env, javaMsg);
}

/*
 * working thread，回调调用MainActivity::updateTimer()更新UI
 *  回调调用JniHelper::updateStatus(String msg)发送消息
 */
void *UpdateTicks(void *context) {
    TickContext *pctx = (TickContext *) context;
    JavaVM *javaVM = pctx->javaVM;
    JNIEnv *env;
    // 重新创建一个JNIEnv对象
    jint res = (*javaVM)->GetEnv(javaVM, (void **) &env, JNI_VERSION_1_6);
    if (res != JNI_OK) { // JNI_EDETACHED(-2)
        LOGE("res != JNI_OK = %d", res);
        res = (*javaVM)->AttachCurrentThread(javaVM, &env, NULL);
        if (JNI_OK != res) {
            LOGE("Failed to AttachCurrentThread, ErrorCode = %d", res);
            return NULL;
        }
    }

    // updateStatus方法ID
    jmethodID statusId = (*env)->GetMethodID(env, pctx->jniHelperClz,
                                             "updateStatus",
                                             "(Ljava/lang/String;)V");
    // 发送消息：initializing
    sendJavaMsg(env, pctx->jniHelperObj, statusId,
                "TickerThread status: initializing...");

    // 获取MainActivity#updateTimer方法ID
    jmethodID timerId = (*env)->GetMethodID(env, pctx->mainActivityClz,
                                            "updateTimer", "()V");

    struct timeval beginTime, curTime, usedTime, leftTime;
    // 常量1s钟
    const struct timeval kOneSecond = {
            (__kernel_time_t) 1,
            (__kernel_suseconds_t) 0
    };
    /**
     * struct timeval {
     *   __kernel_time_t tv_sec; // 秒数
     *   __kernel_suseconds_t tv_usec; // 微秒数
     *  };
     */
    // 发送消息：start ticking
    sendJavaMsg(env, pctx->jniHelperObj, statusId,
                "TickerThread status: start ticking ...");
    while (1) {
        // 获取时间？
        gettimeofday(&beginTime, NULL);
        // lock
        pthread_mutex_lock(&pctx->lock);
        int done = pctx->done;
        if (pctx->done) {
            pctx->done = 0;
        }
        // unlock
        pthread_mutex_unlock(&pctx->lock);
        if (done) {
            break;
        }
        // 调用MainActivity#updateTimer
        (*env)->CallVoidMethod(env, pctx->mainActivityObj, timerId);


        /*
        void timeradd(struct timeval *a, struct timeval *b,
                      struct timeval *res); // res = a + b
        void timersub(struct timeval *a, struct timeval *b,
                      struct timeval *res); // res = a - b
        */

        gettimeofday(&curTime, NULL);
        timersub(&curTime, &beginTime, &usedTime);
        timersub(&kOneSecond, &usedTime, &leftTime);
        /*
        struct timespec
        {
            __time_t tv_sec; // Seconds.
            long   tv_nsec; // Nanoseconds(纳秒).
        };
        */
        struct timespec sleepTime;
        sleepTime.tv_sec = leftTime.tv_sec;
        sleepTime.tv_nsec = leftTime.tv_usec * 1000;

        if (sleepTime.tv_sec <= 1) {
            nanosleep(&sleepTime, NULL);
        } else {
            // 发送消息
            sendJavaMsg(env, pctx->jniHelperObj, statusId,
                        "TickerThread error: processing too long!");
        }
    }
    // 发送消息：ticking stopped
    sendJavaMsg(env, pctx->jniHelperObj, statusId,
                "TickerThread status: ticking stopped");
    // DetachThread
    (*javaVM)->DetachCurrentThread(javaVM);
    return context;
}

JNIEXPORT void JNICALL
Java_com_example_hellojnicallback_MainActivity_startTicks(JNIEnv *env, jobject instance) {
    pthread_t threadInfo_;
    pthread_attr_t threadAttr_;

    pthread_attr_init(&threadAttr_);
    pthread_attr_setdetachstate(&threadAttr_, PTHREAD_CREATE_DETACHED);
    // 初始化lock对象。
    pthread_mutex_init(&g_ctx.lock, NULL);

    // 保存MainActivity类信息和实例。
    jclass clz = (*env)->GetObjectClass(env, instance);
    g_ctx.mainActivityClz = (*env)->NewGlobalRef(env, clz);
    g_ctx.mainActivityObj = (*env)->NewGlobalRef(env, instance);
    // 创建线程
    int result = pthread_create(&threadInfo_, &threadAttr_, UpdateTicks, &g_ctx);
    assert(result == 0);
    //
    pthread_attr_destroy(&threadAttr_);
    LOGE("startTicks");
    (void) result;
}

JNIEXPORT void JNICALL
Java_com_example_hellojnicallback_MainActivity_StopTicks(JNIEnv *env, jobject instance) {
    pthread_mutex_lock(&g_ctx.lock);
    g_ctx.done = 1;
    pthread_mutex_unlock(&g_ctx.lock);
    // 设置done为1，在上面的while循环中，如果done为1，则终止while循环。

    // waiting for ticking thread to flip the done flag
    struct timespec sleepTime;
    memset(&sleepTime, 0, sizeof(sleepTime));
    // 1毫秒(ms)=1000微秒(μs)
    // 1毫秒(ms)=1000000纳秒(ns)
    sleepTime.tv_nsec = 100000000; // 100ms.
    while (g_ctx.done) {
        nanosleep(&sleepTime, NULL);
    }

    // release object we allocated from StartTicks() function
    (*env)->DeleteGlobalRef(env, g_ctx.mainActivityClz);
    (*env)->DeleteGlobalRef(env, g_ctx.mainActivityObj);
    g_ctx.mainActivityObj = NULL;
    g_ctx.mainActivityClz = NULL;
    // destroy线程对象.
    pthread_mutex_destroy(&g_ctx.lock);
}
```

## 代码分析

### JNI_OnLoad
通过System.loadLibrary("")调用时，JNI_OnLoad方法被调用一次，可以用于初始化操作，可以使用GetEnv，创建JNIEnv对象。
```C
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    if ((*vm)->GetEnv(vm, (void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR; // JNI version not supported.
    }
    //
    return JNI_VERSION_1_6;
}
```

### time struct
该例子中涉及到了几个时间相关的struct及函数

1. struct：

```C
struct timeval {
   __kernel_time_t tv_sec; // 秒(s)
   __kernel_suseconds_t tv_usec; // 微秒(μs)
};

struct timespec {
    __time_t tv_sec; // 秒(s)
    long   tv_nsec; // 纳秒(Nanoseconds/ns).
};
```

创建一个timeval结构体
```C
const struct timeval kOneSecond = {
        (__kernel_time_t) 1,
        (__kernel_suseconds_t) 0
};
```

2. 时间转换关系：
```
1毫秒(ms)=1000微秒(μs)
1毫秒(ms)=1000000纳秒(ns)
```

3. 时间操作函数：
```C
void timeradd(struct timeval *a, struct timeval *b, struct timeval *res); // res = a + b
void timersub(struct timeval *a, struct timeval *b, struct timeval *res); // res = a - b
```

## 线程操作

### 启动线程
```C
void startThread() {
    pthread_t threadInfo_;
    pthread_attr_t threadAttr_;
    // 初始化threadAttr_对象，设置状态
    pthread_attr_init(&threadAttr_);
    pthread_attr_setdetachstate(&threadAttr_, PTHREAD_CREATE_DETACHED);
    // 初始化lock对象
    pthread_mutex_init(&g_ctx.lock, NULL);
    // pthread_mutex_init(pthread_mutex_t*, const pthread_mutexattr_t*) __nonnull((1));
    // 创建线程: 后两个参数表示线程执行函数，以及传递给函数的参数对象。
    int result = pthread_create(&threadInfo_, &threadAttr_, UpdateTicks, &g_ctx);
    assert(result == 0);
    // 销毁threadAttr_
    pthread_attr_destroy(&threadAttr_);
    (void) result;
}
```

### 执行线程
```C
void *UpdateTicks(void *context) {
    TickContext *pctx = (TickContext *) context;
    JavaVM *javaVM = pctx->javaVM;
    JNIEnv *env;
    // 重新创建一个JNIEnv对象
    jint res = (*javaVM)->GetEnv(javaVM, (void **) &env, JNI_VERSION_1_6);
    if (res != JNI_OK) { // JNI_EDETACHED(-2)
        res = (*javaVM)->AttachCurrentThread(javaVM, &env, NULL);
        if (JNI_OK != res) {
            LOGE("Failed to AttachCurrentThread, ErrorCode = %d", res);
            return NULL;
        }
    }
    // 循环执行任务操作，直到done为1，表示任务完成
    while (1) {
        // lock
        pthread_mutex_lock(&pctx->lock);
        int done = pctx->done;
        if (pctx->done) {
            pctx->done = 0;
        }
        pthread_mutex_unlock(&pctx->lock);
        // unlock
        if (done) {
            break;
        }
        // work...
    }
}
```

### 停止线程
```C
void stopThread() {
    // 设置done为1，在上面的while循环中，如果done为1，则终止while循环。
    pthread_mutex_lock(&g_ctx.lock);
    g_ctx.done = 1;
    pthread_mutex_unlock(&g_ctx.lock);
    // destroy线程对象.
    pthread_mutex_destroy(&g_ctx.lock);
}
```
