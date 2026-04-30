---
title: "Concurrency en Queues"
weight: 3
draft: false
---

## Concurrentie in Haskell

De Haskell-runtime (GHC) ondersteunt **lichtgewicht threads**: groene threads die door de runtime zelf beheerd worden en op een pool van echte OS-threads draaien. Je kunt tienduizenden Haskell-threads aanmaken zonder noemenswaardige overhead, want ze worden efficiënt ingepland door de runtime-scheduler. Het aantal OS-threads dat de runtime gebruikt, stel je in via de runtime-vlag `+RTS -N` (of `-N4` voor vier cores).

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

---

## Channels en queues

Wanneer threads informatie met elkaar moeten uitwisselen, heb je een communication channel nodig. Eenvoudige gedeelde `IORef`s zijn daarvoor niet geschikt omdat ze niet threadveilig zijn zonder extra synchronisatie. Haskell biedt meerdere types channels, elk met hun eigen eigenschappen.

### `Chan` — onbegrensde FIFO channel

`Chan a` is een onbegrensde, threadveilige FIFO channel. Je schrijft waarden in het channel met `writeChan` en leest ze eruit met `readChan`. Als het channel leeg is, blokkeert `readChan` totdat er een waarde beschikbaar is. Het channel groeit automatisch mee naarmate er meer waarden in worden gezet. Met `dupChan` maak je een duplicaat van het channel zodat twee ontvangers onafhankelijk dezelfde stroom van berichten ontvangen:

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

Dit klassieke producent-consumentpatroon toont hoe twee threads via een channel samenwerken: de producent schrijft asynchroon, de consument leest blokkerend.

---

### `TQueue` — STM-gebaseerde queue (aanbevolen)

`TQueue` is een FIFO queue gebouwd op STM. Ze werkt gelijkaardig aan `Chan`, maar omdat ze STM gebruikt, kun je lees- en schrijfoperaties combineren in bredere atomische transacties. Dat opent de deur naar **niet-blokkerende leesbewerkingen** via `tryReadTQueue`, waarbij je `Nothing` terugkrijgt als de queue leeg is in plaats van te blokkeren. Dit is precies wat je nodig hebt in een game-loop die niet mag pauzeren terwijl hij wacht op invoer:

```haskell
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

newTQueueIO    :: IO (TQueue a)
writeTQueue    :: TQueue a -> a -> STM ()
readTQueue     :: TQueue a -> STM a           -- blokkeert als leeg
tryReadTQueue  :: TQueue a -> STM (Maybe a)   -- niet-blokkerend
isEmptyTQueue  :: TQueue a -> STM Bool
```

Een veelgebruikt patroon in FRP-toepassingen is de **event queue**: een aparte invoerthread schrijft events naar de queue, terwijl de hoofdsimulatie bij elke stap de queue leegmaakt. Dit ontkoppelt de timing van invoerverwerking van de timing van de simulatiestappen. Je kunt de queue leegmaken met een helperfunctie die herhaaldelijk probeert te lezen tot er niets meer is:

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
```

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
