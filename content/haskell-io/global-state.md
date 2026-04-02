---
title: "Global State — IORef and MVar"
weight: 2
draft: false
---

## Why mutable state?

Haskell values are immutable by default. But real programs often need to track state over time: a counter, a configuration setting, a cache. Haskell provides **mutable references** inside `IO` to handle this in a principled way.

The key insight is that mutability is **not hidden**: the type system makes it visible. A mutable reference always lives inside `IO` (or another monad with `IO` capabilities), so you can never accidentally mutate something in a pure function.

---

## `IORef` — a simple mutable cell

`IORef a` is a mutable reference to a value of type `a`. It lives in the `IO` monad.

### Import

```haskell
import Data.IORef
```

### Creating and reading

```haskell
newIORef   :: a -> IO (IORef a)        -- create with initial value
readIORef  :: IORef a -> IO a          -- read current value
writeIORef :: IORef a -> a -> IO ()    -- overwrite with new value
modifyIORef :: IORef a -> (a -> a) -> IO ()  -- apply a pure function
modifyIORef' :: IORef a -> (a -> a) -> IO () -- strict version (avoids thunk build-up)
```

### Example — simple counter

```haskell
import Data.IORef

main :: IO ()
main = do
  counter <- newIORef (0 :: Int)

  -- Increment ten times
  mapM_ (\_ -> modifyIORef' counter (+1)) [1..10]

  val <- readIORef counter
  putStrLn ("Final count: " ++ show val)
```

### When to use `modifyIORef'` vs `modifyIORef`

The tick (`'`) version is **strict**: it evaluates the new value immediately. The non-strict version builds up a chain of thunks, which can cause a space leak in long-running loops. **Always prefer `modifyIORef'`** when modifying inside a loop.

---

## Global variables with `IORef`

A common pattern is to define a top-level `IORef` using `unsafePerformIO`. This should be used sparingly and only when there is genuinely one piece of global state for the whole program (e.g., a global configuration or a logger).

```haskell
import Data.IORef
import System.IO.Unsafe (unsafePerformIO)

-- Global mutable counter — note: not thread-safe
{-# NOINLINE globalCounter #-}
globalCounter :: IORef Int
globalCounter = unsafePerformIO (newIORef 0)

incrementGlobal :: IO ()
incrementGlobal = modifyIORef' globalCounter (+1)

readGlobal :: IO Int
readGlobal = readIORef globalCounter
```

> **Warning:** `unsafePerformIO` bypasses Haskell's purity guarantees. It is correct here only because `globalCounter` is a referentially stable, unique allocation. Never use it to perform arbitrary IO at the top level. For multi-threaded programs, use `MVar` or `TVar` instead (see below).

A **safer** alternative that avoids `unsafePerformIO` is to create the `IORef` in `main` and pass it down explicitly:

```haskell
import Data.IORef

runApp :: IORef Int -> IO ()
runApp counter = do
  modifyIORef' counter (+1)
  val <- readIORef counter
  putStrLn ("Counter is now: " ++ show val)

main :: IO ()
main = do
  counter <- newIORef (0 :: Int)
  runApp counter
  runApp counter
```

This is the recommended approach for most programs.

---

## `MVar` — thread-safe mutable state

`MVar a` is a **mutable variable** designed for safe concurrent access. It works like a box that can be either **full** (holds a value) or **empty**:

- **Taking** from an empty `MVar` blocks until it becomes full.
- **Putting** into a full `MVar` blocks until it becomes empty.

This blocking behaviour makes `MVar` a natural mutex.

### Import

```haskell
import Control.Concurrent.MVar
```

### Core operations

```haskell
newMVar     :: a -> IO (MVar a)             -- create full MVar
newEmptyMVar :: IO (MVar a)                 -- create empty MVar
takeMVar    :: MVar a -> IO a               -- take value (empties it, blocks if empty)
putMVar     :: MVar a -> a -> IO ()         -- put value (blocks if full)
readMVar    :: MVar a -> IO a               -- read without emptying (atomic)
modifyMVar_ :: MVar a -> (a -> IO a) -> IO () -- atomic modify
modifyMVar  :: MVar a -> (a -> IO (a, b)) -> IO b -- atomic modify with result
```

### Example — thread-safe counter

```haskell
import Control.Concurrent
import Control.Concurrent.MVar

main :: IO ()
main = do
  counter <- newMVar (0 :: Int)

  -- Spawn 10 threads, each incrementing the counter 1000 times
  let increment = modifyMVar_ counter (\c -> return (c + 1))
  threads <- mapM (\_ -> forkIO (mapM_ (const increment) [1..1000])) [1..10]

  -- Give threads time to finish
  threadDelay 100000    -- 100 ms

  final <- readMVar counter
  putStrLn ("Expected: 10000, Got: " ++ show final)
```

### `MVar` as a mutex

`MVar` is often used purely as a **lock** — the value inside is `()`, and the presence/absence of the value represents locked/unlocked:

```haskell
import Control.Concurrent.MVar

withLock :: MVar () -> IO a -> IO a
withLock lock action = do
  takeMVar lock          -- acquire
  result <- action
  putMVar lock ()        -- release
  return result

main :: IO ()
main = do
  lock <- newMVar ()
  withLock lock $ putStrLn "Critical section"
```

---

## STM and `TVar` — composable transactions

For more complex concurrent state, the **Software Transactional Memory** (STM) library provides `TVar` — a transactional variable. Multiple `TVar` reads and writes can be grouped into a single atomic transaction.

```haskell
import Control.Concurrent.STM

transfer :: TVar Int -> TVar Int -> Int -> STM ()
transfer from to amount = do
  fromBal <- readTVar from
  toBal   <- readTVar to
  writeTVar from (fromBal - amount)
  writeTVar to   (toBal   + amount)

main :: IO ()
main = do
  accountA <- newTVarIO 1000
  accountB <- newTVarIO 500
  atomically (transfer accountA accountB 200)
  a <- readTVarIO accountA
  b <- readTVarIO accountB
  putStrLn ("A: " ++ show a ++ ", B: " ++ show b)
```

`atomically` runs the transaction; if any `TVar` is modified by another thread during the transaction, the whole transaction **retries automatically**. This eliminates entire classes of race conditions and deadlocks.

---

## Choosing the right tool

| Scenario | Use |
|---|---|
| Simple mutable state, single thread | `IORef` |
| Shared state between threads (simple) | `MVar` |
| Shared state between threads (complex, atomic) | `TVar` / STM |
| Global singleton (use sparingly) | `IORef` + `unsafePerformIO` or pass as argument |
