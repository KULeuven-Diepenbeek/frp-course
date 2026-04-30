# Functional Reactive Programming — Course Website

A Hugo-based course website covering Functional Reactive Programming (FRP) in Haskell, built with the [KU Leuven Diepenbeek Hugo theme](https://github.com/KULeuven-Diepenbeek/hugo-theme-kul).

## Course content

| Section | Pages |
|---|---|
| **Haskell Tools** | `.ghci` configuration, Cabal build tool & dependency manager |
| **I/O and State** | `IO` monad basics, `IORef`/`MVar` global state, concurrency & queues |
| **FRP Concepts** | What is FRP, behaviours, events, push vs pull |
| **Yampa** | Signal functions, events, `reactimate`, switches, advanced patterns |
| **Extra** | References and further reading |

## Prerequisites

Students are expected to have prior knowledge of Haskell syntax, types, type classes, and basic functional programming. This course does **not** contain an introduction to Haskell or functional programming.

## Getting started (development)

```bash
# 1. Clone with submodules (picks up the Hugo theme)
git clone --recurse-submodules <repo-url>
cd frp-course

# 2. If you already cloned without --recurse-submodules:
git submodule update --init

# 3. Serve locally with live reload
hugo server

# 4. Build for production (output goes to docs/)
hugo --minify
```

> **Note:** `enableGitInfo = true` in `config.toml` requires at least one commit in the repository for `hugo` to work without `--noTimes`. Run `git add . && git commit -m "initial"` after cloning if you get a git-related build error.

## Deploying to GitHub Pages

Set the `baseurl` in `config.toml` to your repository's GitHub Pages URL, then push `docs/` (the `publishDir`) as part of your release. A GitHub Actions workflow can automate this — see the other courses in this organisation for an example workflow.

## Theme

Uses the [`hugo-theme-kul`](https://github.com/KULeuven-Diepenbeek/hugo-theme-kul) theme, shared with the Cloud, Databases, and Full-Stack Web Development courses.
