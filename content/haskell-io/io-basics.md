---
title: "I/O programmeren in Haskell"
weight: 1
draft: false
---

## Het `IO`-type

Haskell is een **pure** taal: een functie met type `a -> b` heeft geen neveneffecten. Ze mag geen invoer lezen, geen uitvoer schrijven, geen bestanden openen en geen global state aanpassen. Dit is een fundamentele eigenschap van Haskell die correctness en testbaarheid sterk vereenvoudigt. Toch moeten zinvolle programma's natuurlijk wel iets kunnen uitvoeren. Haskell lost dit op met het `IO`-type.

Een waarde van het type `IO a` is een **actie**: een beschrijving van berekeningen met neveneffecten die, wanneer uitgevoerd, een resultaat van type `a` oplevert. De cruciale nuance is dat een waarde van type `IO a` op zichzelf niets *doet*, ze beschrijft enkel wat er zou moeten gebeuren. De Haskell-runtime voert precies één actie uit: de actie die aan `main` gebonden is. Alle andere I/O-acties worden via `main` aangeroepen.

```haskell
main :: IO ()
main = putStrLn "Hallo, wereld!"
-- `print x` is equivalent aan `putStrLn (show x)`
```

Het type `IO ()` betekent: een actie die side effects heeft en als resultaat de eenheidwaarde `()` returned (niets nuttig). Je kunt `IO` beschouwen als een container die de side effect-informatie bijhoudt, en via het type-system weet de compiler altijd of een functie side effects kan hebben of niet.

---

## `do`-notatie

Op zichzelf beschrijven `IO`-waarden slechts één actie. Om meerdere acties te combineren tot een sequentie, gebruik je `do`-notatie. `do` is syntactische suiker voor de bind-operator `>>=` van de monad, maar het leest als imperatieve code. Elke regel in een `do`-blok beschrijft één stap in de uitvoering.

```haskell
main :: IO ()
main = do
  putStr "Geef je naam in: "
  naam <- getLine
  putStrLn ("Hallo, " ++ naam ++ "!")
```

In dit voorbeeld wordt `getLine` uitgevoerd, en het resultaat — de ingetypte string — wordt gebonden aan de naam `naam` via de `<-`-operator. Let op het verschil met een gewone `let`-binding: `<-` haalt een waarde uit een `IO`-actie, terwijl `let` een puur berekend resultaat bindt zonder neveneffecten!

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

De module `System.IO` (deels automatisch beschikbaar via `Prelude`) biedt de basis primitives voor input en output. Voor output gebruik je `putStr` om te schrijven zonder regelafbreking, `putStrLn` om te schrijven met een afsluitende newline, en `print` om willekeurige waarden met een `Show`-instantie af te drukken. `print` is equivalent aan `putStrLn . show`, maar voegt aanhalingstekens toe rond strings:

```haskell
putStr   :: String -> IO ()
putStrLn :: String -> IO ()
print    :: Show a => a -> IO ()
```

Voor invoer gebruik je `getLine` om een volledige regel te lezen, `getChar` voor één karakter, of `readLn` om een regel te lezen en meteen te parsen naar het gewenste type. Met `readLn` kan je zo een getal direct inlezen:

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

Het pad dat je meegeeft aan `readFile`, `writeFile` en `appendFile` is relatief ten opzichte van de **huidige werkdirectory**. Als je je programma start met `cabal run` of `stack run`, is dat de **root van je project** (de map met het `.cabal`-bestand). Als je het native executable rechtstreeks aanroept, is het de map van waaruit je de terminal gebruikt.

Merk op dat `readFile` **lui** is: de inhoud wordt pas ingelezen wanneer die effectief gebruikt wordt. Dit kan problemen geven als je een bestand wil overschrijven na het te lezen. In die gevallen gebruik je `withFile` met een expliciete handle. `withFile` opent het bestand, geeft een `Handle` door aan een callback-functie, en sluit het bestand gegarandeerd achteraf — ook bij een fout:

```haskell
import System.IO

kopieerBestand :: FilePath -> FilePath -> IO ()
kopieerBestand van naar =
  withFile van ReadMode $ \invoerHandle ->
    withFile naar WriteMode $ \uitvoerHandle -> do
      inhoud <- hGetContents invoerHandle
      hPutStr uitvoerHandle inhoud

main :: IO ()
main = do
  kopieerBestand "invoer.txt" "uitvoer.txt"
  putStrLn "Klaar."
```

