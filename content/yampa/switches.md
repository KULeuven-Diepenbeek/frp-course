---
title: "Switches"
weight: 4
draft: false
---

## Why switching?

A signal function `SF a b` runs continuously. But many systems change their **behaviour** over time: a game character has an idle state, a walking state, an attacking state; a traffic light cycles through red, yellow, and green. Expressing these as a single monolithic `SF` quickly becomes unmanageable.

**Switches** allow a signal function to be **replaced** by another signal function at the moment an event fires. This is the Yampa mechanism for state machines and mode changes.

---

## `switch` — one-shot switch

```haskell
switch :: SF a (b, Event c)
       -> (c -> SF a b)
       -> SF a b
```

`switch sf cont` runs `sf`. As long as `sf` produces `NoEvent` in its second output component, nothing changes. The moment `sf` produces `Event c`, the switch fires:

-  `sf` is replaced by `cont c` for all future time.
- Importantly, the **switch fires in the same instant** as the event (no delay).

### Example — a bouncing ball hitting the floor

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Ball in free fall; transitions to resting when it hits the floor
fallingBall :: SF () Double
fallingBall = switch falling (\_ -> constant 0.0)
  where
    falling :: SF () (Double, Event ())
    falling = proc _ -> do
      vel <- integral -< (-9.8)           -- velocity (starts at 0)
      pos <- (100 +) ^<< integral -< vel  -- position (starts at 100)
      let hitFloor = if pos <= 0 then Event () else NoEvent
      returnA -< (max 0 pos, hitFloor)
```

When `pos` drops to 0, the `hitFloor` event fires, and the ball transitions permanently to `constant 0.0`.

---

## `dSwitch` — delayed switch

```haskell
dSwitch :: SF a (b, Event c)
        -> (c -> SF a b)
        -> SF a b
```

Identical to `switch` except the switch happens **one infinitesimal step later**. The event instant still produces output from the old SF, and the new SF takes over from the *next* step.

`dSwitch` is used when the event is produced by the same SF that is being switched — this avoids a potential causality loop.

---

## `rSwitch` — restartable switch

```haskell
rSwitch :: SF a b -> SF (a, Event (SF a b)) b
```

Unlike `switch` (which fires once and is done), `rSwitch` allows the active signal function to be replaced **repeatedly** from an external event. Each time a new `SF a b` arrives in the event, the active SF is swapped.

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Switches between two oscillators when an event fires
oscillatorSwitch :: SF ((), Event (SF () Double)) Double
oscillatorSwitch = rSwitch (oscillator 1.0 1.0)

oscillator :: Double -> Double -> SF () Double
oscillator amp freq = proc _ -> do
  t <- time -< ()
  returnA -< amp * sin (2 * pi * freq * t)
```

Usage: send a new `SF` in the event channel to hot-swap the active behaviour.

---

## `kSwitch` — keyed switch

```haskell
kSwitch :: SF a b
        -> SF (a, b) (Event c)
        -> (SF a b -> c -> SF a b)
        -> SF a b
```

`kSwitch sf test cont`:

1. Runs `sf` to produce outputs.
2. Simultaneously runs `test` on the pair `(input, output)` to detect a transition event.
3. When the event fires with value `c`, the whole system restarts using `cont sf c` — the continuation receives the **old SF** (so it can reuse it or wrap it).

`kSwitch` is more flexible than `switch` because transition detection can depend on both input and output, and the continuation has access to the old SF.

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Bounce with elasticity: when pos hits floor, reverse velocity
-- (simplified: using kSwitch to pass the current SF into continuation)
bouncingBall :: Double -> SF () Double
bouncingBall initHeight = kSwitch (freefall initHeight) detectHit bounce
  where
    freefall h = proc _ -> do
      vel <- integral -< (-9.8)
      pos <- (h +) ^<< integral -< vel
      returnA -< pos

    detectHit :: SF ((), Double) (Event Double)
    detectHit = proc (_, pos) -> do
      returnA -< if pos <= 0 then Event (abs pos * 0.8) else NoEvent
      -- carry the "rebound height" in the event

    bounce _oldSF reboundHeight = bouncingBall reboundHeight
```

---

## `drSwitch` and `dkSwitch`

Like `dSwitch`, the `d`-prefixed variants (`drSwitch`, `dkSwitch`) delay the switch by one infinitesimal step to avoid causality issues when the event is produced by the switching SF itself.

---

## Parallel switches — `pSwitch` and `dpSwitch`

For collections of signal functions that can be added, removed, or modified dynamically:

```haskell
pSwitch  :: Functor col
          => (forall sf. (a -> col sf -> col (ext, sf)))
          -> col (SF ext b)
          -> SF (a, col b) (Event c)
          -> (col (SF ext b) -> c -> SF a (col b))
          -> SF a (col b)
```

`pSwitch` runs a **collection** of signal functions in parallel. A supervision SF watches the outputs and can reorganise the collection (fire, add, or remove SFs) when an event occurs.

This is the foundation for particle systems, entity-component systems, and multi-agent simulations:

```haskell
-- Conceptual example: pool of particles managed by pSwitch
type Particle = SF () (Double, Double)   -- SF producing (x, y) position

-- Each particle drifts with its own velocity
particle :: (Double, Double) -> Particle
particle (vx, vy) = proc _ -> do
  x <- integral -< vx
  y <- integral -< vy
  returnA -< (x, y)

-- The collection starting state
initialParticles :: [Particle]
initialParticles = map particle [(1.0, 0.5), (-0.5, 1.0), (0.8, -0.8)]
```

Full `pSwitch` wiring is complex; see the Advanced section.

---

## Implementing a state machine with `switch`

State machines are a natural fit for `switch`. Each state is its own `SF`, and transitions are events:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

data TrafficLight = Red | Yellow | Green deriving (Show, Eq)

trafficLight :: SF () TrafficLight
trafficLight = redPhase

redPhase :: SF () TrafficLight
redPhase = switch
  (proc _ -> do
    done <- after 30.0 () -< ()    -- red for 30 seconds
    returnA -< (Red, done))
  (\_ -> greenPhase)

greenPhase :: SF () TrafficLight
greenPhase = switch
  (proc _ -> do
    done <- after 25.0 () -< ()    -- green for 25 seconds
    returnA -< (Green, done))
  (\_ -> yellowPhase)

yellowPhase :: SF () TrafficLight
yellowPhase = switch
  (proc _ -> do
    done <- after 5.0 () -< ()     -- yellow for 5 seconds
    returnA -< (Yellow, done))
  (\_ -> redPhase)
```

Each phase uses `after` to fire a transition event, then `switch` replaces itself with the next phase.

---

## Summary

| Switch | Description |
|---|---|
| `switch sf cont` | One-shot. When event fires, replace SF with `cont eventVal`. Switch happens immediately. |
| `dSwitch sf cont` | Like `switch`, but one step delayed. Avoids causality issues. |
| `rSwitch sf` | Repeated hot-swap. New SF arrives in the event channel. |
| `drSwitch sf` | Like `rSwitch`, but delayed. |
| `kSwitch sf test cont` | Filtered switch. Transition detection can use output; continuation receives old SF. |
| `dkSwitch sf test cont` | Like `kSwitch`, but delayed. |
| `pSwitch` | Parallel collection of SFs with dynamic membership. |
| `dpSwitch` | Like `pSwitch`, but delayed. |
