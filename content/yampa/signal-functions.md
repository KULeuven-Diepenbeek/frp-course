---
title: "Signal Functions"
weight: 1
draft: false
---

## Het type `SF a b`

De fundamentele building block in Yampa is de **signal function**: een waarde van het type `SF a b`. Conceptueel is een signal function een transformatie die een continue signaalstroom van type `a` omzet naar een continue signaalstroom van type `b`. Je kunt het zien als een 'wire' in een signaalgrafiek: aan het ene einde stroomt invoer naar binnen, aan het andere einde stroomt uitvoer naar buiten. De transformatie kan time-dependent zijn — een signal function heeft intern toegang tot de elapsed time en kan daarmee integreren, differentieren, vertraging realiseren, enzovoort.

Het time-dependent karakter maakt signal functions fundamenteel anders dan gewone functies `a -> b`. Een gewone functie geeft altijd dezelfde uitvoer voor dezelfde invoer. Een signal function kan uitvoer produceren die afhangt van de **geschiedenis** van de invoer (via integratie) of van de **elapsed time** (via time sources). Tegelijkertijd zijn signal functions strikt **causaal**: een signal function mag niet vooruitkijken in de time — de uitvoer op time point *t* mag alleen afhangen van de invoer op time points vóór of op *t*.

---

## Eenvoudige signal functions

De eenvoudigste signal function is `arr`, die een gewone functie opheft naar signal functions. `arr f` past `f` op elk time point toe op de invoer, zonder enige time storage:

```haskell
import FRP.Yampa

verdubbel :: SF Double Double
verdubbel = arr (* 2)

optellen :: SF (Double, Double) Double
optellen = arr (uncurry (+))
```

De identiteitssignaal function `identity` geeft de invoer onveranderd door. `constant v` negeert de invoer volledig en produceert altijd de waarde `v`:

```haskell
doorgeven :: SF a a
doorgeven = identity

altijdTien :: SF a Double
altijdTien = constant 10.0
```

---

## Seriële compositie met `>>>`

Signal functions combineer je serieel met de `>>>` operator (uit de `Control.Arrow`-klasse). `sf1 >>> sf2` stuurt de uitvoer van `sf1` rechtstreeks naar de invoer van `sf2`. Dit is de meest gebruikte compositionele operator in Yampa en vormt de ruggengraat van elke signaalketen. In termen van typen: als `sf1 :: SF a b` en `sf2 :: SF b c`, dan is `sf1 >>> sf2 :: SF a c`:

```haskell
schaal :: SF Double Double
schaal = arr (* 3) >>> arr (+ 1)   -- vermenigvuldig met 3, tel 1 op
```

---

## Parallelle compositie met `***` en `&&&`

Naast seriële compositie ondersteunt Yampa ook parallelle compositie. Met `***` laat je twee signal functions onafhankelijk naast elkaar lopen: `sf1 *** sf2` neemt een paar `(a, c)` als invoer en produceert een paar `(b, d)` als uitvoer, waarbij `sf1 :: SF a b` de eerste component verwerkt en `sf2 :: SF c d` de tweede:

```haskell
beheerder :: SF (Double, Double) (Double, Double)
beheerder = arr (* 2) *** arr (+ 10)
-- invoer: (x, y)  ->  uitvoer: (2*x, y+10)
```

Met `&&&` stuur je dezelfde invoer naar twee signal functions tegelijk en verzamel je hun uitvoer in een paar. Dit is handig als je op meerdere manieren tegelijk wil antwoorden op dezelfde invoerstroom:

```haskell
dubbel :: SF Double (Double, Double)
dubbel = arr (* 2) &&& arr (* 3)
-- invoer: x  ->  uitvoer: (2*x, 3*x)
```

---

## Tijdsintegratie met `integral`

Een van de meest essentiële signal functions in Yampa is `integral`, die de invoer integreert over de tijd met behulp van de Euler-methode. Als de invoer de snelheid van een object is (in eenheden per seconde), dan geeft `integral` de positie terug. Als de invoer een versnelling is, geeft twee maal `integral` (de positie als functie van versnelling):

```haskell
-- positie als functie van snelheid
positie :: SF Double Double
positie = integral

-- positie als functie van versnelling
positieVanVersnelling :: SF Double Double
positieVanVersnelling = integral >>> integral
```

Yampa geeft `integral` automatisch de elapsed time (`DTime`) bij elke stap, zodat jij je niet hoeft te bekommeren om de time step.

---

