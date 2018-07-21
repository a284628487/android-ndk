[SourceCode](https://github.com/a284628487/android-ndk/tree/master/hello-jni)

Android Studio 3.0 提供了新的NDK支持，使用[CMake Android plugin](https://developer.android.com/studio/projects/add-native-code.html)插件来编译。

## build.gradle

app的`build.gradle`文件，添加NDK支持

```
android {
    
    compileSdkVersion 26
    
    defaultConfig {
        ...
        externalNativeBuild {
            cmake {
                cppFlags ""
            }
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    externalNativeBuild {
        cmake {
            path "CMakeLists.txt" // 指定编译文件的地址，CMakeLists.txt是固定文件名。
        }
    }

    // 指定CPU平台
    flavorDimensions 'cpuArch'

    productFlavors {
        arm7 {
            dimension 'cpuArch'
            ndk {
                abiFilter 'armeabi-v7a'
            }
        }
        arm8 {
            dimension 'cpuArch'
            ndk {
                abiFilters 'arm64-v8a'
            }
        }
        x86 {
            dimension 'cpuArch'
            ndk {
                abiFilter 'x86'
            }
        }
        x86_64 {
            dimension 'cpuArch'
            ndk {
                abiFilter 'x86_64'
            }
        }
        universal {
            dimension 'cpuArch'
            // include all default ABIs. with NDK-r16,  it is:
            // armeabi-v7a, arm64-v8a, x86, x86_64
        }
    }    
}
```

## CMakeLists.txt

```
# 指定版本
cmake_minimum_required(VERSION 3.4.1)

# 通过提供源文件(可能为多个)创建并命令一个库，可以设置为 STATIC 或者 SHARED。
# 可以配置多个库文件，CMake会自动编译它们。
add_library( hello-jni # The name of the library.
             # 设置库为共享库
             SHARED
             # Provides a relative path to your source file(s).
             src/main/cpp/hello-jni.cpp )

# 查找一个系统库，并提供一个别名。
find_library( # 指定别名
              log-lib
              # 指定要映射的NDK库名称
              log )

# 指定需要链接的库，可以指定多个库文件，包括自定义的，第三方的，或者是系统的
target_link_libraries( # 指定目标输出库名称
                       hello-jni
                       # 指定依赖链接库
                       ${log-lib} )
```

## hello-jni.c

```C
#include <string.h>
#include <jni.h>

/* This is a trivial JNI example where we use a native method
 * to return a new VM String. See the corresponding Java source
 * file located at:
 *
 *   hello-jni/app/src/main/java/com/example/hellojni/HelloJni.java
 */
JNIEXPORT jstring JNICALL
Java_com_example_hellojni_HelloJni_stringFromJNI( JNIEnv* env,
                                                  jobject thiz )
{
#if defined(__arm__)
    #if defined(__ARM_ARCH_7A__)
    #if defined(__ARM_NEON__)
      #if defined(__ARM_PCS_VFP)
        #define ABI "armeabi-v7a/NEON (hard-float)"
      #else
        #define ABI "armeabi-v7a/NEON"
      #endif
    #else
      #if defined(__ARM_PCS_VFP)
        #define ABI "armeabi-v7a (hard-float)"
      #else
        #define ABI "armeabi-v7a"
      #endif
    #endif
  #else
   #define ABI "armeabi"
  #endif
#elif defined(__i386__)
#define ABI "x86"
#elif defined(__x86_64__)
#define ABI "x86_64"
#elif defined(__mips64)  /* mips64el-* toolchain defines __mips__ too */
#define ABI "mips64"
#elif defined(__mips__)
#define ABI "mips"
#elif defined(__aarch64__)
#define ABI "arm64-v8a"
#else
#define ABI "unknown"
#endif

    return (*env)->NewStringUTF(env, "Hello from JNI !  Compiled with ABI " ABI ".");
}
```
