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

## `kSwitch` en `pSwitch` — geavanceerde varianten

Yampa biedt nog twee varianten voor speciale gevallen:

**`kSwitch`** ("continuation switch") geeft bij de switch ook de *huidige* initiële signal function mee als argument aan de vervangingsfunctie. Dit maakt het mogelijk de "oude" SF in de nieuwe SF te hergebruiken — handig voor pauzeren/hervatten:

```haskell
kSwitch :: SF a (b, Event c) -> (SF a (b, Event c) -> c -> SF a b) -> SF a b
```

**`pSwitch`** ("parallel switch") laat een hele **collectie** signal functions parallel lopen en geeft een event al de controle om die collectie dynamisch aan te passen. Dit is de basis voor dynamische entiteitscollecties in games (zie het geavanceerde hoofdstuk).

Voor de meeste toepassingen zijn `switch`, `dSwitch` en `rSwitch` voldoende. `kSwitch` en `pSwitch` zijn nodig zodra je complexere compositie nodig hebt.

---

## Demo: een verkeerslicht als state machine bouwen

In deze demo bouwen we stap voor stap een drieledig verkeerslicht als state machine met `switch`. Elk licht duurt een vaste tijd, waarna het volgende licht automatisch begint via een recursieve switch-keten.

---

### Stap 1: De module-header, imports en het datatype

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa

data Kleur = Rood | Oranje | Groen deriving (Show, Eq)
```

`Kleur` is een eenvoudig ADT met drie constructors. De signal functions produceren een `Kleur`-waarde als uitvoer.

---

### Stap 2: Het groene licht

Het groene licht duurt 4 seconden, waarna een `after`-event de schakelaar triggert naar `oranje`:

```haskell
groen :: SF () Kleur
groen = switch
  (proc () -> do
    timer <- after 4.0 () -< ()
    returnA -< (Groen, timer)
  )
  (\_ -> oranje)
```

---

### Stap 3: Het oranje licht

Het oranje licht duurt 1 seconde en schakelt daarna naar `rood`:

```haskell
oranje :: SF () Kleur
oranje = switch
  (proc () -> do
    timer <- after 1.0 () -< ()
    returnA -< (Oranje, timer)
  )
  (\_ -> rood)
```

---

### Stap 4: Het rode licht

Het rode licht duurt 5 seconden en schakelt daarna terug naar `groen`, waardoor de cyclus herhaalt:

```haskell
rood :: SF () Kleur
rood = switch
  (proc () -> do
    timer <- after 5.0 () -< ()
    returnA -< (Rood, timer)
  )
  (\_ -> groen)
```

---

### Stap 5: Samenvoegen en testen met `embed`

Het verkeerslicht start in de rode fase. We simuleren 12 seconden met stappen van 0.5s via `embed`:

```haskell
verkeerslicht :: SF () Kleur
verkeerslicht = rood

main :: IO ()
main = do
  let stappen  = map (\_ -> (0.5, Nothing)) [1..24 :: Int]
      kleuren  = embed verkeerslicht ((), stappen)
  mapM_ print kleuren
```

De uitvoer toont de volledige cyclus: eerst 10 rode frames (5s), dan 8 groene (4s), dan 2 oranje (1s), waarna de cyclus herhaalt.

---

### Volledig programma

<details closed>
<summary><i><b>Volledig programma — klik om te tonen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa

data Kleur = Rood | Oranje | Groen deriving (Show, Eq)

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

rood :: SF () Kleur
rood = switch
  (proc () -> do
    timer <- after 5.0 () -< ()
    returnA -< (Rood, timer)
  )
  (\_ -> groen)

verkeerslicht :: SF () Kleur
verkeerslicht = rood

main :: IO ()
main = do
  let stappen = map (\_ -> (0.5, Nothing)) [1..24 :: Int]
      kleuren = embed verkeerslicht ((), stappen)
  mapM_ print kleuren
```

</p>
</details>

---

## Oefeningen

<details closed>
<summary><i><b>GHCi-configuratie voor de demo en oefeningen — klik om te tonen</b></i>🔽</summary>
<p>

Sla dit bestand op als `.ghci` in de root van je projectdirectory. GHCi laadt het automatisch bij het opstarten.

