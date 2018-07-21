[SourceCode](https://github.com/a284628487/android-ndk/tree/master/hello-gl2)

使用`jni`来创建`opengl`场景。流程上和Java上层直接调用相同，基本都是相同名称的api。

## gl_code.cpp

```C
#include <jni.h>
#include <android/log.h>

#include <GLES2/gl2.h>
#include <GLES2/gl2ext.h>

#include <stdio.h>
#include <stdlib.h>
#include <math.h>

#define  LOG_TAG    "libgl2jni"
#define  LOGI(...)  __android_log_print(ANDROID_LOG_INFO,LOG_TAG,__VA_ARGS__)
#define  LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)

static void printGLString(const char *name, GLenum s) {
    const char *v = (const char *) glGetString(s);
    LOGI("GL %s = %s\n", name, v);
}

static void checkGlError(const char *op) {
    for (GLint error = glGetError(); error; error
                                                    = glGetError()) {
        LOGI("after %s() glError (0x%x)\n", op, error);
    }
}

// 顶点着色器
auto gVertexShader =
        "attribute vec4 vPosition;\n"
                "void main() {\n"
                "  gl_Position = vPosition;\n"
                "}\n";

// 片段着色器
auto gFragmentShader =
        "precision mediump float;\n"
                "void main() {\n"
                "  gl_FragColor = vec4(0.0, 1.0, 0.0, 1.0);\n"
                "}\n";

// 加载Shader
GLuint loadShader(GLenum shaderType, const char *pSource) {
    // 创建Shader
    GLuint shader = glCreateShader(shaderType);
    if (shader) {
        // 为Shader指定source
        glShaderSource(shader, 1, &pSource, NULL);
        // 编译
        glCompileShader(shader);
        GLint compiled = 0;
        glGetShaderiv(shader, GL_COMPILE_STATUS, &compiled);
        LOGI("loadShader.compiled = %d", compiled);
        if (!compiled) {
            GLint infoLen = 0;
            glGetShaderiv(shader, GL_INFO_LOG_LENGTH, &infoLen);
            LOGI("loadShader.infoLen = %d", infoLen);
            if (infoLen) {
                char *buf = (char *) malloc(infoLen);
                if (buf) {
                    glGetShaderInfoLog(shader, infoLen, NULL, buf);
                    LOGE("Could not compile shader %d:\n%s\n",
                         shaderType, buf);
                    free(buf);
                }
                glDeleteShader(shader);
                shader = 0;
            }
        }
    }
    return shader;
}

// 创建Program，可在Java层的onSurfaceChanged或者onSurfaceCreated层调用。
GLuint createProgram(const char *pVertexSource, const char *pFragmentSource) {
    GLuint vertexShader = loadShader(GL_VERTEX_SHADER, pVertexSource);
    if (!vertexShader) {
        return 0;
    }

    GLuint pixelShader = loadShader(GL_FRAGMENT_SHADER, pFragmentSource);
    if (!pixelShader) {
        return 0;
    }

    GLuint program = glCreateProgram();
    if (program) {
        glAttachShader(program, vertexShader);
        checkGlError("glAttachShader");
        glAttachShader(program, pixelShader);
        checkGlError("glAttachShader");
        glLinkProgram(program); // 链接program程序。
        GLint linkStatus = GL_FALSE;
        glGetProgramiv(program, GL_LINK_STATUS, &linkStatus); // 获取链接状态。
        if (linkStatus != GL_TRUE) {
            GLint bufLength = 0;
            glGetProgramiv(program, GL_INFO_LOG_LENGTH, &bufLength);
            if (bufLength) {
                char *buf = (char *) malloc(bufLength);
                if (buf) {
                    glGetProgramInfoLog(program, bufLength, NULL, buf);
                    LOGE("Could not link program:\n%s\n", buf);
                    free(buf);
                }
            }
            glDeleteProgram(program);
            program = 0;
        }
    }
    return program;
}

GLuint gProgram;
GLuint gvPositionHandle;

// 根据Java层传递过来的width和height配置ViewPort窗口。
bool setupGraphics(int w, int h) {
    printGLString("Version", GL_VERSION);
    printGLString("Vendor", GL_VENDOR);
    printGLString("Renderer", GL_RENDERER);
    printGLString("Extensions", GL_EXTENSIONS);

    LOGI("setupGraphics(%d, %d)", w, h);
    gProgram = createProgram(gVertexShader, gFragmentShader);
    if (!gProgram) {
        LOGE("Could not create program.");
        return false;
    }
    // 获取顶点着色器的vPosition句柄。
    gvPositionHandle = glGetAttribLocation(gProgram, "vPosition");
    checkGlError("glGetAttribLocation");
    LOGI("glGetAttribLocation(\"vPosition\") = %d\n", gvPositionHandle);
    //
    glViewport(0, 0, w, h);
    checkGlError("glViewport");
    return true;
}

// 顶点坐标
const GLfloat gTriangleVertices[] = {
        0.0f, 0.5f, // 上
        -0.5f, -0.5f, // 左
        0.5f, -0.5f  // 右
};

// 渲染画面，对应Java层的onDrawFrame
void renderFrame() {
    static float grey;
    grey += 0.01f;
    if (grey > 1.0f) {
        grey = 0.0f;
    }
    // grey = 0.9f;
    // 设置背景颜色
    glClearColor(grey, grey, grey, 1.0f);
    checkGlError("glClearColor");
    // 清屏
    glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
    checkGlError("glClear");
    // 使用创建出来的gProgram
    glUseProgram(gProgram);
    checkGlError("glUseProgram");
    // glVertexAttribPointer(GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const void *pointer);
    // 第二个参数表示每个顶点所占的坐标值个数，在这些是二维坐标，所以一个基点就只有两个坐标值。
    glVertexAttribPointer(gvPositionHandle, 2, GL_FLOAT, GL_FALSE, 0, gTriangleVertices);
    checkGlError("glVertexAttribPointer");
    // 启用顶点
    glEnableVertexAttribArray(gvPositionHandle);
    checkGlError("glEnableVertexAttribArray");
    // 以GL_TRIANGLES绘制顶点。
    glDrawArrays(GL_TRIANGLES, 0, 3);
    checkGlError("glDrawArrays");
}

extern "C" {
JNIEXPORT void JNICALL
Java_com_android_gl2jni_GL2JNILib_init(JNIEnv *env, jobject obj, jint width, jint height);
JNIEXPORT void JNICALL Java_com_android_gl2jni_GL2JNILib_step(JNIEnv *env, jobject obj);
};

// invoked from GLSurfaceView.Renderer#onSurfaceChanged
// 获取到Surface的宽和高后调用。
JNIEXPORT void JNICALL
Java_com_android_gl2jni_GL2JNILib_init(JNIEnv *env, jobject obj, jint width, jint height) {
    setupGraphics(width, height);
}

// invoked from GLSurfaceView.Renderer#onDrawFrame
// 绘制内容。
JNIEXPORT void JNICALL Java_com_android_gl2jni_GL2JNILib_step(JNIEnv *env, jobject obj) {
    renderFrame();
}
```

## GL2JNIView.java

继承**GLSurfaceView**，设置`EGLContextClientVersion`和`Renderer`以及`RenderMode`。

```Java
class GL2JNIView extends GLSurfaceView {
    private static String TAG = "GL2JNIView";
    private static final boolean DEBUG = false;

    public GL2JNIView(Context context) {
        super(context);
        init(false, 0, 0);
    }

    public GL2JNIView(Context context, boolean translucent, int depth, int stencil) {
        super(context);
        init(translucent, depth, stencil);
    }

    private void init(boolean translucent, int depth, int stencil) {
        /* Set the renderer responsible for frame rendering */
        setEGLContextClientVersion(2);
        setRenderer(new Renderer());
        setRenderMode(GLSurfaceView.RENDERMODE_WHEN_DIRTY);
    }

```

### Renderer.java

`Renderer`类中，直接调用**GL2JNILib**的`native`方法，来进行渲染。

```Java
public class Renderer implements GLSurfaceView.Renderer {
    public void onDrawFrame(GL10 gl) {
        GL2JNILib.step();
    }

    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GL2JNILib.init(width, height);
    }

    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        // Do nothing.
    }
}
```

### GL2JNILib.java

```Java
public class GL2JNILib {

     static {
         System.loadLibrary("gl2jni");
     }

    /**
     * @param width the current view width
     * @param height the current view height
     */
     public static native void init(int width, int height);
     public static native void step();
}
```

## CMakeLists.txt

```Shell
add_library(gl2jni SHARED
            src/main/cpp/gl_code.cpp)

# add lib dependencies
target_link_libraries(gl2jni
                      android
                      log
                      EGL
                      GLESv2)
```
