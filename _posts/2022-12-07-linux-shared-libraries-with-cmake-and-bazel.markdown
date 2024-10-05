---
layout: post
title: Linux shared libraries with CMake and Bazel
date: '2022-12-07 19:55:07'
---

A great introduction to shared libraries is available from Michael Kerrisk:

- [https://man7.org/conf/lca2006/shared\_libraries/index.html](https://man7.org/conf/lca2006/shared_libraries/index.html)

A detailed analysis of shared libraries can be found in Ulrich Drepper's paper:

- [https://akkadia.org/drepper/dsohowto.pdf](https://akkadia.org/drepper/dsohowto.pdf)

This paper is a must read if you want to understand the mechanics behind shared libraries. I will focus only on how shared libraries can be produced using various methods and build systems. The code can be found here:

- [https://github.com/ltekieli/shared_library_examples](https://github.com/ltekieli/shared_library_examples)

## Building shared libraries manually

A shared library is a collection of object files that contain usable symbols, which are loaded by the dynamic linker when running an executable. First, the object files need to be created:

    "${CC}" -fPIC -c "${SCRIPT_DIR}/f2/f2.cpp" -o "${SCRIPT_DIR}/build/f2/f2.o"
    
    "${CC}" -fPIC -I "${SCRIPT_DIR}" -c "${SCRIPT_DIR}/f1/f1.cpp" -o "${SCRIPT_DIR}/build/f1/f1.o"

Two cpp files, f1.cpp and f2.cpp are compiled using the -fPIC flag. Refer to Ulrich Drepper's paper for full explanation, but in short, instead of fixed addresses of the symbols, addresses relative to some starting point are generated. This is then used by the dynamic linker to setup the library inside the executable.

In the example code base, the f1 implementation depends on f2, so two shared libraries can be created:

    "${CC}" -fPIC -shared -Wl,-soname,libf2.so.0 \
        "${SCRIPT_DIR}/build/f2/f2.o" \
        -o "${SCRIPT_DIR}/build/f2/libf2.so.0.1"
    
    "${CC}" -fPIC -shared -Wl,-soname,libf1.so.0 -Wl,-rpath,\$ORIGIN/../f2 \
        "${SCRIPT_DIR}/build/f1/f1.o" "${SCRIPT_DIR}/build/f2/libf2.so.0.1" \
        -o "${SCRIPT_DIR}/build/f1/libf1.so.0.1"

The -shared flag is added, so that the linker is told to create a shared object.. The -soname sets the shared object name which is embedded as a dynamic tag (SONAME) inside the library. -rpath sets the dynamic linker search path (dynamic tag RUNPATH) to look for needed libraries during run time.

The library naming follows the pattern libname.major.minor (and optional .patch can be included) which is the standard semantic versioning: [https://semver.org/](https://semver.org/)

In case of shared libraries, the SONAME is set to the major version only. When a library is created additional symbolic links are needed for proper functioning:

    $ ln -s libf2.so.0.1 "${SCRIPT_DIR}/build/f2/libf2.so.0"
    $ ln -s libf2.so.0.1 "${SCRIPT_DIR}/build/f2/libf2.so"

libf2.so.0.1 is the real name of the shared object. libf2.so.0 is the shared object name. libf2.so is the linker name.

When a shared library is linked there are two possibilities. First, is to directly specify the full path to the shared object. Second, is to specify the library search path and part of the linker name:

    "${CC}" -fPIC -shared -Wl,-soname,libf1.so.0 -Wl,-rpath,\$ORIGIN/../f2 \
        "${SCRIPT_DIR}/build/f1/f1.o" "${SCRIPT_DIR}/build/f2/libf2.so.0.1" \
        -o "${SCRIPT_DIR}/build/f1/libf1.so.0.1"

vs.

    "${CC}" -fPIC -shared -Wl,-soname,libf1.so.0 -Wl,-rpath,\$ORIGIN/../f2 \
        "${SCRIPT_DIR}/build/f1/f1.o" -L "${SCRIPT_DIR}/build/f2" -lf2 \
        -o "${SCRIPT_DIR}/build/f1/libf1.so.0.1"

In both cases the SONAME of the dependency will be embedded into the resulting library's DT\_NEEDED tag.

Linking an executable, to library f1 will result in the following output from ldd:

    $ ldd build/main 
    	linux-vdso.so.1 (0x00007ffd4a37f000)
    	libf1.so.0 => /home/lukasz/workspace/shared_library_examples/build/f1/libf1.so.0 (0x00007f8101256000)
    	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f810101b000)
    	libf2.so.0 => /home/lukasz/workspace/shared_library_examples/build/f1/../f2/libf2.so.0 (0x00007f8101016000)
    	/lib64/ld-linux-x86-64.so.2 (0x00007f8101262000)
    	libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f8100dec000)
    	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f8100d05000)
    	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f8100ce3000)

