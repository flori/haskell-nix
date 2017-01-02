# Dependencies

This project now depends on the `turtle` package in order to illustrate some
dependency management basics.  The `Main.hs` executable is a contrived program
that uses `turtle` gratuitously for the `echo` function:

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Turtle

main :: IO ()
main = echo "Hello, world!"
```

The `release0.nix` file is the same as for the previous project except that we
now name our package `project1` instead of `project0`:

```nix
let
  pkgs = import <nixpkgs> { };

in
  { project1 = pkgs.haskellPackages.callPackage ./default.nix { };
  }
```

... but our `default.nix` file has changed.  This is because our
`project1.cabal` file now has a new `turtle` dependency:

```cabal
name: project1
version: 1.0.0
license: BSD3
license-file: LICENSE
cabal-version: >= 1.18
build-type: Simple

executable project1
    build-depends: base < 5, turtle
    main-is: Main.hs
    default-language: Haskell2010
```

We'll see the corresponding change in the corresponding `default.nix` generated
by `cabal2nix`:

```nix
{ mkDerivation, base, stdenv, turtle }:
mkDerivation {
  pname = "project1";
  version = "1.0.0";
  src = ./.;
  isLibrary = false;
  isExecutable = true;
  executableHaskellDepends = [ base turtle ];
  license = stdenv.lib.licenses.bsd3;
}
```

Notice how neither file specifies what version of `turtle` to depend on.  This
is because Nix resembles `stack` and provides a curated package set of
Haskell packages that build together.  If you don't specify a version then Nix
will pick a version for you.

You can find the latest curated package set here:

* [Curated Hackage package set][hackage-packages]

... which corresponds roughly to the `nixpkgs-unstable` release.  If you would
like to see what package versions Nix selects for a stable release such as
`nixOS-16.09`, then change the branch name in the URL from `master` to
`release-16.09`, like this:

* [Curated Hackage package set for NixOS-16.09][hackage-packages-16.09]

These curated package sets correspond roughly to Stackage resolvers.  Stable
releases like `nixOS-16.09` correspond to Stackage LTS resolvers, and
the `nixpkgs-unstable` release corresponds roughly to a Stackage nightly
resolver.  The main difference is that Nix's package set curation extends beyond
Haskell packages: `nixpkgs` also curates non-Haskell dependencies, too.

We can also see which version our project selects if we build our project
using `nix-build`:

```bash
$ nix-build -A project1 release0.nix
these derivations will be built:
  /nix/store/8g54hjpim7l9s41c9wpcn1h2q8m254m5-project1-1.0.0.drv
building path(s) ‘/nix/store/pi47yvw46xv346brajyrblwqhjmglhaj-project1-1.0.0’
...
Configuring project1-1.0.0...
Dependency base <5: using base-4.9.0.0
Dependency turtle -any: using turtle-1.2.8
...
/nix/store/pi47yvw46xv346brajyrblwqhjmglhaj-project1-1.0.0
```

The log output from `nix-build` notes that `turtle-1.2.8` was chosen for this
build.  Your results might vary depending on which version of the `nixpkgs`
channel that you have installed.

# Changing versions

Suppose that we want to build against `turtle-1.3.0` for whatever reason.  This
requires a much larger change to our project derivation which we can see in
`release1.nix`:

```nix
# Note: This should fail to build
let
  config = {
    packageOverrides = pkgs: rec {
      haskellPackages = pkgs.haskellPackages.override {
        overrides = haskellPackagesNew: haskellPackgesOld: rec {
          project1 =
            haskellPackagesNew.callPackage ./default.nix { };

          turtle =
            haskellPackagesNew.callPackage ./turtle.nix { };
        };
      };
    };
  };

  pkgs = import <nixpkgs> { inherit config; };

in
  { project1 = pkgs.haskellPackages.project1;
  }
