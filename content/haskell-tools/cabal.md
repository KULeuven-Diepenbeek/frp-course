---
title: "Cabal — Build Tool & Dependency Manager"
weight: 2
draft: false
---

## What is Cabal?

**Cabal** (Common Architecture for Building Applications and Libraries) is the standard build system and package manager for Haskell. It serves two roles:

1. **Build tool**: it knows how to compile your modules, link executables, and run tests.
2. **Dependency manager**: it resolves, downloads, and builds the Haskell packages your project needs from [Hackage](https://hackage.haskell.org/), the central Haskell package repository.

Modern Cabal (>= 3.0) uses a **per-project dependency store** (inside `~/.cabal/store`) so that different projects can use different versions of the same library without conflict.

---

## Installing Cabal

Cabal is distributed with the [GHCup](https://www.haskell.org/ghcup/) toolchain installer. Install it with:

```bash
ghcup install cabal latest
```

Verify the installation:

```bash
cabal --version
# cabal-install version 3.12.x.x
```

---

## Creating a new project

```bash
cabal init --interactive
```

This wizard asks for:
- The project name.
- The type (library, executable, or both).
- The GHC version to target.
- The license.

The result is a directory structure like:

```
my-project/
  my-project.cabal      -- project description file
  app/
    Main.hs             -- entry point for the executable
  src/                  -- library source (if library was chosen)
  CHANGELOG.md
```

---

## The `.cabal` file

The `.cabal` file is the heart of a Cabal project. It declares everything: metadata, source directories, language extensions, dependencies, and build targets.

### Package-level metadata

```cabal
cabal-version:      3.4
name:               my-project
version:            0.1.0.0
synopsis:           A short description
description:        A longer description of the package.
license:            MIT
author:             Your Name
maintainer:         you@example.com
build-type:         Simple
```

### Declaring an executable

```cabal
executable my-project
  main-is:          Main.hs
  hs-source-dirs:   app
  build-depends:
      base          ^>=4.18
    , text          ^>=2.0
    , containers    ^>=0.6
  default-language: Haskell2010
  ghc-options:      -Wall
```

Key fields:

| Field | Purpose |
|---|---|
| `main-is` | The `.hs` file containing the `main` function |
| `hs-source-dirs` | Directories where GHC looks for source files |
| `build-depends` | List of required packages with version constraints |
| `default-language` | Haskell standard (`Haskell2010` or `GHC2021`) |
| `ghc-options` | Flags passed directly to GHC |

### Declaring a library

```cabal
library
  exposed-modules:
      MyLib
    , MyLib.Helpers
  hs-source-dirs:   src
  build-depends:
      base          ^>=4.18
  default-language: Haskell2010
```

### Declaring tests

```cabal
test-suite my-project-test
  type:             exitcode-stdio-1.0
  main-is:          Spec.hs
  hs-source-dirs:   test
  build-depends:
      base          ^>=4.18
    , my-project
    , hspec         ^>=2.11
  default-language: Haskell2010
```

### Language extensions in .cabal

Instead of adding language extensions to every source file manually, declare them once in the `.cabal` file under `default-extensions`:

```cabal
executable my-project
  ...
  default-extensions:
      OverloadedStrings
    , ScopedTypeVariables
    , LambdaCase
```

---

## Version constraints

Cabal uses a concise syntax for specifying compatible package versions:

| Constraint | Meaning |
|---|---|
| `== 2.0` | Exactly version 2.0 |
| `>= 2.0` | 2.0 or newer |
| `>= 2.0 && < 3.0` | Any 2.x version |
| `^>= 2.0` | `>= 2.0 && < 3` (the **caret operator**, most common) |

The caret operator `^>=` is the recommended choice for most dependencies. It expresses "this version or any compatible minor update".

---

## Common Cabal commands

### Building and running

```bash
# Build all targets in the project
cabal build

# Build and run the executable
cabal run

# Run a specific executable (when multiple are defined)
cabal run my-project -- --some-flag

# Start GHCi with the project loaded
cabal repl
```

### Testing

```bash
cabal test
```

### Adding a dependency

1. Open the `.cabal` file.
2. Add the package to the `build-depends` field.
3. Run `cabal build` — Cabal will resolve and install the new dependency automatically.

Example: adding Yampa:

```cabal
build-depends:
    base   ^>=4.18
  , Yampa  ^>=0.14
```

Then:

```bash
cabal build
```

### Searching for packages

```bash
# List all packages matching a keyword
cabal list yampa

# Show details of a specific package
cabal info Yampa
```

You can also browse [https://hackage.haskell.org/](https://hackage.haskell.org/) directly.

### Updating the package index

Cabal keeps a local copy of the Hackage package index. Refresh it periodically to see new releases:

```bash
cabal update
```

---

## `cabal.project` — multi-package workspaces

For larger projects or when you want to pin everything precisely, create a `cabal.project` file in the root of your workspace:

```
-- cabal.project
packages: .
           ../some-other-package

-- Pin a specific version of a dependency
constraints: text == 2.0.2
```

This file is optional for single-package projects but becomes essential when working with multiple related packages in one workspace.

---

## `cabal.project.freeze` — reproducible builds

```bash
cabal freeze
```

This writes a `cabal.project.freeze` file that records the exact version of every dependency (and transitive dependency) that was solved. Commit this file to version control to guarantee that every developer and every CI run uses identical dependencies.

---

## Typical development workflow

```bash
# 1. Create the project (once)
cabal init --interactive

# 2. Update the package index (periodically)
cabal update

# 3. Add dependencies in the .cabal file, then build
cabal build

# 4. Work interactively
cabal repl

# 5. Run tests
cabal test

# 6. Freeze dependencies before sharing
cabal freeze
```

---

## Project layout example

A typical Haskell project with Yampa:

```
frp-game/
  frp-game.cabal
  cabal.project.freeze
  app/
    Main.hs          -- reactimate loop, entry point
  src/
    Game/
      Logic.hs       -- signal functions
      Rendering.hs   -- output side effects
  test/
    Spec.hs
```

`frp-game.cabal`:

```cabal
cabal-version: 3.4
name:          frp-game
version:       0.1.0.0
build-type:    Simple

executable frp-game
  main-is:          Main.hs
  hs-source-dirs:   app, src
  build-depends:
      base   ^>=4.18
    , Yampa  ^>=0.14
    , time   ^>=1.12
  default-language:   Haskell2010
  default-extensions: OverloadedStrings
                       ScopedTypeVariables
  ghc-options:        -Wall -O2
```
