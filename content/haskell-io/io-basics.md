---
title: "I/O programmeren in Haskell"
weight: 1
draft: false
---

## Het `IO`-type

Haskell is een **pure** taal: een functie met type `a -> b` heeft geen neveneffecten. Ze mag geen invoer lezen, geen uitvoer schrijven, geen bestanden openen en geen global state aanpassen. Dit is een fundamentele eigenschap van Haskell die correctheidsredenering en testbaarheid sterk vereenvoudigt. Toch moeten zinvolle programma's natuurlijk wel iets kunnen uitvoeren. Haskell lost dit op met het `IO`-type.

Een waarde van het type `IO a` is een **actie**: een beschrijving van berekeningen met neveneffecten die, wanneer uitgevoerd, een resultaat van type `a` oplevert. De cruciale nuance is dat een waarde van type `IO a` op zichzelf niets *doet* — ze beschrijft enkel wat er zou moeten gebeuren. De Haskell-runtime voert precies één actie uit: de actie die aan `main` gebonden is. Alle andere I/O-acties worden via `main` aangeroepen.

```haskell
main :: IO ()
main = putStrLn "Hallo, wereld!"
```

Het type `IO ()` betekent: een actie die neveneffecten heeft en als resultaat de eenheidwaarde `()` retourneert (niets nuttig). Je kunt `IO` beschouwen als een container die de neveneffect-informatie bijhoudt, en via het type-systeem weet de compiler altijd of een functie neveneffecten kan hebben of niet.

---

## `do`-notatie

Op zichzelf beschrijven `IO`-waarden slechts één actie. Om meerdere acties te combineren tot een sequentie, gebruik je `do`-notatie. `do` is syntactische suiker voor de bind-operator `>>=` van de monade, maar het leest als imperatieve code. Elke regel in een `do`-blok beschrijft één stap in de uitvoering.

```haskell
main :: IO ()
main = do
  putStr "Geef je naam in: "
  naam <- getLine
  putStrLn ("Hallo, " ++ naam ++ "!")
```

In dit voorbeeld wordt `getLine` uitgevoerd, en het resultaat — de ingetypte string — wordt gebonden aan de naam `naam` via de `<-`-operator. Let op het verschil met een gewone `let`-binding: `<-` haalt een waarde uit een `IO`-actie, terwijl `let` een puur berekend resultaat bindt zonder neveneffecten.

De regels in een `do`-blok volgen een vaste structuur. Acties die enkel uitgevoerd worden voor hun neveneffect (zoals `putStrLn`) staan op een eigen regel zonder binding. Acties waarvan je de resultaatwaarde verder wil gebruiken, bind je met `<-`. Pure berekeningen die geen I/O vereisen bind je met `let`.

```haskell
main :: IO ()
main = do
  let begroeting = "Hallo"          -- puur, geen I/O
  naam <- getLine                   -- I/O: resultaat binden
  putStrLn (begroeting ++ ", " ++ naam ++ "!")  -- I/O: neveneffect, geen binding
```

---

## Standaard I/O-functies

De module `System.IO` (deels automatisch beschikbaar via `Prelude`) biedt de basisprimitieven voor invoer en uitvoer. Voor uitvoer gebruik je `putStr` om te schrijven zonder regelafbreking, `putStrLn` om te schrijven met een afsluitende newline, en `print` om willekeurige waarden met een `Show`-instantie af te drukken. `print` is equivalent aan `putStrLn . show`, maar voegt aanhalingstekens toe rond strings:

```haskell
putStr   :: String -> IO ()
putStrLn :: String -> IO ()
print    :: Show a => a -> IO ()
```

Voor invoer gebruik je `getLine` om een volledige regel te lezen, `getChar` voor één karakter, of `readLn` om een regel te lezen en meteen te parseren naar het gewenste type. Met `readLn` kan je zo een getal direct inlezen:

```haskell
main :: IO ()
main = do
  putStr "Geef een getal: "
  n <- readLn :: IO Int
  putStrLn ("Het dubbele is: " ++ show (n * 2))
```

---

