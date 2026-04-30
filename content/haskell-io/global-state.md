---
title: "Globale State — IORef en MVar"
weight: 2
draft: false
---

## Mutable state in een pure taal

Haskell-waarden zijn standaard onveranderlijk. Als je eenmaal een waarde aan een naam hebt gebonden, verandert die waarde nooit meer. Dit is een grote sterkte van de taal: het maakt het eenvoudig om over code te redeneren en elimineert een hele klasse fouten die met gedeelde mutable state gepaard gaan. In de praktijk zijn er echter situaties waarbij je state wil bijhouden die evolueert doorheen de uitvoering van het programma: een teller, een cache, de huidige configuratie, of de positie van een spelentiteit.

Voor zulke gevallen biedt Haskell **mutable referenties** binnen de `IO`-monad. Het sleutelprincipe is dat de variabele altijd zichtbaar is in het type: een functie die werkt met een `IORef` of `MVar` heeft altijd een type dat `IO` bevat, zodat de compiler weet dat er side effects in het spel zijn. Je kunt een `IORef` nooit per ongeluk aanpassen in een pure functie — het type-system verhindert dat.

---

## `IORef` — een eenvoudige mutable cel

`IORef a` is een mutable reference naar een waarde van type `a`. Je maakt één aan met `newIORef`, leest de huidige waarde met `readIORef`, overschrijft de waarde met `writeIORef`, en past de waarde aan met een pure functie via `modifyIORef'`. Al deze operaties leven in de `IO`-monad.

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

Er bestaan twee versies: de strenge versie met apostrof (`modifyIORef'`) en de luie versie zonder (`modifyIORef`). De luie versie bouwt bij herhaald gebruik in een loop een keten van niet-geëvalueerde thunks op, wat memory leaks veroorzaakt. **Gebruik altijd de strenge versie `modifyIORef'`** wanneer je een `IORef` in een loop aanpast.

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

`IORef` is niet threadsafe: als meerdere threads tegelijkertijd een `IORef` lezen en schrijven, kunnen er raceconditions optreden. Voor shared state tussen threads gebruik je `MVar a`, een mutable variabele die ontworpen is voor veilig concurrent access.

Een `MVar` werkt als een doos die ofwel **vol** is (bevat een waarde) ofwel **leeg** is. De operatie `takeMVar` neemt de waarde uit de doos en maakt hem leeg — als de doos al leeg was, blokkeert de aanroepende thread totdat een andere thread er een waarde in zet. `putMVar` zet een waarde in een lege doos — als de doos al vol was, blokkeert de aanroepende thread totdat een andere thread de waarde heeft weggenomen. Dit blocking gedrag maakt `MVar` een natural mutex-mechanisme.

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

---

## Demo: interactieve notitielijst

In deze demo bouwen we stap voor stap een klein programma dat een notitielijst bijhoudt in een `IORef`. De gebruiker kan notities toevoegen, bekijken en verwijderen via een eenvoudig tekstmenu.

---

### Stap 1: De module-header en imports

```haskell
module Main where

import Data.IORef
import Control.Monad (forM_)
```

We importeren `Data.IORef` voor de mutable referentie en `forM_` om de notitielijst genummerd af te drukken.

---

### Stap 2: De `IORef` aanmaken

De notitielijst wordt bewaard in een `IORef [String]`. We maken hem aan in `main` met een lege beginlijst en geven hem door aan alle functies die hem nodig hebben:

```haskell
main :: IO ()
main = do
  notities <- newIORef ([] :: [String])
  menuLoop notities
```

Door de `IORef` als argument door te geven in plaats van hem globaal op te slaan, is altijd zichtbaar welke functies de toestand kunnen aanpassen.

---

### Stap 3: Een notitie toevoegen

`modifyIORef'` past de lijst aan met een pure functie. We voegen de nieuwe notitie achteraan toe met `(++ [tekst])`:

```haskell
voegToe :: IORef [String] -> IO ()
voegToe notities = do
  putStr "Notitie: "
  tekst <- getLine
  modifyIORef' notities (++ [tekst])
  putStrLn "Notitie toegevoegd."
```

---

### Stap 4: De notitielijst tonen en verwijderen

`readIORef` geeft de huidige waarde terug. Met `zip [1..]` voegen we nummers toe aan de lijst:

```haskell
toonAlles :: IORef [String] -> IO ()
toonAlles notities = do
  lijst <- readIORef notities
  if null lijst
    then putStrLn "(Geen notities)"
    else forM_ (zip [1..] lijst) $ \(i, n) ->
      putStrLn (show (i :: Int) ++ ". " ++ n)

verwijderLaatste :: IORef [String] -> IO ()
verwijderLaatste notities = do
  lijst <- readIORef notities
  if null lijst
    then putStrLn "Geen notities om te verwijderen."
    else modifyIORef' notities init >> putStrLn "Laatste notitie verwijderd."
```

---

### Stap 5: De menuloop

De menuloop leest de keuze van de gebruiker en roept via `case` de juiste actie aan. Bij keuze `"4"` stopt de recursie:

```haskell
menuLoop :: IORef [String] -> IO ()
menuLoop notities = do
  putStrLn "\n=== Notitielijst ==="
  putStrLn "1. Notitie toevoegen"
  putStrLn "2. Alle notities tonen"
  putStrLn "3. Laatste notitie verwijderen"
  putStrLn "4. Afsluiten"
  putStr "Keuze: "
  keuze <- getLine
  case keuze of
    "1" -> voegToe notities          >> menuLoop notities
    "2" -> toonAlles notities        >> menuLoop notities
    "3" -> verwijderLaatste notities >> menuLoop notities
    "4" -> putStrLn "Tot ziens!"
    _   -> putStrLn "Ongeldige keuze." >> menuLoop notities
```

---

### Volledig programma

<details closed>
<summary><i><b>Volledig programma — klik om te tonen</b></i>🔽</summary>
<p>

```haskell
module Main where

import Data.IORef
import Control.Monad (forM_)

voegToe :: IORef [String] -> IO ()
voegToe notities = do
  putStr "Notitie: "
  tekst <- getLine
  modifyIORef' notities (++ [tekst])
  putStrLn "Notitie toegevoegd."

toonAlles :: IORef [String] -> IO ()
toonAlles notities = do
  lijst <- readIORef notities
  if null lijst
    then putStrLn "(Geen notities)"
    else forM_ (zip [1..] lijst) $ \(i, n) ->
      putStrLn (show (i :: Int) ++ ". " ++ n)

verwijderLaatste :: IORef [String] -> IO ()
verwijderLaatste notities = do
  lijst <- readIORef notities
  if null lijst
    then putStrLn "Geen notities om te verwijderen."
    else modifyIORef' notities init >> putStrLn "Laatste notitie verwijderd."

menuLoop :: IORef [String] -> IO ()
menuLoop notities = do
  putStrLn "\n=== Notitielijst ==="
  putStrLn "1. Notitie toevoegen"
  putStrLn "2. Alle notities tonen"
  putStrLn "3. Laatste notitie verwijderen"
  putStrLn "4. Afsluiten"
  putStr "Keuze: "
  keuze <- getLine
  case keuze of
    "1" -> voegToe notities          >> menuLoop notities
    "2" -> toonAlles notities        >> menuLoop notities
    "3" -> verwijderLaatste notities >> menuLoop notities
    "4" -> putStrLn "Tot ziens!"
    _   -> putStrLn "Ongeldige keuze." >> menuLoop notities

main :: IO ()
main = do
  notities <- newIORef ([] :: [String])
  menuLoop notities
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
-- .ghci --- global-state

-- Prompt
:set prompt "ghci> "
:set prompt-cont "     | "

-- Warnings
:set -Wall

-- Imports die in de demo en oefeningen gebruikt worden
import Data.IORef
import Control.Concurrent.MVar
import Control.Concurrent.STM
import Control.Concurrent.STM.TVar
```

</p>
</details>

_De gegeven oplossingen zijn EEN mogelijke oplossing, soms zijn meerdere mogelijkheden juist. Is het gewenste gedrag bereikt, dan is je oplossing correct!_

### Oefeningenreeks 1: IORef

- Maak een `IORef Int` aan met beginwaarde `0`. Verhoog hem 5 keer met `modifyIORef'` en druk de eindwaarde af.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Data.IORef

