---
title: DRAFT Inside these little fellas called files - the ELF
layout: post
categories: software
---

![ELF Executable and Linkable Format diagram from Ange Albertini](https://en.wikipedia.org/wiki/File:ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)


## What is the ELF?

> the Executable and Linkable Format[2] (ELF, formerly named Extensible Linking Format), is a common standard file format for executable files, object code, shared libraries, and core dumps.

In Wikipedia there is a [great page](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) that explains the ELF details, so I am not digging there.
What we can do instead, is trying to read some ELF files to learn something in the process.

So, we compile a very simple program and read what ELF is produced.
The program is the ever-green hello world in C (did you miss it?)

```C
/* hello-world.c */
#include <stdio.h>

int main(void) {
  printf("Hello world!");
  return 0;
}
```

which we compile with

```
gcc hello-world.c
```

We have just generated the executable `a.out`.
To get a first look of what is inside the `a.out` file we can use the hex editor `xxd`

```
xxd a.out
```

This is an excerpt of the output:
```
[..]
000003b0: 0200 0000 0600 0000 0100 0000 0600 0000  ................
000003c0: 0000 8100 0000 0000 0600 0000 0000 0000  ................
000003d0: d165 ce6d 0000 0000 0000 0000 0000 0000  .e.m............
000003e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003f0: 1000 0000 1200 0000 0000 0000 0000 0000  ................
00000400: 0000 0000 0000 0000 4a00 0000 2000 0000  ........J... ...
00000410: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000420: 2200 0000 1200 0000 0000 0000 0000 0000  "...............
00000430: 0000 0000 0000 0000 6600 0000 2000 0000  ........f... ...
00000440: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000450: 7500 0000 2000 0000 0000 0000 0000 0000  u... ...........
00000460: 0000 0000 0000 0000 0100 0000 2200 0000  ............"...
00000470: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000480: 005f 5f63 7861 5f66 696e 616c 697a 6500  .__cxa_finalize.
00000490: 5f5f 6c69 6263 5f73 7461 7274 5f6d 6169  __libc_start_mai
000004a0: 6e00 7072 696e 7466 006c 6962 632e 736f  n.printf.libc.so
000004b0: 2e36 0047 4c49 4243 5f32 2e32 2e35 0047  .6.GLIBC_2.2.5.G
000004c0: 4c49 4243 5f32 2e33 3400 5f49 544d 5f64  LIBC_2.34._ITM_d
000004d0: 6572 6567 6973 7465 7254 4d43 6c6f 6e65  eregisterTMClone
000004e0: 5461 626c 6500 5f5f 676d 6f6e 5f73 7461  Table.__gmon_sta
000004f0: 7274 5f5f 005f 4954 4d5f 7265 6769 7374  rt__._ITM_regist
[..]
```

We can scroll all output to read it byte by byte, or generate an artistic graphical of the file with [this](https://github.com/matteo-briani/bytes-to-image).
The picture shows each byte of the `a.out` ELF file as a pixel-plot.
White pixel indicate a `0000` byte.

![a.out pixel plot](/assets/images/a_out_pixelplot.png)

One this that stands out immediately is the whiteness of the picture.
The information is localized in specific portions of the ELF file, leaving zero bytes in between.
Are the white spaces functional? Not all of them, and we can prove it by brutally write something on the `a.out` binary.

Let's take a text editor and do some hex mangling.
`xxd` support the conversion back and forth of the hex dump, so we can get the hex output with something like `xxd -c 127`, modify the output and revert back to binary with `xxd -r`.
The `-c` option allows you to decide how many octect per string you want to edit, thus becomes really easy to match the size of the picture to make some graphical editing.
Just check that your editor does not decide to do something fancy when saving the file, like a newline at the end.
In Vim, you prevent unwanted mangling by setting `:set binary` and `set noeol`.
This is our artistic result: making "hello world" even more hello-worldier.

![a.out mangled pixel plot](/assets/images/a_out_mangled_pixelplot.png)


