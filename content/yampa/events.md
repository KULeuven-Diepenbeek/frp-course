---
title: "Events"
weight: 2
draft: false
---

## Het type `Event a`

In Yampa zijn **events** discrete gebeurtenissen die op specifieke time points optreden. Ze worden gemodelleerd met het type `Event a`, dat vergelijkbaar is met `Maybe a`: ofwel is er een event met een waarde (`Event v`), ofwel is er geen event (`NoEvent`). Het verschil met `Maybe` zit in de semantische betekenis: `Event` drukt een tijdelijk discrete voorval uit, terwijl `Maybe` een optionele waarde uitdrukt.

```haskell
data Event a = NoEvent | Event a
```

Een signal function die events produceert heeft als uitvoertype `Event a`. Een signal function die op events reageert heeft als invoertype `Event a`. In de meeste Yampa-programma's stromen events door het signaalnetwerk naast gewone continue waarden.

---

## Events genereren

### `never` en `now`

De eenvoudigste eventgenerator is `never :: SF a (Event b)`, die nooit een event produceert. Dit is handig als placeholder wanneer je een event verwacht maar er nog geen gedrag aan wil koppelen. Tegenovergesteld produceert `now :: b -> SF a (Event b)` precies één event — met de opgegeven waarde — op het allereerste time point (t = 0), en daarna `NoEvent`:

```haskell
import FRP.Yampa

startEvent :: SF () (Event String)
startEvent = now "Gestart!"

nooitEvent :: SF () (Event Int)
nooitEvent = never
```

### `after`

`after :: Time -> b -> SF a (Event b)` wacht een opgegeven aantal seconden en produceert dan precies één event. Dit is de meest gebruikte manier om een vertraagd event te maken — bijvoorbeeld een timer die na vijf seconden afgaat:

```haskell
timer :: SF () (Event ())
timer = after 5.0 ()    -- één event na 5 seconden
```

### `repeatedly`

`repeatedly :: Time -> b -> SF a (Event b)` produceert op regelmatige tijdsintervallen een event, voor onbepaalde duur. Dit is handig voor periodieke acties, zoals het spawnen van vijanden elke twee seconden:

```haskell
tick :: SF () (Event ())
tick = repeatedly 1.0 ()    -- elke seconde een event
```

### `edge`

`edge :: SF Bool (Event ())` detecteert een stijgende flank in een boolesignaal: het produceert een event op het moment dat de invoer verandert van `False` naar `True`. Dit is bijzonder nuttig om toetsdrukken te detecteren: je geeft een continu `Bool`-signaal (is de toets ingedrukt?) en krijgt een event terug op het exacte moment van indrukken:

```haskell
toets :: SF Bool (Event ())
toets = edge
-- invoer: False False True True True False True
-- uitvoer: NoEvent NoEvent Event() NoEvent NoEvent NoEvent Event()
```

---

## Events verwerken

### `hold`

`hold :: a -> SF (Event a) a` zet een eventstroom om in een continu signaal door de laatste eventwaarde te "onthouden". De eerste parameter is de beginwaarde die geldt zolang er nog geen event is opgetreden. Na elk event wordt de waarde bijgewerkt:

```haskell
huidigeLetter :: SF (Event Char) Char
huidigeLetter = hold 'A'
-- invoer: NoEvent, Event 'B', NoEvent, NoEvent, Event 'Z'
-- uitvoer: 'A', 'B', 'B', 'B', 'Z'
```

### `accumHold`

`accumHold :: a -> (a -> b -> a) -> SF (Event b) a` combineert `hold` met accumulatie. Bij elk event wordt de huidige state bijgewerkt via de opgegeven functie. Dit is de standaardmanier om een teller of score bij te houden:

```haskell
teller :: SF (Event ()) Int
teller = accumHold 0 (\n () -> n + 1)
-- elke keer dat er een event arriveert, wordt n met 1 verhoogd
```

---

## Events combineren

### `mergeEvents`

Wanneer je meerdere eventstroom wilt samenvoegen tot één, gebruik je `mergeEvents :: [Event a] -> Event a`. Als meerdere events tegelijk optreden, geeft `mergeEvents` het eerste terug (van links naar rechts). Dit is eigenlijk een vereenvoudigde versie van `lMerge`:

