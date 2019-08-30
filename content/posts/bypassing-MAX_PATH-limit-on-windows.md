---
title: "Bypassing MAX_PATH Limit with GHC on Windows"
date: 2019-08-28T23:49:39+01:00
draft: false
---

> *TL;DR:* `GHC 8.10+` Will ship with a custom `GCC+Binutils` bindist that will remove the
> `MAX_PATH` limitation to files.  The GHC binaries themselves must still be in
> installed in a path shorter than `MAX_PATH`.  `GHC 8.6` and `GHC 8.8` are also
> supported if installed via chocolatey by installing the extension ghc-jailbreak.
> For cabal to work correctly you need a cabal compiled with GHC 8.6+, on
> chocolatey this means cabal 2.4+.

<pre class="light">
NOTE: I know that some people will consider this a hack, and while it is a hack it is
the best solution that works on older platforms as well. For the Linux users
reading this, while Windows does have an `LD_PRELOAD` equivalent system DLLs are
exempt from this. On ELF platforms the approach taken here is equivalent to
patching `DT_NEEDED` entries.
</pre>

## Introduction

On Windows the ancient `ANSI` APIs are generally constrained in the length of
file paths to **~260** characters.  In the `Windows.h` header this limit is defined
as the `MAX_PATH` constant.

So how does this manifest itself? Usually as a `file not found` error as the
path is silently truncated to this maximum when these APIs are called.  See for
example how `GHC 8.0.1` handles long paths:

```
> C:\ghc\ghc-8.0.1\bin\ghc.exe .\hello.hs -o "R:\<very long path>\a.exe" -fforce-recomp
<command line>: error: directory portion of "R:\\<very long path>\\a.exe" does not exist (used with "-o" option.)
```

## Background

When Microsoft introduced the `Wide` versions of the `Win32` APIs they added the
ability to bypass this limit (and other naming restrictions) by using an extended
format for paths.

Windows PATHs are categorized in `Namespaces` and the namespaces we're interested
in using are the `Win32 device` and `Win32 file` namespaces.