`hGetContents`, `hPutStr` en `hPutStrLn` zijn de handle-varianten van `getContents`, `putStr` en `putStrLn`. De `ReadMode`/`WriteMode`/`AppendMode`-waarden bepalen hoe het bestand geopend wordt.

---

{{% notice info %}}
`$` past een functie toe op een argument (haakjes vermijden); `.` stelt twee functies samen tot één nieuwe functie (geen argument nodig).<br/>
`print $ show 42        -- $ : pas print toe op (show 42)`<br/>
`(print . show) 42      -- . : combineer print en show, pas toe op 42`<br/><br/>
`\` is de lambda-notatie in Haskell - het introduceert een anonieme functie. De syntax is:<br/>
`\argument -> expressie`
{{% /notice %}}

## Loops in I/O

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

Soms wil je niet alleen side effects uitvoeren, maar ook de resultaten opvangen. `replicateM` voert een actie een bepaald aantal keren uit en verzamelt de resultaten in een lijst. `mapM` doet hetzelfde voor een lijst van waarden:

```haskell
import Control.Monad (replicateM)

leesGetallen :: Int -> IO [Int]
leesGetallen n = replicateM n (do
  putStr "Getal: "
  readLn)

main :: IO ()
main = do
  getallen <- leesGetallen 3
  putStrLn ("Ingevoerde getallen: " ++ show getallen)
  putStrLn ("Som: " ++ show (sum getallen))
```

---

## `return` en `pure`

De functie `return` (synoniem: `pure`) is een manier om een gewone waarde in de `IO`-monade te tillen, zonder enig side effect. Ze *stopt* de uitvoering niet — dat is een veelgemaakte verwarring voor programmeurs die komen van talen als C of Java. In Haskell is `return 42` gewoon een `IO`-actie die bij uitvoering de waarde `42` produceert zonder iets te doen:

```haskell
begroet :: String -> IO String
begroet naam = return ("Hallo, " ++ naam)

main :: IO ()
main = do
  bericht <- begroet "Alice"
  putStrLn bericht
```

---

## Error handling

Voor het opvangen van runtime-errors biedt `Control.Exception` de `try`-functie. Die probeert een `IO`-actie uit te voeren en geeft een `Either` terug: `Left error` als er een uitzondering optrad, `Right value` als alles goed verliep:

```haskell
import Control.Exception

main :: IO ()
main = do
  resultaat <- try (readFile "bestaat-niet.txt") :: IO (Either SomeException String)
  case resultaat of
    Left fout    -> putStrLn ("Fout: " ++ show fout)
    Right inhoud -> putStrLn ("Gelezen: " ++ take 100 inhoud)
```

---

## Demo: een interactieve rekenmachine
In deze demo bouwen we stap voor stap een klein interactief programma dat twee getallen van de gebruiker inleest, een bewerking vraagt en het resultaat afdrukt. Onderweg komen alle concepten uit dit hoofdstuk aan bod.

---

### Stap 1: De module-header en imports

```haskell
module Main where

import Control.Monad (when)
import Control.Exception (try, SomeException)
```

We importeren `when` voor conditionele I/O en `try` voor foutafhandeling bij het parsen van invoer.

---

### Stap 2: Een getal inlezen met errorhandling

`readLn` gooit een exception als de user geen geldig getal intypt. We vangen dit op met `try` en vragen opnieuw als het mislukt:

```haskell
leesGetal :: String -> IO Double
leesGetal prompt = do
  putStr prompt
  resultaat <- try readLn :: IO (Either SomeException Double)
  case resultaat of
    Left _     -> do
      putStrLn "Ongeldige invoer, probeer opnieuw."
      leesGetal prompt
    Right getal -> return getal
```

`leesGetal` is recursief: bij een fout roept het zichzelf opnieuw aan. Dit is de idiomatische Haskell-manier om een while-loop te schrijven zonder `while`.

---

### Stap 3: De bewerking kiezen

```haskell
leesBewerking :: IO Char
leesBewerking = do
  putStr "Bewerking (+, -, *, /): "
  invoer <- getLine
  case invoer of
    [c] | c `elem` "+-*/" -> return c
    _ -> do
      putStrLn "Ongeldige bewerking, kies uit +, -, *, /."
      leesBewerking
