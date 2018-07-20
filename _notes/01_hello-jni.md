Android Studio 3.0 提供了新的NDK支持，使用CMake来编译。

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
add_library( native-lib # The name of the library.
             # 设置库为共享库
             SHARED
             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )

# 查找一个系统库，并提供一个别名。
find_library( # 指定别名
              log-lib
              # 指定要映射的NDK库名称
              log )

# 指定需要链接的库，可以指定多个库文件，包括自定义的，第三方的，或者是系统的
target_link_libraries( # 指定目标输出库名称
                       native-lib
                       # 指定依赖链接库
                       ${log-lib} )
```

## native-lib.cpp

```C++
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring

JNICALL
Java_com_example_ccfyyn_prov3_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

[Link](https://developer.android.com/studio/projects/add-native-code)
