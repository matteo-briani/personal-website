---
title: DRAFT Zig and CLAP plugin - Part 1
layout: post
categories: audio
---

![Zig and CLAP logos]({{site.baseurl}}/assets/images/zig_and_clap.png)


## Zig and CLAP, what are we talking about


Directly from [zilang homepage](https://ziglang.org), we read that

> Zig is a general purpose programming language and toolchain for maintaining robust, optimal and reusable software.

More specifically, Zig is a new programming language focused on simplicity and no hidden control flow or allocations.
You can start writing Zig code in less than a week, just start playing with [this repository](https://github.com/ratfactor/ziglings) and you immediately get what I mean.

Another cool thing about Zig is that it is really easy to import and use C/C++ program or library.

Speaking of C library, [CLAP](https://github.com/free-audio/clap) - CLever Audio Plugin - is an audio plugin standard that offers a [very clear C interface](https://github.com/free-audio/clap).
Most importantly, CLAP is open source and liberally licensed.

Inspired by [a great example in C++](https://nakst.gitlab.io/tutorial/clap-part-1.html), we want to write a CLAP plugin in Zig, directly accessing the CLAP C interface.

## A CLAP overview

Writing a CLAP plugin means building a shared library that follows a specific structure, depending on our plugin features.
Luckily for us, the [CLAP C bindings](https://github.com/free-audio/clap) are well commented.

To make a CLAP plugin, we have to build a shared library with a `clap_plugin_entry_t` as our structure entry point.
Besides the entry point, the CLAP format follows the structure portrait in the following image.
Disclaimer: it is not UML on purpose (not a fan of it), it is more a scratch paper note.

![CLAP structure]({{site.baseurl}}/assets/images/clap_structure.svg)

With the overall picture in mind, we can start implementing our plugin in Zig.
For your information, CLAP has incredibly useful comments in its source code.
So, if you ever feel in doubt of anything, just take a look at the [source code](https://github.com/free-audio/clap); most probably your answer is laying down there.

## Get started

We get a copy of the CLAP client repository

```
git clone git@github.com:free-audio/clap.git
```

In this repo, we are only interested in the `include/clap/clap.h` header file, that contains all the `C` interfaces to build a CLAP plugin.
Our goal is to read this header file and provide a minimal implementation of the core functionality of a plugin.
To keep the example at the absolute bare minimum, we are going to create a plugin that output a constant sine wave at a fixed frequency.
Right now, we are just interested in building a plugin with Zig, later on we will expand and explore the CLAP features.
A CLAP plugin, from a coding prospective, is nothing more than a dynamic library; so let's initialize a zig library with

```
zig init-lib
```

This command will generate `build.zig` and `src/main.zig` files. BTW, if you poke your head inside `build.zig` you'll notice one of the cool feature of Zig: build instruction are written in Zig itself.
To build the library, we just launch

```
zig build
```

Up to now, CLAP and Zig are not interacting, let's change this.

## Bring CLAP definition within Zig

In our `build.zig` we want to include our CLAP definitions.
Moreover, we want to specify that our library is a shared one named "zig-clap-example".
We achieve this by modifying the `build.zig` with these lines:
```zig
const lib = b.addSharedLibrary(.{
    .name = "zig-clap-example",
    // In this case the main source file is merely a path, however, in more
    // complicated build scripts, this could be a generated file.
    .root_source_file = .{ .path = "src/main.zig" },
    .target = target,
    .optimize = optimize,
});
lib.addIncludePath(".");
```


Now, this is our new `build.zig`:

```zig
const std = @import("std");

// Although this function looks imperative, note that its job is to
// declaratively construct a build graph that will be executed by an external
// runner.
pub fn build(b: *std.Build) void {
    // Standard target options allows the person running `zig build` to choose
    // what target to build for. Here we do not override the defaults, which
    // means any target is allowed, and the default is native. Other options
    // for restricting supported target set are available.
    const target = b.standardTargetOptions(.{});

    // Standard optimization options allow the person running `zig build` to select
    // between Debug, ReleaseSafe, ReleaseFast, and ReleaseSmall. Here we do not
    // set a preferred release mode, allowing the user to decide how to optimize.
    const optimize = b.standardOptimizeOption(.{});

    const lib = b.addSharedLibrary(.{
        .name = "zig-clap-example",
        // In this case the main source file is merely a path, however, in more
        // complicated build scripts, this could be a generated file.
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });
    lib.addIncludePath(".");

    // This declares intent for the library to be installed into the standard
    // location when the user invokes the "install" step (the default step when
    // running `zig build`).
    b.installArtifact(lib);

    // Creates a step for unit testing. This only builds the test executable
    // but does not run it.
    const main_tests = b.addTest(.{
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    const run_main_tests = b.addRunArtifact(main_tests);

    // This creates a build step. It will be visible in the `zig build --help` menu,
    // and can be selected like this: `zig build test`
    // This will evaluate the `test` step rather than the default, which is "install".
    const test_step = b.step("test", "Run library tests");
    test_step.dependOn(&run_main_tests.step);
}
```

We are now able to import `clap.h` in `main.zig` using 

```zig
const clap = @cImport({
    @cInclude("clap/include/clap/clap.h");
});
```
That's it, simple as it gets, we just added all the CLAP definition that are needed to make our plugin.

It 's now time to start writing some Zig code!
<!-- ## Interfacing with CLAP -->

<!-- Taking a look at the CLAP source code, we need to implement and expose the following structure: -->

<!-- ```c -->
<!-- typedef struct clap_plugin_entry { -->
<!--    clap_version_t clap_version; // initialized to CLAP_VERSION -->

<!--    // This function must be called first, and can only be called once. -->
<!--    // -->
<!--    // It should be as fast as possible, in order to perform a very quick scan of the plugin -->
<!--    // descriptors. -->
<!--    // -->
<!--    // It is forbidden to display graphical user interface in this call. -->
<!--    // It is forbidden to perform user interaction in this call. -->
<!--    // -->
<!--    // If the initialization depends upon expensive computation, maybe try to do them ahead of time -->
<!--    // and cache the result. -->
<!--    // -->
<!--    // If init() returns false, then the host must not call deinit() nor any other clap -->
<!--    // related symbols from the DSO. -->
<!--    bool(CLAP_ABI *init)(const char *plugin_path); -->

<!--    // No more calls into the DSO must be made after calling deinit(). -->
<!--    void(CLAP_ABI *deinit)(void); -->

<!--    // Get the pointer to a factory. See factory/plugin-factory.h for an example. -->
<!--    // -->
<!--    // Returns null if the factory is not provided. -->
<!--    // The returned pointer must *not* be freed by the caller. -->
<!--    const void *(CLAP_ABI *get_factory)(const char *factory_id); -->
<!-- } clap_plugin_entry_t; -->
<!-- ``` -->

<!-- Does it means we have to translate every line of the previous snippet in Zig?  -->
<!-- No, we can hack our way out of this by re-building the plugin with `zig build` and poke our head inside the directory `zig-cache`. -->

<!-- After some crawling around, we find out what Zig generated when reading our C code: -->

<!-- ```zig -->
<!-- pub const struct_clap_plugin_entry = extern struct { -->
<!--     clap_version: clap_version_t, -->
<!--     init: ?*const fn ([*c]const u8) callconv(.C) bool, -->
<!--     deinit: ?*const fn () callconv(.C) void, -->
<!--     get_factory: ?*const fn ([*c]const u8) callconv(.C) ?*const anyopaque, -->
<!-- }; -->
<!-- pub const clap_plugin_entry_t = struct_clap_plugin_entry; -->
<!-- ``` -->

<!-- Well, now we know what we should implement in Zig to bring the plugin to life. -->

<!-- Let's find out `clap_version_t` type as well -->

<!-- ```zig -->
<!-- pub const struct_clap_version = extern struct { -->
<!--     major: u32, -->
<!--     minor: u32, -->
<!--     revision: u32, -->
<!-- }; -->
<!-- pub const clap_version_t = struct_clap_version; -->
<!-- ``` -->

