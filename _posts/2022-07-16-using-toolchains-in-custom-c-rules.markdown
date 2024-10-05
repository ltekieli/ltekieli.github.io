---
layout: post
title: Using toolchains in custom Bazel C++ rules
date: '2022-07-16 17:33:08'
---

After writing a sketch of own C++ rules, which you can find here:

- [Writing custom Bazel C++ rules](/writing-custom-bazel-c-rules/)

we might extend this example with toolchain support. In order to not conflict with existing C++ toolchain implementation we need to define a new toolchain type using a special rule:

    toolchain_type(
        name = "cxx_toolchain_type",
        visibility = ["//visibility:public"],
    )

Next, the toolchain needs to be declared:

    toolchain(
        name = "toolchain",
        exec_compatible_with = [
            "@platforms//os:linux",
            "@platforms//cpu:x86_64",
        ],
        target_compatible_with = [
            "@platforms//os:linux",
            "@platforms//cpu:x86_64",
        ],
        toolchain = ":default_cxx_toolchain",
        toolchain_type = ":cxx_toolchain_type",
    )

Here we declared a toolchain called " **_toolchain_**" which can run on Linux, x86\_64 systems and produces code for the same target. The toolchain type, as well as the implementation rule is assigned. The "_ **:default\_cxx\_toolchain** _" can be any rule that returns **_platform\_common.ToolchainInfo_** provider.

    load("//bazel/rules/cxx:toolchain.bzl", "cxx_toolchain")
    
    cxx_toolchain(
        name = "default_cxx_toolchain",
    )

The implementation of the **_cxx\_toolchain_** rule can be as simple as:

    def _cxx_toolchain_impl(ctx):
        return [
            platform_common.ToolchainInfo(
                compiler = "g++",
                archiver = "ar",
            )
        ]
    
    cxx_toolchain = rule(
        implementation = _cxx_toolchain_impl,
        attrs = {},
    )

The rule does not need any attributes and it returns the provider with a key-value list that can be accessed from the rules depending on this toolchain type. Here we hard-coded the compiler and the archiver, so that the rules get those tools injected from outside.

Important is to let Bazel know about or newly defined toolchain by registering it in the WORKSPACE:

    register_toolchains("//bazel/rules/cxx:toolchain")

The rules can access the toolchain directly from the context:

    toolchain = ctx.toolchains["//bazel/rules/cxx:cxx_toolchain_type"]
    ...
    toolchain.compiler // equals "g++"

Every rule that want's to use this toolchain type needs to declare it:

    cxx_binary = rule(
        _cxx_binary_impl,
        attrs = {
            "srcs": attr.label_list(
                allow_files = [".cpp", ".h"],
                doc = "Source files to compile for this binary",
            ),
            "deps": attr.label_list(
                providers = [CxxInfo],
            ),
        },
        doc = "Builds an executable program from C++ source code",
        executable = True,
        toolchains = ["//bazel/rules/cxx:cxx_toolchain_type"],
    )

With this our build should now use the defined toolchain for building. If we don't register this toolchain we will get the following error:

    ERROR: /home/tekieli/workspace/bazel_rules_cxx/lib/func2/BUILD:3:19: While resolving toolchains for target //lib/func2:func2: no matching toolchains found for types //bazel/rules/cxx:cxx_toolchain_type

