---
layout: post
title: Bazel AVR toolchain from scratch (no_legacy_features)
date: '2023-06-28 17:36:43'
---

Even when you define your own cross compilation toolchain in Bazel, it still comes with a set of predefined, 'sane' defaults for the compilation and linking flags. But if you want to take full control over the generated commands, you can turn off all the default features, that are added by Bazel. This is done using the 'no\_legacy\_features' feature.

Configuring a cross compilation toolchain has some boilerplate that I've described here:

- [Cross compiling with Bazel](/cross-compiling-with-bazel/)

We will only focus on getting the compile and link commands working.

Full code for this excercise can be found here:

- [https://github.com/ltekieli/bazel_avr_toolchain](https://github.com/ltekieli/bazel_avr_toolchain)

Let's start with the return value of the toolchain configuration implementation function:

<figure class="kg-card kg-code-card"><pre><code>def _impl(ctx):
    
    ...

    return cc_common.create_cc_toolchain_config_info(
        ctx = ctx,
        action_configs = action_configs,
        features = features,
        toolchain_identifier = "unkown",
        target_system_name = "unknown",
        target_cpu = "unknown",
        target_libc = "unknown",
        compiler = "unknown",
        tool_paths = tool_paths,
    )</code></pre>
<figcaption>i</figcaption></figure>

"features" attribtue should point to a list of "feature" data structures, one of which is the "no\_legacy\_feature":

    no_legacy_features = feature(name = "no_legacy_features")
    
    features = [
        ...
        no_legacy_features,
    ]

This will effectively disable any command generation and we are on our own now. Bazel will complain that certain actions need to be defined:

    File "/virtual_builtins_bzl/common/cc/cc_library.bzl", line 52, column 72, in _cc_library_impl
    Error in compile: Expected action_config for 'c++-compile' to be configured

These actions are:

- c++compile
- c++-link-executable
- c++-link-static-library
- strip

c++compile and strip are mandatory. c++-link-static-library might be optional if you never define a cc\_library target.

Let's define those actions:

        cpp_compile_action = action_config(
            action_name = ACTION_NAMES.cpp_compile,
            tools = [
                tool(
                    path = "wrappers/avr-gcc",
                ),
            ],
        )
    
        cpp_link_executable_action = action_config(
            action_name = ACTION_NAMES.cpp_link_executable,
            tools = [
                tool(
                    path = "wrappers/avr-gcc",
                ),
            ],
        )
    
        cpp_link_static_library_action = action_config(
            action_name = ACTION_NAMES.cpp_link_static_library,
            tools = [
                tool(
                    path = "wrappers/avr-ar",
                ),
            ],
        )
    
        strip_action = action_config(
            action_name = ACTION_NAMES.strip,
            tools = [
                tool(
                    path = "wrappers/avr-strip",
                ),
            ],
        )

Minimal definition consists only of the tool that needs to be used.

In order to set some arguments to the tools, default features need to be defined. Default features are those marked with "enabled = True".

##### Compilation

        default_compiler_flags = feature(
            name = "default_compiler_flags",
            enabled = True,
            flag_sets = [
                flag_set(
                    actions = [ACTION_NAMES.cpp_compile],
                    flag_groups = [
                        #
                        # Compile only.
                        #
                        flag_group(
                            flags = [
                                "-c",
                            ],
                        ),
                        #
                        # Use C++.
                        #
                        flag_group(
                            flags = [
                                "-xc++",
                            ],
                        ),
                        #
                        # Optimize for size by default
                        #
                        flag_group(
                            flags = [
                                "-Os",
                            ],
                        ),
                        #
                        # Do not canonicalize paths. This is needed because everything is a symlink
                        # in Bazel. Canonicalizing will resolve those and confuse Bazel, which will think
                        # access outside of sandobox is requested.
                        #
                        flag_group(
                            flags = [
                                "-no-canonical-prefixes",
                                "-fno-canonical-system-headers",
                            ],
                        ),
                        #
                        # Override autogenerated macros with fixed values.
                        #
                        flag_group(
                            flags = [
                                "-Wno-builtin-macro-redefined",
                                "-D __DATE__ =\"redacted\"",
                                "-D __TIMESTAMP__ =\"redacted\"",
                                "-D __TIME__ =\"redacted\"",
                            ],
                        ),
                        #
                        # Generates dependencies to headers for every object file.
                        # Bazel uses this to find usage of headers outside of the sandbox.
                        #
                        flag_group(
                            flags = [
                                "-MD",
                                "-MF",
                                "%{dependency_file}",
                            ],
                        ),
                        #
                        # Add defines.
                        #
                        flag_group(
                            iterate_over = "preprocessor_defines",
                            flags = [
                                "-D%{preprocessor_defines}",
                            ],
                        ),
                        #
                        # Use input file and declare output file.
                        #
                        flag_group(
                            flags = [
                                "%{source_file}",
                                "-o",
                                "%{output_file}",
                            ],
                        ),
                    ],
                ),
            ],
        )

##### Archive creation

    default_archive_flags = feature(
        name = "default_archive_flags",
        enabled = True,
        flag_sets = [
            flag_set(
                actions = [ACTION_NAMES.cpp_link_static_library],
                flag_groups = [
                    #
                    # Create archive and and add object files.
                    #
                    flag_group(
                        flags = [
                            "rc",
                        ],
                    ),
                    #
                    # Name of the archive
                    #
                    flag_group(
                        flags = [
                            "%{output_execpath}",
                        ],
                    ),
                    #
                    # Object files to archive.
                    #
                    flag_group(
                        iterate_over = "libraries_to_link",
                        flags = [
                            "%{libraries_to_link.name}",
                        ],
                    ),
                ],
            ),
        ],
    )

##### Linking

    default_linker_flags = feature(
        name = "default_linker_flags",
        enabled = True,
        flag_sets = [
            flag_set(
                actions = [
                    ACTION_NAMES.cpp_link_executable,
                ],
                flag_groups = [
                    #
                    # Declare output file.
                    #
                    flag_group(
                        flags = [
                            "-o",
                            "%{output_execpath}",
                        ],
                    ),
                    #
                    # Always link with math library
                    #
                    flag_group(
                        flags = [
                            "-lm",
                        ],
                    ),
                    #
                    # Link object files or archives.
                    #
                    flag_group(
                        iterate_over = "libraries_to_link",
                        flags = [
                            "%{libraries_to_link.name}",
                        ],
                    ),
                    #
                    # Bazel stores linker flags in a param file which needs to be given as input.
                    #
                    flag_group(
                        flags = [
                            "@%{linker_param_file}",
                        ],
                    ),
                ],
            ),
        ],
    )

In case of linking, note the linker\_param\_file. This file contains all the arguments collected in one file, which is then passed to the tool. This is done to prevent too long invocations.

In case of AVR platform the "-mmcu" flag needs to be defined both for compilation and linking. We will define it as optional features, which need to be then enabled by specifying features on the build command:

        atmega32 = feature(
            name = "atmega32",
            enabled = False,
            provides = ["avr_mcu_type"],
            flag_sets = [
                flag_set(
                    actions = [
                        ACTION_NAMES.cpp_compile,
                        ACTION_NAMES.cpp_link_executable,
                    ],
                    flag_groups = [
                        #
                        # Setup MCU type.
                        #
                        flag_group(
                            flags = [
                                "-mmcu=atmega32",
                            ],
                        ),
                    ],
                ),
            ],
        )
    
        atmega32u4 = feature(
            name = "atmega32u4",
            provides = ["avr_mcu_type"],
            enabled = False,
            flag_sets = [
                flag_set(
                    actions = [
                        ACTION_NAMES.cpp_compile,
                        ACTION_NAMES.cpp_link_executable,
                    ],
                    flag_groups = [
                        #
                        # Setup MCU type.
                        #
                        flag_group(
                            flags = [
                                "-mmcu=atmega32u4",
                            ],
                        ),
                    ],
                ),
            ],
        )

Those are then chosen during build by a flag:

    $ bazel build ***other args*** --features=atmega32u4

With all the above definitions, we are now able to build a cross compiled executable:

    $ bazel build -s --config=avr //:blink
    Starting local Bazel server and connecting to it...
    INFO: Analyzed target //:blink (40 packages loaded, 1816 targets configured).
    INFO: Found 1 target...
    SUBCOMMAND: # //:delay [action 'Compiling delay.cpp', configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495, execution platform: @local_config_platform//:host]
    (cd /home/lukasz/.cache/bazel/_bazel_lukasz/3af514bd0efe34b0bbf2457ff9d9d73b/execroot/ __main__ && \
      exec env - \
        PATH=/home/lukasz/.cache/bazelisk/downloads/bazelbuild/bazel-6.2.1-linux-x86_64/bin:/home/lukasz/local/bin:/home/lukasz/.local/bin:/home/lukasz/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin \
        PWD=/proc/self/cwd \
      bazel/toolchain/avr/wrappers/avr-gcc '-mmcu=atmega32u4' -c -xc++ -Os -no-canonical-prefixes -fno-canonical-system-headers -Wno-builtin-macro-redefined '-D __DATE__ ="redacted"' '-D __TIMESTAMP__ ="redacted"' '-D __TIME__ ="redacted"' -MD -MF bazel-out/k8-fastbuild/bin/_objs/delay/delay.d '-DF_CPU=16000000UL' '-DBAZEL_CURRENT_REPOSITORY=""' delay.cpp -o bazel-out/k8-fastbuild/bin/_objs/delay/delay.o)
    # Configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495
    # Execution platform: @local_config_platform//:host
    SUBCOMMAND: # //:blink [action 'Compiling blink.cpp', configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495, execution platform: @local_config_platform//:host]
    (cd /home/lukasz/.cache/bazel/_bazel_lukasz/3af514bd0efe34b0bbf2457ff9d9d73b/execroot/ __main__ && \
      exec env - \
        PATH=/home/lukasz/.cache/bazelisk/downloads/bazelbuild/bazel-6.2.1-linux-x86_64/bin:/home/lukasz/local/bin:/home/lukasz/.local/bin:/home/lukasz/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin \
        PWD=/proc/self/cwd \
      bazel/toolchain/avr/wrappers/avr-gcc '-mmcu=atmega32u4' -c -xc++ -Os -no-canonical-prefixes -fno-canonical-system-headers -Wno-builtin-macro-redefined '-D __DATE__ ="redacted"' '-D __TIMESTAMP__ ="redacted"' '-D __TIME__ ="redacted"' -MD -MF bazel-out/k8-fastbuild/bin/_objs/blink/blink.d '-DBAZEL_CURRENT_REPOSITORY=""' blink.cpp -o bazel-out/k8-fastbuild/bin/_objs/blink/blink.o)
    # Configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495
    # Execution platform: @local_config_platform//:host
    SUBCOMMAND: # //:delay [action 'Linking libdelay.a', configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495, execution platform: @local_config_platform//:host]
    (cd /home/lukasz/.cache/bazel/_bazel_lukasz/3af514bd0efe34b0bbf2457ff9d9d73b/execroot/ __main__ && \
      exec env - \
        PATH=/home/lukasz/.cache/bazelisk/downloads/bazelbuild/bazel-6.2.1-linux-x86_64/bin:/home/lukasz/local/bin:/home/lukasz/.local/bin:/home/lukasz/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin \
        PWD=/proc/self/cwd \
      bazel/toolchain/avr/wrappers/avr-ar rc bazel-out/k8-fastbuild/bin/libdelay.a bazel-out/k8-fastbuild/bin/_objs/delay/delay.o)
    # Configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495
    # Execution platform: @local_config_platform//:host
    SUBCOMMAND: # //:blink [action 'Linking blink', configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495, execution platform: @local_config_platform//:host]
    (cd /home/lukasz/.cache/bazel/_bazel_lukasz/3af514bd0efe34b0bbf2457ff9d9d73b/execroot/ __main__ && \
      exec env - \
        PATH=/home/lukasz/.cache/bazelisk/downloads/bazelbuild/bazel-6.2.1-linux-x86_64/bin:/home/lukasz/local/bin:/home/lukasz/.local/bin:/home/lukasz/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin \
        PWD=/proc/self/cwd \
      bazel/toolchain/avr/wrappers/avr-gcc @bazel-out/k8-fastbuild/bin/blink-2.params)
    # Configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495
    # Execution platform: @local_config_platform//:host
    Target //:blink up-to-date:
      bazel-bin/blink
    INFO: Elapsed time: 4.805s, Critical Path: 0.26s
    INFO: 8 processes: 4 internal, 4 linux-sandbox.
    INFO: Build completed successfully, 8 total actions

In order to be able to write our program to the AVR device, we need to have the hex format of the executable which we can obtain by writing a custom rule:

    load("@bazel_tools//tools/cpp:toolchain_utils.bzl", "find_cpp_toolchain", "use_cpp_toolchain")
    
    def _impl(ctx):
        src = ctx.attr.src.files.to_list()[0]
        binary = ctx.actions.declare_file(ctx.label.name + ".hex")
        toolchain = find_cpp_toolchain(ctx)
    
        args = ctx.actions.args()
        args.add("-O", "ihex")
        args.add(src)
        args.add(binary)
    
        ctx.actions.run(
            executable = toolchain.objcopy_executable,
            outputs = [binary],
            inputs = depset(
                direct = [src],
                transitive = [toolchain.all_files],
            ),
            arguments = [args],
            mnemonic = "Firmware",
            use_default_shell_env = True,
        )
    
        return [
            DefaultInfo(
                files = depset([binary]),
            ),
        ]
    
    cc_firmware = rule(
        implementation = _impl,
        attrs = {
            "src": attr.label(allow_single_file = True),
        },
        toolchains = use_cpp_toolchain(),
    )

This rule uses the C++ toolchain. It will access the objcopy\_executable. In order to allow this, our toolchain definition needs to be extended with common tool paths:

        #
        # In order to get access to tools from custom rules, need to specify them here.
        # Because of the "no_legacy_features", those won't be anyway used.
        #
        tool_paths = [
            tool_path(
                name = "ar",
                path = "not-used",
            ),
            tool_path(
                name = "cpp",
                path = "not-used",
            ),
            tool_path(
                name = "gcc",
                path = "not-used",
            ),
            tool_path(
                name = "ld",
                path = "not-used",
            ),
            tool_path(
                name = "nm",
                path = "not-used",
            ),
            tool_path(
                name = "objcopy",
                path = "wrappers/avr-objcopy",
            ),
            tool_path(
                name = "objdump",
                path = "not-used",
            ),
            tool_path(
                name = "strip",
                path = "not-used",
            ),
        ]

Unfortunately, the "cc\_common.create\_cc\_toolchain\_config\_info" API requires to specify all the tools. However, dummy values can be provided for the tools we don't want to expose.

And with that implemented, building firmware for AVR platform is now possible:

    $ bazel build -s --config=avr //:blink_firmware
    INFO: Analyzed target //:blink_firmware (0 packages loaded, 1 target configured).
    INFO: Found 1 target...
    SUBCOMMAND: # //:blink_firmware [action 'Firmware blink_firmware.hex', configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495, execution platform: @local_config_platform//:host]
    (cd /home/lukasz/.cache/bazel/_bazel_lukasz/3af514bd0efe34b0bbf2457ff9d9d73b/execroot/ __main__ && \
      exec env - \
        PATH=/home/lukasz/.cache/bazelisk/downloads/bazelbuild/bazel-6.2.1-linux-x86_64/bin:/home/lukasz/local/bin:/home/lukasz/.local/bin:/home/lukasz/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin \
      bazel/toolchain/avr/wrappers/avr-objcopy -O ihex bazel-out/k8-fastbuild/bin/blink bazel-out/k8-fastbuild/bin/blink_firmware.hex)
    # Configuration: 8f48f689bace00c389f03f237b0b75c2bce1802398fae79de2de08f16edc6495
    # Execution platform: @local_config_platform//:host
    Target //:blink_firmware up-to-date:
      bazel-bin/blink_firmware.hex
    INFO: Elapsed time: 0.203s, Critical Path: 0.05s
    INFO: 2 processes: 1 internal, 1 linux-sandbox.
    INFO: Build completed successfully, 2 total actions

