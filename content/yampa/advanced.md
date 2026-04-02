---
title: "Advanced Yampa"
weight: 5
draft: false
---

## Topics covered

This section collects patterns that go beyond the basics:

- Recursive signal functions (self-referential `SF`)
- Dynamic collections with `dpSwitch`
- Continuations and higher-order signal functions
- Integration with real I/O (full worked example)
- Performance considerations

---

## Recursive signal functions

Yampa supports **recursive signal function networks** using `loop` (from the `ArrowLoop` class):

```haskell
loop :: SF (a, c) (b, c) -> SF a b
```

The second output component is fed back as an additional input component, with a **one-sample delay** to break the cycle.

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Manual double-integration using loop
-- (integrate velocity to get position,
--  the position feeds back conceptually — here just shown structurally)
doubleIntegral :: SF Double Double
doubleIntegral = proc accel -> do
  vel <- integral -< accel
  pos <- integral -< vel
  returnA -< pos
```

A practical use for `loop`: a PID controller where the error is computed from the difference between a target and the current (fed-back) output.

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Simple P controller: output = Kp * (target - current)
-- Uses loopPre to feed output back
pController :: Double -> SF Double Double
pController kp = loopPre 0 $ proc (target, prev) -> do
  let err    = target - prev
      output = kp * err
  returnA -< (output, output)
```

---

## Higher-order signal functions

Signal functions are **first-class values** in Haskell. You can pass them as arguments, store them in data structures, and return them from functions — enabling higher-order FRP patterns.

### Parameterised SFs

```haskell
import FRP.Yampa

-- A spring: returns a SF for a spring with given stiffness k and damping d
springSystem :: Double -> Double -> SF Double Double
springSystem k d = proc force -> do
  rec
    pos <- (0 +) ^<< integral -< vel
    vel <- (0 +) ^<< integral -< (force - k * pos - d * vel)
  returnA -< pos
```

### Selecting between SFs at runtime

```haskell
choose :: Bool -> SF a b -> SF a b -> SF a b
choose True  sf _  = sf
choose False _  sf = sf
```

Of course, `choose` ignores time — for *dynamic* switching use `rSwitch` (see Switches section).

---

## Dynamic collections with `dpSwitch`

`dpSwitch` manages a **dynamic collection** of parallel signal functions. Use it when entities can be added or removed at runtime (enemy waves, particles, UI widgets).

```haskell
dpSwitch :: Functor col
          => (forall sf. (a -> col sf -> col (ext, sf)))
          -> col (SF ext b)
          -> SF (a, col b) (Event c)
          -> (col (SF ext b) -> c -> SF a (col b))
          -> SF a (col b)
```

Compared to `pSwitch`, `dpSwitch` delays the reorganisation by one step.

**Conceptual entity system:**

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa
import qualified Data.IntMap.Strict as IM

type EntityId  = Int
type EntityMap = IM.IntMap

data EntityInput = EntityInput { deltaTime :: DTime }

-- Each entity is its own SF
enemy :: EntityId -> SF EntityInput (Double, Double)
enemy eid = proc inp -> do
  t <- time -< ()
  let x = fromIntegral eid * 50.0 + sin t * 10.0
      y = 100.0
  returnA -< (x, y)

-- Route: give each entity the global input
routeEntities :: EntityInput
              -> EntityMap (SF EntityInput (Double, Double))
              -> EntityMap (EntityInput, SF EntityInput (Double, Double))
routeEntities inp = IM.map (\sf -> (inp, sf))

-- Supervision: remove entities that go off-screen (y < 0)
supervise :: SF (EntityInput, EntityMap (Double, Double)) (Event (EntityMap (SF EntityInput (Double, Double)) -> EntityMap (SF EntityInput (Double, Double))))
supervise = proc (_, positions) -> do
  let offscreen = IM.filter (\(_, y) -> y < 0) positions
  returnA -< if IM.null offscreen
               then NoEvent
               else Event (IM.filterWithKey (\k _ -> not (IM.member k offscreen)))