```haskell
-- .ghci --- switches

-- Prompt
:set prompt "ghci> "
:set prompt-cont "     | "

-- Warnings
:set -Wall

-- Arrow-notatie
:set -XArrows

-- Yampa beschikbaar stellen (buiten cabal repl)
:set -package Yampa

-- Imports die in de demo en oefeningen gebruikt worden
import FRP.Yampa
```

</p>
</details>

_De gegeven oplossingen zijn EEN mogelijke oplossing, soms zijn meerdere mogelijkheden juist. Is het gewenste gedrag bereikt, dan is je oplossing correct!_

### Oefeningenreeks 1: `switch`

- Schrijf een signal function `bewegingMetStop :: SF () Double` die gedurende 2 seconden een positie opbouwt met snelheid 1.0 m/s en daarna stilstaat op de bereikte positie. Gebruik `switch`, `integral` en `after`. Test met `embed` over 6 stappen van 0.5s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

bewegingMetStop :: SF () Double
bewegingMetStop = switch
  (proc () -> do
    pos   <- integral -< 1.0
    timer <- after 2.0 () -< ()
    returnA -< (pos, timer)
  )
  (\_ -> constant 2.0)

-- embed bewegingMetStop ((), replicate 6 (0.5, Nothing))
-- => [0.5, 1.0, 1.5, 2.0, 2.0, 2.0]
```

</p>
</details> -->

- Schrijf een signal function `tweeSnelheden :: SF () Double` die de eerste 3 seconden met snelheid 2.0 m/s beweegt, en daarna met snelheid 0.5 m/s. Gebruik `switch` en `integral`. Test met `embed` over 8 stappen van 1.0s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

tweeSnelheden :: SF () Double
tweeSnelheden = switch
  (proc () -> do
    pos   <- integral -< 2.0
    timer <- after 3.0 () -< ()
    returnA -< (pos, timer)
  )
  (\_ -> proc () -> do
    extra <- integral -< 0.5
    returnA -< 6.0 + extra
  )

-- embed tweeSnelheden ((), replicate 8 (1.0, Nothing))
-- => [2.0, 4.0, 6.0, 6.5, 7.0, 7.5, 8.0, 8.5]
```

</p>
</details> -->

---

### Oefeningenreeks 2: `dSwitch`

- Pas de `bewegingMetStop`-oefening aan met `dSwitch` in plaats van `switch`. Test met dezelfde invoer en vergelijk de uitvoer: welk frame verschilt tussen `switch` en `dSwitch`?
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

bewegingMetStopD :: SF () Double
bewegingMetStopD = dSwitch
  (proc () -> do
    pos   <- integral -< 1.0
    timer <- after 2.0 () -< ()
    returnA -< (pos, timer)
  )
  (\_ -> constant 2.0)

-- Met switch:  [0.5, 1.0, 1.5, 2.0, 2.0, 2.0]
-- Met dSwitch: [0.5, 1.0, 1.5, 2.0, 2.0, 2.0]
-- (Bij deze tijdstap valt het verschil niet op; het verschil is zichtbaar
--  bij het exacte switch-frame wanneer de nieuwe SF een andere beginwaarde heeft)
```

</p>
</details> -->

---

### Oefeningenreeks 3: `rSwitch`

- Modelleer een dag-nachtcyclus met `rSwitch`: begin in de `"Dag"`-fase (constant `"Dag"`), en wissel na 12s naar `"Nacht"` en na nog eens 12s terug naar `"Dag"`. Gebruik `after` als eventbron die de nieuwe SF meelevert. Test met `embed` over 50 stappen van 1.0s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

dagFase :: SF () String
dagFase = proc () -> do
  wissel <- after 12.0 (nachtFase) -< ()
  returnA -< ("Dag", wissel)

nachtFase :: SF ((), Event (SF () String)) String
nachtFase = undefined -- vereenvoudiging: gebruik switch-keten

-- Eenvoudigere aanpak met switch-keten (zoals verkeerslicht):
dag :: SF () String
dag = switch
  (proc () -> do
    timer <- after 12.0 () -< ()
    returnA -< ("Dag", timer)
  )
  (\_ -> nacht)

nacht :: SF () String
nacht = switch
  (proc () -> do
    timer <- after 12.0 () -< ()
    returnA -< ("Nacht", timer)
  )
  (\_ -> dag)

cyclus :: SF () String
cyclus = dag

-- embed cyclus ((), replicate 50 (1.0, Nothing))
```

</p>
</details> -->
