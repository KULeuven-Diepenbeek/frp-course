---
title: "Concurrency en Queues"
weight: 3
draft: false
---

## Concurrentie in Haskell

De Haskell-runtime (GHC) ondersteunt **lightweight threads**: green threads die door de runtime zelf beheerd worden en op een pool van echte OS-threads draaien. Je kunt tienduizenden Haskell-threads aanmaken zonder noemenswaardige overhead, want ze worden efficiënt ingepland door de runtime-scheduler. Het aantal OS-threads dat de runtime gebruikt, stel je in via de runtime-vlag `+RTS -N` (of `-N4` voor vier cores).

Een nieuwe thread aanmaken doe je met `forkIO`. Deze functie neemt een `IO ()`-actie en voert die uit in een aparte thread, terwijl de aanroepende thread onmiddellijk verder gaat. De return-waarde is een `ThreadId` waarmee je de thread later kunt inspecteren of stoppen:

```haskell
import Control.Concurrent

main :: IO ()
main = do
  forkIO $ do
    threadDelay 500000    -- wacht 0,5 seconden
    putStrLn "Thread klaar"
  putStrLn "Main gaat verder"
  threadDelay 1000000     -- houd main lang genoeg in leven
```

De functie `threadDelay` pauzeert de huidige thread voor het opgegeven aantal **microseconden**. Dit is de standaard manier om een loop op een vaste framerate te laten draaien, of om te wachten op externe events.

{{% notice warning %}}
De precisie van `threadDelay` is **microseconden**, maar de effectieve minimumvertraging is in de praktijk veel groter. Het OS-schedulinginterval bedraagt typisch **~15–20 ms** op Windows en **~1–4 ms** op Linux/macOS. Een aanroep `threadDelay 1` (1 µs) wacht dus in werkelijkheid minstens ~15 ms, omdat het OS pas na die cyclus opnieuw checkt. Gebruik `threadDelay` niet voor tijdkritische toepassingen die sub-milliseconde precisie vereisen.
{{% /notice %}}

---

## Channels en queues

Wanneer threads informatie met elkaar moeten uitwisselen, heb je een communication channel nodig. Eenvoudige gedeelde `IORef`s zijn daarvoor niet geschikt omdat ze niet threadsafe zijn zonder extra synchronisatie. Haskell biedt meerdere types channels, elk met hun eigen eigenschappen.

### `Chan` — onbegrensd FIFO channel

`Chan a` is een onbegrensd, threadsafe FIFO channel. Je schrijft waarden in het channel met `writeChan` en leest ze eruit met `readChan`. Als het channel leeg is, blokkeert `readChan` totdat er een waarde beschikbaar is. Het channel groeit automatisch mee naarmate er meer waarden in worden gezet. Met `dupChan` maak je een duplicaat van het channel zodat twee ontvangers onafhankelijk dezelfde stroom van berichten ontvangen:

```haskell
import Control.Concurrent
import Control.Concurrent.Chan

producent :: Chan Int -> IO ()
producent kanaal = mapM_ (\i -> do
    putStrLn ("Produceren: " ++ show i)
    writeChan kanaal i
    threadDelay 200000
  ) [1..5]

consument :: Chan Int -> IO ()
consument kanaal = do
  waarde <- readChan kanaal
  putStrLn ("Consumeren: " ++ show waarde)
  consument kanaal

main :: IO ()
main = do
  kanaal <- newChan
  forkIO (consument kanaal)
  producent kanaal
  threadDelay 2000000
```

Dit klassieke producer-consumerpatroon toont hoe twee threads via een channel samenwerken: de producent schrijft asynchroon, de consument leest blocking.

---

### `TQueue` — STM-gebaseerde queue (aanbevolen)

`TQueue` is een FIFO queue gebouwd op STM. Ze werkt gelijkaardig aan `Chan`, maar omdat ze STM gebruikt, kun je lees- en schrijfoperaties combineren in bredere atomische transacties. Dat opent de deur naar **niet-blokkerende leesbewerkingen** via `tryReadTQueue`, waarbij je `Nothing` terugkrijgt als de queue leeg is in plaats van te blokkeren. Dit is precies wat je nodig hebt in een game/robot-loop die niet mag pauzeren terwijl hij wacht op invoer:

```haskell
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

newTQueueIO    :: IO (TQueue a)
writeTQueue    :: TQueue a -> a -> STM ()
readTQueue     :: TQueue a -> STM a           -- blokkeert als leeg
tryReadTQueue  :: TQueue a -> STM (Maybe a)   -- niet-blokkerend
isEmptyTQueue  :: TQueue a -> STM Bool
```

