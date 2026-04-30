---
title: "Globale State — IORef en MVar"
weight: 2
draft: false
---

## Mutable state in een pure taal

Haskell-waarden zijn standaard onveranderlijk. Als je eenmaal een waarde aan een naam hebt gebonden, verandert die waarde nooit meer. Dit is een grote sterkte van de taal: het maakt het eenvoudig om over code te redeneren en elimineert een hele klasse fouten die met gedeelde mutable state gepaard gaan. In de praktijk zijn er echter situaties waarbij je state wil bijhouden die evolueert doorheen de uitvoering van het programma: een teller, een cache, de huidige configuratie, of de positie van een spelentiteit.

Voor zulke gevallen biedt Haskell **mutable referenties** binnen de `IO`-monade. Het sleutelprincipe is dat de veranderlijkheid altijd zichtbaar is in het type: een functie die werkt met een `IORef` of `MVar` heeft altijd een type dat `IO` bevat, zodat de compiler weet dat er neveneffecten in het spel zijn. Je kunt een `IORef` nooit per ongeluk aanpassen in een pure functie — het type-systeem verhindert dat.

---

## `IORef` — een eenvoudige mutable cel

`IORef a` is een mutable referentie naar een waarde van type `a`. Je maakt één aan met `newIORef`, leest de huidige waarde met `readIORef`, overschrijft de waarde met `writeIORef`, en past de waarde aan met een pure functie via `modifyIORef'`. Al deze operaties leven in de `IO`-monade.

```haskell
import Data.IORef

newIORef    :: a -> IO (IORef a)
readIORef   :: IORef a -> IO a
writeIORef  :: IORef a -> a -> IO ()
modifyIORef' :: IORef a -> (a -> a) -> IO ()
```

Een klassiek voorbeeld is een teller die tien keer verhoogd wordt. De `modifyIORef'`-functie neemt een pure functie `(a -> a)` en past die toe op de huidige waarde:

```haskell
import Data.IORef

main :: IO ()
main = do
  teller <- newIORef (0 :: Int)
  mapM_ (\_ -> modifyIORef' teller (+1)) [1..10]
  waarde <- readIORef teller
  putStrLn ("Eindwaarde: " ++ show waarde)
```

### `modifyIORef'` versus `modifyIORef`

Er bestaan twee versies: de strenge versie met apostrof (`modifyIORef'`) en de luie versie zonder (`modifyIORef`). De luie versie bouwt bij herhaald gebruik in een loop een keten van niet-geëvalueerde thunks op, wat geheugenlekkage veroorzaakt. **Gebruik altijd de strenge versie `modifyIORef'`** staan wanneer je een `IORef` in een loop aanpast.

---

## Een `IORef` doorgeven als argument

De meest idiomatische manier om mutable state in Haskell te beheren, is om de `IORef` aan te maken in `main` en door te geven als argument aan de functies die hem nodig hebben. Dit houdt de code transparant: elke functie die de state kan veranderen, heeft een `IORef` als parameter, en dat is direct zichtbaar in zijn type.

```haskell
import Data.IORef

verwerkItem :: IORef Int -> String -> IO ()
verwerkItem teller item = do
  putStrLn ("Verwerken: " ++ item)
  modifyIORef' teller (+1)

main :: IO ()
main = do
  teller <- newIORef (0 :: Int)
  let items = ["appel", "peer", "banaan"]
  mapM_ (verwerkItem teller) items
  n <- readIORef teller
  putStrLn ("Verwerkt: " ++ show n ++ " items")
```

---

## `MVar` — threadsafe mutable state

`IORef` is niet threadveilig: als meerdere threads tegelijkertijd een `IORef` lezen en schrijven, kunnen er raceconditities optreden. Voor gedeelde state tussen threads gebruik je `MVar a`, een mutable variabele die ontworpen is voor veilig concurrente toegang.