So both libraries, f1 and f2 are correctly found. However, looking at dynamic tags, it is clear that the f2 dependency comes directly form f1.

    $ readelf -d build/main | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [libf1.so.0]
     0x0000000000000001 (NEEDED) Shared library: [libc.so.6]
     0x000000000000001d (RUNPATH) Library runpath: [$ORIGIN/f1]
     
     $ readelf -d build/f1/libf1.so.0.1 | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [libf2.so.0]
     0x000000000000001d (RUNPATH) Library runpath: [$ORIGIN/../f2]

## Building shared libraries with CMake

The manual process above can be reproduced using CMake:

    cmake_minimum_required(VERSION 3.22)
    project(shlibs VERSION 0.1)
    
    add_library(f2 f2/f2.cpp)
    target_include_directories(f2
        PUBLIC
            ${CMAKE_CURRENT_SOURCE_DIR}
    )
    set_target_properties(f2 PROPERTIES
        SOVERSION "${shlibs_VERSION_MAJOR}"
        VERSION "${shlibs_VERSION}"
    )
    
    add_library(f1 f1/f1.cpp)
    target_link_libraries(f1
        PRIVATE
            f2
    )
    set_target_properties(f1 PROPERTIES
        SOVERSION "${shlibs_VERSION_MAJOR}"
        VERSION "${shlibs_VERSION}"
    )
    
    add_executable(main main.cpp)
    target_link_libraries(main f1)

Standard command add\_library and add\_executable are needed. Additionally by using target\_link\_libraries, the dependency chain is created. Notice the usage of PRIVATE keyword for f1 dependencies. This will prevent CMake from transitively forwarding the dependencies of f1 to all targets that depend on f1. Using public will result in the executable having both f1 and f2 specified in NEEDED tags.

Additionally the VERSION and SOVERSION properties are set for the library targets, so that they have a proper shared object name and symlinks are automatically created.

CMake by default does not build shared libraries and needs to be instructed to do so:

    $ cmake -B build-cmake . -GNinja -DBUILD_SHARED_LIBS=ON
    $ cmake --build build-cmake/ --verbose

The result is similar to what can be achived with manual steps:

    $ tree -L 1 build-cmake/
    build-cmake/
    ├── build.ninja
    ├── CMakeCache.txt
    ├── CMakeFiles
    ├── cmake_install.cmake
    ├── libf1.so -> libf1.so.0
    ├── libf1.so.0 -> libf1.so.0.1
    ├── libf1.so.0.1
    ├── libf2.so -> libf2.so.0
    ├── libf2.so.0 -> libf2.so.0.1
    ├── libf2.so.0.1
    └── main

## Building shared libraries with Bazel

The above setup can be replicated with a simple BUILD file using Bazel:

    cc_library(
        name = "f1",
        srcs = ["f1/f1.cpp"],
        hdrs = ["f1/f1.h"],
        deps = [":f2"],
    )
    
    cc_library(
        name = "f2",
        srcs = ["f2/f2.cpp"],
        hdrs = ["f2/f2.h"],
    )
    
    cc_binary(
        name = "main",
        srcs = ["main.cpp"],
        deps = [":f1"],
    )
    
    cc_test(
        name = "main_test",
        srcs = ["main.cpp"],
        deps = [":f1"],
    )

As a result we will see both static and shared libraries created:

    $ tree -L 1 bazel-bin/
    bazel-bin/
    ├── libf1.a
    ├── libf1.a-2.params
    ├── libf1.so
    ├── libf1.so-2.params
    ├── libf2.a
    ├── libf2.a-2.params
    ├── libf2.so
    ├── libf2.so-2.params
    ├── main
    ├── main-2.params
    ├── main.runfiles
    ├── main.runfiles_manifest
    ├── main_test
    ├── main_test-2.params
    ├── main_test.runfiles
    ├── main_test.runfiles_manifest
    ├── _objs
    └── _solib_k8

However, if we look closely we will see that few things are missing

    $ readelf -a bazel-bin/main | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [libstdc++.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libc.so.6]
     
     $ readelf -a bazel-bin/main_test | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [liblibf1.so]
     0x0000000000000001 (NEEDED) Shared library: [liblibf2.so]
     0x0000000000000001 (NEEDED) Shared library: [libstdc++.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libc.so.6]
     0x000000000000001d (RUNPATH) Library runpath: [$ORIGIN/_solib_k8/]
    
    $ readelf -a bazel-bin/libf1.so | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [libstdc++.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libc.so.6]
    
    $ readelf -a bazel-bin/libf2.so | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [libstdc++.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libc.so.6]

