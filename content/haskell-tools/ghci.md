---
title: "Configuring GHCi with .ghci"
weight: 1
draft: false
---

## What is `.ghci`?

When GHCi starts, it looks for a configuration file called `.ghci`. This file is executed as a sequence of GHCi commands before the interactive prompt appears. It is the standard way to customise the GHCi session for a project without passing flags on the command line every time.

GHCi looks for the file in the following order:

1. `./.ghci` — a project-level file in the current working directory.
2. `~/.ghci` — a user-level file in the home directory.

The project-level file takes precedence; settings in the user-level file act as global defaults.

---

## Basic syntax

Every line in a `.ghci` file is either:

- A **GHCi meta-command** starting with `:` (e.g. `:set`, `:load`, `:module`).
- A **Haskell expression** that is evaluated on startup (useful to import modules or define helpers).
- A **comment** starting with `--`.

```haskell
-- .ghci example
:set prompt "λ> "
:set -Wall
import Data.List (sort)
```

---

## Common settings

### Changing the prompt

The default GHCi prompt is `Prelude>`. A custom prompt makes it easier to see which modules are loaded and gives the session a personal touch.

```haskell
:set prompt "ghci> "
-- or use ANSI colours (supported in modern terminals)
:set prompt "\ESC[1;32mghci\ESC[0m> "
```

For multi-line prompts (continuation lines):

```haskell
:set prompt-cont "  | "
```

### Enabling compiler warnings

```haskell
-- Enable all warnings (equivalent to -Wall on the command line)
:set -Wall

-- Enable some specific groups
:set -Wincomplete-patterns
:set -Wunused-imports
```

`-Wall` is strongly recommended in a course setting — it catches incomplete pattern matches, unused bindings, and many other common mistakes before they become bugs.

### Enabling language extensions

You can activate GHC language extensions inside `.ghci` using `:set -X<ExtensionName>`:

```haskell
:set -XOverloadedStrings
:set -XScopedTypeVariables
:set -XTupleSections
```

This is equivalent to writing `{-# LANGUAGE OverloadedStrings #-}` at the top of every file you load in that session.

> **Warning:** Extensions enabled globally in `.ghci` apply only to the interactive session, not to compiled code. For permanent project-wide extensions, add them to your `.cabal` file (see the Cabal section).

### Setting the editor

GHCi's `:edit` command opens a file in an external editor. Configure which editor to use:

```haskell
:set editor vim
-- or
:set editor "code --wait"
```

### Loading modules automatically

You can ask GHCi to load a specific module every time it starts in a given project directory:

```haskell
:load Main
```

Or import frequently used modules so they are always available:

```haskell
import Control.Monad (forM_, when, unless)
import Data.Maybe (fromMaybe, mapMaybe)
```

---

## A practical project-level `.ghci`

Below is a realistic `.ghci` file for a small Haskell project:

```haskell
-- .ghci --- project settings

-- Custom prompt showing the current module
:set prompt "\ESC[34m[ghci]\ESC[0m \ESC[33m%s\ESC[0m> "
:set prompt-cont "      | "

-- Warnings
:set -Wall
:set -Wno-name-shadowing

-- Useful language extensions for interactive work
:set -XOverloadedStrings
:set -XScopedTypeVariables

-- Load the project's main module
:load Main

-- A helper to re-run main quickly
let rerun = main
```

After saving this file, simply run `ghci` from the project root and it will pick up all these settings automatically.

---

## User-level vs. project-level `.ghci`

| Scope | Location | When to use |
|---|---|---|
| Project | `./.ghci` | Project-specific extensions, loading `Main`, custom prompts per project |
| User | `~/.ghci` | Personal defaults: preferred prompt style, always-imported utilities |

It is a good habit to keep the user-level file minimal and put project-specific settings in the project-level file. This prevents settings from one project accidentally affecting another.

---

## Security note

GHCi reads and evaluates the `.ghci` file automatically. Never trust a `.ghci` file from an unknown source: it can execute arbitrary Haskell IO actions on startup (e.g. delete files, make network connections). Always inspect `.ghci` before running GHCi in a directory you did not create yourself.

---

## Summary

| Setting | Example |
|---|---|
| Custom prompt | `:set prompt "λ> "` |
| Continuation prompt | `:set prompt-cont " \| "` |
| All warnings | `:set -Wall` |
| Language extension | `:set -XOverloadedStrings` |
| Editor | `:set editor "code --wait"` |
| Auto-load module | `:load Main` |
