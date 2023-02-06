# Development

Find here some useful information about how to install all dependencies per platform in order to start hacking.

## Windows

### Tools and other items to install

Apart from the usual Windows development tools such as MSVS (2019), its tools and
Windows SDKs (10.0.19041.0) there is no much left to install. The following list
enumerates these other additional tools to install.

- Python 3.
- Bazel 5.3.0.
- LLVM 11.0.0 (given the one shipped with MSVS 2019 12.0.0 does not work well with Bazel 5.3.0).
- Git Bash terminal (usually installed along with Git for Windows).

Mediapipe version 0.8.11 depends on OpenCV 3.4.10. While the ML Transformers
library does not depend on it the Windows Mediapipe build process depends on it. 
We have uploaded a pre build OpenCV version that will be used by both Mediapipe and ML-Transformers for linkage.

For the reader reference the list of items here to install is synthesized from
info at https://google.github.io/mediapipe/getting_started/install.html#installing-on-windows.

### Environment

#### MSVS environment

In order to have all MSVS related tools available under the terminal windows (CMD) being use
just init the environment as usual (e.g. targeting x64):

```
"%VS160COMNTOOLS%"\..\..\VC\Auxiliary\Build\vcvarsall.bat x64
```

The Python build script sets it for Windows when needed.

Note: `VS160COMNTOOLS` points to the common tools folder for MSVS
2019 (e.g. `VS160COMNTOOLS=C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\Common7\Tools\`).

#### Bazel related variables

```
BAZEL_LLVM=C:\Program Files\LLVM
BAZEL_VC=C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC
BAZEL_VC_FULL_VERSION=14.29.30133
BAZEL_WINSDK_FULL_VERSION=10.0.19041.0
```

Learn more about them at https://google.github.io/mediapipe/getting_started/install.html#installing-on-windows (See _Set Bazel variables_ section 6).

#### PATH

Some bin folder has to be added to the PATH environment variable so the scripts in this repo are able
to find the tools they need. Find here such list of folders.

The `bash.exe` command line tool has to found so add its bin folder to the PATH environment variable:

```
set PATH=%ProgramFiles%\\Git\\bin;%PATH%
```

It is important the environment variable is set this way under the CMD terminal
window use for running the Python build script both locally and in any Github
Actions runner building the library.

## Third party libraries

ML Transformers depends on a few third party libraries. Find here some information
about them.

### ANGLE

Medipipe depends on [ANGLE](https://github.com/google/angle). It provides GPU
support for Mediapipe desktop builds for platforms such as macOS and Windows. The
following sections provide detailed information about how it is built and some
random useful comments.

#### Intro

ANGLE source code is mirrored in Github.com [here](https://github.com/google/angle)
and there is some useful information about
development [here](https://github.com/google/angle/blob/main/doc/DevSetup.md).

#### ANGLE for macOS - not in use since we use Metal for MacOS.

#### Pulling the code

We build ANGLE version `chromium/5359`. We apply some changes on top of it. The
patch we apply lives in this repo [here](./ml/angle_chromium_5359_macos_build.patch).

The process we follow so we have the sources ready to build the library is as
follow (Note `depot_tools` tool suite has to be available in your PATH environment
variable):

```bash
$ mkdir angle; cd angle
$ fetch angle
$ git checkout chromium/5359
$ gclient sync --reset -D
git apply <PATH_TO_ANGLE_CHROMIUM_5359_MACOS_BUILD_PATCH>
```

#### Building the library

Run the following commands to build the library as we need for Mediapipe. Note
these commands are tested on a macOS M1 machine.

```bash
$ cd angle
$ gn gen ../out/release/arm64 --args='is_component_build=false is_debug=false angle_assert_always_on=false angle_enable_cl=true angle_build_tests=false angle_enable_swiftshader=false use_android_unwinder_v2=false symbol_level=0'
$ autoninja -C ../out/release/arm64 libEGL_static
$ gn gen ../out/release/x86_64 --args='is_component_build=false is_debug=false angle_assert_always_on=false angle_enable_cl=true angle_build_tests=false angle_enable_swiftshader=false use_android_unwinder_v2=false target_cpu="x64" symbol_level=0'
$ autoninja -C ../out/release/x86_64 libEGL_static
```

Note the commands above get the library built for the two macOS archs we support
`x86_64` and `arm64`. These are built in release build configuration. No need to
build them in debug as of today.

#### Library build details

The build proccess gives us the following binaries:

```
libGLESv2_static.a
libEGL_static.a
libangle_common.a
libchrome_zlib.a
libangle_gpu_info_util.a
libtranslator.a
libpreprocessor.a
libcompression_utils_portable.a
libOpenCL_ANGLE.a
```

The next step is to combine/pack all of them so we have the final binaries to use.

As a reference we have the following list:

```
libANGLE_base.a
libANGLE.a
angle_glslang_wrapper.a
angle_cl_backend.a
angle_gl_backend.a
angle_metal_backend.a
angle_vulkan_backend.a
angle_null_backend.a
angle_version_info.a
angle_libc++.a (folder libc++ added angle to avoid confilcts)
astcenc.a
zlib_adler32_simd.a
zlib_arm_crc32.a
zlib_inflate_chunk_simd.a
xxhash.a
angle_image_util.a
spirv_cross_source.a
glslang_lib_sources.a
vulkan_memory_allocator.a
libvulkan.a
angle_spirv_builder.a
glslang_default_resource_limits_sources.a
angle_spirv_parser.a
angle_vulkan_icd.a
angle_vk_mem_alloc_wrapper.a
```

We combine static build binaries by using `libtool`.

The last step in the process is to create the final binary as a XC framework.

As summary the process we follow so we have the library ready to use by the
Medipipe build is:

- Combine all binaries (objects and intermediate static libraries) into one by using `libtool`.
- Create the final fat static library by merging `arm64` and `x86_64` binaries into one by using `lipo`.
- Create the final XC framework bundle file by using `xcodebuild`.

All this packing process is automated by the [angle.py](../ml/angle.py) script.

Find below the way the developer runs the script:

```bash
$ python3 angle.py --verbose --platform macos
```

As as result we have the `VonageLibAngle.xcframework` XC framework bundle file
that we can distribute and consume in ML Transformers library build time from
any remote source (e.g. AWS S3).

#### ANGLE for Windows - No need to build since we use pre build assets.

#### Pulling the code

We build ANGLE version `chromium/4844`. We apply some changes on top of it. The
patch we apply lives in this repo [here](./ml/angle_chromium_4844_windows_build.patch).

The process we follow so we have the sources ready to build the library is as
follow (Note `depot_tools` tool suite has to be available in your PATH environment
variable):

```bash
$ mkdir angle; cd angle
$ fetch angle
$ git checkout chromium/4844
$ gclient sync --reset -D
git apply <PATH_TO_ANGLE_CHROMIUM_4844_WINDOWS_BUILD_PATCH>
```

#### Building the library

Run the following commands to build the library as we need for Mediapipe.

```bash
$ cd angle
$ gn gen out/Release --args='target_cpu=\"x64\" is_component_build=false is_debug=false use_lld=false use_custom_libcxx=false'
$ autoninja -C out/release libANGLE_static
$ autoninja -C out/release libGLESv2_static
$ autoninja -C out/release libEGL_static
```

Note the commands above get the library built for the Windows `x64` arch we support.

As a result of these commands we have the binaries ready to use. We pack all
resources (header files and .a files) under `libangle` folder in the root that we
compress manually as a zip file we can distribute and consume in ML Transformers
library build time from any remote source (e.g. AWS S3).

See the layout of the `libangle` folder in the root (root repository `angle` folder):

```
.
├── build
│   └── Release
│       └── x64
│           ├── libANGLE_static.lib
│           ├── libEGL_static.lib
│           └── libGLESv2_static.lib
└── include
    ├── CL
    ├── EGL
    ├── GLES
    ├── GLES2
    ├── GLES3
    ├── GLSLANG
    ├── KHR
    ├── WGL
    ├── angle_cl.h
    ├── angle_gl.h
    ├── angle_windowsstore.h
    ├── export.h
    ├── platform
    └── vulkan
```

## OpenCV

### Windows - No need to build since we use pre build assets.
- Go use the installer at https://sourceforge.net/projects/opencvlibrary/files/3.4.10/opencv-3.4.10-vc14_vc15.exe/download
and install it locally.

#### ML-Transformers use we need to build it static.
- Install `cmake` for windows https://cmake.org/download/
- Open with `cmake` .../opencv/source folder.
- Make sure the following:
	- `BUILD_WITH_STATIC_CRT` - checked.
	- `BUILD_SHARED_LIBS` - not checked.
	- `WITH_CAROTENE` - not checked.
	- `WITH_IPP` - not checked.
	- `WITH_WEBP` - not checked.
- `configure` and `open project`
- now in the visual studio just build the code in the old fashion way.
- copy the build assets to the final destination package.
#### MediaPipe needs the pre build dynamic build of OpenCV.
- copy `build` folder to final destination package.

#### upload the artifact as a `zip` to S3.
- for understanding how the package looks see sample in S3.
