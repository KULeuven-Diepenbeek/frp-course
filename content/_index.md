---
title: "Index"
---

# Functional Reactive Programming

Academic year 2025–2026.

## Overview

This course builds on your existing knowledge of Haskell and functional programming to introduce:

1. **Haskell tooling** — configuring GHCi and using Cabal as a build system and dependency manager.
2. **I/O and state in Haskell** — working with the `IO` monad, mutable references, global state, and concurrent queues.
3. **Functional Reactive Programming (FRP)** — the paradigm: what it is, why it exists, and how it models time-varying values and event streams.
4. **Yampa** — a mature, arrow-based FRP library for Haskell, from first signals to advanced switching patterns.

## Prerequisites

You are expected to already know:

- Haskell syntax, types, type classes, and higher-order functions.
- Basic functional programming concepts (map, filter, fold, currying, …).

If you need a refresher visit the Haskell learning resources in the Extra section.

## Course structure

| # | Section | Topics |
|---|---------|--------|
| 1 | Haskell Tools | `.ghci` configuration, Cabal projects |
| 2 | I/O & State | `IO` monad, `IORef`, `MVar`, `TQueue` |
| 3 | FRP Concepts | Time-varying values, event streams, FRP paradigm |
| 4 | Yampa — Basics | Signal functions, `SF a b`, `arr`, `>>>` |
| 5 | Yampa — Events | `Event`, `edge`, `never`, `now` |
| 6 | Yampa — Running | `reactimate`, game loops |
| 7 | Yampa — Switches | `switch`, `dSwitch`, `rSwitch`, `kSwitch` |
| 8 | Yampa — Advanced | State machines, `dpSwitch`, `par`, `pSwitch` |