```

`case invoer of [c] | c 'elem' "+-*/"` matcht enkel als de invoer exact één karakter is dat in de lijst voorkomt. Anders wordt opnieuw gevraagd.

---

### Stap 4: De berekening uitvoeren

Een pure functie zonder enige I/O, zodat de logica apart testbaar is:

```haskell
bereken :: Double -> Char -> Double -> Either String Double
bereken _ '/' 0 = Left "Deling door nul is niet toegestaan."
bereken a op  b = Right $ case op of
  '+' -> a + b
  '-' -> a - b
  '*' -> a * b
  '/' -> a / b
  _   -> 0
```

We returnen een `Either String Double`: `Left` voor een foutmelding, `Right` voor het resultaat. Door de logica puur te houden, is testen eenvoudig.

---

### Stap 5: Alles samenvoegen in `main`

```haskell
main :: IO ()
main = do
  putStrLn "=== Interactieve rekenmachine ==="
  a  <- leesGetal "Eerste getal: "
  b  <- leesGetal "Tweede getal: "
  op <- leesBewerking
  case bereken a op b of
    Left fout      -> putStrLn ("Fout: " ++ fout)
    Right resultaat -> putStrLn ("Resultaat: " ++ show resultaat)
  putStr "Nog een berekening? (j/n): "
  antwoord <- getLine
  when (antwoord == "j") main
```

De laatste drie regels implementeren de herhaling: als de gebruiker `"j"` intypt, roept `main` zichzelf aan. `when` voert de actie enkel uit als de voorwaarde `True` is.

---

### Volledig programma

<details closed>
<summary><i><b>Volledig programma — klik om te tonen</b></i>🔽</summary>
<p>

```haskell
module Main where

import Control.Monad (when)
import Control.Exception (try, SomeException)

leesGetal :: String -> IO Double
leesGetal prompt = do
  putStr prompt
  resultaat <- try readLn :: IO (Either SomeException Double)
  case resultaat of
    Left _      -> do
      putStrLn "Ongeldige invoer, probeer opnieuw."
      leesGetal prompt
    Right getal -> return getal

leesBewerking :: IO Char
leesBewerking = do
  putStr "Bewerking (+, -, *, /): "
  invoer <- getLine
  case invoer of
    [c] | c `elem` "+-*/" -> return c
    _ -> do
      putStrLn "Ongeldige bewerking, kies uit +, -, *, /."
      leesBewerking

bereken :: Double -> Char -> Double -> Either String Double
bereken _ '/' 0 = Left "Deling door nul is niet toegestaan."
bereken a op  b = Right $ case op of
  '+' -> a + b
  '-' -> a - b
  '*' -> a * b
  '/' -> a / b
  _   -> 0

main :: IO ()
main = do
  putStrLn "=== Interactieve rekenmachine ==="
  a  <- leesGetal "Eerste getal: "
  b  <- leesGetal "Tweede getal: "
  op <- leesBewerking
  case bereken a op b of
    Left fout       -> putStrLn ("Fout: " ++ fout)
    Right resultaat -> putStrLn ("Resultaat: " ++ show resultaat)
  putStr "Nog een berekening? (j/n): "
  antwoord <- getLine
  when (antwoord == "j") main
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
-- .ghci --- io-basics

-- Prompt
:set prompt "ghci> "
:set prompt-cont "     | "

-- Warnings
:set -Wall

-- Imports die in de demo en oefeningen gebruikt worden
import Control.Monad (when, unless, forM_, replicateM)
import Control.Exception (try, SomeException)
import System.IO
```

</p>
</details>

_De gegeven oplossingen zijn EEN mogelijke oplossing, soms zijn meerdere mogelijkheden juist. Is het gewenste gedrag bereikt, dan is je oplossing correct!_

### Oefeningenreeks 1: Basisfuncties

- Schrijf een `IO`-actie `begroet` die de gebruiker om zijn naam vraagt en daarna `"Hallo, <naam>!"` afdrukt.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
begroet :: IO ()
begroet = do
  putStr "Naam: "
  naam <- getLine
  putStrLn ("Hallo, " ++ naam ++ "!")
``` -->

- Schrijf een functie `drukLijstAf :: [String] -> IO ()` die elk element op een aparte regel afdrukt met `forM_`.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
import Control.Monad (forM_)