Een veelgebruikt patroon in FRP-toepassingen is de **event queue**: een aparte invoerthread schrijft events naar de queue, terwijl de hoofdsimulatie bij elke stap de queue leegmaakt. Dit ontkoppelt de timing van invoerverwerking van de timing van de simulatiestappen. Je kunt de queue leegmaken met een helperfunction die herhaaldelijk probeert te lezen tot er niets meer is:

```haskell
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

leegmaken :: TQueue a -> IO [a]
leegmaken wachtrij = go []
  where
    go acc = do
      mWaarde <- atomically (tryReadTQueue wachtrij)
      case mWaarde of
        Nothing     -> return (reverse acc)
        Just waarde -> go (waarde : acc)

main :: IO ()
main = do
  wachtrij <- newTQueueIO
  atomically $ do
    writeTQueue wachtrij 10
    writeTQueue wachtrij 20
    writeTQueue wachtrij 30
  waarden <- leegmaken wachtrij
  print waarden       -- [10, 20, 30]
  leeg <- leegmaken wachtrij
  print leeg          -- []
```

`go` is een lokale hulpfunctie die de queue stap voor stap leeghaalt. `acc` staat voor **accumulator**: de lijst van alle gelezen waarden die stap voor stap wordt opgebouwd. `tryReadTQueue` probeert niet-blokkerend één waarde te lezen — als de queue leeg is, krijg je `Nothing` terug en wordt `reverse acc` teruggegeven (omgekeerd omdat elke nieuwe waarde vóóraan werd toegevoegd met `:`). Is er wél een waarde, dan wordt die bij de accumulator gevoegd en roept `go` zichzelf recursief aan. `atomically` is nodig omdat `tryReadTQueue` een `STM`-actie is, niet rechtstreeks een `IO`-actie: het wikkelt de STM-transactie in een `IO`-actie die atomisch wordt uitgevoerd, zodat geen andere thread de queue tussenin kan aanpassen.


---

### Voorbeeld — event queue voor een game loop

Het volgende voorbeeld toont hoe je een invoerthread combineert met een verwerkings loop via een `TQueue`. De invoerthread simuleert toetsaanslagen en schrijft ze naar de queue. De hoofdloop draait op een vaste framerate en verwerkt bij elke frame alle beschikbare events:

```haskell
import Control.Concurrent
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

data InvoerEvent = ToetsW | ToetsS | Stoppen deriving (Show, Eq)

invoerThread :: TQueue InvoerEvent -> IO ()
invoerThread wachtrij = do
  threadDelay 500000
  atomically $ writeTQueue wachtrij ToetsW
  threadDelay 500000
  atomically $ writeTQueue wachtrij ToetsS
  threadDelay 1000000
  atomically $ writeTQueue wachtrij Stoppen

verwerkEvents :: TQueue InvoerEvent -> IO ()
verwerkEvents wachtrij = do
  events <- leegmaken wachtrij
  mapM_ (\e -> putStrLn ("Event: " ++ show e)) events
  let doorgaan = Stoppen `notElem` events
  if doorgaan
    then do
      threadDelay 16667     -- ~60 FPS
      verwerkEvents wachtrij
    else putStrLn "Spel gestopt."

main :: IO ()
main = do
  wachtrij <- newTQueueIO
  forkIO (invoerThread wachtrij)
  verwerkEvents wachtrij
```

Dit patroon — invoerthread schrijft naar een `TQueue`, simulatie loop leegt de queue per frame — is precies wat je gebruikt wanneer je Yampa's `reactimate`-functie koppelt aan echte invoerbronnen. De `TQueue` fungeert als buffer die het verschil in timing tussen de invoerthread en de simulatie opvangt.

---

### `MVar` als enkelvoudig channel

Een `MVar` kan ook fungeren als een synchroon éénplaats channel: de zender blokkeert als de vorige waarde nog niet afgehaald is. Dit is nuttig voor een handshake-patroon waarbij je wil dat de ontvanger expliciet bevestigt dat hij elke waarde heeft verwerkt voordat de volgende verzonden wordt:

```haskell
import Control.Concurrent
import Control.Concurrent.MVar

main :: IO ()
main = do
  kanaal <- newEmptyMVar
  forkIO $ do
    threadDelay 500000
    putMVar kanaal "Bericht van thread"    -- verzenden
  bericht <- takeMVar kanaal               -- ontvangen (blokkeert)
  putStrLn bericht
```

---

## Samenvatting

| Hulpmiddel | Eigenschap | Wanneer te gebruiken |
|---|---|---|
| `IORef` | Niet threadveilig | Enkelvoudige thread, lokale state |
| `MVar` | Mutex / blokkerend éénplaatskanaal | Eenvoudige synchronisatie tussen threads |
| `Chan` | Onbegrensd FIFO, threadveilig | Eenvoudig producent-consumentpatroon |
| `TQueue` | STM FIFO, niet-blokkerend lezen mogelijk | Game loops, event queues (aanbevolen) |
| `TVar` | Atomische transactional state | Complexe gedeelde state met meerdere variabelen |