```

This new derivation now also references a `turtle.nix` file which was generated
by `cabal2nix` by running:

```bash
$ cabal2nix cabal://turtle-1.3.0 > turtle.nix
```

If you try to run that and the command fails with this error message:

```bash
cabal2nix: ~/.cabal/packages/hackage.haskell.org/00-index.tar: openBinaryFile: does not exist (No such file or directory)
```

... then run `cabal update` and then the error should disappear.

The generated `turtle.nix` file looks like this:

```nix
{ mkDerivation, ansi-wl-pprint, async, base, bytestring, clock
, directory, doctest, foldl, hostname, managed, optional-args
, optparse-applicative, process, stdenv, stm, system-fileio
, system-filepath, temporary, text, time, transformers, unix
, unix-compat
}:
mkDerivation {
  pname = "turtle";
  version = "1.3.0";
  sha256 = "0iwsd78zhzc70d3dflshmy1h8qzn3x6mhij0h0gk92rc6iww2130";
  libraryHaskellDepends = [
    ansi-wl-pprint async base bytestring clock directory foldl hostname
    managed optional-args optparse-applicative process stm
    system-fileio system-filepath temporary text time transformers unix
    unix-compat
  ];
  testHaskellDepends = [ base doctest ];
  description = "Shell programming, Haskell-style";
  license = stdenv.lib.licenses.bsd3;
}
```

`nixpkgs` uses a `callPackage` utility function to "tie the knot" when updating
dependencies.  When we change`turtle` this way every package that depends on
`turtle` (including our `project1` package) will pick up this new version of
`turtle`.

The `nixpkgs` manual suggests an alternative approach of specifying package
overrides in a shared `~/.nixpkgs/config.nix` configuration file.  However, I do
not recommend this approach: project build instructions should be checked into
version control alongside the project so that they stay in sync with the
project.

At the time of this writing, if you try to build `release1.nix` then you will
get the following build error:

```bash
$ nix-build -A project1 release1.nix
these derivations will be built:
  /nix/store/r780xwf197a2gxn3008raq5k6xxid8mh-turtle-1.3.0.drv
  /nix/store/y6g5ya8lis6250fcj0mrw1kybjdara9i-project1-1.0.0.drv
building path(s) ‘/nix/store/qmyqayhvy4rxhfkxpdvc1ayc1vyp8nmw-turtle-1.3.0’
...
Configuring turtle-1.3.0...
Setup: Encountered missing dependencies:
optparse-applicative ==0.13.*
builder for ‘/nix/store/r780xwf197a2gxn3008raq5k6xxid8mh-turtle-1.3.0.drv’ failed with exit code 1
cannot build derivation ‘/nix/store/y6g5ya8lis6250fcj0mrw1kybjdara9i-project1-1.0.0.drv’: 1 dependencies couldn't be built
error: build of ‘/nix/store/y6g5ya8lis6250fcj0mrw1kybjdara9i-project1-1.0.0.drv’ failed
```

This error indicates that we can't upgrade `turtle-1.3.0` alone because
`turtle-1.3.0` depends on `optparse-applicative-0.13.*` and the default version
default `optparse-applicative` version that Nix selects is not in this range.
At the time of this writing, Nix picks `optparse-applicative-0.12.1.0` as the
default version.

We can override the `optparse-applicative` version using the exact same trick
and `release2.nix` contains this additional override:

```nix
# Note: This should also fail to build
let
  config = {
    packageOverrides = pkgs: rec {
      haskellPackages = pkgs.haskellPackages.override {
        overrides = haskellPackagesNew: haskellPackgesOld: rec {
          optparse-applicative =
            haskellPackagesNew.callPackage ./optparse-applicative.nix { };

          project1 =
            haskellPackagesNew.callPackage ./default.nix { };

          turtle =
            haskellPackagesNew.callPackage ./turtle.nix { };
        };
      };
    };
  };

  pkgs = import <nixpkgs> { inherit config; };

in
  { project1 = pkgs.haskellPackages.project1;
  }
```

... where `./optparse-applicative.nix` is also generated by `cabal2nix`:

```bash
$ cabal2nix cabal://optparse-applicative-0.13.0.0 > optparse-applicative.nix
```

However, that will still fail to build due to a test suite failure in
`optparse-applicative-0.13.0.0`:

```bash
$ nix-build -A project1 release2.nix
these derivations will be built:
  /nix/store/55b9hxwvknznfdqcksdfp8fqxifgw00p-optparse-applicative-0.13.0.0.drv
  /nix/store/9m816lg47c6wcgmjqpfxsyml4hgc739d-turtle-1.3.0.drv
  /nix/store/las95xi2lzn1lfc75k6acv05yvcw1cc2-project1-1.0.0.drv