Een `MVar` werkt als een doos die ofwel **vol** is (bevat een waarde) ofwel **leeg** is. De operatie `takeMVar` neemt de waarde uit de doos en maakt hem leeg — als de doos al leeg was, blokkeert de aanroepende thread totdat een andere thread er een waarde in zet. `putMVar` zet een waarde in een lege doos — als de doos al vol was, blokkeert de aanroepende thread totdat een andere thread de waarde heeft weggenomen. Dit blokkeringsgedrag maakt `MVar` een natuurlijk mutex-mechanisme.

```haskell
import Control.Concurrent.MVar

newMVar      :: a -> IO (MVar a)
newEmptyMVar :: IO (MVar a)
takeMVar     :: MVar a -> IO a
putMVar      :: MVar a -> a -> IO ()
readMVar     :: MVar a -> IO a           -- lees zonder leeg te maken (atomisch)
modifyMVar_  :: MVar a -> (a -> IO a) -> IO ()  -- atomische aanpassing
```

Een voorbeeld van een threadveilige teller waarbij tien threads elk duizend keer verhogen:

```haskell
import Control.Concurrent
import Control.Concurrent.MVar

main :: IO ()
main = do
  teller <- newMVar (0 :: Int)
  let verhoog = modifyMVar_ teller (\n -> return (n + 1))
  threads <- mapM (\_ -> forkIO (mapM_ (const verhoog) [1..1000])) [1..10]
  threadDelay 200000   -- wacht 200 ms zodat alle threads kunnen eindigen
  eindwaarde <- readMVar teller
  putStrLn ("Verwacht: 10000, Gekregen: " ++ show eindwaarde)
```

### `MVar` als mutex

Een andere frequente toepassing van `MVar` is als een loutere vergrendeling, waarbij de waarde binnenin niet belangrijk is maar de aan/afwezigheid de lock-status aangeeft. Dit wordt ook wel een binaire semafoor of mutex genoemd:

```haskell
import Control.Concurrent.MVar

metSlot :: MVar () -> IO a -> IO a
metSlot slot actie = do
  takeMVar slot     -- vergrendeling acquireren
  resultaat <- actie
  putMVar slot ()   -- vergrendeling vrijgeven
  return resultaat

main :: IO ()
main = do
  slot <- newMVar ()
  metSlot slot $ putStrLn "Kritische sectie"
```

---

## STM en `TVar` — samengestelde transacties

Voor complexere scenario's met meerdere gedeelde variabelen biedt de **Software Transactional Memory** (STM) library `TVar`, een transactionele variabele. Het krachtige aan STM is dat je meerdere lees- en schrijfoperaties op verschillende `TVar`s kunt groeperen in één atomische transactie via `atomically`. Als een andere thread tussenin een van de gebruikte `TVar`s aanpast, wordt de hele transactie automatisch opnieuw uitgevoerd — transparant en zonder deadlocks.

```haskell
import Control.Concurrent.STM

overschrijving :: TVar Int -> TVar Int -> Int -> STM ()
overschrijving van naar bedrag = do
  saldoVan  <- readTVar van
  saldoNaar <- readTVar naar
  writeTVar van  (saldoVan  - bedrag)
  writeTVar naar (saldoNaar + bedrag)

main :: IO ()
main = do
  rekeningA <- newTVarIO 1000
  rekeningB <- newTVarIO 500
  atomically (overschrijving rekeningA rekeningB 200)
  a <- readTVarIO rekeningA
  b <- readTVarIO rekeningB
  putStrLn ("A: " ++ show a ++ ", B: " ++ show b)
```

---

## Wanneer welk hulpmiddel?

| Scenario | Gebruik |
|---|---|
| Eenvoudige mutable state, één thread | `IORef` |
| Gedeelde state tussen threads, eenvoudig | `MVar` |
| Gedeelde state, complexe atomische bewerkingen | `TVar` / STM |
| Globale configuratie (gebruik spaarzaam) | `IORef` als argument in `main` |
