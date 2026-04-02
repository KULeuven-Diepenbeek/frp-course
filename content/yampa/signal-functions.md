---
title: "Signal Functions"
weight: 1
draft: false
---

## Setup

Add Yampa to your Cabal project:

```cabal
build-depends:
    base   ^>=4.18
  , Yampa  ^>=0.14
```

Then in your source file:

```haskell
import FRP.Yampa
```

---

## The `SF` type

The central type in Yampa is the **signal function**:

```haskell
SF a b
```

A signal function transforms a **stream of input values of type `a`** into a **stream of output values of type `b`**, taking into account elapsed time.

You can think of it conceptually as:

$$\text{SF}\ a\ b \cong \text{Signal}\ a \to \text{Signal}\ b$$

where $\text{Signal}\ a = \text{Time} \to a$.

In practice, `SF a b` is an **opaque, stateful transformer**. It may internally accumulate state (e.g. integrate velocity to get position), use the elapsed time between samples, and respond to events. You never construct the internal state directly — you build signal functions by composing smaller ones.

---

## Primitive signal functions

### `arr` — lift a pure function

```haskell
arr :: (a -> b) -> SF a b
```

Converts any pure function into a signal function. No state, no time dependency.

```haskell
double :: SF Int Int
double = arr (*2)

toString :: SF Double String
toString = arr show
```

### `identity`

```haskell
identity :: SF a a
```

Passes the input through unchanged. Equivalent to `arr id`.

### `constant`

```haskell
constant :: b -> SF a b
```

Ignores the input and always outputs the same value.

```haskell
alwaysFive :: SF String Int
alwaysFive = constant 5
```

### `time`

```haskell
time :: SF a Time
```

Ignores the input and outputs the current simulation time (accumulated `DTime` since the start).

```haskell
clockSF :: SF () Double
clockSF = time
```

---

## Composition with `>>>`

Signal functions compose sequentially with `>>>` (from the `Arrow` type class):

```haskell
(>>>) :: SF a b -> SF b c -> SF a c
```

```haskell
pipeline :: SF Double String
pipeline = arr (*2) >>> arr show
-- equivalent to: arr (show . (*2))
```

The output of the first SF feeds directly into the second.

---

## Parallel composition

### `***` — pair inputs and outputs

```haskell
(***) :: SF a b -> SF c d -> SF (a, c) (b, d)
```

Run two signal functions in parallel on a pair of inputs:

```haskell
pairSF :: SF (Double, Double) (Double, Double)
pairSF = arr (*2) *** arr (+1)
-- input: (x, y)  ->  output: (x*2, y+1)
```

### `&&&` — fan-out

```haskell
(&&&) :: SF a b -> SF a c -> SF a (b, c)
```

Feed the same input into two SFs in parallel and pair the results:

```haskell
fanOut :: SF Double (Double, String)
fanOut = arr (*2) &&& arr show
-- input: x  ->  output: (x*2, show x)
```

---

## Integrating over time — `integral`

```haskell
integral :: VectorSpace a s => SF a a
```

Integrates the input signal over time. If the input represents **velocity**, the output is **position**. This is the most fundamental stateful primitive in Yampa.

```haskell
-- Constant downward velocity -> linearly increasing position
fallingPosition :: SF () Double
fallingPosition = constant 9.8 >>> integral
```

`DTime` (delta time) is handled automatically by Yampa's runtime — you do not pass time explicitly in your `SF` definition.

A more realistic free-fall (integrating acceleration to get velocity, then velocity to get position):

```haskell
import FRP.Yampa

gravity :: Double
gravity = -9.8

-- Starting at height 100, falling under gravity
freeFall :: SF () Double
freeFall = proc _ -> do
  velocity <- integral -< gravity
  position <- (100 +) ^<< integral -< velocity
  returnA -< position
```

---

## Arrow notation: `proc` and `-<`

Yampa's signal functions are **arrows**. The `proc` notation is syntactic sugar for arrow combinators and makes complex signal networks readable.

```haskell
{-# LANGUAGE Arrows #-}

mySignal :: SF Double Double
mySignal = proc input -> do
  doubled <- arr (*2)  -< input
  shifted <- arr (+10) -< doubled
  returnA -< shifted
```

Reading the notation:

| Syntax | Meaning |
|---|---|
| `proc x ->` | Name the input `x` |
| `y <- sf -< expr` | Feed `expr` into `sf`, bind result to `y` |
| `returnA -< expr` | Output `expr` as the signal function's output |

The `Arrows` language extension is required for `proc` notation.

A practical example — a simple oscillator:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

oscillator :: Double -> Double -> SF () Double
oscillator amplitude frequency = proc _ -> do
  t <- time -< ()
  returnA -< amplitude * sin (2 * pi * frequency * t)
```

---

## `loopPre` — feedback loops

```haskell
loopPre :: c -> SF (a, c) (b, c) -> SF a b
```

`loopPre initVal sf` creates a feedback loop. The signal function `sf` receives both the current input and the **previous output** (initialised to `initVal`). This is used to express recursive or iterative computations such as Euler integration:

```haskell
-- Manual Euler integrator using loopPre
eulerIntegral :: SF Double Double
eulerIntegral = loopPre 0 $ proc (velocity, prevPos) -> do
  dt <- DTime -- not directly accessible; use integral in practice
  returnA -< (prevPos, prevPos)
```

> In practice you almost always use the built-in `integral` rather than `loopPre`. The latter is more useful for state machines, which we cover in the Switches section.

---

## Testing a signal function with `embed`

Before wiring up a full reactive system, you can test signal functions offline using `embed`:

```haskell
embed :: SF a b -> (a, [(DTime, Maybe a)]) -> [b]
```

It runs the signal function with a list of (time-delta, input) pairs and returns the list of outputs.

```haskell
import FRP.Yampa

main :: IO ()
main = do
  let sf     = integral :: SF Double Double
      input  = (1.0, replicate 5 (0.1, Just 1.0))
      output = embed sf input
  print output
  -- [0.1, 0.2, 0.3, 0.4, 0.5]
```

`embed` is invaluable for unit-testing signal functions without writing a full `reactimate` loop.

---

## Summary

| Combinator | Type | Purpose |
|---|---|---|
| `arr f` | `SF a b` | Lift pure function |
| `identity` | `SF a a` | Pass-through |
| `constant v` | `SF a b` | Always output `v` |
| `time` | `SF a Time` | Current simulation time |
| `sf1 >>> sf2` | `SF a c` | Sequential composition |
| `sf1 *** sf2` | `SF (a,c) (b,d)` | Parallel on pairs |
| `sf1 &&& sf2` | `SF a (b,c)` | Fan-out same input |
| `integral` | `SF a a` | Integrate over time |
| `embed` | test utility | Run SF offline with sample data |