building path(s) ‘/nix/store/bgam9vpvqx5j4x8fkasqwws54cqb0pbs-optparse-applicative-0.13.0.0’
...
Running 1 test suites...
Test suite optparse-applicative-tests: RUNNING...
...
=== prop_drops_back_contexts from tests/test.hs:153 ===
*** Failed! Exception: 'tests/dropback.err.txt: openFile: does not exist (No such file or directory)' (after 1 test): 

=== prop_context_carry from tests/test.hs:162 ===
*** Failed! Exception: 'tests/carry.err.txt: openFile: does not exist (No such file or directory)' (after 1 test): 

=== prop_help_on_empty from tests/test.hs:171 ===
*** Failed! Exception: 'tests/helponempty.err.txt: openFile: does not exist (No such file or directory)' (after 1 test): 

=== prop_help_on_empty_sub from tests/test.hs:180 ===
*** Failed! Exception: 'tests/helponemptysub.err.txt: openFile: does not exist (No such file or directory)' (after 1 test): 
...
Test suite optparse-applicative-tests: FAIL
Test suite logged to:
dist/test/optparse-applicative-0.13.0.0-optparse-applicative-tests.log
0 of 1 test suites (0 of 1 test cases) passed.
builder for ‘/nix/store/55b9hxwvknznfdqcksdfp8fqxifgw00p-optparse-applicative-0.13.0.0.drv’ failed with exit code 1
cannot build derivation ‘/nix/store/9m816lg47c6wcgmjqpfxsyml4hgc739d-turtle-1.3.0.drv’: 1 dependencies couldn't be built
cannot build derivation ‘/nix/store/las95xi2lzn1lfc75k6acv05yvcw1cc2-project1-1.0.0.drv’: 1 dependencies couldn't be built
error: build of ‘/nix/store/las95xi2lzn1lfc75k6acv05yvcw1cc2-project1-1.0.0.drv’ failed
```

However, we can instruct `cabal2nix` to disable the test suite for our
`optparse-applicative` applicative dependency by running:

```bash
$ cabal2nix --no-check cabal://optparse-applicative-0.13.0.0 > optparse-applicative-2.nix
```

`release3.nix` uses this test-free `optparse-applicative-2.nix` file:

```nix
let
  config = {
    packageOverrides = pkgs: rec {
      haskellPackages = pkgs.haskellPackages.override {
        overrides = haskellPackagesNew: haskellPackgesOld: rec {
          optparse-applicative =
            haskellPackagesNew.callPackage ./optparse-applicative-2.nix { };

          project1 =
            haskellPackagesNew.callPackage ./default.nix { };

          turtle =
            haskellPackagesNew.callPackage ./turtle.nix { };
        };
      };
    };
  };

  pkgs = import <nixpkgs> { inherit config; };

in
  { project1 = pkgs.haskellPackages.project1;
  }
```

... and now the build succeeds:

```bash
$ nix-build -A project1 release3.nix 
these derivations will be built:
  /nix/store/f17zxqqk582my4qfig78yvi6nv0vb588-turtle-1.3.0.tar.gz.drv
  /nix/store/vn2m1agg92fbmwj1h9168jywy5wmarah-turtle-1.3.0.drv
  /nix/store/c3vidydlyaa25kq2zs35kx80vv8sz1mk-project1-1.0.0.drv
these paths will be fetched (1.60 MiB download, 16.92 MiB unpacked):
  /nix/store/9kyi7kr3gzpxns4qalda26ww6jfzbpc7-syb-0.6
  /nix/store/9v640kglbndlrfpr6wpsq626gb2amx40-libssh2-1.7.0-dev
  /nix/store/cm1561r3fs320571n7cicggmjlgy3rdi-unix-compat-0.4.2.0
  /nix/store/hfy7dnf85gp1j8dqlhc2v0fdpsmc02vc-nghttp2-1.10.0
  /nix/store/kmqlvjmkrg66f5pnbx5jknfiyqf09jgx-curl-7.51.0-dev
  /nix/store/l06y84abvix2qmsxs5dbxllfsv9aznfk-hscolour-1.24.1
  /nix/store/pq0dcikqwvxxpiknmwki77nyalba3n4a-doctest-0.11.0
  /nix/store/v0wwvy3ygb52flq49z1yk445w2126rhs-base-compat-0.9.1
  /nix/store/vxqdcfj9kf0k1qvladvxl6wm21np8czh-optparse-applicative-0.13.0.0
  /nix/store/w5v8fjg704a5r38sk5dnf3zz7r8vapac-libev-4.22
  /nix/store/wyvv44zpn9j2fn30m2k2kry2cycqqlqv-ghc-paths-0.1.0.9
  /nix/store/xs8caklddfbmnvkgwcn38m0d7ivxdlq9-mirrors-list
  /nix/store/zdkjkdq4fcc6cwnxddm5mxa64n4pavb8-nghttp2-1.10.0-dev
