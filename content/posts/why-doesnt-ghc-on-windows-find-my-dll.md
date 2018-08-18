---
title: "Why doesn't GHCi on Windows find my DLL"
date: 2018-08-18T10:11:47+01:00
draft: false
---

You have a new package you want to try out. You try to install the package only to be greeted with

```
GHCi, version 8.2.2: http://www.haskell.org/ghc/  :? for help
<command line>: user specified .o/.so/.DLL could not be loaded (addDLL: my-cool-library or dependencies not loaded. (Win32 error 126))
Whilst trying to load:  (dynamic) my-cool-library
Additional directories searched: (none)
```

by GHCi.  Confused you look into your `libraries` folder and see `my-cool-library-7.dll`.  Why is the file name incorrect? Why is GHCi looking for `my-cool-library.dll` instead of `my-cool-library-7.dll`.

You rename the file and things *seem* to work.  This is actually quite dangerous and wrong. Unfortunately this is also often suggested as what to do.

To understand why you should not do this, let's take a step back and give a crash course on linkers.

## Import Libraries

First off, let's talk about what you should be giving to the compiler.
On Unix systems the shared library contains internally the `PLT` and the `GOT` sections along with the binding
method required.  On Windows, this information is externalized into a file called an `import library`.

All Microsoft compilers will only accept import libraries to link against. Unfortunately Binutils also accepts DLLs
directly, mostly due to historical reasons.

When it comes to import libraries there are two versions:

1. The official format is documented in the Microsoft PE
specification. Libraries produced by this format typically contain the file extension `.lib`.
2. The second format is an unofficial format invited by and only supported by GNU tools.  This format is based on the normal object code format and typically result in larger and more inefficient libraries.  Libraries in this format typically end with the extensions `.dll.a` or `.a` depending on the tool that created them.  They work using a hack around pseudo symbols that start with `_head_` in the name. GHC and GCC support both formats.

So what is one supposed to do when you only have a DLL?  Create an import library from the Microsoft DEF file.
https://msdn.microsoft.com/en-us/library/28d6s79h.aspx and when creating a shared library, always ask the compiler
to also produce an import library! (This is the default for GHC and Microsoft compilers.)

## DLL Hell

Before Microsoft solved "DLL" hell once and for all using `side-by-side` assemblies, different projects used import libraries to support multiple versions of the same DLL on the same system at the same time.  They did this by versioning the DLL by adding information to the name of the DLL to make them unique. e.g. to support multiple versions of `my-cool-library` on the user's machine the DLLs would be named `my-cool-library-1.dll`, `my-cool-library-2.dll`,..,`my-cool-library-n.dll`. The import library would be named `my-cool-library.lib` so that `-lmy-cool-library` still works.

The import library will point the linker to which version of the DLL to enter into the program's `IAT`. The `Import Address Table` contains a list of `DLL` and `symbols` to import from each.

GHC knows how to properly deal with these redirects.

## Lazy loading

As I mentioned before, Windows externalizes the binding type.  Windows defaults to strict binding, i.e. when the program starts up it forces the resolution of all symbols.  Including ones you may never need.

Lazy bindings (which is the default in System-V) is also supported via the import library. This is a choice made by
the programmer during library creation time.  When lazy loading the library gets a chance to resolve each symbol.  This allows it to do symbol resolution and redirection.

When linking directly against a DLL you may be changing the semantics of the library due to it switch from lazy to eager binding.

## Relocations

On any given compilation pipeline you generally have three main phases. `compiling`, `assembling` and `linking`.  Input source files like `.hs` and `.c` files are compiled into assembly language for the given target in the first phase.  The compiler has no idea where the functions that are defined in another source or library actually is.  The compiler simply emits a `call` or `branch` to the symbol.

Let's look at an example. The following C file

```c
#include <stdio.h>

int main ()
{
   printf("Hello World");
}
```

gets compiled into the following x86_64 assembly file

```
.LC0:
        .ascii "Hello World\0"
        .text
        .globl  main
main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $32, %rsp
        call    __main
        leaq    .LC0(%rip), %rcx
        call    printf
        movl    $0, %eax
        addq    $32, %rsp
        popq    %rbp
        ret
```

This assembly file is then given to the assembler which will produce object code.  Since it doesn't know the address of these function it emits what's called a `relocation` against the symbol¹.

¹ In general any function or global data reference is called a `symbol`.

If we look at the object file we'll get our first hint of the relocation:

```
Disassembly of section .text:

0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   48 83 ec 20             sub    $0x20,%rsp
   8:   e8 00 00 00 00          callq  d <main+0xd>
                        9: R_X86_64_PC32        __main
   d:   48 8d 0d 00 00 00 00    lea    0x0(%rip),%rcx        # 14 <main+0x14>
                        10: R_X86_64_PC32       .rdata
  14:   e8 00 00 00 00          callq  19 <main+0x19>
                        15: R_X86_64_PC32       printf
  19:   b8 00 00 00 00          mov    $0x0,%eax
  1e:   48 83 c4 20             add    $0x20,%rsp
  22:   5d                      pop    %rbp
  23:   c3                      retq
```

As we can see the assembler has emitted a relocation `R_X86_64_PC32` against the `printf` symbol.  The important thing to note here is that this is a `Program Counter relative 32-bit` relocation.  It means the address of the symbol *MUST* fit within a `32-bit` offset.  Put it this way, the function must be within a `4GB` offset from the current instruction.

The final step in this process is the linker that is responsible to finding the the functions used.  This step is called `symbol resolution`. For GHCi and Template Haskell, the GHC runtime has it's own runtime linker.

We don't need to continue beyond this point, but the `32-bit` relocation is important to note.

## Preferred Image base

On a conceptual level, the linking model between Windows and Unix (System-V) are reasonably close, however
implementation wise things are rather different. Windows for one doesn't have PIC (Position Independent Code)
but has a concept of `rebasing`.  Essentially what this means is that when the PE file's preferred load address
is already taken the file will be `rebased` to a new address.  When ASLR is enabled the image base is randomly
chosen.

Why is this important? Because on `64-bit` Windows the preferred load address for shared libraries created with
a Microsoft compiler is `0x180000000`. Which means the address is larger than would fit in a `32-bit` address.

Linking against a DLL directly can, and most often will result in the address being truncated to fit the `R_X86_64_PC32` relocation.  This is why you should never do this.  It is unfortunate, but binutils will allow you to link directly against a DLL (mostly for historical reasons.) but will emit a rather cryptic warning:

```
relocation truncated to fit: R_X86_64_PC32 against `<symbol>'
```

This warning essentially means the address it is using for the DLL is likely invalid! This will result in a segfault at runtime if you're lucky, at worse a random function being called.

When the linker does warn you and you ignore it, this is what typically happens

```
Access violation in generated code when executing data at 000000008000fb50
```

It may not happen the first, or second time, but eventually it will due to ASLR moving around the library. As a concrete example, the official `PostGres` libraries are compiled using a Microsoft compiler. As such it's image base is in the low `64-bit` range.

```
0x000ab360: C:\Program Files\PostgreSQL\10\bin\LIBPQ.dll
      Base   0x180000000  EntryPoint  0x180021968  Size        0x00049000    DdagNode     0x000ab4b0
      Flags  0x000c22ec  TlsIndex    0x00000000  LoadCount   0x00000001    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED
```

To illustrate the difference between linking directly against the DLL and against the import table let's look
at this example program:

```c
#include <stdio.h>
#include <stdint.h>

extern void* PQgetvalue (void);

int main () {
  printf ("PQgetvalue: %p", PQgetvalue);
}
```

This program reports the address of the symbol `PQgetvalue`.
When linking directly against the DLL the address returned is

```
Tamar@Rage MINGW64 /r
$ gcc test.c -o test -lpq -L/E/Program\ Files/PostgreSQL/10/lib/
Tamar@Rage MINGW64 /r
$ ./test.exe
PQgetvalue: 00000000`6440CDE0
```

The function `PQgetvalue` is actually at address `0x000000018000fb50` but has been truncated to `0x8000fb50`.
This is why it segfaults.

However when linking against the import library explicitly

