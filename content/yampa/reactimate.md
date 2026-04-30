---
title: "Reactimate"
weight: 3
draft: false
---

## Yampa uitvoeren: het `reactimate`-patroon

Een signal function beschrijft *wat* er berekend moet worden, maar niet *wanneer* of *hoe* de invoer binnenkomt en de uitvoer verwerkt wordt. Die koppeling aan de echte wereld — een klok, een invoerbron, een renderer — levert Yampa's functie **`reactimate`**. Het is de standaardmanier om een Yampa-simulatie als een echte loop te draaien.

De signatuur van `reactimate` ziet er als volgt uit:

```haskell
reactimate
  :: IO a                          -- initialisatie: eerste invoerwaarde
  -> (Bool -> IO (DTime, Maybe a)) -- sense: haal de volgende invoer op
  -> (Bool -> b -> IO Bool)        -- actuate: verwerk de uitvoer
  -> SF a b                        -- de signal function
  -> IO ()
```

`reactimate` is een oneindige loop. Bij elke iteratie:
1. roept hij **sense** aan om de elapsed time (`DTime`, in seconden) en optioneel een nieuwe invoerwaarde op te halen;
2. voert hij de signal function uit voor die time step;
3. roept hij **actuate** aan met het resultaat;
4. stopt hij als **actuate** `True` teruggeeft.

---

## De drie callbacks

### Initialisatiefunctie

De eerste parameter is een `IO a`-actie die de **beginwaarde** van de invoer produceert. Deze wordt precies één keer uitgevoerd vóór de loop begint:

```haskell
initialiseer :: IO GameInvoer
initialiseer = do
  putStrLn "Spel gestart"
  return beginInvoer
```

### Sense-callback

De sense-callback heeft type `Bool -> IO (DTime, Maybe a)`. De `Bool`-parameter geeft aan of de vorige actuate-stap de uitvoer als "gewijzigd" beschouwde (dit wordt zelden gebruikt). De callback retourneert de elapsed time sinds de vorige stap (in seconden) en een optionele nieuwe invoerwaarde — `Nothing` betekent dat de vorige invoer ongewijzigd blijft:

```haskell
sense :: Bool -> IO (DTime, Maybe GameInvoer)
sense _ = do
  nu       <- getCurrentTime
  invoer   <- leesInvoer
  return (tijdstap, Just invoer)
```

In de praktijk meet je vaak de system time en bereken je het verschil met de vorige frame. Zorg ervoor dat `DTime` altijd strikt positief is: een time step van nul kan problemen geven bij sommige signal functions.

### Actuate-callback

De actuate-callback heeft type `Bool -> b -> IO Bool`. De eerste `Bool`-parameter geeft aan of de uitvoer veranderd is ten opzichte van de vorige stap (Yampa kan dit optimaliseren). De `b`-waarde is de uitvoer van de signal function. De teruggegeven `Bool` geeft aan of de loop moet stoppen (`True` = stop):

```haskell
actuate :: Bool -> GameUitvoer -> IO Bool
actuate _ uitvoer = do
  renderFrame uitvoer
  return (spelVoorbij uitvoer)
```

---

## `embed` voor testen en batch-simulatie

Wanneer je geen echte loop nodig hebt maar een signal function wil uitvoeren over een vooraf bepaalde reeks invoerwaarden — voor unit tests of batchsimulaties — gebruik je `embed`:

```haskell
-- embed :: SF a b -> (a, [(DTime, Maybe a)]) -> [b]
testResultaten :: [GameToestand]
testResultaten = embed gameSF
  ( []
  , [ (0.1, Just [Pijl 1])
    , (0.1, Nothing)
    , (0.1, Just [Pijl 3])
    , (0.1, Just [Stoppen])
    ]
  )
```

`embed` is volledig puur: geen IO, geen side effects, ideaal voor property-based tests met QuickCheck.

---

## Demo: een game loop met `TQueue` en `reactimate`

In deze demo bouwen we stap voor stap een game loop waarbij een aparte invoerthread events schrijft naar een `TQueue`, en `reactimate` die events per frame verwerkt via een game signal function.

---

### Stap 1: De module-header, imports en datatypes

```haskell
module Main where

import FRP.Yampa
import Control.Concurrent
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
import Data.IORef
import Data.Time.Clock

data InvoerEvent  = Pijl Int | Stoppen deriving (Show, Eq)
data GameToestand = GameToestand { positie :: Int, gestopt :: Bool } deriving (Show)
```