...
building path(s) ‘/nix/store/frdmdrpr7rgv24i8p5qs5ifvcxz2jm6d-turtle-1.3.0’
...
Configuring turtle-1.3.0...
Dependency ansi-wl-pprint ==0.6.*: using ansi-wl-pprint-0.6.7.3
Dependency async >=2.0.0.0 && <2.2: using async-2.1.0
Dependency base >=4.6 && <5: using base-4.9.0.0
Dependency bytestring >=0.9.1.8 && <0.11: using bytestring-0.10.8.1
Dependency clock >=0.4.1.2 && <0.8: using clock-0.7.2
Dependency directory >=1.0.7 && <1.3: using directory-1.2.6.2
Dependency doctest >=0.7 && <0.12: using doctest-0.11.0
Dependency foldl >=1.1 && <1.3: using foldl-1.2.1
Dependency hostname <1.1: using hostname-1.0
Dependency managed >=1.0.3 && <1.1: using managed-1.0.5
Dependency optional-args >=1.0 && <2.0: using optional-args-1.0.1
Dependency optparse-applicative ==0.13.*: using optparse-applicative-0.13.0.0
Dependency process >=1.0.1.1 && <1.5: using process-1.4.2.0
Dependency stm <2.5: using stm-2.4.4.1
Dependency system-fileio >=0.2.1 && <0.4: using system-fileio-0.3.16.3
Dependency system-filepath >=0.3.1 && <0.5: using system-filepath-0.4.13.4
Dependency temporary <1.3: using temporary-1.2.0.4
Dependency text <1.3: using text-1.2.2.1
Dependency time <1.7: using time-1.6.0.1
Dependency transformers >=0.2.0.0 && <0.6: using transformers-0.5.2.0
Dependency turtle -any: using turtle-1.3.0
Dependency unix >=2.5.1.0 && <2.8: using unix-2.7.2.0
Dependency unix-compat ==0.4.*: using unix-compat-0.4.2.0
...
Building turtle-1.3.0...
Preprocessing library turtle-1.3.0...
[ 1 of 10] Compiling Turtle.Internal  ( src/Turtle/Internal.hs, dist/build/Turtle/Internal.o )
[ 2 of 10] Compiling Turtle.Line      ( src/Turtle/Line.hs, dist/build/Turtle/Line.o )
[ 3 of 10] Compiling Turtle.Shell     ( src/Turtle/Shell.hs, dist/build/Turtle/Shell.o )
[ 4 of 10] Compiling Turtle.Options   ( src/Turtle/Options.hs, dist/build/Turtle/Options.o )
[ 5 of 10] Compiling Turtle.Pattern   ( src/Turtle/Pattern.hs, dist/build/Turtle/Pattern.o )
[ 6 of 10] Compiling Turtle.Format    ( src/Turtle/Format.hs, dist/build/Turtle/Format.o )
[ 7 of 10] Compiling Turtle.Prelude   ( src/Turtle/Prelude.hs, dist/build/Turtle/Prelude.o )
[ 8 of 10] Compiling Turtle.Bytes     ( src/Turtle/Bytes.hs, dist/build/Turtle/Bytes.o )
[ 9 of 10] Compiling Turtle           ( src/Turtle.hs, dist/build/Turtle.o )
[10 of 10] Compiling Turtle.Tutorial  ( src/Turtle/Tutorial.hs, dist/build/Turtle/Tutorial.o )
Preprocessing test suite 'tests' for turtle-1.3.0...
[1 of 1] Compiling Main             ( test/Main.hs, dist/build/tests/tests-tmp/Main.dyn_o )
Linking dist/build/tests/tests ...
Preprocessing test suite 'regression-broken-pipe' for turtle-1.3.0...
[1 of 1] Compiling Main             ( test/RegressionBrokenPipe.hs, dist/build/regression-broken-pipe/regression-broken-pipe-tmp/Main.dyn_o )
Linking dist/build/regression-broken-pipe/regression-broken-pipe ...
running tests
Running 2 test suites...
Test suite tests: RUNNING...
Test suite tests: PASS
Test suite logged to: dist/test/turtle-1.3.0-tests.log
Test suite regression-broken-pipe: RUNNING...
Test suite regression-broken-pipe: PASS
Test suite logged to: dist/test/turtle-1.3.0-regression-broken-pipe.log
2 of 2 test suites (2 of 2 test cases) passed.
...
Configuring project1-1.0.0...
Dependency base <5: using base-4.9.0.0
Dependency turtle -any: using turtle-1.3.0
...
/nix/store/dv0vxc8pwrl8sdrfhy4y2ajnb1nh6qqy-project1-1.0.0
```

We can look at the difference between `optparse-applicative.nix` and
`optparse-applicative-2.nix` to see what changed:

```haskell
$ diff optparse-applicative.nix optparse-applicative-2.nix 
11a12
>   doCheck = false;
```

The latter file contains a `doCheck = false;` directive which instructs Nix to
not build and run the test suite.

# Github dependencies

Suppose that we wish to retrieve our `turtle` dependency from Github instead of
from Hackage.  All we have to do is run:

```bash
$ cabal2nix https://github.com/Gabriel439/Haskell-Turtle-Library.git --revision ba9c992933ae625cef40a88ea16ee857d1b93e13 > turtle-2.nix
```

... replacing `ba9c992933ae625cef40a88ea16ee857d1b93e13` with the revision we
wish to use.  You can omit the revision if you want `cabal2nix` to select the
revision of the current `master` branch.

`release4.nix` shows an example of depending on `turtle` from Github.  The only
difference is that we now depend on the `turtle-2.nix` file:

```nix
let
  config = {
    packageOverrides = pkgs: rec {
      haskellPackages = pkgs.haskellPackages.override {
        overrides = haskellPackagesNew: haskellPackgesOld: rec {
          optparse-applicative =
            haskellPackagesNew.callPackage ./optparse-applicative-2.nix { };

          project1 =
            haskellPackagesNew.callPackage ./default.nix { };

          turtle =
            haskellPackagesNew.callPackage ./turtle-2.nix { };
        };
      };
    };
  };

  pkgs = import <nixpkgs> { inherit config; };

