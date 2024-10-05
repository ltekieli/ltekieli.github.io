---
layout: post
title: Cross compiling with Bazel
date: '2020-09-17 17:01:32'
tags:
- bazel
- c
- cross-compilation
- toolchain
---

This time a short introduction to Bazel and how to cross-compile with this tool. I will explain how to build for host platform, as well as for multiple different targets.

You can get the partial repositories used in this exercise [here](https://github.com/ltekieli/bazel_cross_compile).

<!--kg-card-begin: markdown-->
## Installing Bazelisk

I highly recommend using [Bazelisk](https://github.com/bazelbuild/bazelisk) for managing your Bazel installation. TLDR:

    $ wget https://github.com/bazelbuild/bazelisk/releases/download/v1.6.1/bazelisk-linux-amd64
    $ sudo mv bazelisk-linux-amd64 /usr/local/bin/bazel
    $ sudo chmod +x /usr/local/bin/bazel

## Compiling "Hello World!"

Simplest possible structure for a C++ projects looks like this:

    $ tree
    .
    ├── BUILD
    ├── main.cpp
    └── WORKSPACE
    
    $ cat BUILD 
    cc_binary(
        name = "hello",
        srcs = ["main.cpp"],
    )
    
    $ cat main.cpp 
    #include <iostream>
    
    int main() {
        std::cout << "Hello World!" << std::endl;
    }

`WORKSPACE` file defines the root of the source tree. In our case it is empty, but usually it contains various definitions needed for your project to build.  
In the `BUILD` file resides a definition of our single target called "hello", which will be constructed from `main.cpp`

Everything you need to know about Bazel's terminology is [here](https://docs.bazel.build/versions/master/build-ref.html).

We can now build:

    $ bazel build //:hello
    2020/09/02 17:21:34 Downloading https://releases.bazel.build/3.4.1/release/bazel-3.4.1-linux-x86_64...
    Extracting Bazel installation...
    Starting local Bazel server and connecting to it...
    INFO: Analyzed target //:hello (14 packages loaded, 47 targets configured).
    INFO: Found 1 target...
    Target //:hello up-to-date:
      bazel-bin/hello
    INFO: Elapsed time: 8.451s, Critical Path: 0.62s
    INFO: 2 processes: 2 linux-sandbox.
    INFO: Build completed successfully, 6 total actions

And run:

    $ bazel run //:hello 
    ...
    Hello World!

or directly:

    $ ./bazel-bin/hello
    Hello World!

## Downloading dependencies

In order to setup our cross compilation environment we need to download our toolchains. Doing this manually is tedious, fortunately Bazel already includes the necessary facilities for that.

For our needs we need to download two files: the compiler and the sysroot.

    $ tree
    .
    ├── BUILD
    ├── main.cpp
    ├── third_party
    │   ├── BUILD
    │   ├── deps.bzl
    │   └── toolchains
    │   ├── aarch64-rpi3-linux-gnu-sysroot.BUILD
    │   ├── aarch64-rpi3-linux-gnu.BUILD
    │   ├── arm-cortex_a8-linux-gnueabihf-sysroot.BUILD
    │   ├── arm-cortex_a8-linux-gnueabihf.BUILD
    │   ├── BUILD
    │   └── toolchains.bzl
    └── WORKSPACE

All the `*.BUILD"` files contain the specification of how to use the content of the downloaded artifacts. The `*.bzl` files contain [Starlark](https://docs.bazel.build/versions/master/skylark/language.html) code that defines the logic of downloading files and exposing them as targets.

The entry point for Bazel to know that it needs to download any external content is in the `WORKSPACE` file:

    $ cat WORKSPACE 
    load("//third_party:deps.bzl", "deps")
    deps()

It says: load a function called deps from the file deps.bzl in third\_party package and then call it.

Similar the `deps.bzl`:

    $ cat third_party/deps.bzl 
    load("//third_party/toolchains:toolchains.bzl", "toolchains")
    
    def deps():
        toolchains()

It provides a level of indirection to hide the details of external packages from the `WORKSPACE` file.

The `toolchains.bzl` on the other hand:

    $ cat third_party/toolchains/toolchains.bzl 
    load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
    
    URL_TOOLCHAIN = "https://github.com/ltekieli/devboards-toolchains/releases/download/v2020.09.01/"
    URL_SYSROOT = "https://github.com/ltekieli/buildroot/releases/download/v2020.09.01/"
    
    def toolchains():
        if "aarch64-rpi3-linux-gnu" not in native.existing_rules():
            http_archive(
                name = "aarch64-rpi3-linux-gnu",
                build_file = Label("//third_party/toolchains:aarch64-rpi3-linux-gnu.BUILD"),
                url = URL_TOOLCHAIN + "aarch64-rpi3-linux-gnu.tar.gz",
                sha256 = "35a093524e35061d0f10e302b99d164255dc285898d00a2b6ab14bfb64af3a45",
            )
    
        if "aarch64-rpi3-linux-gnu-sysroot" not in native.existing_rules():
            http_archive(
                name = "aarch64-rpi3-linux-gnu-sysroot",
                build_file = Label("//third_party/toolchains:aarch64-rpi3-linux-gnu-sysroot.BUILD"),
                url = URL_SYSROOT + "aarch64-rpi3-linux-gnu-sysroot.tar.gz",
                sha256 = "56f3d84c9adf192981a243f27e6970afe360a60b72083ae06a8aa5c0161077a5",
                strip_prefix = "sysroot",
            )
    
        if "arm-cortex_a8-linux-gnueabihf" not in native.existing_rules():
            http_archive(
                name = "arm-cortex_a8-linux-gnueabihf",
                build_file = Label("//third_party/toolchains:arm-cortex_a8-linux-gnueabihf.BUILD"),
                url = URL_TOOLCHAIN + "arm-cortex_a8-linux-gnueabihf.tar.gz",
                sha256 = "6176e47be8fde68744d94ee9276473648e2e3d98d22578803d833d189ee3a6f0",
            )
    
        if "arm-cortex_a8-linux-gnueabihf-sysroot" not in native.existing_rules():
            http_archive(
                name = "arm-cortex_a8-linux-gnueabihf-sysroot",
                build_file = Label("//third_party/toolchains:arm-cortex_a8-linux-gnueabihf-sysroot.BUILD"),
                url = URL_SYSROOT + "arm-cortex_a8-linux-gnueabihf-sysroot.tar.gz",
                sha256 = "89a72cc874420ad06394e2333dcbb17f088c2245587f1147ff9da124bb60328f",
                strip_prefix = "sysroot",
            )
    

First it loads the http\_archive rule, then defines a function which has a repeating pattern inside:

    if "NAME_OF_THE_EXTERNAL_RESOURCE" not in native.existing_rules():
        http_archive(
            name = "NAME_OF_THE_EXTERNAL_RESOURCE",
            build_file = Label("//path/to:buildfile.BUILD"),
            url = SOME_URL,
            sha256 = SOME_SHA256,
        )

Which reads: if there is no such rule defined yet, then define it using the http\_archive with given name, build file, URL and checksum.

A particular `*.BUILD` file contains the definitions of targets coming from the downloaded artifact, for example:

    $ cat third_party/toolchains/aarch64-rpi3-linux-gnu.BUILD 
    package(default_visibility = ['//visibility:public'])
    
    filegroup(
      name = 'toolchain',
      srcs = glob([
        '**',
      ]),
    )

It specifies the default visibility of all the targets in this package to be public, and creates a target "toolchain" which is a handle to all the files inside the artifact.

We can build such a target:

    $ bazel build @aarch64-rpi3-linux-gnu//:toolchain

And peek what Bazel did for us:

    $ tree -L 1 bazel-02_deps/external/aarch64-rpi3-linux-gnu/
    ├── aarch64-rpi3-linux-gnu
    ├── bin
    ├── BUILD.bazel
    ├── build.log.bz2
    ├── include
    ├── lib
    ├── libexec
    ├── share
    └── WORKSPACE

Bazel downloaded the artifact inside his cache, copied our `aarch64-rpi3-linux-gnu.BUILD` file as `BUILD.bazel` and added a `WORKSPACE` file indicating that this is another source tree. We can refer to all targets inside such a package by specifying the repository name: `@aarch64-rpi3-linux-gnu//:toolchain`.

## Setting up custom toolchains

Bazel supports two ways of setting up custom toolchains, the legacy approach with `crosstool_top`, and the new approach with platforms. We will construct our rules so that both approaches are available.

In order to cross compile cc\_rules we need to run bazel with additional arguments pointing to cross compilation toolchain definition:

    bazel build \
        --crosstool_top=//bazel/toolchain/aarch64-rpi3-linux-gnu:gcc_toolchain \
        --cpu=aarch64
        //:hello

It instructs bazel to look for `aarch64` toolchain in the [cc\_toolchain\_suite](https://docs.bazel.build/versions/master/be/c-cpp.html#cc_toolchain_suite) rule named `gcc_toolchain` located in `bazel/toolchain/aarch64-rpi3-linux-gnu` package.

This file contains definitions of all tools we want to use when cross compiling:

    $ cat bazel/toolchain/aarch64-rpi3-linux-gnu/BUILD 
    package(default_visibility = ["//visibility:public"])
    
    load(":cc_toolchain_config.bzl", "cc_toolchain_config")
    
    filegroup(name = "empty")
    
    filegroup(
      name = 'wrappers',
      srcs = glob([
        'wrappers/**',
      ]),
    )
    
    filegroup(
      name = 'all_files',
      srcs = [
        '@aarch64-rpi3-linux-gnu-sysroot//:sysroot',
        '@aarch64-rpi3-linux-gnu//:toolchain',
        ':wrappers',
      ],
    )
    
    cc_toolchain_config(name = "aarch64_toolchain_config")
    
    cc_toolchain(
        name = "aarch64_toolchain",
        toolchain_identifier = "aarch64-toolchain",
        toolchain_config = ":aarch64_toolchain_config",
        all_files = ":all_files",
        compiler_files = ":all_files",
        dwp_files = ":empty",
        linker_files = ":all_files",
        objcopy_files = ":empty",
        strip_files = ":empty",
    )
    
    cc_toolchain_suite(
        name = "gcc_toolchain",
        toolchains = {
            "aarch64": ":aarch64_toolchain",
        },
        tags = ["manual"]
    )

The filegroups are convenience wrappers for files referenced from the externally downloaded artifacts of the compiler and sysroot. This can be more granular as seen from the `cc_toolchain` rule definition, but to keep it simple we will reference all files everywhere.

In order to hide from bazel where do we actually get our compiler from, we need to create some wrapper files:

    $ tree bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/
    bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/
    ├── aarch64-rpi3-linux-gnu-ar -> wrapper
    ├── aarch64-rpi3-linux-gnu-cpp -> wrapper
    ├── aarch64-rpi3-linux-gnu-gcc -> wrapper
    ├── aarch64-rpi3-linux-gnu-gcov -> wrapper
    ├── aarch64-rpi3-linux-gnu-ld -> wrapper
    ├── aarch64-rpi3-linux-gnu-nm -> wrapper
    ├── aarch64-rpi3-linux-gnu-objdump -> wrapper
    ├── aarch64-rpi3-linux-gnu-strip -> wrapper
    └── wrapper

This is needed, because in the tool specifications we cannot reference external repositories with `@` syntax and bazel expects this tool to live relatively to the `cc_toolchain` rule. Therefore we reference the wrapper script, which in the end knows where does the actual tool reside:

    $ cat bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/wrapper 
    #!/bin/bash
     
    NAME=$(basename "$0")
    TOOLCHAIN_BINDIR=external/aarch64-rpi3-linux-gnu/bin
     
    exec "${TOOLCHAIN_BINDIR}"/"${NAME}" "$@"

Bazel will call `bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/aarch64-rpi3-linux-gnu-gcc` which will exec the actuall gcc which resides inside the sandobx in the external directory: `external/aarch64-rpi3-linux-gnu/bin/aarch64-rpi3-linux-gnu-gcc`.

The lines:

    load(":cc_toolchain_config.bzl", "cc_toolchain_config")
    ...
    cc_toolchain_config(name = "aarch64_toolchain_config")

load the toolchain configuration from an additional file, which contains the path for particular toolchain tools, as well as default flags for compilation and linking steps:

    $ cat bazel/toolchain/aarch64-rpi3-linux-gnu/cc_toolchain_config.bzl 
    load("@bazel_tools//tools/build_defs/cc:action_names.bzl", "ACTION_NAMES")
    load("@bazel_tools//tools/cpp:cc_toolchain_config_lib.bzl",
        "feature",
        "flag_group",
        "flag_set",
        "tool_path",
    )
    
    all_link_actions = [
        ACTION_NAMES.cpp_link_executable,
        ACTION_NAMES.cpp_link_dynamic_library,
        ACTION_NAMES.cpp_link_nodeps_dynamic_library,
    ]
    
    all_compile_actions = [
        ACTION_NAMES.assemble,
        ACTION_NAMES.c_compile,
        ACTION_NAMES.clif_match,
        ACTION_NAMES.cpp_compile,
        ACTION_NAMES.cpp_header_parsing,
        ACTION_NAMES.cpp_module_codegen,
        ACTION_NAMES.cpp_module_compile,
        ACTION_NAMES.linkstamp_compile,
        ACTION_NAMES.lto_backend,
        ACTION_NAMES.preprocess_assemble,
    ]
    
    def _impl(ctx):
        tool_paths = [
            tool_path(
                name = "ar",
                path = "wrappers/aarch64-rpi3-linux-gnu-ar",
            ),
            tool_path(
                name = "cpp",
                path = "wrappers/aarch64-rpi3-linux-gnu-cpp",
            ),
            tool_path(
                name = "gcc",
                path = "wrappers/aarch64-rpi3-linux-gnu-gcc",
            ),
            tool_path(
                name = "gcov",
                path = "wrappers/aarch64-rpi3-linux-gnu-gcov",
            ),
            tool_path(
                name = "ld",
                path = "wrappers/aarch64-rpi3-linux-gnu-ld",
            ),
            tool_path(
                name = "nm",
                path = "wrappers/aarch64-rpi3-linux-gnu-nm",
            ),
            tool_path(
                name = "objdump",
                path = "wrappers/aarch64-rpi3-linux-gnu-objdump",
            ),
            tool_path(
                name = "strip",
                path = "wrappers/aarch64-rpi3-linux-gnu-strip",
            ),
        ]
    
        default_compiler_flags = feature(
            name = "default_compiler_flags",
            enabled = True,
            flag_sets = [
                flag_set(
                    actions = all_compile_actions,
                    flag_groups = [
                        flag_group(
                            flags = [
                                "--sysroot=external/aarch64-rpi3-linux-gnu-sysroot",
                                "-no-canonical-prefixes",
                                "-fno-canonical-system-headers",
                                "-Wno-builtin-macro-redefined",
                                "-D __DATE__ =\"redacted\"",
                                "-D __TIMESTAMP__ =\"redacted\"",
                                "-D __TIME__ =\"redacted\"",
                            ],
                        ),
                    ],
                ),
            ],
        )
        
        default_linker_flags = feature(
            name = "default_linker_flags",
            enabled = True,
            flag_sets = [
                flag_set(
                    actions = all_link_actions,
                    flag_groups = ([
                        flag_group(
                            flags = [
                                "--sysroot=external/aarch64-rpi3-linux-gnu-sysroot",
                                "-lstdc++",
                            ],
                        ),
                    ]),
                ),
            ],
        )
    
        features = [
            default_compiler_flags,
            default_linker_flags,
        ]
    
        return cc_common.create_cc_toolchain_config_info(
            ctx = ctx,
            features = features,
            toolchain_identifier = "aarch64-toolchain",
            host_system_name = "local",
            target_system_name = "unknown",
            target_cpu = "unknown",
            target_libc = "unknown",
            compiler = "unknown",
            abi_version = "unknown",
            abi_libc_version = "unknown",
            tool_paths = tool_paths,
        )
    
    cc_toolchain_config = rule(
        implementation = _impl,
        attrs = {},
        provides = [CcToolchainConfigInfo],
    )

What is happening here is that we create a new rule which will return a `CcToolchainConfigInfo` data structure containing all the tools Bazel needs for compiling. Additionally we set up features, which are a way of specifying behaviors of the toolchain, in our case we specify default flags for compiling and linking C++ code.

With all the code above we can now build for aarch64:

    $ bazel build --crosstool_top=//bazel/toolchain/aarch64-rpi3-linux-gnu:gcc_toolchain --cpu=aarch64 //:hello 
    INFO: Build options --cpu and --crosstool_top have changed, discarding analysis cache.
    INFO: Analyzed target //:hello (0 packages loaded, 7129 targets configured).
    INFO: Found 1 target...
    Target //:hello up-to-date:
      bazel-bin/hello
    INFO: Elapsed time: 0.970s, Critical Path: 0.03s
    INFO: 0 processes.
    INFO: Build completed successfully, 1 total action
    
    $ file bazel-bin/hello
    bazel-bin/hello: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 5.5.5, not stripped

## Setting up custom platforms

There are few additional steps needed to use platforms with the above setup. First we need to define our new platform:

    $ cat bazel/platforms/BUILD 
    platform(
        name = "rpi",
        constraint_values = [
            "@platforms//cpu:aarch64",
            "@platforms//os:linux",
        ],
    )

Second, we need to create new platform-compatible toolchain target:

    $ cat bazel/toolchain/aarch64-rpi3-linux-gnu/BUILD 
    ...
    
    toolchain(
        name = "aarch64_linux_toolchain",
        exec_compatible_with = [
            "@platforms//os:linux",
            "@platforms//cpu:x86_64",
        ],
        target_compatible_with = [
            "@platforms//os:linux",
            "@platforms//cpu:aarch64",
        ],
        toolchain = ":aarch64_toolchain",
        toolchain_type = "@bazel_tools//tools/cpp:toolchain_type",
    )

Third, register the toolchain in the WORKSPACE file:

    $ cat bazel/toolchain/toolchain.bzl 
    def register_all_toolchains():
        native.register_toolchains(
            "//bazel/toolchain/aarch64-rpi3-linux-gnu:aarch64_linux_toolchain",
        )

    $ cat WORKSPACE 
    ...
    
    load("//bazel/toolchain:toolchain.bzl", "register_all_toolchains")
    register_all_toolchains()

With that done, Bazel needs different command line arguments to use platforms:

    $ bazel build \
        --incompatible_enable_cc_toolchain_resolution \
        --platforms=//bazel/platforms:rpi \
        //:hello 
    Starting local Bazel server and connecting to it...
    INFO: Analyzed target //:hello (19 packages loaded, 7123 targets configured).
    INFO: Found 1 target...
    Target //:hello up-to-date:
      bazel-bin/hello
    INFO: Elapsed time: 25.510s, Critical Path: 1.41s
    INFO: 2 processes: 2 linux-sandbox.
    INFO: Build completed successfully, 6 total actions
    
    $ file bazel-bin/hello
    bazel-bin/hello: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 5.5.5, not stripped

## Setting up .bazelrc

For convenience all of the additional command line arguments can be hidden in the `.bazelrc` file:

    $ cat .bazelrc 
    build:rpi --crosstool_top=//bazel/toolchain/aarch64-rpi3-linux-gnu:gcc_toolchain --cpu=aarch64
    build:bbb --crosstool_top=//bazel/toolchain/arm-cortex_a8-linux-gnueabihf:gcc_toolchain --cpu=armv7
    
    build:platform_build --incompatible_enable_cc_toolchain_resolution
    build:rpi-platform --config=platform_build --platforms=//bazel/platforms:rpi
    build:bbb-platform --config=platform_build --platforms=//bazel/platforms:bbb

Invocation simplifies to:

    $ bazel build --config=rpi //:hello
    
    $ bazel build --config=rpi-platform //:hello

## Summary

Those steps should be valid for most of the C++ toolchains with slight modifications. Although the process of setting it up might be complicated in the end the benefits are really worth it. You get out-of-the-box cached, distributed and one-command build.

<!--kg-card-end: markdown-->