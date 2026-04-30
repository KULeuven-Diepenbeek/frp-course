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

## `reactimate` met een `TQueue` voor invoer

In echte toepassingen komt invoer (toetsaanslagen, muisbewegingen, netwerkberichten) uit een aparte thread. De aanbevolen aanpak is een `TQueue` gebruiken als buffer: de invoerthread schrijft events naar de queue, de sense-callback leest ze non-blokkerend uit. Dit ontkoppelt de timing van invoerverwerking van de timing van de simulatiestap volledig:

```haskell
{-# LANGUAGE Arrows #-}
import FRP.Yampa
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
import Control.Concurrent (forkIO, threadDelay)
import Data.Time.Clock

data InvoerEvent = Pijl Int | Stoppen deriving (Show, Eq)
data GameToestand = GameToestand { positie :: Int, gestopt :: Bool }

leegmaken :: TQueue a -> IO [a]
leegmaken q = atomically $ do
  first <- tryReadTQueue q
  case first of
    Nothing -> return []
    Just v  -> do
      rest <- flushTQueue q
      return (v : rest)

gameSF :: SF [InvoerEvent] GameToestand
gameSF = proc events -> do
  let delta = sum [ d | Pijl d <- events ]
  pos <- accumHold 0 (+) -< if null events then NoEvent else Event delta
  let stop = any (== Stoppen) events
  returnA -< GameToestand pos stop

main :: IO ()
main = do
  wachtrij <- newTQueueIO
  forkIO $ do
    threadDelay 300000
    atomically $ writeTQueue wachtrij (Pijl 1)
    threadDelay 300000
    atomically $ writeTQueue wachtrij (Pijl 2)
    threadDelay 300000
    atomically $ writeTQueue wachtrij Stoppen
  startTijd <- getCurrentTime
  tijdRef   <- newIORef startTijd
  reactimate
    (return [])
    (\_ -> do
        nu       <- getCurrentTime
        vorige   <- readIORef tijdRef
        writeIORef tijdRef nu
        events   <- leegmaken wachtrij
        return (realToFrac (diffUTCTime nu vorige), Just events)
    )
    (\_ uitvoer -> do
        print uitvoer
        return (gestopt uitvoer)
    )
    gameSF
```

Dit volledige voorbeeld toont het volledige reactimate-patroon: een aparte invoerthread, een IORef voor tijdmeting, een TQueue als buffer, en een game signal function.

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
