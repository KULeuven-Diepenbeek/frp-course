---
title: "Events in Yampa"
weight: 2
draft: false
---

## The `Event` type

In Yampa, an event is represented by the type:

```haskell
data Event a = NoEvent | Event a
```

At any given moment in time, an event either:
- **has not occurred** — `NoEvent`
- **has occurred** — `Event v`, carrying a value `v`

This is essentially `Maybe`, but with the semantic meaning "did something happen at this instant?". Unlike a behaviour (which always has a value), an event is only defined *at* the moment it fires.

---

## Producing events

### `never`

```haskell
never :: SF a (Event b)
```

Never produces an event. Useful as a placeholder or default.

```haskell
noOpSF :: SF () (Event String)
noOpSF = never
```

### `now`

```haskell
now :: b -> SF a (Event b)
```

Fires exactly once at time 0 (at the very start of the simulation), carrying the given value.

```haskell
initEvent :: SF () (Event String)
initEvent = now "simulation started"
```

### `after`

```haskell
after :: Time -> b -> SF a (Event b)
```

Fires once after the given amount of time has elapsed.

```haskell
-- Fires after 3 seconds
timer3s :: SF () (Event String)
timer3s = after 3.0 "time's up!"
```

### `repeatedly`

```haskell
repeatedly :: Time -> b -> SF a (Event b)
```

Fires at regular intervals, carrying the same value each time.

```haskell
-- Fires every 0.5 seconds
ticker :: SF () (Event ())
ticker = repeatedly 0.5 ()
```

### `edge`

```haskell
edge :: SF Bool (Event ())
```

Detects a **rising edge**: fires an event when the input transitions from `False` to `True`.

```haskell
-- Detects the moment a key is pressed (input: is key held down?)
keyPressEvent :: SF Bool (Event ())
keyPressEvent = edge
```

There is also `iEdge` (inclusive edge — fires on both rising and falling), and `edgeTag`:

```haskell
edgeTag :: a -> SF Bool (Event a)
edgeTag v = edge >>> arr (fmap (const v))
```

### `edgeJust`

```haskell
edgeJust :: SF (Maybe a) (Event a)
```

Fires an event when the input changes from `Nothing` to `Just v`, carrying `v`.

---

## Transforming events

Events form a `Functor`, so you can apply a function to the carried value:

```haskell
fmap :: (a -> b) -> Event a -> Event b
-- via SF:
arr (fmap f) :: SF (Event a) (Event b)
```

Or more naturally with `arr`:

```haskell
labelEvent :: SF (Event ()) (Event String)
labelEvent = arr (fmap (const "clicked"))
```

### `tag`

```haskell
tag :: Event a -> b -> Event b
tag NoEvent  _ = NoEvent
tag (Event _) v = Event v
```

Replace the event's value with a constant.

```haskell
import FRP.Yampa

taggedTimer :: SF () (Event String)
taggedTimer = after 1.0 () >>> arr (`tag` "one second passed")
```

### `mergeEvents`

```haskell
mergeEvents :: [Event a] -> Event a
```

Returns the first `Event` from a list (left-biased), or `NoEvent` if all are `NoEvent`. Useful for combining multiple event sources.

```haskell
anyKey :: SF (Bool, Bool) (Event ())
anyKey = proc (keyA, keyB) -> do
  evA <- edge -< keyA
  evB <- edge -< keyB
  returnA -< mergeEvents [evA, evB]
```

### `catEvents`

```haskell
catEvents :: [Event a] -> Event [a]
```

Collects all occurring events into a list. Fires only when at least one event occurs.

---

## Filtering events

```haskell
gate :: Event a -> Bool -> Event a
gate NoEvent   _     = NoEvent
gate (Event v) True  = Event v
gate (Event v) False = NoEvent
```

Example — only pass through events when a "game active" flag is set:

```haskell
{-# LANGUAGE Arrows #-}

filteredEvent :: SF (Event (), Bool) (Event ())
filteredEvent = proc (evt, active) -> do
  returnA -< evt `gate` active
```

---

## Converting events to and from behaviours

### `hold`

```haskell
hold :: a -> SF (Event a) a
```

Holds the most recently received event value. The initial value is used until the first event fires.

```haskell
-- Tracks the last key pressed, starting with ' '
lastKey :: SF (Event Char) Char
lastKey = hold ' '
```

### `dHold`

Like `hold`, but with a one-sample delay (the update propagates in the next step).

### `accum`

```haskell
accum :: a -> SF (Event (a -> a)) a
```

Accumulates values using a function carried by the event.

```haskell
-- Counter that increments when an event fires
counter :: SF (Event ()) Int
counter = accumHold 0 (arr (fmap (const (+1))))
```

Or equivalently:

```haskell
counter :: SF (Event ()) Int
counter = arr (fmap (const (+1))) >>> accumHold 0
```

### `accumHold`

```haskell
accumHold :: a -> SF (Event (a -> a)) a
```

Like `accum` but immediately returns the current accumulated value (no initial delay).

---

## Practical example — a simple click counter

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Input: Bool (True = mouse button is currently down)
-- Output: Int (number of clicks so far)
clickCounter :: SF Bool Int
clickCounter = proc mouseDown -> do
  clickEvent <- edge        -< mouseDown
  count      <- accumHold 0 -< fmap (const (+1)) clickEvent
  returnA    -< count
```

Testing with `embed`:

```haskell
main :: IO ()
main = do
  let sf     = clickCounter
      -- mouse: up, up, down, down, up, down, up
      input  = (False, [ (0.1, Just False)
                       , (0.1, Just True)    -- click 1
                       , (0.1, Just True)
                       , (0.1, Just False)
                       , (0.1, Just True)    -- click 2
                       , (0.1, Just False)
                       ])
      output = embed sf input
  print output
  -- [0, 0, 1, 1, 1, 2]
```

---

## Summary

| Function | Type | Purpose |
|---|---|---|
| `never` | `SF a (Event b)` | Never fires |
| `now v` | `SF a (Event a)` | Fires once at t=0 |
| `after t v` | `SF a (Event a)` | Fires once after `t` seconds |
| `repeatedly t v` | `SF a (Event a)` | Fires every `t` seconds |
| `edge` | `SF Bool (Event ())` | Rising edge detection |
| `edgeTag v` | `SF Bool (Event a)` | Rising edge with value |
| `hold v` | `SF (Event a) a` | Latch: remember last event value |
| `accumHold v` | `SF (Event (a->a)) a` | Fold over events |
| `mergeEvents` | `[Event a] -> Event a` | Merge multiple event sources |
| `gate` | `Event a -> Bool -> Event a` | Conditional event filter |
