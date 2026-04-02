---
title: "I/O Programming in Haskell"
weight: 1
draft: false
---

## The `IO` type

Haskell is a **pure** language: a function with type `a -> b` cannot have side effects. I/O (reading from stdin, writing to stdout, reading files, …) is a side effect. Haskell handles this with the `IO` type.

A value of type `IO a` is an **action** that, when executed, performs some side effects and produces a result of type `a`. No side effect happens until the runtime executes the action. The Haskell runtime runs exactly one action: `main`.

```haskell
main :: IO ()
main = putStrLn "Hello, world!"
```

`IO ()` means: an action that performs side effects and returns the unit value `()` (nothing useful).

---

## `do` notation

The `do` block is syntactic sugar for chaining `IO` actions with the bind operator `>>=`. It reads like imperative code while remaining fully pure.

```haskell
main :: IO ()
main = do
  putStr "Enter your name: "
  name <- getLine            -- bind the result of getLine to `name`
  putStrLn ("Hello, " ++ name ++ "!")
```

Rules for `do` notation:

| Expression | Meaning |
|---|---|
| `action` | Execute `action`, discard the result |
| `x <- action` | Execute `action`, bind the result to `x` |
| `let x = expr` | Pure binding, no side effect |
| `return v` | Wrap a pure value `v` in `IO` without any effect |

---

## Standard I/O functions

### Output

```haskell
putStr   :: String -> IO ()   -- print without a newline
putStrLn :: String -> IO ()   -- print with a newline
print    :: Show a => a -> IO ()  -- print any Show-able value (adds quotes for strings)
```

### Input

```haskell
getLine  :: IO String          -- read one line from stdin
getChar  :: IO Char            -- read one character
getContents :: IO String       -- read all of stdin lazily
```

### File I/O

```haskell
import System.IO

readFile  :: FilePath -> IO String
writeFile :: FilePath -> String -> IO ()
appendFile :: FilePath -> String -> IO ()

-- Lower-level handles for more control
withFile :: FilePath -> IOMode -> (Handle -> IO r) -> IO r
```

Example — read a file and count its lines:

```haskell
import System.IO

main :: IO ()
main = do
  contents <- readFile "data.txt"
  let lineCount = length (lines contents)
  putStrLn ("Line count: " ++ show lineCount)
```

---

## `return` and `pure`

`return` (or its synonym `pure`) lifts a pure value into the `IO` monad. It does **not** stop execution (unlike `return` in C or Java).

```haskell
greet :: String -> IO String
greet name = return ("Hello, " ++ name)

main :: IO ()
main = do
  msg <- greet "Alice"
  putStrLn msg
```

---

## Looping with `IO`

Haskell does not have `for` or `while` loops. Looping is expressed with recursion or with combinators from `Control.Monad`.

### Recursive loop

```haskell
countDown :: Int -> IO ()
countDown 0 = putStrLn "Blast off!"
countDown n = do
  print n
  countDown (n - 1)
```

### `forM_` — iterating over a list

```haskell
import Control.Monad (forM_)

main :: IO ()
main = forM_ [1..5] $ \i -> do
  putStrLn ("Item: " ++ show i)
```

### `forever` — infinite loop

```haskell
import Control.Monad (forever)

echoLoop :: IO ()
echoLoop = forever $ do
  line <- getLine
  putStrLn ("You said: " ++ line)
```

### `when` and `unless`

```haskell
import Control.Monad (when, unless)

main :: IO ()
main = do
  x <- readLn :: IO Int
  when (x > 0) $ putStrLn "Positive"
  unless (x > 0) $ putStrLn "Non-positive"
```

---

## Sequencing actions

Sometimes you want to collect results from a list of `IO` actions into a list:

```haskell
import Control.Monad (replicateM)

-- Ask for n numbers and return them as a list
readNumbers :: Int -> IO [Int]
readNumbers n = replicateM n readLn
```

Or sequence a list of actions discarding results:

```haskell
import Control.Monad (mapM_)

printAll :: [String] -> IO ()
printAll = mapM_ putStrLn
```

---

## Exception handling

Use `Control.Exception` to catch runtime errors:

```haskell
import Control.Exception

safeDivide :: Int -> Int -> IO Int
safeDivide _ 0 = throwIO (userError "Division by zero")
safeDivide x y = return (x `div` y)

main :: IO ()
main = do
  result <- try (safeDivide 10 0) :: IO (Either SomeException Int)
  case result of
    Left err  -> putStrLn ("Error: " ++ show err)
    Right val -> print val
```

---

## Interacting with the command line

```haskell
import System.Environment (getArgs, getProgName)

main :: IO ()
main = do
  progName <- getProgName
  args     <- getArgs
  putStrLn ("Program: " ++ progName)
  putStrLn ("Arguments: " ++ show args)
```

---

## Summary

| Concept | Key function/type |
|---|---|
| I/O action | `IO a` |
| Print to terminal | `putStr`, `putStrLn`, `print` |
| Read from terminal | `getLine`, `getChar`, `readLn` |
| File operations | `readFile`, `writeFile`, `withFile` |
| Wrap pure value | `return` / `pure` |
| Iterate list | `forM_`, `mapM_` |
| Infinite loop | `forever` |
| Conditional action | `when`, `unless` |
| Collect results | `replicateM`, `mapM` |