---

## Demo: achtergrondverwerking met TQueue

In deze demo bouwen we een programma waarbij een aparte werkerthread taken verwerkt die via een `TQueue` worden aangeleverd. De hoofdthread dient taken in en wacht via een `MVar` op de bevestiging dat alles verwerkt is.

---

### Stap 1: De module-header en imports

```haskell
module Main where

import Control.Concurrent
import Control.Concurrent.MVar
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
```

We importeren `forkIO` via `Control.Concurrent`, `MVar` voor synchronisatie en de STM/TQueue-modules voor de taakwachtrij.

---

### Stap 2: Het taaktype definiëren

We definiëren een algebraïsch datatype voor taken. `Stoppen` fungeert als afsluitsignaal voor de werkerthread:

```haskell
data Taak = VerwerkTekst String | Stoppen deriving (Show)
```

Door `Stoppen` in het type op te nemen hebben we geen aparte stopsignaal-`MVar` nodig. De werker weet via patternmatching wanneer hij klaar is.

---

### Stap 3: De werkerthread

De werker leest taken blocking uit de `TQueue` via `readTQueue`. Bij `VerwerkTekst` verwerkt hij de taak en roept zichzelf recursief aan. Bij `Stoppen` signaleert hij via `putMVar` dat hij klaar is:

```haskell
werker :: TQueue Taak -> MVar () -> IO ()
werker wachtrij klaar = do
  taak <- atomically (readTQueue wachtrij)
  case taak of
    VerwerkTekst tekst -> do
      putStrLn ("Verwerkt: " ++ tekst ++ " (" ++ show (length (words tekst)) ++ " woorden)")
      threadDelay 300000
      werker wachtrij klaar
    Stoppen -> putMVar klaar ()
```

---

### Stap 4: Taken indienen en synchroniseren

Taken worden atomisch aan de wachtrij toegevoegd. Na alle taken voegen we `Stoppen` toe als eindsignaal. `takeMVar klaar` blokkeert `main` totdat de werker `putMVar klaar ()` aanroept:

```haskell
let taken = [ "Haskell is een pure functionele taal"
            , "STM maakt gedeelde state veilig"
            , "TQueue is een threadveilige FIFO queue"
            ]
mapM_ (\t -> atomically (writeTQueue wachtrij (VerwerkTekst t))) taken
atomically (writeTQueue wachtrij Stoppen)
takeMVar klaar
```

---

### Stap 5: Alles samenvoegen in `main`

```haskell
main :: IO ()
main = do
  wachtrij <- newTQueueIO
  klaar    <- newEmptyMVar
  forkIO (werker wachtrij klaar)
  let taken = [ "Haskell is een pure functionele taal"
              , "STM maakt gedeelde state veilig"
              , "TQueue is een threadveilige FIFO queue"
              ]
  mapM_ (\t -> atomically (writeTQueue wachtrij (VerwerkTekst t))) taken
  atomically (writeTQueue wachtrij Stoppen)
  takeMVar klaar
  putStrLn "Alle taken verwerkt."
```

`newTQueueIO` en `newEmptyMVar` maken de gedeelde structuren aan. `forkIO` start de werker als lichtweight thread, waarna de mainthread taken indient en wacht.

---

### Volledig programma

<details closed>
<summary><i><b>Volledig programma — klik om te tonen</b></i>🔽</summary>
<p>

```haskell
module Main where

import Control.Concurrent
import Control.Concurrent.MVar
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

data Taak = VerwerkTekst String | Stoppen deriving (Show)

werker :: TQueue Taak -> MVar () -> IO ()
werker wachtrij klaar = do
  taak <- atomically (readTQueue wachtrij)
  case taak of
    VerwerkTekst tekst -> do
      putStrLn ("Verwerkt: " ++ tekst ++ " (" ++ show (length (words tekst)) ++ " woorden)")
      threadDelay 300000
      werker wachtrij klaar
    Stoppen -> putMVar klaar ()

main :: IO ()
main = do
  wachtrij <- newTQueueIO
  klaar    <- newEmptyMVar
  forkIO (werker wachtrij klaar)
  let taken = [ "Haskell is een pure functionele taal"
              , "STM maakt gedeelde state veilig"
              , "TQueue is een threadveilige FIFO queue"
              ]
  mapM_ (\t -> atomically (writeTQueue wachtrij (VerwerkTekst t))) taken
  atomically (writeTQueue wachtrij Stoppen)
  takeMVar klaar
  putStrLn "Alle taken verwerkt."
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
-- .ghci --- concurrency

-- Prompt
:set prompt "ghci> "
:set prompt-cont "     | "

-- Warnings
:set -Wall

-- Imports die in de demo en oefeningen gebruikt worden
import Control.Concurrent
import Control.Concurrent.MVar
import Control.Concurrent.Chan
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue
```

