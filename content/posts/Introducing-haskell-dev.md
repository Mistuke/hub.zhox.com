---
title: "The easiest way to setup a Haskell environment on Windows"
date: 2018-08-30T07:14:25+01:00
draft: true
---

## What is chocolatey

Windows has had quite a push lately to provide script-able ways to install
packages. One such attempt that has gained quite a lot of traction is
`Chocolatey` (https://chocolatey.org/). Chocolatey's populary is quickly growing.

Chocolatey is supported out of the box in all major cloud CI services including
`AppVeyor`, `CircleCI`, `Travis CI` and `Azure Pipelines`.  It's also supported
by full system building software such as `BoxStarter` that are used by companies
to provision new machines and also supported by system management software such
as `Puppet` and `Ansible`.

The reason for this is that `Chocolatey` is not only easy to use, but safe and
secure. Every package is initially hand verified through moderation until the
package becomes trusted.  Packages are always passed through automated verification
and packages are downloaded and uploaded through https.  The package metadata
itself is verified after downloaded which checks that the package wasn't
tampered with during download.  Furthermore any files downloaded (such as `cabal`
or `ghc`) have to have a `sha256` checksum provided for them which will be
verified.  If you don't trust the signatures you can provide your own that it
uses to check against.  Once uploaded every binary the package installs is ran
through a host of virus checkers and the results publicly posted.

Read more on Chocolatey security [here](https://chocolatey.org/security) and see
for yourself why it's trusted by so many companies.

## Introduction

`Chocolatey` contains some of my own packages for installing `GHC` and `cabal`

The `GHC` package (https://chocolatey.org/packages/ghc) goes all the way back to `GHC 6.10.1`
and the `cabal-install` package (https://chocolatey.org/packages/cabal) to `cabal-install 0.6.0`.

These are quite handy for those who use the `cabal+ghc` workflow like myself and like to switch
compiler versions easily, or script environment setups.

After installing [Chocolatey](https://chocolatey.org/install#installing-chocolatey)
 one can install `GHC` simply using the command

```
choco install ghc cabal
refreshenv
```

<pre class="light">
NOTE: For a longer introduction see <a style="color: #000" href="https://hub.zhox.com/posts/chocolatey-introduction/">Introduction to Chocolatey</a>.
</pre>

For the longest time the complaint here was that these packages do not install `msys2` and so are unsuitable to fully
replace the `Haskell Platform`.

With the release of `Cabal 3.0.0.0` this is no longer the case.

## What's new in the chocolatey packages

There are a couple of new things in both the `Cabal 3.0.0.0` and `GHC 8.8.1` packages

<pre class="light">
NOTE: At the time of writing the no 32-bit GHC 8.8.1 package has produced by GHC HQ.
Publishing the GHC 8.8.1 package without a 32-bit release would block 32-bit users from using the unversioned package head (i.e. from being able to install the latest GHC without needing to give an explicit version).

As such the GHC 8.8.1 package will install GHC 8.6.5 on 32-bit machines so that you get a working compiler. If 32-bits is important to you please let them know by filing a ticket at https://gitlab.haskell.org/ghc/ghc/issues
</pre>

### Cabal 3.0+

The `cabal` package on chocolatey will now automatically detect the presence of
`msys2` on the system and configure it to work with `cabal` seamlessly.  It also
configures some default `cabal` options such that `cabal` works fully out of the
box on `Windows`.  This means that packages such as `network` or `threadscope`
will install without problem from *ANY* shell, including `cmd` and `powershell`.

A second change is that the package will detect the presence of it being installed
on `AppVeyor` and will configure the `cabal` such that it works without any issue
there using the `AppVeyor` provided `msys2`.

Since this functionality is provided by the `cabal 3.0+` package, you can use any
`GHC` version (or even multiple by using the choco flag `-m` when installing `GHC`)
and it will just work.

Because of this change `AppVeyor` scripts can be much simpler.  As an example the
`Win32` script looks like this now

```
clone_folder: "c:\\WORK"
clone_depth: 5

# Do not build feature branch with open Pull Requests
skip_branch_with_pr: true

platform:
  - x86_64
  - x86
cache:
 - "C:\\SR"

environment:
  global:
    CABOPTS:  --store-dir=C:\\SR --http-transport=plain-http
    CHOCOCMD: ghc --version %GHCVER%
  matrix:
    - GHCVER: 8.6.2
    - GHCVER: 8.4.2
    - GHCVER: 8.2.2
    - GHCVER: 8.0.2
    - GHCVER: 7.10.3.2
    - GHCVER: 7.8.4.1
    - GHCVER: 7.6.3.1

for:
  -
    matrix:
      only:
        - platform: x86
    environment:
      global:
        GHCOPTS: --forcex86

install:
 - choco source add -n mistuke -s https://www.myget.org/F/mistuke/api/v2
 - choco install %CHOCOCMD% -y %GHCOPTS% %CHOCOPTS% --ignore-dependencies
 - choco install -y cabal %CHOCOPTS%
 - refreshenv

before_build:
 - cabal --version
 - ghc --version
 - cabal %CABOPTS% v2-update
 - IF EXIST configure.ac bash -c "autoreconf -i"

build_script:
 - echo packages:. > cabal.project
 - cabal %CABOPTS% v2-build -j all
```

This script should work for `99%` of all packages.

The last thing the cabal package now provides is a new command `mingw64-pkg`.
One of the common mistakes people make is installing the `wrong` package from msys2
causing ghc(i) to not be able to find or load the package.

`mingw64-pkg` is a script that can be called from any shell to download the right packages
into `msys2` such that `GHC` and `cabal` will find them without hassle.

```
mingw64-pkg install pkg-config
mingw64-pkg install gtk2
```

Will give you a fully working and configured `gtk2` and `pkg-config`.   This
script also provides additional functionality documented on the [cabal chocolatey](https://chocolatey.org/packages/cabal) page.

### GHC 8.8+

The `GHC 8.8+` packages will automatically detect `Travis CI` as well, however
due to the way `Travis CI` is setup, calling the default `refreshenv` will have
no effect since travis doesn't use a Windows shell by default and instead uses
a unix shell such as `bash`.

The `GHC 8.8+` packages will register a `bash` alias `refreshenv` such which will
re-export your environment variables such that they pick up new `GHC` installs.

`Travis CI` does not install `msys2` automatically like `AppVeyor`, as such we have to
install it manually. On travis

```
choco install ghc cabal msys2
refreshenv
```

gets you all you need.

### Haskell-dev

In order to make this easier for beginners chocolatey has a new package called
[Haskell-Dev](https://chocolatey.org/packages/haskell-dev).

Haskell-Dev is a full replacement for the minimal Haskell-Platform with no additional
configuration needed. It will set up a 100% working Haskell environment using a single
chocolatey command:

```
choco install haskell-dev
```

The benefits of using this package are that you can afterwards still manage each
individual component (such as upgrading `ghc` alone) or upgrade everything as a
whole.

<pre class="light">
NOTE: Don't use this package for AppVeyor, you may needlessly duplicate work as it
already provides `msys2`. Please install the `cabal` and `ghc` packages directly.
</pre>

```
choco upgrade haskell-dev
```

Will always get you the most up-to-date version of everything.  Speaking of updates
one of the major advantages of using chocolatey to install `msys2` is the fact that
it does not use a stale tarball for `msys2`. During the installation a full `msys2`
system update is done. Giving you the latest working version of `msys2`.  This is
very important for security and compatibility issues.  Having an too out of date `msys2`
install will also prohibit you from installing newer versions of packages and if it gets
too out of date `pacman` may no longer have the right `pgp` keys to work.

But what about setting up an `IDE`? I have a package that I am still putting together and testing.
The package will install all of the above but on top of this install `vscode` and pre-built `hie` binaries
and automatically configure the extensions to work together out of the box.

This will be available soon.

<pre class="light">
NOTE: I don't particularly use anything in an IDE aside from syntax highlighting so I may not be fully in tune with the needs of people who do.  As such I am looking for two things:

- What do people want the IDE package to do/come with out of the box.
- I am looking for co-maintainers, as mentioned I won't be using this much myself so I'd like people who have slightly more skin in the game to help shape this.  The only hard requirement is that this *has* to stay `cabal` centric.
</pre>

-- Tamar