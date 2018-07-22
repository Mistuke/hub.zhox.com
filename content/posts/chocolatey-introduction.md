---
title: "Haskell & AppVeyor Chocolatey Introduction"
date: 2018-07-22T13:14:25+01:00
draft: false
---

For those who are unaware, Windows has had quite a push lately to provide script-able
ways to install packages. One such attempt that has gained quite a lot of traction is
`Chocolatey` https://chocolatey.org/.

`Chocolatey` also contains some of my own packages for installing `GHC` and `cabal`
(along with packages to fully automate setting up a GHC dev environment etc, more on these
at a later date).

The `GHC` package https://chocolatey.org/packages/ghc goes all the way back to `GHC 6.10.1`
and the `cabal-install` package https://chocolatey.org/packages/cabal to `cabal-install 0.6.0`.

These are quite handy for those who use the `cabal+ghc` workflow like myself and like to switch
compiler versions easy, or script environment setups.

After installing [Chocolatey](https://chocolatey.org/install#installing-chocolatey)
 one can install `GHC` simply using the command

```
choco install ghc
refreshenv
```

which will install and reconfigure the latest released `GHC` along with the latest
`cabal-install`. The `refreshenv` is needed in order to reload the `PATH` variables which
an install of the package changes.  This is normally not needed for a Chocolatey package
but for `GHC` it is needed to preserve an assumption the `GHC` bindists make about their layout.

For older packages this dependencies on `cabal` isn't specified so one way that works
with all versions is:

```
choco install cabal ghc
refreshenv
```

You can also use `chocolatey` to install the latest `GHC` `Alpha`, `Beta` and `RC` simply by adding
`-pre` to the install command. e.g.:

```
choco install cabal ghc -pre
refreshenv
```

An existing install can be upgraded using the `upgrade` command

```
choco upgrade ghc
refreshenv
```

and specific versions can be installed using `--version`:

```
choco install ghc --version 8.4.1
refreshenv
```

The chocolatey packages are self contained, which means you can also install multiple `GHC`
versions side-by-side and switch between them using the full versioned form. e.g.

```
choco install ghc --version 8.4.1
choco install ghc --version 8.2.2 -m
refreshenv
```

Will install both `GHC 8.4.1` and `8.2.2` at once, with `GHC 8.4.1` being accessible
by the un-suffixed `ghc` command. The `-m` option means allow `multiple` which won't
force only one version to be installed. Using `cabal new-build`'s `-w` flag you can
easily switch between the two versions:

```
cabal new-build -w ghc-8.4.1
cabal new-build -w ghc-8.2.2
```

So take a look at the Chocolatey based distributions, which can also be used to install a full
Haskell dev environment using e.g. `vscode`.

```
choco install ghc vscode
Update-SessionEnvironment
code --install-extension justusadam.language-haskell
```

For those who like to test their packages against the bleeding edge, just like `Hvr`'s
excellent [PPA](https://launchpad.net/~hvr/+ppa-packages) for Ubuntu I provide nightlies
for both [GHC](https://www.myget.org/feed/mistuke/package/nuget/ghc-head) and
[cabal-install](https://www.myget.org/feed/mistuke/package/nuget/cabal-head) under my `NuGet` feed.

The current build frequency for both is Nightly, but depending on popularity/activity I may scale them
to weekly or longer. Builds are retained for a year before they are deleted, just as before I may extend or
shorten the period in the future depending on feedback. `GHC` builds are currently **64 bit ONLY**.

To use them first add my `NuGet` feed as a `Chocolatey source` and that will give you access to the
packages `cabal-head` and `ghc-head`.


```
choco source add -n mistuke -s https://www.myget.org/F/mistuke/api/v2
choco install ghc-head cabal-head -pre
refreshenv
```

`ghc-head` will always install `cabal-head`, this is because newer `GHC` may require a bleeding edge
`Cabal`. If you have them already installed you can just `upgrade` existing `nightlies` or `downgrade`
as long as the build is still around.  These are unsupported builds, If you do feel like you've found
a genuine bug, when reporting them my package feed can tell you which `commit` the build was made from.

In the future I will also be providing some of my own internal builds for wider testing before they get
merged into `upstream master`.

## AppVeyor Continuous Integration

`AppVeyor` comes with `Chocolatey` pre-installed. Which means you can use the exact same commands
as above to set up a CI script for your project using `AppVeyor`.  For instance the [Win32](https://github.com/haskell/win32/blob/master/appveyor.yml) library
I maintain uses the following `AppVeyor` script for Windows testing:


```
clone_folder: "c:\\WORK"

environment:
  global:
    CABOPTS:  "--store-dir=C:\\SR --http-transport=plain-http"
  matrix:
    # 64 bit builds
    - GHCVER: "8.4.2"
      CHOCOPTS:
    - GHCVER: "8.2.2"
      CHOCOPTS:
    - GHCVER: "8.0.2"
      CHOCOPTS:
    - GHCVER: "7.10.3.2"
      CHOCOPTS:
    - GHCVER: "7.8.4.1"
      CHOCOPTS:
    # 32 bit builds
    - GHCVER: "8.4.2"
      CHOCOPTS: --forcex86
    - GHCVER: "8.2.2"
      CHOCOPTS: --forcex86
    - GHCVER: "8.0.2"
      CHOCOPTS: --forcex86
    - GHCVER: "7.10.3.2"
      CHOCOPTS: --forcex86
    - GHCVER: "7.8.4.1"
      CHOCOPTS: --forcex86

cache:
- "C:\\SR"

install:
 - "choco install -y ghc --version %GHCVER% %CHOCOPTS%"
 - "choco install -y cabal %CHOCOPTS%"
 - "refreshenv"
 - "set PATH=C:\\ghc\\ghc-%GHCVER%:C:\\msys64\\mingw64\\bin;C:\\msys64\\usr\\bin;%PATH%"
 - "cabal --version"
 - "ghc --version"
 - "cabal %CABOPTS% update -v"

build: off

test_script:
 - IF EXIST configure.ac bash -c "autoreconf -i"
 - "echo packages:. > cabal.project"
 - "cabal %CABOPTS% new-build -j1 all"
```

This tests both `x86_64` and `x86` versions of the last 5 `GHC` against any pull requests to `Win32`.

To point out some important bits:

```
  global:
    CABOPTS:  "--store-dir=C:\\SR --http-transport=plain-http"
```

The `--store-dir` flag is used to shorten the path of the `new-build nix` style package store folder so
it doesn't hit `MAX_PATH` as easily. (***NOTE:*** __`GHC 8.6` and newer no longer have this limitation for
end user programs. So in the future once `cabal` is compiled with `GHC 8.6` this won't be needed, but for
backwards compatibility should still be used for now.__)

The `http-transport` is unimportant, it can be omitted but I prefer to not have it use `curl` from `msys2`.

```
cache:
- "C:\\SR"
```

This caches the `nix` style package cache so that dependencies don't get built over and over again. This can
save a significant amount of time when building your project. `AppVeyor` has an hour build limit.

```
 - IF EXIST configure.ac bash -c "autoreconf -i"
```

This allows packages like `network` to be used from the repo where there is no final `configure` script yet.

```
 - "choco install -y ghc --version %GHCVER% %CHOCOPTS%"
 - "choco install -y cabal %CHOCOPTS%"
```

This command installs the compiler and the latest `cabal-install` which is backward compatible with older `GHC`s.
The explicit `cabal` install is for older packages which didn't have a dependency on `cabal`.  For newer `GHC` this
would be a no-op. The `-y` makes `Chocolatey` suppress the script execution question with a default answer of `yes`.

```
 - "refreshenv"
 - "set PATH=C:\\ghc\\ghc-%GHCVER%:C:\\msys64\\mingw64\\bin;C:\\msys64\\usr\\bin;%PATH%"
```

As before we reload the `environment variables`, which unfortunately also clears the custom ones `AppVeyor` sets.
To restore the ones we need we modify the `PATH` to contain `msys2` and where we expect older `GHC`s to have been installed.

***NOTE:*** __If you actually intend to use `pacman` packages, please set the appropriate `mingw` for the architecture you intend
to use. The above examples all set the `64 bit` paths.__

If you want to report a bug against the packaging (the chocolatey packages and not the binaries themselves) please do so on my
GitHub project for the packages. [Cabal](https://github.com/Mistuke/CabalChoco) and [GHC](https://github.com/Mistuke/GhcChoco).

I do plan on adding a `Haskell Platform` package in the near future. Happy hacking.

-- Tamar