`InvoerEvent` modelleert de twee soorten invoer: een positieverschuiving (`Pijl`) en een stopsignaal. `GameToestand` bevat de spelstate die elke frame wordt berekend en afgedrukt.

---

### Stap 2: De `leegmaken`-hulpfunctie

De sense-callback moet de queue elke frame non-blokkerend leegmaken. We doen dit in één atomische STM-transactie via `flushTQueue`:

```haskell
leegmaken :: TQueue a -> IO [a]
leegmaken q = atomically $ do
  eerste <- tryReadTQueue q
  case eerste of
    Nothing -> return []
    Just v  -> do
      rest <- flushTQueue q
      return (v : rest)
```

`flushTQueue` leest alle resterende elementen in één keer. Door te beginnen met `tryReadTQueue` vermijden we een lege lijst te returnen als de queue al leeg was.

---

### Stap 3: De game signal function

`gameSF` verwerkt een lijst van events per frame. `accumHold` accumuleert de positieverschuivingen; het stopsignaal wordt direct in de toestand gezet:

```haskell
gameSF :: SF [InvoerEvent] GameToestand
gameSF = proc events -> do
  let delta = sum [ d | Pijl d <- events ]
  pos  <- accumHold 0 (+) -< if null events then NoEvent else Event delta
  let stop = any (== Stoppen) events
  returnA -< GameToestand pos stop
```

---

### Stap 4: De invoerthread

Een aparte thread simuleert toetsaanslagen door events na een vertraging naar de queue te schrijven:

```haskell
invoerThread :: TQueue InvoerEvent -> IO ()
invoerThread wachtrij = do
  threadDelay 300000
  atomically $ writeTQueue wachtrij (Pijl 1)
  threadDelay 300000
  atomically $ writeTQueue wachtrij (Pijl 2)
  threadDelay 300000
  atomically $ writeTQueue wachtrij Stoppen
```

---

### Stap 5: Alles samenvoegen met `reactimate`

`main` initialiseert de queue en een `IORef` voor tijdmeting, start de invoerthread en roept `reactimate` aan met de drie callbacks:

```haskell
main :: IO ()
main = do
  wachtrij <- newTQueueIO
  forkIO (invoerThread wachtrij)
  startTijd <- getCurrentTime
  tijdRef   <- newIORef startTijd
  reactimate
    (return [])
    (\_ -> do
        nu     <- getCurrentTime
        vorige <- readIORef tijdRef
        writeIORef tijdRef nu
        events <- leegmaken wachtrij
        return (realToFrac (diffUTCTime nu vorige), Just events)
    )
    (\_ toestand -> do
        print toestand
        return (gestopt toestand)
    )
    gameSF
```

De sense-callback meet de elapsed time via `diffUTCTime` en leegt de queue. De actuate-callback drukt de toestand af en stopt de loop zodra `gestopt` `True` is.

---

### Volledig programma

<details closed>
<summary><i><b>Volledig programma — klik om te tonen</b></i>🔽</summary>
<p>

```haskell
module Main where

import FRP.Yampa
import Control.Concurrent
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
import Data.IORef
import Data.Time.Clock

data InvoerEvent  = Pijl Int | Stoppen deriving (Show, Eq)
data GameToestand = GameToestand { positie :: Int, gestopt :: Bool } deriving (Show)

leegmaken :: TQueue a -> IO [a]
leegmaken q = atomically $ do
  eerste <- tryReadTQueue q
  case eerste of
    Nothing -> return []
    Just v  -> do
      rest <- flushTQueue q
      return (v : rest)

gameSF :: SF [InvoerEvent] GameToestand
gameSF = proc events -> do
  let delta = sum [ d | Pijl d <- events ]
  pos  <- accumHold 0 (+) -< if null events then NoEvent else Event delta
  let stop = any (== Stoppen) events
  returnA -< GameToestand pos stop

invoerThread :: TQueue InvoerEvent -> IO ()
invoerThread wachtrij = do
  threadDelay 300000
  atomically $ writeTQueue wachtrij (Pijl 1)
  threadDelay 300000
  atomically $ writeTQueue wachtrij (Pijl 2)
  threadDelay 300000
  atomically $ writeTQueue wachtrij Stoppen

main :: IO ()
main = do
  wachtrij <- newTQueueIO
  forkIO (invoerThread wachtrij)
  startTijd <- getCurrentTime
  tijdRef   <- newIORef startTijd
  reactimate
    (return [])
    (\_ -> do
        nu     <- getCurrentTime
        vorige <- readIORef tijdRef
        writeIORef tijdRef nu
        events <- leegmaken wachtrij
        return (realToFrac (diffUTCTime nu vorige), Just events)
    )
    (\_ toestand -> do
        print toestand
        return (gestopt toestand)
    )
    gameSF
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
-- .ghci --- reactimate

-- Prompt
:set prompt "ghci> "
:set prompt-cont "     | "

-- Warnings
:set -Wall

-- Arrow-notatie
:set -XArrows

-- Yampa beschikbaar stellen (buiten cabal repl)
:set -package Yampa
:set -package stm

-- Imports die in de demo en oefeningen gebruikt worden
import FRP.Yampa
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
```