## Arrow-notatie met `proc` en `-<`

Voor complexere signal functions is de infix-compositorstijl snel onleesbaar. Haskell biedt een speciale syntax voor arrows — vergelijkbaar met `do`-notatie voor Monads — die het werken met signal functions veel leesbaarder maakt. Je gebruikt het sleutelwoord `proc` om de invoer te binden en `-<` om een signal function toe te passen:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

-- Een bal met zwaartekracht
-- invoer: beginsnelheid (Double)
-- uitvoer: hoogte (Double)
vallendebal :: SF Double Double
vallendebal = proc beginSnelheid -> do
  let versnelling = -9.81
  snelheid   <- integral -< versnelling + beginSnelheid
  hoogte     <- integral -< snelheid
  returnA -< hoogte
```

In dit voorbeeld lees je `proc beginSnelheid ->` als "neem de invoer en noem hem `beginSnelheid`". De pijl `variabele <- signalFunction -< invoer` past `signalFunction` toe op `invoer` en slaat het resultaat op in `variabele`. `returnA -< uitvoer` produceert de einduitvoer van de signal function. Met `let` kun je gewone berekeningen uitvoeren zonder tijdsaspect.

De `proc`/`-<`-syntax vereist de language extension `{-# LANGUAGE Arrows #-}`, die je best in je `.ghci`-bestand of je `.cabal`-bestand zet.

---

## Signal functions testen met `embed`

Yampa biedt de functie `embed` om signal functions te testen zonder een volledig reactief systeem op te starten. `embed sf samples` voert de signal function `sf` uit op een lijst van tijdsstempels met bijbehorende invoerwaarden en geeft een lijst van uitvoerwaarden terug:

```haskell
import FRP.Yampa

-- embed :: SF a b -> (a, [(DTime, Maybe a)]) -> [b]
resultaten :: [Double]
resultaten = embed positie
  ( 5.0                              -- beginwaarde invoer
  , [ (0.1, Nothing)                 -- na 0.1s, zelfde invoer
    , (0.1, Nothing)
    , (0.1, Just 0.0)                -- na 0.1s, nieuwe invoer: 0.0
    , (0.5, Nothing)
    ]
  )
```

De eerste component is de beginwaarde van de invoer. Elk element in de lijst is een paar `(DTime, Maybe a)`: het time difference ten opzichte van de vorige stap, en de optionele nieuwe invoerwaarde (`Nothing` houdt de vorige invoer). `embed` is bijzonder nuttig voor unit tests van signal functions zonder externe dependencies.

---

## Demo: een vallendesimulator bouwen

In deze demo bouwen we stap voor stap een simulatie van een vallende bal. Onderweg komen `arr`, `>>>`, `integral`, `proc`-notatie en `embed` systematisch aan bod.

---

### Stap 1: De module-header en imports

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa
```

De `Arrows`-language extension is nodig voor de `proc`/`-<`-syntax. `FRP.Yampa` exporteert `arr`, `integral`, `embed`, `constant` en alle compositie-operatoren.

---

### Stap 2: Eenvoudige transformaties met `arr`

We definiëren een hulp-signal function die een signaal schaalt. Ze is tijdloos: elke tijdpunt wordt onafhankelijk getransformeerd:

```haskell
schalen :: Double -> SF Double Double
schalen factor = arr (* factor)
```

---

### Stap 3: Snelheid en positie via `integral`

`integral` accumuleert de invoer over de tijd. Door twee maal te integreren gaan we van versnelling naar snelheid naar positie:

```haskell
-- versnelling -> snelheid -> positie
positieVanVersnelling :: SF Double Double
positieVanVersnelling = integral >>> integral
```

---

### Stap 4: De vallendebal in `proc`-notatie

We voegen zwaartekracht en integratie samen in een leesbaar `proc`-blok. Invoer is de beginsnelheid, uitvoer is de hoogte:

```haskell
vallendebal :: SF Double Double
vallendebal = proc beginSnelheid -> do
  let versnelling = -9.81
  snelheid <- integral -< beginSnelheid + versnelling
  hoogte   <- integral -< snelheid
  returnA -< max 0.0 hoogte
```

`max 0.0` zorgt ervoor dat de bal niet door de vloer zakt.

---

### Stap 5: Testen met `embed`

`embed` voert de signal function uit over een vaste reeks tijdsstappen — geen IO nodig. We simuleren 2 seconden met stappen van 0.1s:

```haskell
main :: IO ()
main = do
  let dt      = 0.1
      stappen = replicate 20 (dt, Nothing)
      hoogtes = embed vallendebal (0.0, stappen)
  mapM_ print hoogtes
```

De bal start met beginsnelheid 0.0 en valt 2 seconden. De uitvoer is een lijst van hoogtes die richting nul dalen.

---

### Volledig programma

<details closed>
<summary><i><b>Volledig programma — klik om te tonen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa

vallendebal :: SF Double Double
vallendebal = proc beginSnelheid -> do
  let versnelling = -9.81
  snelheid <- integral -< beginSnelheid + versnelling
  hoogte   <- integral -< snelheid
  returnA -< max 0.0 hoogte

main :: IO ()
main = do
  let dt      = 0.1
      stappen = replicate 20 (dt, Nothing)
      hoogtes = embed vallendebal (0.0, stappen)
  mapM_ print hoogtes
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
-- .ghci --- signal-functions

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

### Oefeningenreeks 1: `arr` en compositie

- Schrijf een signal function `omzetten :: SF Double Double` die de invoer vermenigvuldigt met 2 en daarna 5 optelt. Gebruik `arr` en `>>>`. Test met `embed omzetten (1.0, [(0.1, Nothing), (0.1, Just 3.0)])`.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
omzetten :: SF Double Double
omzetten = arr (* 2) >>> arr (+ 5)
``` -->

- Schrijf een signal function `splitter :: SF Double (Double, Double)` die de invoer zowel verdubbelt als halveert via `&&&`. Test met `embed splitter (4.0, [(0.1, Nothing)])`.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
splitter :: SF Double (Double, Double)
splitter = arr (* 2) &&& arr (/ 2)
``` -->

- Schrijf een signal function `verschil :: SF (Double, Double) Double` die het verschil tussen twee invoerkanalen berekent via `***` en daarna `arr (uncurry (-))`. Test met `embed verschil ((10.0, 3.0), [(0.1, Just (8.0, 5.0))])`.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
verschil :: SF (Double, Double) Double
verschil = arr (uncurry (-))
``` -->

---

### Oefeningenreeks 2: `integral` en tijdsintegratie

- Schrijf een signal function `afstand :: SF () Double` die een constante snelheid van 2.0 m/s integreert naar een afstand. Gebruik `constant 2.0 >>> integral`. Test over 5 stappen van 0.2s en controleer of de uitvoer `[0.4, 0.8, 1.2, 1.6, 2.0]` is.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
afstand :: SF () Double
afstand = constant 2.0 >>> integral

-- embed afstand ((), replicate 5 (0.2, Nothing))
-- => [0.4, 0.8, 1.2, 1.6, 2.0]
``` -->

- Schrijf een signal function `vallenMetBeginSnelheid :: Double -> SF () Double` die een object laat vallen met een gegeven beginsnelheid (omhoog positief) en zwaartekracht -9.81. Gebruik `proc`-notatie en zorg dat de hoogte niet negatief wordt. Test met beginsnelheid 5.0 over 15 stappen van 0.1s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

vallenMetBeginSnelheid :: Double -> SF () Double
vallenMetBeginSnelheid v0 = proc () -> do
  snelheid <- integral -< v0 + (-9.81)
  hoogte   <- integral -< snelheid
  returnA -< max 0.0 hoogte

-- embed (vallenMetBeginSnelheid 5.0) ((), replicate 15 (0.1, Nothing))
```

</p>
</details> -->

---

### Oefeningenreeks 3: `proc`-notatie

- Schrijf een signal function `gemiddelde :: SF (Double, Double) Double` in `proc`-notatie die twee invoerkanalen gemiddeld neemt. Test met `embed gemiddelde ((2.0, 8.0), [(0.1, Just (4.0, 6.0))])`.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

gemiddelde :: SF (Double, Double) Double
gemiddelde = proc (x, y) -> returnA -< (x + y) / 2.0
``` -->

- Schrijf een signal function `snelheidsMeter :: SF Double (Double, Double)` in `proc`-notatie die zowel de huidige snelheid (invoer) als de afgelegde afstand (integraal van de snelheid) als paar teruggeeft. Test over 10 stappen van 0.1s met constante invoersnelheid 3.0.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

snelheidsMeter :: SF Double (Double, Double)
snelheidsMeter = proc snelheid -> do
  afstand <- integral -< snelheid
  returnA -< (snelheid, afstand)

-- embed snelheidsMeter (3.0, replicate 10 (0.1, Nothing))
```

</p>
</details> -->
