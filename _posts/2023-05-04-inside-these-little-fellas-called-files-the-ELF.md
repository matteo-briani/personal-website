---
title: DRAFT Inside these little fellas called files - the ELF
layout: post
categories: software
---

![ELF Executable and Linkable Format diagram from Ange Albertini](https://upload.wikimedia.org/wikipedia/commons/e/e4/ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)


## What is the ELF?

> the Executable and Linkable Format[2] (ELF, formerly named Extensible Linking Format), is a common standard file format for executable files, object code, shared libraries, and core dumps.

In Wikipedia there is a [great page](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) that explains the ELF details, so I am not digging there, at least not now.
What we can do instead, is trying to play with some ELF files and learn something in the process.

To start, we compile a very simple program and read what ELF is produced.
The program is the ever-green hello world in C (did you miss it?)

```C
#include <stdio.h>

int main(void) {
  printf("Hello world!");
  return 0;
}
```

which we compile with

```
gcc hello.c
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

We can scroll all output to read it byte by byte, but to have a better overview of the file I wrote [this](https://github.com/matteo-briani/bytes-to-image) script to render the file as a `png` image.
The picture shows each byte of the `a.out` ELF file as a pixel-plot.
White pixels indicate a `0000` byte.

![a.out pixel plot]({{site.baseurl}}/assets/images/a_out_pixelplot.png)

One thing that stands out immediately is the whiteness of the picture.
The information is localized in specific portions of the ELF file, leaving zero bytes in between.
Are the white spaces functional? Not all of them, and we can prove it by brutally write something on the `a.out` binary.

## Digression, edit a hex file - the simple tools way

Let's take a text editor and do some hex mangling.
`xxd` support the conversion back and forth from the binary to the hex dump.
So, we can get the hex output with `xxd -c 127`, modify the output and revert back to binary with `xxd -r`.
By the way, the number `127` isn't magical, it just the size of the image rendered with [this tool](https://github.com/matteo-briani/bytes-to-image).
The `-c` option allows you to decide how many octects per string you want to edit, thus becomes really easy to match the size of the picture and make some graphical editing.
On last extra note: just check that your editor does not decide to do something fancy when saving the file, like a newline at the end of it when saving it.
In Vim, you prevent unwanted extra mangling by setting `:set binary` and `set noeol`.

## Can we feel a bit of that emptiness?

So, with the help of the hex editor we can modify the ELF file and change some zero bytes.
This is our artistic result: making "hello world" even more hello-worldier, and the program still works!

![a.out mangled pixel plot]({{site.baseurl}}/assets/images/a_out_mangled_pixelplot.png)

Of course, we couldn't just write just anywhere in the ELF, but I did my homework and I choose a nice spot.
You don't need to trust me that the ELF keeps on working fine; just download both files: [the original]({{site.baseurl}}/assets/code/the-elf-hello/a.out.original) and [the modified]({{site.baseurl}}/assets/code/the-elf-hello/a.out.mangled).
Anyway, if you don't want to download the files, here is the output of both and their `md5sum`:

```
>./a.out.original
Hello world!
>./a.out.mangled
Hello world!
>md5sum a.out*
2c29a699f239839dc15c2edc9213dc9b  a.out.mangled
64edee7d4de7cd40edd9d2219b154031  a.out.original
```

## An explanation, please

