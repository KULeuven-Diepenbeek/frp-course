---
title: "What is Functional Reactive Programming?"
weight: 1
draft: false
---

## Motivation

Many programs are essentially about **change over time**: a game character moves between frames, a sensor reading updates every millisecond, a user interface responds to mouse clicks. Imperative code handles this with mutable state, callbacks, and event loops — an approach that tends to produce tangled, hard-to-test code.

**Functional Reactive Programming** (FRP) offers an alternative: model time-varying values and events as first-class values in the language, and express program logic as compositions of pure functions over those values.

---

## Two core abstractions

All FRP systems are built on two dual concepts. Different FRP libraries use different names, but the ideas are the same:

### 1. Behaviours (Signals)

A **behaviour** (also called a **signal**) is a value that varies continuously over time. It can be thought of as a function from time to a value:

$$\text{Behaviour}(a) \cong \text{Time} \to a$$

Examples:
- The $x$-position of a moving object: `Behaviour Double`
- The current time: `Behaviour Time`
- Whether a key is held down: `Behaviour Bool`

### 2. Events

An **event** is a sequence of discrete occurrences, each carrying a value. Events happen at specific moments in time and are **not** defined between occurrences:

$$\text{Event}(a) \cong [(\ \text{Time},\ a\ )]$$

Examples:
- A mouse click carrying the position: `Event (Int, Int)`
- A key press carrying the character: `Event Char`
- A timer firing every second: `Event ()`

---

## How FRP differs from callbacks

Consider detecting when a falling ball hits the floor.

**Imperative approach (callbacks):**

```python
ball_y = 500.0

def on_update(dt):
    global ball_y
    ball_y -= 9.8 * dt
    if ball_y <= 0:
        on_hit_floor()

def on_hit_floor():
    ball_y = 0
    start_bounce_animation()
```

The logic is scattered. The `on_hit_floor` callback is registered somewhere else. Mutable state is shared globally. Testing requires simulating the whole update loop.

**FRP approach (conceptual):**

```
ballY     : Behaviour Double
           = integral gravity 500

hitFloor  : Event ()
           = when (ballY <= 0)

ballY'    : Behaviour Double
           = switch ballY hitFloor (const bounceAnimation)
```

The program is a **declarative description** of relationships between time-varying values. There is no mutation — each definition is a pure equation. The FRP runtime handles executing the updates.

---

## A brief history

| Year | System | Significance |
|---|---|---|
| 1997 | **FRAN** (Functional Reactive ANimation) | First FRP system, by Conal Elliott & Paul Hudak |
| 2002 | **FrTime** | FRP in Scheme |
| 2003 | **Yampa** | Arrow-based FRP in Haskell, still widely used |
| 2010s | **Reactive Banana**, **Reflex**, **reactive-banana** | Push-based FRP for GUI/web |
| 2013+ | **Elm** | FRP-inspired language for front-end web |
| 2010s | **RxJS / RxJava** | Reactive extensions, inspired by FRP concepts, mainstream |

This course focuses on **Yampa**, which is well-suited for game development and simulations because it models continuous time explicitly.

---

## Push vs. pull

FRP implementations differ in how they propagate changes:

- **Pull-based**: values are computed on demand. The system "pulls" the current value of a behaviour whenever it needs it. Simple to implement but potentially wasteful (recomputing everything every frame).
- **Push-based**: changes propagate automatically from sources to dependents. More efficient but harder to implement correctly (requires dependency tracking).

Yampa uses a **pull-based** model: each simulation step, the system pulls the current output from the signal network given the current input and elapsed time.

---

## Why Yampa in this course?

Yampa is:

- **Mature**: actively maintained, used in real game projects (including the [Keera Hails](https://github.com/keera-studios/keera-hails) framework).
- **Arrow-based**: its API aligns with Haskell's `Arrow` type class, which provides principled composition of stateful computations.
- **Explicit about time**: the `DTime` (delta time) parameter is first-class, making physics and animation straightforward.
- **Teachable**: the core concepts (signal functions, `reactimate`, switches) can be learned incrementally.

---

## Summary

| Concept | Description |
|---|---|
| FRP | Programming paradigm for time-varying and event-driven systems |
| Behaviour / Signal | Value that changes continuously over time |
| Event | Discrete occurrence at a specific moment, carrying a value |
| Pull-based FRP | Values computed on request (Yampa's model) |
| Push-based FRP | Changes propagate automatically (Reflex, reactive-banana) |