</p>
</details>

_De gegeven oplossingen zijn EEN mogelijke oplossing, soms zijn meerdere mogelijkheden juist. Is het gewenste gedrag bereikt, dan is je oplossing correct!_

### Oefeningenreeks 1: `embed` voor testen

- Test `gameSF` met `embed`: geef een reeks events mee die eerst `Pijl 1`, dan `Pijl (-1)` en tot slot `Stoppen` bevatten. Druk alle tussenliggende `GameToestand`-waarden af en verifieer de posities.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

data InvoerEvent  = Pijl Int | Stoppen deriving (Show, Eq)
data GameToestand = GameToestand { positie :: Int, gestopt :: Bool } deriving (Show)

gameSF :: SF [InvoerEvent] GameToestand
gameSF = proc events -> do
  let delta = sum [ d | Pijl d <- events ]
  pos  <- accumHold 0 (+) -< if null events then NoEvent else Event delta
  let stop = any (== Stoppen) events
  returnA -< GameToestand pos stop

-- embed gameSF
--   ( []
--   , [ (0.1, Just [Pijl 1])
--     , (0.1, Just [Pijl (-1)])
--     , (0.1, Just [Stoppen])
--     ]
--   )
-- => [GameToestand 1 False, GameToestand 0 False, GameToestand 0 True]
``` -->

- Schrijf een eenvoudige signal function `teller :: SF () Int` die bij elke stap met 1 verhoogd wordt (via `accumHold` en `now ()`). Test met `embed` over 5 stappen van 1.0s.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa

teller :: SF () Int
teller = proc () -> do
  puls <- repeatedly 1.0 () -< ()
  n    <- accumHold 0 (\x () -> x + 1) -< puls
  returnA -< n

-- embed teller ((), replicate 5 (1.0, Nothing))
-- => [0, 1, 2, 3, 4]
```

</p>
</details> -->

---

### Oefeningenreeks 2: sense en actuate opbouwen

- Schrijf een sense-callback die altijd een vaste tijdstap van 0.016s teruggeeft (60 FPS) en als invoer een constante `()` levert. Verifieer het type: `Bool -> IO (DTime, Maybe ())`.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
senseFps60 :: Bool -> IO (DTime, Maybe ())
senseFps60 _ = return (0.016, Just ())
``` -->

- Schrijf een actuate-callback voor een `Int`-uitvoer die de waarde afdrukt en stopt wanneer de uitvoer groter is dan 10.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
actuateInt :: Bool -> Int -> IO Bool
actuateInt _ n = do
  print n
  return (n > 10)
``` -->

---

### Oefeningenreeks 3: volledige `reactimate`-loop

- Bouw een minimale `reactimate`-loop die een teller laat oplopen van 0 tot en met 5. Gebruik `repeatedly 0.5 ()` als signal function, tel met `accumHold` en stop via actuate zodra de teller 5 bereikt. Gebruik een vaste tijdstap van 0.5s in de sense-callback.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
{-# LANGUAGE Arrows #-}
module Main where

import FRP.Yampa

tellerSF :: SF () Int
tellerSF = proc () -> do
  puls <- repeatedly 0.5 () -< ()
  n    <- accumHold 0 (\x () -> x + 1) -< puls
  returnA -< n

main :: IO ()
main = reactimate
  (return ())
  (\_ -> return (0.5, Just ()))
  (\_ n -> do
      print n
      return (n >= 5)
  )
  tellerSF
```

</p>
</details> -->