```
Tamar@Rage MINGW64 /r
$ gcc test.c -o test -llibpq -L/E/Program\ Files/PostgreSQL/10/lib/
Tamar@Rage MINGW64 /r
$ ./test.exe
PQgetvalue: 00000000`004015A0
```

you get an address far inside the `32-bit` range as expected.

GHCi is able to correctly handle function symbols when given a DLL to load.  That is, we create a `trampoline/indirection` on any symbol required from a DLL.  The `trampoline` is ensured to be created in
the `4GB` range and will then use a `jmp` to any `64-bit` location.  If there's no more space in the `4GB` range
we abort.  However `GHCi` cannot know from just the DLL what the original semantics of all the symbols in the
`export` table of the DLL is.  It is forced to assume everything is a function.

Import library essentially contain stubs, or direct the linker to emit stubs within the program's `.text` section.
This ensures that the stub itself is within range of the `R_X86_64_PC32` relocation and that the truncation doesn't happen.

## Data references (GOT)

Like System-V, Windows allows data to be exported from the DLL.  However, because data is accessed directly we cannot use an indirection to access the data.  The address returned for the symbol *must* be the symbol itself.

Which means when using a relocation, it *must* point to the data directly.  The compiler knows about which symbols
are data and which are functions, and will appropriately mark them so in the import library.

If GHC or Binutils is given a DLL directly, they have no way to determine this information.  If this were to happen
then you would access the redirection instead of the data value you were expecting.

## Library Search Order

On all platforms, including Windows we do honor the `LD_LIBRARY_PATH` and `LIBRARY_PATH` environment variables.
`LD_LIBRARY_PATH` will be used to look for shared dynamic libraries and `LIBRARY_PATH` for static archives and
import libraries.  We also search the user `PATH` and `extra-lib-dirs` along with any paths given via `-L`.

The current search order in GHCi is:

```ini
For non-Haskell libraries (e.g. gmp, iconv):
first look in library-dirs for a dynamic library (on User paths only)
(libfoo.so)
then  try looking for import libraries on Windows (on User paths only)
(.dll.a, .lib)
first look in library-dirs for a dynamic library (on GCC paths only)
(libfoo.so)
then  check for system dynamic libraries (e.g. kernel32.dll on windows)
then  try looking for import libraries on Windows (on GCC paths only)
(.dll.a, .lib)
then  look in library-dirs for a static library (libfoo.a)
then look in library-dirs and inplace GCC for a dynamic library (libfoo.so)
then  try looking for import libraries on Windows (.dll.a, .lib)
then  look in library-dirs and inplace GCC for a static library (libfoo.a)
then  try "gcc --print-file-name" to search gcc's search path
    for a dynamic library (#5289)
otherwise, assume loadDLL can find it

The logic is a bit complicated, but the rationale behind it is that
loading a shared library for us is O(1) while loading an archive is
O(n). Loading an import library is also O(n) so in general we prefer
shared libraries because they are simpler and faster.
```

This has been tweaked through bug reports from various different releases. For most user libraries
we will do the right thing as long as you don't give it a path to your `DLL` but an import library.
The current search directory is assuming any dll you give it only has functions.  In the future I'd like to
promote the searching for import libraries above shared libraries.

For now this order is safe for the majority of cases as we do extra work to ensure loading a dll directly will
work for the majority of cases. In the mean time before we change the order (and ignore backwards compatibility)
you can control which one it picks using different paths for linking and running.

To figure out where GHCi searched for libraries just pass it an extra `-v`

```
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "my-cool-library.lib"
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "libmy-cool-library.lib"
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "libmy-cool-library.dll.a"
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "my-cool-library.dll.a"
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "my-cool-library.dll"
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "libmy-cool-library.dll"
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "libmy-cool-library.a"
*** gcc:
"E:\msys64\mingw64\lib/../mingw/bin/gcc.exe" "-fno-stack-protector" "-DTABLES_NEXT_TO_CODE" "--print-file-name" "my-cool-library.a"
Loading object (dynamic) my-cool-library ... failed.
*** Deleting temp files:
Deleting:
*** Deleting temp dirs:
Deleting:
<command line>: user specified .o/.so/.DLL could not be loaded (addDLL: my-cool-library or dependencies not loaded. (Win32 error 126))
Whilst trying to load:  (dynamic) my-cool-library
Additional directories searched: (none)
```

So where does the error you as the user sees comes from? When GHC's runtime linker cannot find the path to a
library, it is forced to assume the library must be a DLL on the user's system search path. The error `126`
comes from the Windows loader which means

```
ERROR_MOD_NOT_FOUND

    126 (0x7E)

    The specified module could not be found.
```

More at https://docs.microsoft.com/en-us/windows/desktop/Debug/system-error-codes--0-499-

Error 126 can be caused because it either can't find the DLL or one of it's transitive dependencies.
To figure out the exact reason for the failure one can enable `loader snaps` and get debug output from
the Windows's dynamic loader.

## Conclusion

Hopefully this has shown why the GHC runtime linker is unable to find your DLL and how to do proper linking on Windows.  If there's anything I hope people take away from this it's: _never link directly against a dll_ on Windows.

Starting with `GHC 7.10.x` I have started to add import library support to `GHC`. With `GHC 8.x` this work has been
mostly completed. The remaining issues are just internal organization issues rather than functional changes.

This work is also the basis of getting a dynamically linked GHC to work on Windows. More on this at a later date.