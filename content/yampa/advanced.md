---
title: "Geavanceerde Yampa"
weight: 5
draft: false
---

## Feedback met `loop`

Tot nu toe zijn alle signal functions **acyclisch** in hun gegevensstroom: de uitvoer beïnvloedt de invoer niet. Yampa biedt echter `loop` om **feedback** te realiseren: een deel van de uitvoer wordt teruggevoerd als extra invoer:

```haskell
loop :: SF (a, c) (b, c) -> SF a b
```

`loop sf` verpakt een signal function `sf` die een paar `(a, c)` als invoer verwacht en een paar `(b, c)` produceert. De tweede component `c` van de uitvoer wordt teruggevoerd als tweede component van de invoer bij de *volgende* time step. Dit is vertraagde feedback: er is altijd één time step vertraging in de loop, wat causaliteit garandeert.

Een klassiek gebruik is een filter met geheugen, zoals een lopend gemiddelde:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

lopendGemiddelde :: Int -> SF Double Double
lopendGemiddelde n = proc invoer -> do
  rec
    som    <- iPre 0.0 -< som + invoer - oudste
    oudste <- iPre 0.0 -< invoer
  returnA -< som / fromIntegral n
```

`iPre v` ("initialized pre") geeft de vorige waarde van een signaal terug, geïnitialiseerd op `v`. Dit is de laagste building block voor feedback en recursieve signal functions.

---

## Higher-order signal functions

Yampa's signal functions zijn gewone Haskell-waarden en kunnen daarom doorgegeven, opgeslagen, en dynamisch gewisseld worden. Het is mogelijk om signal functions te maken die zichzelf of andere signal functions als invoer ontvangen. Dit noemt men **higher-order signal functions**.

Een eenvoudig voorbeeld: een "verzwakker" die een signal function inpakt en zijn uitvoer halveert:

```haskell
verzwakker :: SF a Double -> SF a Double
verzwakker sf = sf >>> arr (* 0.5)
```

Een ingewikkelder voorbeeld: een signal function die dynamisch tussen twee gedragingen wisselt op basis van een booleaans signaal:

```haskell
schakelaar :: SF a b -> SF a b -> SF (a, Bool) b
schakelaar sf1 sf2 = proc (invoer, keuze) ->
  if keuze
    then sf1 -< invoer
    else sf2 -< invoer
```

Let op: dit is **niet** hetzelfde als een switch. Hier draait altijd maar één van de twee SFs per time step. Beide SFs bewaren hun interne state onafhankelijk.

---

## Dynamische entiteitscollecties met `dpSwitch`

Een van de meest geavanceerde toepassingen van Yampa is het beheren van een **dynamische verzameling entiteiten**: spelletjesfiguren die aangemaakt en verwijderd worden tijdens het spel. Yampa biedt hiervoor `dpSwitch` (en `pSwitch`).

Het idee: je onderhoudt een collectie (een `Map` of lijst) van signal functions voor actieve entiteiten. Een aparte signal function bewaakt de collectie en produceert events die entiteiten toevoegen of verwijderen:

```haskell
import FRP.Yampa
import qualified Data.Map.Strict as Map

type EntiteitId = Int
type Entiteiten a b = Map.Map EntiteitId (SF a b)

-- dpSwitch laat een hele Map van SFs parallel lopen
-- en geeft een event om de Map te wijzigen
dpSwitch
  :: Functor col
  => (forall sf. col sf -> input -> col (input, sf))
  -> col (SF input output)
  -> SF (input, col output) (Event switchevent)
  -> (col (SF input output) -> switchevent -> SF input (col output))
  -> SF input (col output)
```

In de praktijk gebruikt men `dpSwitch` via een helper library zoals `yampa-utils` of schrijft men een wrapper die eenvoudiger te gebruiken is. De kern van het patroon is:

```haskell
beheerEntiteiten :: SF GameInvoer (Map EntiteitId EntiteitUitvoer)
beheerEntiteiten = dpSwitch
  routeerInvoer           -- verspreid invoer naar elk entiteits-SF
  beginEntiteiten         -- beginverzameling van SF's
  detecteerWijzigingen    -- SF die bijhoudt welke entiteiten toegevoegd/verwijderd moeten worden
  pasAan                  -- update de collectie bij elke switch
```

---

## Volledig voorbeeld — eenvoudige 2D-simulatie

Het volgende voorbeeld combineert alle technieken: meerdere objecten met zwaartekracht, elk met hun eigen signal function, beheerd via een lijst:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

data Object = Object { objX :: Double, objY :: Double } deriving (Show)

objectSF :: (Double, Double) -> SF () Object
objectSF (startX, startY) = proc () -> do
  let zwaartekracht = -9.81
  vy  <- integral -< zwaartekracht
  x   <- integral -< 0.0          -- geen horizontale versnelling
  y   <- integral -< vy
  returnA -< Object (startX + x) (startY + y)

simulatie :: SF () [Object]
simulatie = proc () -> do
  obj1 <- objectSF (0.0, 10.0) -< ()
  obj2 <- objectSF (5.0, 20.0) -< ()
  obj3 <- objectSF (2.0, 15.0) -< ()
  returnA -< [obj1, obj2, obj3]

testSimulatie :: [Object]
testSimulatie = head $ embed simulatie
  ((), [(0.016, Nothing)])
```

Dit voorbeeld toont hoe je meerdere objecten met hun eigen state tegelijk kunt simuleren. Elke `objectSF` houdt zijn eigen positie bij, volledig onafhankelijk van de anderen.

---

## Performancetips

Yampa is ontworpen voor efficiënt gebruik in realtime toepassingen, maar er zijn een paar aandachtspunten:

**Vermijd onnodige `arr`-kettingen**: Elke `arr` voegt een functieaanroep toe. In kritische code kun je meerdere transformaties samenvoegen in één `arr (f . g . h)` in plaats van `arr f >>> arr g >>> arr h`.

**Gebruik `iPre` met zorg**: `iPre` introduceert een vertraging van één time step. In sommige gevallen is dat gewenst, maar het kan ook leiden tot subtiele timing-problemen als je er te vrijelijk mee omgaat.

**Houd time steps klein en consistent**: Grote time steps (> 0.1s) kunnen leiden tot inconsistente fysica, met name bij integratie. In games streef je naar ~60 FPS (time step ~0.016s).

**Profileer met GHC's profiler**: Voor serieuze toepassingen compileer je met `-prof -fprof-auto` en analyseer je welke signal functions de meeste tijd verbruiken.

**Overweeg `BangPatterns` voor strikte evaluatie**: Haskell evalueert standaard lui, wat in Yampa loops tot stapeling van thunks kan leiden. `{-# LANGUAGE BangPatterns #-}` en expliciete `!`-patronen helpen dit te voorkomen.