drukLijstAf :: [String] -> IO ()
drukLijstAf xs = forM_ xs putStrLn
``` -->

- Schrijf een `IO`-actie die drie getallen van de gebruiker inleest (met `readLn :: IO Int`) en hun som afdrukt.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
main :: IO ()
main = do
  putStr "Getal 1: "
  a <- readLn :: IO Int
  putStr "Getal 2: "
  b <- readLn :: IO Int
  putStr "Getal 3: "
  c <- readLn :: IO Int
  putStrLn ("Som: " ++ show (a + b + c))
```

</p>
</details> -->

---

### Oefeningenreeks 2: Loops

- Schrijf een programma dat de gebruiker herhaaldelijk om een getal vraagt en het dubbele afdrukt, totdat de gebruiker `0` invoert.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Monad (when)

loop :: IO ()
loop = do
  putStr "Getal (0 om te stoppen): "
  n <- readLn :: IO Int
  when (n /= 0) $ do
    putStrLn ("Dubbel: " ++ show (n * 2))
    loop

main :: IO ()
main = loop
```

</p>
</details> -->

- Gebruik `replicateM` om exact 4 namen van de gebruiker in te lezen en die daarna allemaal samen af te drukken.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Monad (replicateM, forM_)

main :: IO ()
main = do
  namen <- replicateM 4 $ do
    putStr "Naam: "
    getLine
  putStrLn "Ingevoerde namen:"
  forM_ namen putStrLn
```

</p>
</details> -->

---

### Oefeningenreeks 3: Bestanden

- Schrijf een programma dat de inhoud van een bestand `invoer.txt` inleest en het aantal regels afdrukt.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
main :: IO ()
main = do
  inhoud <- readFile "invoer.txt"
  print (length (lines inhoud))
``` -->

- Schrijf een programma dat de gebruiker om vijf regels tekst vraagt en die wegschrijft naar `uitvoer.txt`.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Monad (replicateM)

main :: IO ()
main = do
  regels <- replicateM 5 $ do
    putStr "> "
    getLine
  writeFile "uitvoer.txt" (unlines regels)
  putStrLn "Opgeslagen in uitvoer.txt."
```

</p>
</details> -->

- Schrijf een programma dat een bestand `log.txt` probeert te lezen. Als het bestand niet bestaat, druk dan `"Logbestand niet gevonden."` af. Als het wel bestaat, druk dan de laatste regel af.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Exception (try, SomeException)

main :: IO ()
main = do
  resultaat <- try (readFile "log.txt") :: IO (Either SomeException String)
  case resultaat of
    Left _       -> putStrLn "Logbestand niet gevonden."
    Right inhoud ->
      let regels = lines inhoud
      in if null regels
           then putStrLn "Bestand is leeg."
           else putStrLn ("Laatste regel: " ++ last regels)
```

</p>
</details> -->

---

### Oefeningenreeks 4: Pure logica scheiden van I/O

- Schrijf een pure functie `somLijst :: [Int] -> Int` die de som van een lijst berekent. Schrijf daarna een `IO`-actie die een getal `n` inleest, vervolgens `n` getallen inleest en de som via `somLijst` afdrukt.
<!-- EXSOL -->
<!-- <details closed>
<summary><i><b><span style="color: #03C03C;">Solution:</span> Klik hier om de code te zien/verbergen</b></i>🔽</summary>
<p>

```haskell
import Control.Monad (replicateM)

somLijst :: [Int] -> Int
somLijst = sum

main :: IO ()
main = do
  putStr "Hoeveel getallen? "
  n <- readLn :: IO Int
  getallen <- replicateM n $ do
    putStr "  Getal: "
    readLn :: IO Int
  putStrLn ("Som: " ++ show (somLijst getallen))
```

</p>
</details> -->

- Schrijf een pure functie `palindroom :: String -> Bool` die controleert of een string een palindroom is. Schrijf dan een `IO`-actie die de gebruiker een woord laat invoeren en het resultaat afdrukt.
<!-- EXSOL -->
<!-- _**<span style="color: #03C03C;">Solution:</span>**_
```haskell
palindroom :: String -> Bool
palindroom s = s == reverse s

main :: IO ()
main = do
  putStr "Woord: "
  woord <- getLine
  if palindroom woord
    then putStrLn "Is een palindroom."
    else putStrLn "Is geen palindroom."
``` -->
