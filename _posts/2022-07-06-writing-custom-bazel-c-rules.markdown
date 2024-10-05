---
layout: post
title: Writing custom Bazel C++ rules
date: '2022-07-06 15:17:53'
tags:
- bazel
- c
- rule
- automation
- build-system
---

Implementing a language support in Bazel can help one better understand the existing functionalities that this build system already provides. The migration process for Bazel C++ rules is still not finished at the moment, so the implementation for C++ rules is not fully done in starlark. We can try to implement the simplest possible full-starlark support for C++. For this the following operations are needed:

- Compilation

    $ g++ -c -o file1.o file1.cpp

This line will compile the file **_file1.cpp_** to an object file **_file1.o_**

- Archive creation

    ar crs lib1.a file1.o file2.o

Object files **_file1.o_** and **_file2.o_** will be put into an archive **_lib1.a_**

- Linking

    $ g++ -o app main.o file3.o lib1.a

Object files **_main.o_** , **_file3.o_** and archive **_lib1.a_** are linked together into one application app.

The following code is available at:

- [https://github.com/ltekieli/bazel_rules_cxx](https://github.com/ltekieli/bazel_rules_cxx)

Let's create functions that implement the above commands. Those functions will be later used in language support rules.

    def _cxx_compile(ctx, src, hdrs, out):
        args = ctx.actions.args()
        args.add("-c")
        args.add("-o", out)
        args.add("-iquote", ".")
        args.add(src)
    
        ctx.actions.run(
            executable = "g++",
            outputs = [out],
            inputs = [src] + hdrs,
            arguments = [args],
            mnemonic = "CxxCompile",
            use_default_shell_env = True,
        )

_ **\_cxx\_compile** _ function creates an action for compiling a given C++ file. We use the ctx object, that is supplied for all rules. All inputs and outputs needs to be declared, so that during execution the files are available in the sandboxed environment. This means all dependent headers needs to also be known. Since Bazel always runs the build by default from the top workspace directory, we use a little hack to add the current working directory in the include paths with _ **-iquote ".".** _

Similarly, the implementation for archiving object files follows:

    def _cxx_archive(ctx, objs, out):
        args = ctx.actions.args()
        args.add("crs", out)
        args.add_all(objs)
    
        ctx.actions.run(
            executable = "ar",
            outputs = [out],
            inputs = objs,
            arguments = [args],
            mnemonic = "CxxArchive",
            use_default_shell_env = True,
        )

Last but not least, the linking of object files:

    def _cxx_link(ctx, objs, out):
        args = ctx.actions.args()
        args.add("-o", out)
        args.add_all(objs)
    
        ctx.actions.run(
            executable = "g++",
            outputs = [out],
            inputs = objs,
            arguments = [args],
            mnemonic = "CxxLink",
            use_default_shell_env = True,
        )

The compile and archive functions should be enough to create a rule that will produce archives:

    def _cxx_static_library_impl(ctx):
        hdrs = _collect_headers(ctx)
        objs = _compile_sources(ctx, hdrs)
    
        static_library = ctx.actions.declare_file(ctx.label.name + ".a")
        _cxx_archive(
            ctx,
            objs = objs,
            out = static_library,
        )
    
        return [
            DefaultInfo(
                files = depset([static_library]),
            ),
            CxxInfo(
                hdrs = depset(ctx.files.hdrs, transitive = [dep[CxxInfo].hdrs for dep in ctx.attr.deps]),
                archives = depset([static_library], transitive = [dep[CxxInfo].archives for dep in ctx.attr.deps]),
            ),
        ]
    
    cxx_static_library = rule(
        _cxx_static_library_impl,
        attrs = {
            "hdrs": attr.label_list(
                allow_files = [".h"],
                doc = "Public header files for this static library",
            ),
            "srcs": attr.label_list(
                allow_files = [".cpp", ".h"],
                doc = "Source files to compile for this binary",
            ),
            "deps": attr.label_list(
                providers = [CxxInfo],
            ),
        },
        doc = "Builds a static library from C++ source code",
    )

The rules has three attributes specified:

- hdrs, which holds the public headers of this archive
- srcs, which holds source files and private headers
- deps, which might point to other archives

Data transfer between the rules is realized by introducing a new provider CxxInfo, that holds public headers from all dependencies as well as the archives needed for linking:

    CxxInfo = provider(
        fields = {
            "hdrs": "depset of header files",
            "archives": "depset of archives",
        },
    )

The rule, first collects all the public headers and stores them in a list. Those are then used to compile source files. The resulting object files are then used to create the archive. The CxxInfo provider is returned with the headers and an archive produced in this rule,as well as the transitive dependencies.

The binary rule is very similar:

    def _cxx_binary_impl(ctx):
        hdrs = _collect_headers(ctx)
        objs = _compile_sources(ctx, hdrs)
        objs += depset(transitive = [dep[CxxInfo].archives for dep in ctx.attr.deps]).to_list()
    
        executable = ctx.actions.declare_file(ctx.label.name)
        _cxx_link(
            ctx,
            objs = objs,
            out = executable,
        )
    
        return [DefaultInfo(
            files = depset([executable]),
            executable = executable,
        )]
    
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
    )

The differences are:

- it does not have a hdrs attribute
- it does not return a CxxInfo provider
- it is marked executable
- the object files as well as archives from _ **deps** _ are used for linking

Setting up a build with those rules can produce a following result:

    $ bazel run //app:main 
    INFO: Analyzed target //app:main (6 packages loaded, 13 targets configured).
    INFO: Found 1 target...
    Target //app:main up-to-date:
      bazel-bin/app/main
    INFO: Elapsed time: 5.092s, Critical Path: 4.49s
    INFO: 9 processes: 3 internal, 6 linux-sandbox.
    INFO: Build completed successfully, 9 total actions
    INFO: Build completed successfully, 9 total actions
    Hello World!
    Hello func1!
    Hello func2!

The rules work, but they are very limited:

- no support for shared libraries
- no support for customization of compiler and linker flags
- hard coded tools

Supporting shared libraries might be tricky, but customizing flags and tools can be done with the support of toolchains.