</p>
</details>

_De gegeven oplossingen zijn EEN mogelijke oplossing, soms zijn meerdere mogelijkheden juist. Is het gewenste gedrag bereikt, dan is je oplossing correct!_

### Oefeningenreeks 1: forkIO en synchronisatie

- Start een thread die `"Hallo van een thread!"` afdrukt. Gebruik een `MVar ()` om te garanderen dat `main` wacht tot de thread klaar is.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Control.Concurrent
import Control.Concurrent.MVar

main :: IO ()
main = do
  klaar <- newEmptyMVar
  forkIO $ do
    putStrLn "Hallo van een thread!"
    putMVar klaar ()
  takeMVar klaar
``` -->

- Start 5 threads, waarbij elke thread zijn threadnummer afdrukt. Gebruik een `MVar Int` als teller om te detecteren wanneer alle 5 threads klaar zijn, en wacht dan in `main`.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Concurrent
import Control.Concurrent.MVar

main :: IO ()
main = do
  teller <- newMVar (0 :: Int)
  klaar  <- newEmptyMVar
  let aantalThreads = 5
  mapM_ (\i -> forkIO $ do
    putStrLn ("Thread " ++ show i)
    n <- takeMVar teller
    let nieuw = n + 1
    putMVar teller nieuw
    if nieuw == aantalThreads
      then putMVar klaar ()
      else return ()
    ) [1..aantalThreads]
  takeMVar klaar
```

</p>
</details> -->

---

### Oefeningenreeks 2: Chan

- Maak een `Chan Int`. Start een producent-thread die de getallen `1` t/m `5` naar het channel schrijft. Lees ze in `main` met `readChan` en druk elk getal af.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Control.Concurrent
import Control.Concurrent.Chan

main :: IO ()
main = do
  kanaal <- newChan
  forkIO $ mapM_ (\i -> writeChan kanaal (i :: Int)) [1..5]
  mapM_ (\_ -> readChan kanaal >>= print) [1..5 :: Int]
``` -->

- Start twee producent-threads die elk 3 berichten naar een `Chan String` schrijven. Lees alle 6 berichten in `main` en druk ze af. Gebruik `threadDelay 200000` na het forken zodat de producenten kunnen schrijven.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Concurrent
import Control.Concurrent.Chan

producent :: Chan String -> String -> [String] -> IO ()
producent kanaal prefix berichten =
  mapM_ (\b -> writeChan kanaal (prefix ++ ": " ++ b)) berichten

main :: IO ()
main = do
  kanaal <- newChan
  forkIO (producent kanaal "A" ["a1", "a2", "a3"])
  forkIO (producent kanaal "B" ["b1", "b2", "b3"])
  threadDelay 200000
  mapM_ (\_ -> readChan kanaal >>= putStrLn) [1..6 :: Int]
```

</p>
</details> -->

---

### Oefeningenreeks 3: TQueue

- Schrijf een werkerthread die taken leest uit een `TQueue` en ze afdrukt. Gebruik een ADT `data Bericht = Tekst String | Stoppen` als signaaltype. Push 4 taken en synchroniseer met een `MVar ()`.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Concurrent
import Control.Concurrent.MVar
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

data Bericht = Tekst String | Stoppen

werker :: TQueue Bericht -> MVar () -> IO ()
werker q klaar = do
  b <- atomically (readTQueue q)
  case b of
    Tekst s -> putStrLn ("Taak: " ++ s) >> werker q klaar
    Stoppen -> putMVar klaar ()

main :: IO ()
main = do
  q     <- newTQueueIO
  klaar <- newEmptyMVar
  forkIO (werker q klaar)
  mapM_ (\t -> atomically (writeTQueue q (Tekst t)))
    ["taak 1", "taak 2", "taak 3", "taak 4"]
  atomically (writeTQueue q Stoppen)
  takeMVar klaar
```

</p>
</details> -->

- Vul een `TQueue Int` met de waarden `1` t/m `10`. Gebruik daarna de `leegmaken`-functie uit de theorie om alle waarden niet-blokkerend te lezen en druk hun som af.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

leegmaken :: TQueue a -> IO [a]
leegmaken wachtrij = go []
  where
    go acc = do
      mWaarde <- atomically (tryReadTQueue wachtrij)
      case mWaarde of
        Nothing     -> return (reverse acc)
        Just waarde -> go (waarde : acc)

main :: IO ()
main = do
  q <- newTQueueIO
  mapM_ (\i -> atomically (writeTQueue q i)) [1..10 :: Int]
  waarden <- leegmaken q
  putStrLn ("Som: " ++ show (sum waarden))
``` -->