```haskell
gecombineerd :: SF () (Event String)
gecombineerd = proc () -> do
  eA <- after 1.0 "A" -< ()
  eB <- after 2.0 "B" -< ()
  returnA -< mergeEvents [eA, eB]
```

### `gate`

`gate :: SF (Event a, Bool) (Event a)` filtert een eventstroom: het event passeert alleen als het `Bool`-channel `True` is. Dit is nuttig om events conditioneel toe te laten afhankelijk van de player state:

```haskell
-- laat events alleen door als het spel actief is
gated :: SF (Event () , Bool) (Event ())
gated = gate
```

---

## Demo: een stuiterendebal-simulator bouwen

In deze demo bouwen we stap voor stap een simulatie van een stuiterende bal. De bal valt door zwaartekracht, een `edge`-detector registreert het moment dat de bal de grond raakt en `accumHold` telt het aantal stuiteringen.

---

### Stap 1: De module-header en imports

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa
```

---

### Stap 2: Hoogte berekenen met `integral`

De bal valt door een constante versnelling van -9.81 m/s². Door twee maal te integreren krijgen we de hoogte. We knippen de hoogte af op nul zodat de bal niet door de vloer zakt:

```haskell
hoogteSF :: SF () Double
hoogteSF = proc () -> do
  snelheid <- integral -< -9.81
  hoogte   <- integral -< snelheid
  returnA -< max 0.0 hoogte
```

---

### Stap 3: Stuitering detecteren met `edge`

`edge` detecteert een stijgende flank: het produceert een event op het moment dat de invoer van `False` naar `True` gaat. We gebruiken dit om te detecteren wanneer de bal de grond raakt (hoogte ≤ 0):

```haskell
stuiteringsSF :: SF Double (Event ())
stuiteringsSF = arr (<= 0.0) >>> edge
```

`arr (<= 0.0)` zet de hoogte om naar een `Bool`-signaal. `edge` produceert een event op het exacte moment dat dat signaal `True` wordt.

---

### Stap 4: Stuiteringen tellen met `accumHold`

`accumHold` houdt de huidige teller bij en verhoogt die bij elk stuiterings-event:

```haskell
tellerSF :: SF (Event ()) Int
tellerSF = accumHold 0 (\n () -> n + 1)
```

---

### Stap 5: Alles samenvoegen met `proc` en testen met `embed`

We verbinden alle stukken in één `proc`-blok en testen de simulatie met `embed`:

```haskell
stuiterendebal :: SF () (Double, Int)
stuiterendebal = proc () -> do
  hoogte     <- hoogteSF    -< ()
  stuitering <- stuiteringsSF -< hoogte
  aantal     <- tellerSF    -< stuitering
  returnA -< (hoogte, aantal)

main :: IO ()
main = do
  let stappen  = replicate 30 (0.1, Nothing)
      uitvoer  = embed stuiterendebal ((), stappen)
  mapM_ print uitvoer
```

---

### Volledig programma

<details closed>
<summary><i><b>Volledig programma — klik om te tonen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa

hoogteSF :: SF () Double
hoogteSF = proc () -> do
  snelheid <- integral -< -9.81
  hoogte   <- integral -< snelheid
  returnA -< max 0.0 hoogte

stuiteringsSF :: SF Double (Event ())
stuiteringsSF = arr (<= 0.0) >>> edge

tellerSF :: SF (Event ()) Int
tellerSF = accumHold 0 (\n () -> n + 1)

stuiterendebal :: SF () (Double, Int)
stuiterendebal = proc () -> do
  hoogte     <- hoogteSF      -< ()
  stuitering <- stuiteringsSF -< hoogte
  aantal     <- tellerSF      -< stuitering
  returnA -< (hoogte, aantal)

main :: IO ()
main = do
  let stappen = replicate 30 (0.1, Nothing)
      uitvoer = embed stuiterendebal ((), stappen)
  mapM_ print uitvoer
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
-- .ghci --- events

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

### Oefeningenreeks 1: Events genereren

- Gebruik `after 2.0 "Klaar"` om een signal function te maken die na 2 seconden één event met de string `"Klaar"` produceert. Test met `embed` over 5 stappen van 0.5s en verifieer dat precies op stap 4 een event verschijnt.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
klaarTimer :: SF () (Event String)
klaarTimer = after 2.0 "Klaar"

-- embed klaarTimer ((), replicate 5 (0.5, Nothing))
-- => [NoEvent, NoEvent, NoEvent, Event "Klaar", NoEvent]
``` -->

