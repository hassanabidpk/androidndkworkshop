# Add Native Code Part 2

1. Create a new C++ source file `RendererES2.cpp` in `cpp` folder and add the following code.

```c++ 

/*
 * Copyright 2013 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include "glesjni.h"
#include <EGL/egl.h>

static const char VERTEX_SHADER[] =
        "#version 100\n"
                "uniform mat2 scaleRot;\n"
                "uniform vec2 offset;\n"
                "attribute vec2 pos;\n"
                "attribute vec4 color;\n"
                "varying vec4 vColor;\n"
                "void main() {\n"
                "    gl_Position = vec4(scaleRot*pos + offset, 0.0, 1.0);\n"
                "    vColor = color;\n"
                "}\n";

static const char FRAGMENT_SHADER[] =
        "#version 100\n"
                "precision mediump float;\n"
                "varying vec4 vColor;\n"
                "void main() {\n"
                "    gl_FragColor = vColor;\n"
                "}\n";

class RendererES2: public Renderer {
public:
    RendererES2();
    virtual ~RendererES2();
    bool init();

private:
    virtual float* mapOffsetBuf();
    virtual void unmapOffsetBuf();
    virtual float* mapTransformBuf();
    virtual void unmapTransformBuf();
    virtual void draw(unsigned int numInstances);

    const EGLContext mEglContext;
    GLuint mProgram;
    GLuint mVB;
    GLint mPosAttrib;
    GLint mColorAttrib;
    GLint mScaleRotUniform;
    GLint mOffsetUniform;

    float mOffsets[2*MAX_INSTANCES];
    float mScaleRot[4*MAX_INSTANCES];   // array of 2x2 column-major matrices
};

Renderer* createES2Renderer() {
    RendererES2* renderer = new RendererES2;
    if (!renderer->init()) {
        delete renderer;
        return NULL;
    }
    return renderer;
}

RendererES2::RendererES2()
        :   mEglContext(eglGetCurrentContext()),
            mProgram(0),
            mVB(0),
            mPosAttrib(-1),
            mColorAttrib(-1),
            mScaleRotUniform(-1),
            mOffsetUniform(-1)
{}

bool RendererES2::init() {
    mProgram = createProgram(VERTEX_SHADER, FRAGMENT_SHADER);
    if (!mProgram)
        return false;
    mPosAttrib = glGetAttribLocation(mProgram, "pos");
    mColorAttrib = glGetAttribLocation(mProgram, "color");
    mScaleRotUniform = glGetUniformLocation(mProgram, "scaleRot");
    mOffsetUniform = glGetUniformLocation(mProgram, "offset");

    glGenBuffers(1, &mVB);
    glBindBuffer(GL_ARRAY_BUFFER, mVB);
    glBufferData(GL_ARRAY_BUFFER, sizeof(QUAD), &QUAD[0], GL_STATIC_DRAW);

    ALOGV("Using OpenGL ES 2.0 renderer");
    return true;
}

RendererES2::~RendererES2() {
    /* The destructor may be called after the context has already been
     * destroyed, in which case our objects have already been destroyed.
     *
     * If the context exists, it must be current. This only happens when we're
     * cleaning up after a failed init().
     */
    if (eglGetCurrentContext() != mEglContext)
        return;
    glDeleteBuffers(1, &mVB);
    glDeleteProgram(mProgram);
}

float* RendererES2::mapOffsetBuf() {
    return mOffsets;
}

void RendererES2::unmapOffsetBuf() {
}

float* RendererES2::mapTransformBuf() {
    return mScaleRot;
}

void RendererES2::unmapTransformBuf() {
}

void RendererES2::draw(unsigned int numInstances) {
    glUseProgram(mProgram);

    glBindBuffer(GL_ARRAY_BUFFER, mVB);
    glVertexAttribPointer(mPosAttrib, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (const GLvoid*)offsetof(Vertex, pos));
    glVertexAttribPointer(mColorAttrib, 4, GL_UNSIGNED_BYTE, GL_TRUE, sizeof(Vertex), (const GLvoid*)offsetof(Vertex, rgba));
    glEnableVertexAttribArray(mPosAttrib);
    glEnableVertexAttribArray(mColorAttrib);

    for (unsigned int i = 0; i < numInstances; i++) {
        glUniformMatrix2fv(mScaleRotUniform, 1, GL_FALSE, mScaleRot + 4*i);
        glUniform2fv(mOffsetUniform, 1, mOffsets + 2*i);
        glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    }
}

```

Since we added another native file. Lets add this to our CMake script too. Add following below `src/main/cpp/glesjni.cpp`

```bash
src/main/cpp/RendererES2.cpp
```


2. Add following lines to your `CMakeLists.txt` file below `cmake_minimum_required(VERSION 3.4.1)`

```bash

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fno-rtti -fno-exceptions -Wall")

if (${ANDROID_PLATFORM_LEVEL} LESS 11)
  message(FATAL_ERROR "OpenGL 2 is not supported before API level 11 (currently using ${ANDROID_PLATFORM_LEVEL}).")
  return()
else ()
  set(OPENGL_LIB GLESv2)
endif (${ANDROID_PLATFORM_LEVEL} LESS 11)


```

Add following in `target_link_libraries` section of `CMakeLists.txt`

```bash

target_link_libraries( # Specifies the target library.
                       native-lib
                       ${log-lib}
                       ${OPENGL_LIB}
                       android
                       EGL
                       m)
```

3. Sync ![](https://codelabs.developers.google.com/codelabs/android-studio-jni/img/a0e53a92d95d4098.png), Build ![](https://codelabs.developers.google.com/codelabs/android-studio-jni/img/48d2cace55b491ec.png), there should be no `errors` from Android Studio.
4. Let's add the remain code in our native source files. 

In `glesjni.cpp`, add the following code in corresponding methods

```c++

JNIEXPORT void JNICALL
Java_com_hassanabid_gdgfestgles3_GLESJNILib_init(JNIEnv *env, jclass type) {

    if (g_renderer) {
        delete g_renderer;
        g_renderer = NULL;
    }

    printGlString("Version", GL_VERSION);
    printGlString("Vendor", GL_VENDOR);
    printGlString("Renderer", GL_RENDERER);
    printGlString("Extensions", GL_EXTENSIONS);

    const char* versionStr = (const char*)glGetString(GL_VERSION);
    if (strstr(versionStr, "OpenGL ES 2.") || strstr(versionStr, "OpenGL ES 3.")) {
        g_renderer = createES2Renderer();
    } else {
        ALOGE("Unsupported OpenGL ES version");
    }

}

```

```c++

JNIEXPORT void JNICALL
Java_com_hassanabid_gdgfestgles3_GLESJNILib_resize(JNIEnv *env, jclass type, jint width,
                                                   jint height) {

    if (g_renderer) {
        g_renderer->resize(width, height);
    }

}

```

```c++

JNIEXPORT void JNICALL
Java_com_hassanabid_gdgfestgles3_GLESJNILib_step(JNIEnv *env, jclass type) {

    if (g_renderer) {
        g_renderer->render();
    }

}

```

5. Sync ![](https://codelabs.developers.google.com/codelabs/android-studio-jni/img/a0e53a92d95d4098.png), Build ![](https://codelabs.developers.google.com/codelabs/android-studio-jni/img/48d2cace55b491ec.png), there should be no `errors` from Android Studio. Let's move to the next chapter where we will add code on Java side! 

