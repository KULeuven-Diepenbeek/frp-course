---
title: "Switches"
weight: 4
draft: false
---

## State machines in Yampa

In reactieve systemen wil je vaak dat een systeem van **gedrag verandert** op het moment dat een bepaald event optreedt: een vijand gaat van "patrouillerend" naar "achtervolgend" zodra hij de speler ziet; een verkeerslicht schakelt van rood naar groen na een bepaald aantal seconden; een menu-item wordt geselecteerd bij een muisklik. Dit soort discrete overgangen heten **switches** in Yampa.

Een switch bestaat altijd uit twee delen: een **initiële signal function** die loopt totdat een event optreedt, en een **vervangingsfunctie** die op basis van het event een nieuwe signal function terugggeeft. Vanaf het moment van de switch wordt de nieuwe signal function actief en loopt de oude niet meer.

---

## `switch` — eenvoudige switch

De meest fundamentele switch-operator is:

```haskell
switch :: SF a (b, Event c) -> (c -> SF a b) -> SF a b
```

De initiële signal function `SF a (b, Event c)` produceert tegelijkertijd een uitvoerwaarde `b` en een optioneel event `Event c`. Zodra het event `Event c` optreedt, wordt de functie `c -> SF a b` aangeroepen met de eventwaarde, en de teruggegeven signal function neemt het roer over. De nieuwe signal function is permanent: `switch` schakelt precies één keer.

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Beweeg naar rechts gedurende 3 seconden, dan stilstaan
beweging :: SF () Double
beweging = switch
  (proc () -> do
    pos    <- integral -< 1.0          -- snelheid 1 m/s
    timer  <- after 3.0 ()  -< ()
    returnA -< (pos, timer)
  )
  (\_ -> constant 3.0)                 -- na 3s: vaste positie 3.0
```

Een belangrijk detail: `switch` evalueert zijn initiële signal function ook op time point 0. Als het event *al* op time point 0 optreedt (zoals `now`), schakelt hij onmiddellijk.

---

## `dSwitch` — vertraagde switch

`dSwitch` ("delayed switch") werkt als `switch`, maar met één verschil: de **uitvoer van de initiële signal function wordt nog één keer gebruikt** op het time point van de switch, vóórdat de nieuwe signal function overneemt. Dit is nuttig als je het exacte frame van de overgang ook nog wil renderen:

```haskell
dSwitch :: SF a (b, Event c) -> (c -> SF a b) -> SF a b
```

Het verschil is subtiel maar relevant bij animaties: `switch` geeft al de uitvoer van de nieuwe SF op het switchframe, terwijl `dSwitch` nog de uitvoer van de oude SF laat zien.

---

## `rSwitch` — herhaaldelijk schakelende switch

`switch` schakelt slechts één keer. Voor systemen die meerdere keren van gedrag moeten wisselen — zoals een verkeerslicht dat door een cyclus loopt — gebruik je `rSwitch` ("recurring switch"):

```haskell
rSwitch :: SF a b -> SF (a, Event (SF a b)) b
```

`rSwitch` neemt een begingedrag en een eventstroom waarbij elk event een geheel nieuwe signal function meelevert. Elke keer dat er een event arriveert met een nieuwe SF, schakelt `rSwitch` over naar die SF en vervangt de vorige volledig. Dit geeft de volledige flexibiliteit van een state machine waarbij de overgangen zelf door de invoer bepaald worden.

---

## Voorbeeld — verkeerslicht als state machine

Het volgende voorbeeld modelleert een drieledig verkeerslicht als state machine met `switch`. Elk licht duurt een vaste tijd, waarna het volgende licht automatisch begint:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

data Kleur = Rood | Oranje | Groen deriving (Show, Eq)

rood :: SF () Kleur
rood = switch
  (proc () -> do
    timer <- after 5.0 () -< ()
    returnA -< (Rood, timer)
  )
  (\_ -> groen)

groen :: SF () Kleur
groen = switch
  (proc () -> do
    timer <- after 4.0 () -< ()
    returnA -< (Groen, timer)
  )
  (\_ -> oranje)

oranje :: SF () Kleur
oranje = switch
  (proc () -> do
    timer <- after 1.0 () -< ()
    returnA -< (Oranje, timer)
  )
  (\_ -> rood)

verkeerslicht :: SF () Kleur
verkeerslicht = rood
```

`embed` toont de cyclus over 12 seconden:

```haskell
test :: [Kleur]
test = embed verkeerslicht
  ((), map (\_ -> (0.5, Nothing)) [1..24])
-- [Rood,Rood,...,Groen,Groen,...,Oranje,Rood,...]
```

---

## `kSwitch` en `pSwitch` — geavanceerde varianten

Yampa biedt nog twee varianten voor speciale gevallen:

**`kSwitch`** ("continuation switch") geeft bij de switch ook de *huidige* initiële signal function mee als argument aan de vervangingsfunctie. Dit maakt het mogelijk de "oude" SF in de nieuwe SF te hergebruiken — handig voor pauzeren/hervatten:

```haskell
kSwitch :: SF a (b, Event c) -> (SF a (b, Event c) -> c -> SF a b) -> SF a b
```

**`pSwitch`** ("parallel switch") laat een hele **collectie** signal functions parallel lopen en geeft een event al de controle om die collectie dynamisch aan te passen. Dit is de basis voor dynamische entiteitscollecties in games (zie het geavanceerde hoofdstuk).

Voor de meeste toepassingen zijn `switch`, `dSwitch` en `rSwitch` voldoende. `kSwitch` en `pSwitch` zijn nodig zodra je complexere compositie nodig hebt.
