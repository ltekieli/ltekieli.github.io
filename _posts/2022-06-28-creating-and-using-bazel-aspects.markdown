---
layout: post
title: Using Bazel aspects to run clang-format
date: '2022-06-28 16:04:26'
tags:
- bazel
- clang-format
- c
- aspect
- rule
---

Bazel's aspects are next to rules the major building block used to extend the functionality of your build system. Contrary to rules, aspects allow to run particular functionality over an existing dependency graph. Aspect are documented pretty well on the bazel documentation page:

- [https://bazel.build/rules/aspects](https://bazel.build/rules/aspects)


Let's build on top of the file counter example and extend it in the following way:

- instead of counting files, store the collected files in a new provider structure
- attach the aspect to a custom rule
- use the rule to dump the collected files to a text file
- run clang-format over the files from the above text files

The example code can be found in github:
- [https://github.com/ltekieli/bazel_aspect_clang_format](https://github.com/ltekieli/bazel_aspect_clang_format)

First, let's start with defining the aspect:

    collected_cpp_files_aspect = aspect(
        implementation = _collected_cpp_files_aspect_impl,
        attr_aspects = ['deps'],
    )

This defines that our aspect **_collected\_cpp\_files\_aspect_** will propagate over the _ **deps** _ attribute of the rule it is attached to. It specifies to use the following implementation function:

    def _collected_cpp_files_aspect_impl(target, ctx):
        collected_cpp_files = []
        collected_cpp_files += _extract_cpp_files(ctx.rule, 'srcs')
        collected_cpp_files += _extract_cpp_files(ctx.rule, 'hdrs')
        collected_cpp_files_deps = [dep[CollectedCppFilesInfo].files for dep in ctx.rule.attr.deps]
        return [CollectedCppFilesInfo(files = depset(collected_cpp_files, transitive=collected_cpp_files_deps))]

This aspect will collect files from **_srcs_** and **_hdrs_** attributes. The return type is a list of providers, in this case we return only the _ **CollectedCppFilesInfo** _ provider, which holds a depset of source files collected in the current rule, as well as transitive providers of the same type coming directly from dependencies of this rule. This custom provider is defined as follows:

    CollectedCppFilesInfo = provider(
        fields = {
            'files': 'depset of cpp files'
        }
    )

In order to make use of our freshly created aspect, let's attach it to a custom rule:

    collect_cpp_files = rule(
        implementation = _collect_cpp_files_impl,
        attrs = {
            "deps": attr.label_list(
                        aspects = [collected_cpp_files_aspect],
                        providers = [CcInfo])
        },
    )

This rule has one attribute **_deps_** , which can hold rules that produce a **_CcInfo_** provider. We attach our aspect to this rule, which means that the rule will have access to **_CollectedCppFilesInfo_** provider as well.

The implementation function can resolve the depsets and store the collected sources in a text file:

    accumulated = []
    for dep in ctx.attr.deps:
        accumulated.append(dep[CollectedCppFilesInfo].files)
    
    content = ""
    for cpp_filename in depset(transitive=accumulated).to_list():
        content += "%s\n" % cpp_filename.path

Having the aspect and the rule, we can now setup the pieces to run clang-format based on the created text files.

To ease the work a bit, let's introduce a macro that will combine all the boilerplate:

    def clang_format(name="empty"):
        if "clang_format" in native.existing_rules():
            fail("clang_format rule already defined")
    
        deps = []
        for rule_name, rule in native.existing_rules().items():
            if rule["kind"] in ["cc_binary", "cc_library"]:
                deps.append(rule_name)
    
        collect_cpp_files(
            name = "clang_format_cpp_files",
            deps = deps
        )
    
        native.py_binary(
            name = "clang_format",
            srcs = ["//bazel/tools/clang_format:clang_format_runner.py"],
            main = "clang_format_runner.py",
            args = [
                "$(location :clang_format_cpp_files)",
            ],
            data = [":clang_format_cpp_files"]
        )

The macro needs to be added to the end of every BUILD file which contains the targets we want to run clang-format for. It will pick up all defined **_cc\_binary_** and **_cc\_library_** rules and put those as the **_deps_** argument of the **_collect\_cpp\_files_** rule. Last but not least, it will setup a **_py\_binary_** running the **_clang\_format\_runner.py_** script, with the text files created by the cpp source collection rule.

The result of running this **_py\_binary_** looks like this:

    $ bazel run //:clang_format 
    INFO: Analyzed target //:clang_format (0 packages loaded, 0 targets configured).
    INFO: Found 1 target...
    Target //:clang_format up-to-date:
      bazel-bin/clang_format
    INFO: Elapsed time: 0.232s, Critical Path: 0.00s
    INFO: 1 process: 1 internal.
    INFO: Build completed successfully, 1 total action
    INFO: Build completed successfully, 1 total action
    Running clang-format for: lib/log/log.cpp
    Running clang-format for: lib/log/log.h
    Running clang-format for: main.cpp

clang-format was applied to the //:hello target as well as to the dependency //lib/log:log.

The important thing is that in this setup the clang-format is applied to the files that are used for building given targets. This means, if the setup contain conditional configurations, formatting will respect those. Example:

    $ bazel run //:clang_format --//:use_main_2=true
    INFO: Build option --//:use_main_2 has changed, discarding analysis cache.
    INFO: Analyzed target //:clang_format (0 packages loaded, 329 targets configured).
    INFO: Found 1 target...
    Target //:clang_format up-to-date:
      bazel-bin/clang_format
    INFO: Elapsed time: 0.639s, Critical Path: 0.01s
    INFO: 2 processes: 2 internal.
    INFO: Build completed successfully, 2 total actions
    INFO: Build completed successfully, 2 total actions
    Running clang-format for: lib/log/log.cpp
    Running clang-format for: lib/log/log.h
    Running clang-format for: main2.cpp