```

---

## A full worked example — a simple game

This example ties together `reactimate`, signal functions, events, and switches in a minimal console game: a ball falls and you press ENTER to "bounce" it.

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa
import Control.Concurrent
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
import Data.Time.Clock
import Data.IORef
import System.IO

-- Input events from the player
data GameInput = Bounce | Tick deriving (Show, Eq)

-- Output drawn each frame
data GameOutput = GameOutput
  { goHeight :: Double
  , goScore  :: Int
  } deriving (Show)

-- The ball: position and score, resets on Bounce event
ball :: SF [GameInput] GameOutput
ball = proc inputs -> do
  let bounceEvt = if Bounce `elem` inputs then Event () else NoEvent
  score    <- accumHold 0 -< fmap (const (+1)) bounceEvt
  height   <- ballHeight   -< bounceEvt
  returnA  -< GameOutput height score

ballHeight :: SF (Event ()) Double
ballHeight = switch (falling 100.0) (\_ -> ballHeight)
  where
    falling h = proc bounceEvt -> do
      vel <- integral        -< (-50.0)
      pos <- (h +) ^<< integral -< vel
      let hitEvt = if pos <= 0 then Event () else NoEvent
      let merged = mergeEvents [bounceEvt, hitEvt]
      returnA -< (max 0 pos, merged)

-- Sense: read from TQueue + measure time
senseInput :: TQueue GameInput
           -> IORef UTCTime
           -> Bool
           -> IO (DTime, Maybe [GameInput])
senseInput q lastRef _ = do
  threadDelay 16667          -- ~60 FPS
  now    <- getCurrentTime
  prev   <- readIORef lastRef
  writeIORef lastRef now
  inputs <- drainQueue q
  return (realToFrac (diffUTCTime now prev), Just inputs)

drainQueue :: TQueue a -> IO [a]
drainQueue q = go []
  where
    go acc = do
      mv <- atomically (tryReadTQueue q)
      case mv of
        Nothing -> return (reverse acc)
        Just v  -> go (v : acc)

-- Input thread: read keyboard asynchronously
inputThread :: TQueue GameInput -> IO ()
inputThread q = do
  hSetBuffering stdin NoBuffering
  hSetEcho stdin False
  let loop = do
        c <- getChar
        when (c == '\n') $ atomically (writeTQueue q Bounce)
        loop
  loop

main :: IO ()
main = do
  inputQ   <- newTQueueIO
  lastTime <- newIORef =<< getCurrentTime
  forkIO (inputThread inputQ)

  putStrLn "Press ENTER to bounce the ball. Ctrl-C to quit."

  reactimate
    (return [])                             -- init
    (senseInput inputQ lastTime)            -- sense
    (\_ out -> do                           -- actuate
        let h = round (goHeight out) :: Int
        putStr ("\rheight=" ++ show h ++ "  score=" ++ show (goScore out) ++ "   ")
        return False)
    ball
```

---

## Performance considerations

### Avoiding time leaks

Yampa's `integral` can accumulate large floats over time. For long-running simulations, consider resetting the clock-based position occasionally or using fixed-point arithmetic.

### `dpSwitch` vs `pSwitch`

Prefer `dpSwitch` in most cases — the one-step delay breaks causality cycles and generally leads to more predictable behaviour. Only use `pSwitch` if you need the reorganisation to take effect in the same instant as the event.

### Profiling

Add `+RTS -N -sstderr` when running to see GC pressure. Strict operations (`modifyIORef'`, `seq`, `deepseq`) help avoid space leaks in long loops.

---

## Useful packages

| Package | Description |
|---|---|
| `Yampa` | Core FRP library |
| `yampa-test` | Property-based testing for SFs using QuickCheck |
| `bearriver` | Drop-in Yampa replacement, dunai-based |
| `dunai` | Generalised reactive systems (monadic stream functions) |
| `keera-hails` | Full game framework built on Yampa |
| `sdl2` | SDL2 bindings for rendering/input (pairs well with Yampa) |
| `gloss` | Simple 2D graphics in Haskell, easy to drive with Yampa |

---

## Summary

| Pattern | When to use |
|---|---|
| `loop` / `loopPre` | Recursive/feedback signal networks |
| Parameterised SFs | Reusable physics, controllers |
| `dpSwitch` | Dynamic collections of entities |
| `rSwitch` | Hot-swap the active SF from an event stream |
| `TQueue` + `forkIO` | Decouple async input from simulation step |
| `embed` | Unit-test signal functions without a real loop |
