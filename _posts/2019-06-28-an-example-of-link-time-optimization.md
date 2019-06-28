---
layout: post
title: An Example of Link-time Optimization
---

**Link-time optimization (LTO)** is a technique to perform some program
optimizations at link time. We know that it is the compiler's job to compile
source files into object files with possible optimizations at compilation time.
And the linker's job is to link object files into an executable program at link
time. Then why do we still need to do optimization at link time if object files
are already optimized at compilation time?

The reason is that some optimizations are impossible to do at compilation time.
Programming languages like C compile one source file at a time, which means that
the compiler can only perform optimizations using the information of a specific
file. But most nontrivial programs contain multiple source files relying on each
other. In that case, the compiler's hands are tied because it can't see the
whole picture.

That's why we need to do optimization at link time when all object files are
ready. Let me show you the power of LTO with a simple example. In this example,
I use GCC 8.3.0 and I assume that you already know some basics about GCC and
assembly language.

Here is a single-file program:

``` c
// prog1.c

int inc(int a) {
  return a + 1;
}

int main(int argc, char* argv[]) {
  int r = 0;
  for (int i = 0; i < 100; i++) {
    r = inc(r);
  }
  return r;
}
```

This program *prog1* calls `inc` 100 times and returns 100 as a result. If we
compile the program and check the executable, we will get something like this:

```
$ gcc prog1.c -o prog1 -O2
$ objdump -d prog1

...

0000000000001040 <main>:
    1040:    b8 64 00 00 00          mov    $0x64,%eax
    1045:    c3                      retq
    1046:    66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
    104d:    00 00 00

...
```

As we can see, the compiler is so clever that it eliminates the whole loop and
returns 100 directly!

Next, let's see a similar program with two source files:

``` c
// prog2.c

int inc(int a);

int main(int argc, char* argv[]) {
  int r = 0;
  for (int i = 0; i < 100; i++) {
    r = inc(r);
  }
  return r;
}

// inc.c

int inc(int a) {
  return a + 1;
}
```

This program *prog2* does the same thing as *prog1*, but it defines the `inc`
function in another file `inc.c`. However, the compiler is not that clever this
time:

```
$ gcc prog2.c inc.c -o prog2 -O2
$ objdump -d prog2

...

0000000000001040 <main>:
    1040:    53                       push   %rbx
    1041:    31 c0                    xor    %eax,%eax
    1043:    bb 64 00 00 00           mov    $0x64,%ebx
    1048:    0f 1f 84 00 00 00 00     nopl   0x0(%rax,%rax,1)
    104f:    00
    1050:    89 c7                    mov    %eax,%edi
    1052:    e8 f9 00 00 00           callq  1150 <inc>
    1057:    83 eb 01                 sub    $0x1,%ebx
    105a:    75 f4                    jne    1050 <main+0x10>
    105c:    5b                       pop    %rbx
    105d:    c3                       retq
    105e:    66 90                    xchg   %ax,%ax

...
```

The compilation options are the same, but the generated executable is quite
different: it calls `inc` 100 times. The result is right, of course, but it is
inefficient. This is because when the compiler compiles `prog2.c`, it has no
idea what `inc` does. The compiler doesn't know that `inc` is so simple a
function to be inline. So the compiler just does what we tell it to do and let
the linker resolve the address of `inc` at link time. That's all it can do
without LTO.

Now is the show time for LTO:

```
$ gcc prog2.c inc.c -o prog2 -O2 -flto
$ objdump -d prog2

...

0000000000001040 <main>:
    1040:    b8 64 00 00 00           mov    $0x64,%eax
    1045:    c3                       retq
    1046:    66 2e 0f 1f 84 00 00     nopw   %cs:0x0(%rax,%rax,1)
    104d:    00 00 00

...
```

The added `-flto` option tells GCC to enable link time optimization. This result
is exactly the same as *prog1*, well done!

GCC uses some magic behind this. When LTO is enabled, GCC dumps its internal
representation to object files instead of generating actual instructions, so
that all the compilation units can be optimized as a single module at link time.
We can get some insight from the difference between object files generated with
and without `-flto`:

```
# Compile without LTO #

$ gcc -c inc.c -O2
$ objdump -d inc.o

inc.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <inc>:
   0:    8d 47 01                 lea    0x1(%rdi),%eax
   3:    c3                       retq

$ objdump -h inc.o

inc.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000004  0000000000000000  0000000000000000  00000040  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  00000044  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000044  2**0
                  ALLOC
  3 .comment      00000024  0000000000000000  0000000000000000  00000044  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000068  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000030  0000000000000000  0000000000000000  00000068  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA

# Compile with LTO #

$ gcc -c inc.c -O2 -flto
$ objdump -d inc.o

inc.o:     file format elf64-x86-64

$ objdump -h inc.o

inc.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000000  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000040  2**0
                  ALLOC
  3 .gnu.lto_.profile.6e057e736196e6ee 0000000e  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, READONLY, EXCLUDE
  4 .gnu.lto_.icf.6e057e736196e6ee 00000018  0000000000000000  0000000000000000  0000004e  2**0
                  CONTENTS, READONLY, EXCLUDE
  5 .gnu.lto_.jmpfuncs.6e057e736196e6ee 00000019  0000000000000000  0000000000000000  00000066  2**0
                  CONTENTS, READONLY, EXCLUDE
  6 .gnu.lto_.inline.6e057e736196e6ee 00000035  0000000000000000  0000000000000000  0000007f  2**0
                  CONTENTS, READONLY, EXCLUDE
  7 .gnu.lto_.pureconst.6e057e736196e6ee 00000012  0000000000000000  0000000000000000  000000b4  2**0
                  CONTENTS, READONLY, EXCLUDE
  8 .gnu.lto_inc.6e057e736196e6ee 00000103  0000000000000000  0000000000000000  000000c6  2**0
                  CONTENTS, READONLY, EXCLUDE
  9 .gnu.lto_.symbol_nodes.6e057e736196e6ee 00000026  0000000000000000  0000000000000000  000001c9  2**0
                  CONTENTS, READONLY, EXCLUDE
 10 .gnu.lto_.refs.6e057e736196e6ee 0000000e  0000000000000000  0000000000000000  000001ef  2**0
                  CONTENTS, READONLY, EXCLUDE
 11 .gnu.lto_.decls.6e057e736196e6ee 00000181  0000000000000000  0000000000000000  000001fd  2**0
                  CONTENTS, READONLY, EXCLUDE
 12 .gnu.lto_.symtab.6e057e736196e6ee 00000013  0000000000000000  0000000000000000  0000037e  2**0
                  CONTENTS, READONLY, EXCLUDE
 13 .gnu.lto_.opts 0000006f  0000000000000000  0000000000000000  00000391  2**0
                  CONTENTS, READONLY, EXCLUDE
 14 .comment      00000024  0000000000000000  0000000000000000  00000400  2**0
                  CONTENTS, READONLY
 15 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000424  2**0
                  CONTENTS, READONLY
```

When LTO is enabled, `inc.o` has nothing in the `.text` section at all (size 0).
Instead, a few `.gnu.lto_.*` sections are added. The meaning of these LTO
sections can be found in [Link Time
Optimization](https://gcc.gnu.org/onlinedocs/gcc-8.3.0/gccint/LTO.html#LTO).
That's it, I can't guide you further in the internal, but I hope you can get a
sense about LTO through the above example.

The last question is: when should we enable LTO? Well, this is a trade-off
between compilation time and run time performance. LTO takes longer time to
compile but produces more efficient programs. I suggest that we should enable
LTO in production for better performance but disable LTO in development for
faster compilation (except for benchmark).