The executable does not link against shared objects. The shared objects themselves do not specify their dependencies.

Linking cc\_binary statically is the default behavior, contrary to cc\_test, which is linked dynamically.

Setting the linkstatic attribute of cc\_library to true. will disable the creation of shared libraries completely. Setting it to false on a cc\_library does not change the default behavior, so still both static and dynamic versions are created.

Setting the linkstatic attribute of cc\_binary to false, will force the executable to be linked dynamically.

This is not exactly what we want to achieve so let's take a look at another approach:

    cc_library(
        name = "f1_hdrs",
        hdrs = ["f1/f1.h"],
    )
    
    cc_binary(
        name = "f1",
        srcs = ["f1/f1.cpp"],
        deps = ["f1_hdrs", ":f2"],
        linkshared = True,
    )
    
    cc_library(
        name = "f2_hdrs",
        hdrs = ["f2/f2.h"],
    )
    
    cc_binary(
        name = "f2",
        srcs = ["f2/f2.cpp"],
        deps = [":f2_hdrs"],
        linkshared = True,
    )
    
    cc_binary(
        name = "main",
        srcs = ["main.cpp"],
        deps = [":f1"],
    )

Here, we try to exploit the possibility of setting the linkshared attribute of cc\_binary to true, which will create a shared library. We need to separately specify header libraries to expose the headers.

Unfortunately, this approach will not work at all. First problem is that, the libraries still don't depend on each other. Second problem is, the cc\_binary in deps of another binary is simply ignored. According to Bazel's documentation, the linkshared option is only useful to create shared libraries which can be then loaded in runtime.

In order to get a bit closer to our ideal solution, Bazel has currently (version 5.3.2) experimental support for shared libraries, which is enabled with a flag, that exposes the cc\_shared\_library rule.

    $ bazel build --experimental_cc_shared_library //...

The BUILD file can be written like this:

    cc_library(
        name = "f1_base",
        srcs = ["f1/f1.cpp"],
        hdrs = ["f1/f1.h"],
        deps = [":f2_base"],
    )
    
    cc_shared_library(
        name = "f1",
        dynamic_deps = [":f2"],
        roots = [":f1_base"],
    )
    
    cc_library(
        name = "f2_base",
        srcs = ["f2/f2.cpp"],
        hdrs = ["f2/f2.h"],
    )
    
    cc_shared_library(
        name = "f2",
        roots = [":f2_base"],
    )
    
    cc_binary(
        name = "main",
        srcs = ["main.cpp"],
        dynamic_deps = [":f1"],
        deps = ["f1_base"],
    )

What is happening here, is that we specify the regular cc\_library/cc\_binary combination and dependencies, but additionally we create cc\_shared\_library rules, which are an addon that lives on top of the regular dependency hierarchy. The roots attribute specifies which cc\_libraries to combine into a shared object. The dynamic\_deps attribute allows to create dependencies between shared objects. This attribute is available from the cc\_binary as well. The dynamic\_deps essentially tells the rule which of the regular dependencies should be linked dynamically.

The resulting shared objects have now correctly specified dependencies:

    $ readelf -a bazel-bin/libf1.so | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [liblibf2.so]
     0x0000000000000001 (NEEDED) Shared library: [libstdc++.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libc.so.6]
     0x000000000000001d (RUNPATH) Library runpath: [$ORIGIN/_solib_k8/]

However, the executable still links directly both, instead of relying on transitive resolution:

    $ readelf -a bazel-bin/main | grep -e NEEDED -e RUNPATH
     0x0000000000000001 (NEEDED) Shared library: [liblibf1.so]
     0x0000000000000001 (NEEDED) Shared library: [liblibf2.so]
     0x0000000000000001 (NEEDED) Shared library: [libstdc++.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED) Shared library: [libc.so.6]
     0x000000000000001d (RUNPATH) Library runpath: [$ORIGIN/_solib_k8/]

Bazel has another experimental feature that adds a new attribute to cc\_binary and cc\_library rules called experimental\_cc\_implementation\_deps, which allows to hide private implementations. However, currently (Bazel version 5.3.2) the combination of both features seem not to work correctly. The shared libraries do not have dependencies specified, just forwarded to the executables.

Additionally Bazel does not support out of the box symlinking for semantic versioning and manipulating rpath.