## Bestandsoperaties

Haskell biedt handige hoog-niveau functies voor eenvoudige bestandsoperaties. `readFile` leest de volledige inhoud van een bestand als een `String`, `writeFile` schrijft een `String` naar een bestand (overschrijft bestaande inhoud) en `appendFile` voegt tekst toe aan het einde van een bestand. Voor meer controle gebruik je `withFile` met een bestandshandle:

```haskell
import System.IO

aantalRegels :: FilePath -> IO Int
aantalRegels pad = do
  inhoud <- readFile pad
  return (length (lines inhoud))

main :: IO ()
main = do
  n <- aantalRegels "data.txt"
  putStrLn ("Aantal regels: " ++ show n)
```

Merk op dat `readFile` **lui** is: de inhoud wordt pas ingelezen wanneer die effectief gebruikt wordt. Dit kan problemen geven als je een bestand wil overschrijven na het te lezen. In die gevallen is het beter om `withFile` te gebruiken met expliciete handles, of `Data.Text.IO` voor strikte tekstverwerking.

---

## Herhaling in I/O

Haskell heeft geen imperatieve `for`- of `while`-loops, maar herhaling in I/O-context wordt uitgedrukt via recursie of via combinatoren uit `Control.Monad`. De functie `forM_` itereert over een lijst en voert een I/O-actie uit voor elk element:

```haskell
import Control.Monad (forM_, forever, when, unless)

main :: IO ()
main = forM_ [1..5] $ \i ->
  putStrLn ("Item " ++ show i)
```

Voor een oneindige loop gebruik je `forever`, dat een actie herhaalt tot het programma gestopt wordt:

```haskell
echoProgramma :: IO ()
echoProgramma = forever $ do
  regel <- getLine
  putStrLn ("Je schreef: " ++ regel)
```

De functies `when` en `unless` zijn handig om een actie conditioneel uit te voeren. Ze nemen een `Bool` als eerste argument en voeren de actie enkel uit als die `Bool` respectievelijk `True` of `False` is. Zonder `when` zou je steeds een `if`-expressie nodig hebben met een lege `else`-tak:

```haskell
main :: IO ()
main = do
  invoer <- readLn :: IO Int
  when   (invoer > 0) $ putStrLn "Positief getal"
  unless (invoer > 0) $ putStrLn "Nul of negatief"
```

---

## Resultaten verzamelen

Soms wil je niet alleen neveneffecten uitvoeren, maar ook de resultaten collecteren. `replicateM` voert een actie een bepaald aantal keren uit en verzamelt de resultaten in een lijst. `mapM` doet hetzelfde voor een lijst van waarden:

```haskell
import Control.Monad (replicateM)

leesGetallen :: Int -> IO [Int]
leesGetallen n = replicateM n (do
  putStr "Getal: "
  readLn)
```

---

## `return` en `pure`

De functie `return` (synoniem: `pure`) is een manier om een gewone waarde in de `IO`-monade te tillen, zonder enig neveneffect. Ze *stopt* de uitvoering niet — dat is een veelgemaakte verwarring voor programmeurs die komen van talen als C of Java. In Haskell is `return 42` gewoon een `IO`-actie die bij uitvoering de waarde `42` produceert zonder iets te doen:

```haskell
begroet :: String -> IO String
begroet naam = return ("Hallo, " ++ naam)

main :: IO ()
main = do
  bericht <- begroet "Alice"
  putStrLn bericht
```

---

## Foutafhandeling

Voor het opvangen van runtime-fouten biedt `Control.Exception` de `try`-functie. Die probeert een `IO`-actie uit te voeren en geeft een `Either` terug: `Left fout` als er een uitzondering optrad, `Right waarde` als alles goed verliep:

```haskell
import Control.Exception

main :: IO ()
main = do
  resultaat <- try (readFile "bestaat-niet.txt") :: IO (Either SomeException String)
  case resultaat of
    Left fout    -> putStrLn ("Fout: " ++ show fout)
    Right inhoud -> putStrLn ("Gelezen: " ++ take 100 inhoud)
```