in
  { project1 = pkgs.haskellPackages.project1;
  }
```

... and the difference between `turtle.nix` and `turtle-2.nix` is that the
dependency list changed and there is a new `src` field pointing to the Github
repository:

```diff
2,5c2,5
< , directory, doctest, foldl, hostname, managed, optional-args
< , optparse-applicative, process, stdenv, stm, system-fileio
< , system-filepath, temporary, text, time, transformers, unix
< , unix-compat
---
> , directory, doctest, fetchgit, foldl, hostname, managed
> , optional-args, optparse-applicative, process, stdenv, stm
> , system-fileio, system-filepath, temporary, text, time
> , transformers, unix, unix-compat
10c10,14
<   sha256 = "0iwsd78zhzc70d3dflshmy1h8qzn3x6mhij0h0gk92rc6iww2130";
---
>   src = fetchgit {
>     url = "https://github.com/Gabriel439/Haskell-Turtle-Library.git";
>     sha256 = "1gib4m85xk7h8zdrxpi5sxnjd35l3xnprg9kdy3cflacdzfn9pak";
>     rev = "ba9c992933ae625cef40a88ea16ee857d1b93e13";
>   };
```

# Source dependencies

You can also depend on local source checkouts of a given dependency.  For
example, if you checkout the `turtle` repository in some other directory then
all you have to do is run:

```haskell
$ cabal2nix /path/to/turtle > turtle.nix
```

... and then reference `./turtle.nix` in your `release.nix` file.  Now your
build will automatically pull in any changes you make to your source checkout of
`turtle`.

# Conclusion

This concludes basic dependency management in Nix.  You're now ready to begin
using Nix for your own Haskell projects!

[hackage-packages]: https://raw.githubusercontent.com/NixOS/nixpkgs/master/pkgs/development/haskell-modules/hackage-packages.nix
[hackage-packages-16.09]: https://raw.githubusercontent.com/NixOS/nixpkgs/release-16.09/pkgs/development/haskell-modules/hackage-packages.nix