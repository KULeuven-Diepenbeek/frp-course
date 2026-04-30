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

## Volledig voorbeeld — springende bal

Het volgende voorbeeld combineert events en continues signalen. Een bal valt door zwaartekracht. Wanneer de hoogte nul bereikt, produceert een event een stuitering. `accumHold` telt het aantal stuiteringen:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

stuiterendebal :: SF () (Double, Int)
stuiterendebal = proc () -> do
  hoogte     <- integral <<< integral -< (-9.81)
  stuitering <- edge -< hoogte <= 0.0
  aantStu    <- accumHold 0 (\n () -> n + 1) -< stuitering
  returnA -< (max 0.0 hoogte, aantStu)
```

Dit voorbeeld is bewust vereenvoudigd (de bal stopt niet echt bij de grond zonder switch), maar toont hoe events en continues signalen naadloos samenwerken in een `proc`-blok.