main :: IO ()
main = do
  teller <- newIORef (0 :: Int)
  mapM_ (\_ -> modifyIORef' teller (+1)) [1..5]
  waarde <- readIORef teller
  putStrLn ("Eindwaarde: " ++ show waarde)
``` -->

- Schrijf een functie `telLetters :: IORef Int -> String -> IO ()` die het aantal letters van een string bij de `IORef`-teller optelt. Test hem met de strings `"appel"`, `"peer"` en `"banaan"` en druk het totaal af.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Data.IORef

telLetters :: IORef Int -> String -> IO ()
telLetters teller s = modifyIORef' teller (+ length s)

main :: IO ()
main = do
  teller <- newIORef (0 :: Int)
  mapM_ (telLetters teller) ["appel", "peer", "banaan"]
  totaal <- readIORef teller
  putStrLn ("Totaal letters: " ++ show totaal)
``` -->

- Schrijf een programma dat de gebruiker herhaaldelijk getallen laat invoeren totdat de gebruiker `"stop"` typt. Druk daarna het gemiddelde af. Gebruik een `IORef [Double]` om de getallen bij te houden.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Data.IORef

leesLus :: IORef [Double] -> IO ()
leesLus getallen = do
  putStr "Getal (of 'stop'): "
  invoer <- getLine
  case invoer of
    "stop" -> return ()
    _      -> do
      let n = read invoer :: Double
      modifyIORef' getallen (n :)
      leesLus getallen

main :: IO ()
main = do
  getallen <- newIORef []
  leesLus getallen
  lijst <- readIORef getallen
  if null lijst
    then putStrLn "Geen getallen ingevoerd."
    else putStrLn ("Gemiddelde: " ++ show (sum lijst / fromIntegral (length lijst)))
```

</p>
</details> -->

---

### Oefeningenreeks 2: MVar

- Maak een `MVar String` aan met beginwaarde `"hallo"`. Lees de waarde uit met `readMVar` en druk hem af. Vervang hem daarna via `modifyMVar_` door de string omgezet naar hoofdletters (gebruik `map toUpper` uit `Data.Char`) en druk het resultaat opnieuw af.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Control.Concurrent.MVar
import Data.Char (toUpper)

main :: IO ()
main = do
  v <- newMVar "hallo"
  voor <- readMVar v
  putStrLn ("Voor: " ++ voor)
  modifyMVar_ v (\s -> return (map toUpper s))
  na <- readMVar v
  putStrLn ("Na: " ++ na)
``` -->

- Schrijf een functie `wisselMVar :: MVar a -> MVar a -> IO ()` die de waarden van twee `MVar`s wisselt. Test met twee `MVar Int`s.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Control.Concurrent.MVar

wisselMVar :: MVar a -> MVar a -> IO ()
wisselMVar ma mb = do
  a <- takeMVar ma
  b <- takeMVar mb
  putMVar ma b
  putMVar mb a

main :: IO ()
main = do
  x <- newMVar (1 :: Int)
  y <- newMVar (2 :: Int)
  wisselMVar x y
  a <- readMVar x
  b <- readMVar y
  putStrLn ("x: " ++ show a ++ ", y: " ++ show b)
``` -->

---

### Oefeningenreeks 3: STM en TVar (niet nodig)

- Maak twee `TVar Int` aan met waarden `10` en `20`. Schrijf een `atomically`-blok dat hun som berekent en opslaat in een derde `TVar`. Druk het resultaat af.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Control.Concurrent.STM

main :: IO ()
main = do
  a      <- newTVarIO (10 :: Int)
  b      <- newTVarIO (20 :: Int)
  totaal <- newTVarIO (0  :: Int)
  atomically $ do
    x <- readTVar a
    y <- readTVar b
    writeTVar totaal (x + y)
  resultaat <- readTVarIO totaal
  putStrLn ("Som: " ++ show resultaat)
``` -->

- Implementeer `overschrijving :: TVar Int -> TVar Int -> Int -> STM ()` die een bedrag atomisch van de ene rekening aftrekt en bij de andere optelt. Test met twee rekeningen van elk `500` en een overschrijving van `150`.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

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
  rekeningA <- newTVarIO (500 :: Int)
  rekeningB <- newTVarIO (500 :: Int)
  atomically (overschrijving rekeningA rekeningB 150)
  a <- readTVarIO rekeningA
  b <- readTVarIO rekeningB
  putStrLn ("A: " ++ show a ++ ", B: " ++ show b)
```

</p>
</details> -->
