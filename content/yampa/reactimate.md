---
title: "Running Yampa — reactimate"
weight: 3
draft: false
---

## The simulation loop

A signal function `SF a b` is a description. To actually **run** it, you need:

1. A way to **sense** the current input (e.g. read keyboard state, sample a sensor).
2. A way to **actuate** on the output (e.g. draw to screen, send a network packet).
3. A timing source to know how much time has passed since the last step.

Yampa's `reactimate` function wires these three things together into a continuous simulation loop.

---

## `reactimate`

```haskell
reactimate :: IO a                        -- init: produce the first input
           -> (Bool -> IO (DTime, Maybe a))  -- sense: produce subsequent inputs
           -> (Bool -> b -> IO Bool)         -- actuate: consume output
           -> SF a b                         -- the signal function to run
           -> IO ()
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `init` | `IO a` | Called once at the start. Returns the first input sample. |
| `sense` | `Bool -> IO (DTime, Maybe a)` | Called each step. Returns elapsed time and optionally a new input. If `Nothing`, the previous input is reused. |
| `actuate` | `Bool -> b -> IO Bool` | Called with each output. Returns `True` to stop the loop, `False` to continue. |
| `sf` | `SF a b` | The signal function to drive. |

The `Bool` passed to `sense` and `actuate` is `True` on the very first call (can be used for initialisation logic).

---

## Minimal example — a simple timer

```haskell
import FRP.Yampa
import Control.Concurrent (threadDelay)
import Data.IORef
import Data.Time.Clock

-- Signal function: output = total elapsed time in seconds
timerSF :: SF () Double
timerSF = time

main :: IO ()
main = do
  lastTime <- newIORef =<< getCurrentTime

  reactimate
    (return ())                              -- init: unit input

    (\_ -> do                                -- sense: measure elapsed time
        now  <- getCurrentTime
        prev <- readIORef lastTime
        writeIORef lastTime now
        let dt = realToFrac (diffUTCTime now prev) :: DTime
        return (dt, Nothing))                -- reuse previous input ()

    (\_ t -> do                              -- actuate: print elapsed time
        putStrLn ("t = " ++ show (round t :: Int) ++ "s")
        return (t >= 5.0))                   -- stop after 5 seconds

    timerSF
```

---

## Controlling the loop frequency

`reactimate` calls `sense` and `actuate` as fast as possible. To run at a fixed frame rate, sleep in the `sense` callback:

```haskell
import Control.Concurrent (threadDelay)

senseWithRate :: Int -> IORef UTCTime -> Bool -> IO (DTime, Maybe ())
senseWithRate fps lastRef _ = do
  threadDelay (1000000 `div` fps)   -- sleep for one frame duration
  now  <- getCurrentTime
  prev <- readIORef lastRef
  writeIORef lastRef now
  return (realToFrac (diffUTCTime now prev), Nothing)
```

---

## Integrating with a `TQueue` for external input

In a real application (game, robotics, GUI), inputs come from external sources (keyboard, sensors). The clean pattern is:

1. Run an **input thread** that writes events to a `TQueue`.
2. In the `sense` callback, **drain the queue** to collect all events since the last step.

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa
import Control.Concurrent
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
import Data.Time.Clock
import Data.IORef

data InputEvent = KeyW | KeyS | Quit deriving (Show, Eq)

-- Fake input thread: sends a few events and then Quit
inputThread :: TQueue InputEvent -> IO ()
inputThread q = do
  threadDelay 500000
  atomically $ writeTQueue q KeyW
  threadDelay 500000
  atomically $ writeTQueue q KeyS
  threadDelay 1000000
  atomically $ writeTQueue q Quit

-- Drain all pending events from the queue (non-blocking)
drainQueue :: TQueue a -> IO [a]
drainQueue q = go []
  where
    go acc = do
      mVal <- atomically (tryReadTQueue q)
      case mVal of
        Nothing -> return (reverse acc)
        Just v  -> go (v : acc)

-- Signal function: count W keypresses
gameLogic :: SF [InputEvent] Int
gameLogic = proc events -> do
  let wCount = length (filter (== KeyW) events)
  total <- accumHold 0 -< if wCount > 0 then Event (+wCount) else NoEvent
  returnA -< total

main :: IO ()
main = do
  inputQ   <- newTQueueIO
  lastTime <- newIORef =<< getCurrentTime
  forkIO (inputThread inputQ)

  reactimate
    (drainQueue inputQ)          -- init: collect any events already queued

    (\_ -> do                    -- sense
        threadDelay 16667        -- ~60 FPS
        now    <- getCurrentTime
        prev   <- readIORef lastTime
        writeIORef lastTime now
        events <- drainQueue inputQ
        return (realToFrac (diffUTCTime now prev), Just events))

    (\_ wPresses -> do           -- actuate
        putStrLn ("W presses: " ++ show wPresses)
        -- Check if Quit was received
        return False)            -- simplified: never quit from here

    gameLogic
```

> In a production game you would check the event list for `Quit` inside the actuate callback and return `True` when found.

---

## `reactimateB` — returning a behaviour

```haskell
reactimateB :: SF a b -> IO (a -> IO b)
```

(Note: this is `reactimateB` from some Yampa versions / wrappers, not always available.) More commonly, a variant using `reactimate` is standard.

---

## `embed` — offline testing

For testing without a real event loop, use `embed`:

```haskell
embed :: SF a b -> (a, [(DTime, Maybe a)]) -> [b]
```

This runs the `SF` on a fixed list of (time-delta, input) pairs and returns all outputs.

```haskell
import FRP.Yampa

main :: IO ()
main = do
  let result = embed (arr (*2) >>> integral)
                     (1.0, replicate 10 (0.1, Nothing))
  print result
```

---

## `reactimateNSteps` — running for N steps

If you want to run a signal function for a fixed number of steps (useful in simulations or tests):

```haskell
import FRP.Yampa

simulateNSteps :: Int -> DTime -> a -> SF a b -> [b]
simulateNSteps n dt initInput sf =
  embed sf (initInput, replicate n (dt, Nothing))
```

---

## Summary

| Function | Purpose |
|---|---|
| `reactimate init sense actuate sf` | Run SF in a continuous I/O loop |
| `sense :: Bool -> IO (DTime, Maybe a)` | Measure elapsed time and optionally new input |
| `actuate :: Bool -> b -> IO Bool` | Consume output, return `True` to stop |
| `embed sf (i0, steps)` | Run SF offline on fixed input list |
| `TQueue` + `drainQueue` | Decouple input threads from the simulation step |