- Gebruik `repeatedly 1.0 ()` om een puls te maken die elke seconde een event geeft. Combineer dit met `accumHold 0 (\n () -> n + 1)` om een teller bij te houden. Test over 10 stappen van 0.5s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

pulsTeller :: SF () Int
pulsTeller = proc () -> do
  puls   <- repeatedly 1.0 () -< ()
  teller <- accumHold 0 (\n () -> n + 1) -< puls
  returnA -< teller

-- embed pulsTeller ((), replicate 10 (0.5, Nothing))
-- => [0, 0, 1, 1, 2, 2, 3, 3, 4, 4]
```

</p>
</details> -->

- Schrijf een signal function `dalendeSFEdge :: SF () (Event ())` die een event produceert op het moment dat een lineair afnemende waarde (van 5.0 met snelheid -1.0 m/s) de nul passeert. Gebruik `constant (-1.0) >>> integral` voor de positie, `arr (<= 0.0)` en `edge`. Test met `embed`.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

dalendeSFEdge :: SF () (Event ())
dalendeSFEdge = proc () -> do
  pos <- (arr (+ 5.0) <<< integral) -< -1.0
  e   <- edge -< pos <= 0.0
  returnA -< e

-- embed dalendeSFEdge ((), replicate 12 (0.5, Nothing))
```

</p>
</details> -->

---

### Oefeningenreeks 2: Events verwerken

- Gebruik `hold 'A'` om een signal function te maken die de laatste ontvangen `Char`-event vasthoudt. Test met `embed houdChar (NoEvent, [(0.1, Just (Event 'B')), (0.1, Nothing), (0.1, Just (Event 'Z'))])`.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
houdChar :: SF (Event Char) Char
houdChar = hold 'A'

-- embed houdChar (NoEvent, [(0.1, Just (Event 'B')), (0.1, Nothing), (0.1, Just (Event 'Z'))])
-- => ['A', 'B', 'B', 'Z']
``` -->

- Gebruik `accumHold` om een score bij te houden die met 10 verhoogd wordt bij elk event. Combineer met `repeatedly 1.0 ()` als puls en test over 6 stappen van 0.5s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

scoreSF :: SF () Int
scoreSF = proc () -> do
  puls  <- repeatedly 1.0 () -< ()
  score <- accumHold 0 (\s () -> s + 10) -< puls
  returnA -< score

-- embed scoreSF ((), replicate 6 (0.5, Nothing))
-- => [0, 0, 10, 10, 20, 20]
```

</p>
</details> -->

---

### Oefeningenreeks 3: Events combineren

- Schrijf een signal function die twee events samenvoegt: één na 1.0s met waarde `"A"` en één na 2.0s met waarde `"B"`. Gebruik `mergeEvents`. Test met `embed` over 5 stappen van 0.5s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

samengevoegd :: SF () (Event String)
samengevoegd = proc () -> do
  eA <- after 1.0 "A" -< ()
  eB <- after 2.0 "B" -< ()
  returnA -< mergeEvents [eA, eB]

-- embed samengevoegd ((), replicate 5 (0.5, Nothing))
-- => [NoEvent, Event "A", NoEvent, Event "B", NoEvent]
```

</p>
</details> -->

- Schrijf een signal function `gefilterdeTick :: SF Bool (Event ())` die een puls van `repeatedly 0.5 ()` alleen doorlaat als de `Bool`-invoer `True` is. Gebruik `gate`. Test met `embed gefilterdeTick (True, [(0.5, Nothing), (0.5, Just False), (0.5, Nothing), (0.5, Just True)])`.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

gefilterdeTick :: SF Bool (Event ())
gefilterdeTick = proc actief -> do
  puls <- repeatedly 0.5 () -< ()
  returnA -< gate (puls, actief)
```

</p>
</details> -->