Paths using these namespaces start with `\\.\` and `\\?\` respectively.  These
are the paths you see when you look at the Windows mount table (using the `mountvol` command).

The `device` namespace is used when you require raw/direct access to a device, e.g. `\\.\COM1`,
whereas the `file` namespace is used for normal filesystem access.  `UNC` (Uniform Naming Convention,
also known as network location) paths are accessible as `\\?\UNC\`. More at [MSDN](https://docs.microsoft.com/en-us/windows/desktop/FileIO/naming-a-file#fully-qualified-vs-relative-paths).

Using these namespaces bypass all API level limitations as the path is passed directly to the
underlying filesystem driver.  This lifts the filepath length limit to about `32,767` characters.
There is however one downside.  Since the Win32 API no longer processes the paths, having a `/` in a path is invalid along with relative paths and special components such as `.` and `..` in paths.

Starting with GHC `8.6` I added code into GHC that wraps the usage of the deprecated POSIX
APIs with their modern equivalent.  This allowed Haskell programs like `cabal` and `ghc` itself
to be able to use these longer paths. The code added to GHC does some processing to ensure that any paths it gives to the filesystem is valid.  This processing is disabled if a
namespaced path is used as an input.

So what happens when we run GHC `8.6.x` on a long path then?

```
> ghc .\hello.hs -o "R:\<very long path>\a.exe" -fforce-recomp
[1 of 1] Compiling Main             ( hello.hs, hello.o )
Linking R:\<very long path>\a.exe ...
R:\ghc-8.6.4\lib\../mingw/bin/windres.exe: can't open file `R:\<very long path>\a.exe.manifest': No such file or directory
`windres.exe' failed in phase `Windres'. (Exit code: 1)
```

Still crashed, not quite what was expected...

The problem is that while GHC now understands long paths, the underlying toolchain that we use
does not.  More specifically GCC and Binutils do not support the long paths.

Unfortunately patching GCC and Binutils to support these newer APIs pose both technical and
legal challenges.  These projects do not centralize the way they open files.  While `libiberty`
exists and is in use for some paths, most places just do `fopen` directly.

For the longest time the tickets and frustrations of Windows users have been piling up with the
tag `"upstream"`.  Others have devised workaround such as using `DOS 8.3` shortnames, however these
have multiple issues:

<ul class="list">
<li> They're a performance bottleneck. You do extra writes in your <code>MFT</code> for each entry and the algorithm to find a unique shortname does a linear probe.</li>
<li> They can be disabled, and then things won't work.</li>
<li> A shortname is extra data to an existing path, which means you can't have a shortname to a non-existent file.</li>
</ul>

## Solution

So what can we do about it without changing the source? We can change the binaries!

Essentially what we're going to do is replace every call to certain functions in GCC and Binutils with
new and improved ones.  There are three ways this "hooking" can be done:

<ul class="list">
<li> <code>Dynamic</code>: The program is created suspended, the headers of the program is modified in memory and addresses for the functions we want to hook will be overridden by the new ones.</li>
<li> <code>Static</code>: The header is modified directly in the file on disk.</li>
<li> <code>Local redirect</code>: Windows has a feature where a <code>.local</code> <a href="https://docs.microsoft.com/en-us/windows/desktop/dlls/dynamic-link-library-redirection">forwarder file</a> can be used to redirect access to a DLL with one that's local (in a subfolder of the local file).  This won't work as certain system DLLs such as <code>msvcrt.dll</code> are exempt from being redirected.</li>
</ul>

`Dynamic` is nice as it makes no lasting changes to the binary, but it's also slower and requires changes to `GHC` itself. `Static` on the other hand will have no performance overhead but if the binaries are signed then the signatures would be invalidated.

Luckily the binaries are not signed, so I went with the `Static` approach.

Essentially the game plan is in every binary in the `mingw` folder of `ghc`
we do the following:

Find the IAT (`Import Address Table`) entry for `msvcrt.dll` (the Microsoft C runtime) and replace these with a new runtime `phxcrt.dll`.  That name was chosen as it has the same length as the original file we're replacing.  After the replacement is done the `PE header checksum` is re-calculated and the new values written out.

![IAT](/images/import_descriptor.png "Import Address Table")

The `phxcrt.dll` is a thin `forwarder only` DLL. It uses a little known feature of the Windows system loader to allow one to forward a call to another DLL. The final address is the one that the loader will write back into the `import descriptor`, so this cost is small and paid only once.  (Note that this does require strict binding of functions. Lazy binding can also be overridden but by a different mechanism.)

`phxcrt.dll` will forward the calls we don't care about to `msvcrt.dll` and the ones we do will be forwarded to `muxcrt.dll` (I'll leave the details of this to another post).

So let's see it in action:

```
> rm -rf /r/ghc-8.6.4/
> tar xJf ghc-8.6.4-x86_64-unknown-mingw32.tar.xz
> make
gcc -O2 -DUNICODE -D__USE_MINGW_ANSI_STDIO=1 -std=c99 -municode -DWINVER=0x0601 -D_WIN32_WINNT=0x0601 fs.c iat-patcher.c -o iat-patcher.exe -ldbghelp -limagehlp
gcc -shared phxcrt.c fs.c -o x86_64/muxcrt.dll
gcc -shared x86_64/msvcrt.def -o x86_64/phxcrt.dll

> Import-Module .\Patch-GHC.psm1 -Force; Use-PatchGHC ghc "iat-patcher.exe" "ghc-jailbreak\x86_64" $true
Done patching GHC's Mingw-w64 distribution. Good to go.
```

At this point GCC etc. should be good to go! Let's try out that command from before:


```
> ghc .\hello.hs -o "R:\<very long path>\a.exe" -fforce-recomp
[1 of 1] Compiling Main             ( hello.hs, hello.o )
Linking R:\<very long path>\a.exe ...

> /r/<very long path>/a.exe
Hello World
```

Success! To check if as expected the redirects are cheap we inspect the API trace for `windres.exe`

![IAT](/images/forward.png "Import Address Table")

And as expected, the `IAT` has a mix of `muxcrt.dll` and `msvcrt.dll` but no `phxcrt.dll` apart from the initialization calls.

## How to get it

The patcher is currently available for wide testing and can be installed through `chocolatey`.
Because of the nature of it it's currently only available on my private chocolatey feed:

```
choco source add -n mistuke -s https://www.myget.org/F/mistuke/api/v2
choco install ghc-jailbreak
```

and you're good to go! This will patch the GHC that's on your path.  It
currently supports patching only one GHC automatically.  Though you can call
the patch CmdLets to patch multiple GHC installs.

To undo the patching just uninstall the patcher using

```
choco uninstall ghc-jailbreak
```

This will undo the changes to all the binaries.

For the full source visit the project [GitHub](https://github.com/Mistuke/Ghc-jailbreak).
Please give it a try, and if you encounter any problems open a ticket so I can
fix these up.

-- Tamar