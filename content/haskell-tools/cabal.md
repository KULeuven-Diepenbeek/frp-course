---
title: "Cabal: Build-tool en Dependency Manager"
weight: 2
draft: false
---

## Wat is Cabal?

**Cabal** (Common Architecture for Building Applications and Libraries) is het standaard build-systeem en de package manager voor Haskell. Het vervult twee rollen tegelijkertijd: enerzijds is het een **build-tool** die weet hoe je Haskell-modules compileert, uitvoerbare bestanden linkt en tests uitvoert; anderzijds is het een **dependency manager** die de libraries die je project nodig heeft automatisch ophaalt van [Hackage](https://hackage.haskell.org/), de centrale Haskell package repository, en ze bouwt voordat je eigen project gecompileerd wordt.

Moderne Cabal (versie 3.0 en hoger) werkt met een **per-project dependency store** in `~/.cabal/store`. Dit betekent dat verschillende projecten op je machine zonder conflicten verschillende versies van dezelfde library kunnen gebruiken. Je hoeft nooit manueel versieconflicten op te lossen zoals dat bij sommige andere ecosystemen het geval is.

---

## Cabal installeren

Cabal wordt meegeleverd met de [GHCup](https://www.haskell.org/ghcup/) toolchain-installer, de aanbevolen manier om Haskell op je machine te zetten. Eenmaal GHCup geïnstalleerd, installeer je de nieuwste versie van Cabal met:

```bash
ghcup install cabal latest
```

Controleer daarna of de installatie gelukt is:

```bash
cabal --version
```

Dit zou iets moeten uitvoeren als `cabal-install version 3.12.x.x`. Als dit werkt, ben je klaar om je eerste project aan te maken.

---

## Een nieuw project aanmaken

Cabal biedt een interactieve wizard om snel een nieuw project op te zetten. Navigeer naar de directory waar je het project wil aanmaken en voer uit:

```bash
cabal init --interactive
```

De wizard stelt je een reeks vragen: de projectnaam, het type (uitvoerbaar programma, bibliotheek of beide), welke GHC-versie je wil gebruiken en welke licentie je project krijgt. Na het doorlopen van de wizard heb je een basisstructuur:

```
mijn-project/
  mijn-project.cabal      -- projectbeschrijvingsbestand
  app/
    Main.hs               -- ingangspunt voor het uitvoerbare programma
  src/                    -- broncode voor de library (indien van toepassing)
  CHANGELOG.md
```

{{% notice warning %}}
Opgelet, cabal projecten kunnen moeilijk overweg met directories waarvan het absolute pad spaties of speciale tekens bevatten. Je kan dus eventueel een folder rechtstreeks op je C-schijf aanmaken voor deze projecten.
{{% /notice %}}

---

## Het `.cabal`-bestand

Het `.cabal`-bestand is het hart van elk Cabal-project. Het beschrijft de metadata van de package, de source directory's, de language extensions, de dependencies en de build-targets. Het is een declaratief bestand: je beschrijft *wat* je project is en *waarvan* het afhangt, en Cabal lost de rest op.

### Package metadata

Bovenaan het bestand staat de algemene informatie over het package:

```cabal
cabal-version:  3.4
name:           mijn-project
version:        0.1.0.0
synopsis:       Een korte beschrijving
description:    Een langere beschrijving van het package.
license:        MIT
author:         Jouw Naam
maintainer:     jij@voorbeeld.com
build-type:     Simple
```

### Een executable declareren

De meeste projecten die we gaan maken zijn executables. Je declareert ze met het keyword `executable`:

```cabal
executable mijn-project
  main-is:          Main.hs
  hs-source-dirs:   app
  build-depends:
      base          ^>=4.18
    , text          ^>=2.0
    , containers    ^>=0.6
  default-language: Haskell2010
  ghc-options:      -Wall
```

De belangrijkste velden zijn: `main-is` (het `.hs`-bestand dat de `main`-functie bevat), `hs-source-dirs` (de directory's waar GHC naar bronbestanden zoekt), `build-depends` (de lijst met vereiste packages), `default-language` (de Haskell-standaard) en `ghc-options` (flags die rechtstreeks aan GHC worden doorgegeven).

### Een library declareren

Als je code wil schrijven die door andere projecten hergebruikt kan worden, declareer je een library:

```cabal
library
  exposed-modules:
      MijnLib
    , MijnLib.Hulpfuncties
  hs-source-dirs:   src
  build-depends:
      base          ^>=4.18
  default-language: Haskell2010
```

Het veld `exposed-modules` somt op welke modules publiek toegankelijk zijn voor andere projecten. Modules die je niet vermeldt zijn interne implementatiedetails.

### Tests declareren

Goede projecten hebben tests. Je voegt ze toe als een aparte `test-suite`:

```cabal
test-suite mijn-project-test
  type:             exitcode-stdio-1.0
  main-is:          Spec.hs
  hs-source-dirs:   test
  build-depends:
      base          ^>=4.18
    , mijn-project
    , hspec         ^>=2.11
  default-language: Haskell2010
```

### Language extensions centraliseren

In plaats van `{-# LANGUAGE ... #-}`-pragma's bovenaan elk bronbestand te herhalen, kun je language extensions eenmalig declareren in het `.cabal`-bestand via `default-extensions`. Dit geldt dan voor alle modules in dat build-target:

```cabal
executable mijn-project
  ...
  default-extensions:
      OverloadedStrings
    , ScopedTypeVariables
    , Arrows
```

---

## Versieconstraints

Cabal gebruikt een beknopte syntaxis om aan te geven welke versies van een afhankelijkheid compatibel zijn. De aanbevolen operator is het dakje `^>=`, ook wel de **caret operator** genoemd. Deze drukt uit "minimaal deze versie en elke compatibele minorversie erna", wat in de praktijk betekent: dezelfde hoofdversie, hogere patchversies zijn toegestaan.

```cabal
build-depends:
    base     ^>=4.18      -- versie >= 4.18 en < 5
  , text     ^>=2.0       -- versie >= 2.0 en < 3
  , Yampa    ^>=0.14      -- versie >= 0.14 en < 0.15 (of < 1 voor Yampa)
```

De caret operator is voor de meeste gevallen de beste keuze. Het is streng genoeg om onverwachte breaking changes te voorkomen, maar soepel genoeg om automatische bugfixupdates toe te staan.

---

## Veelgebruikte Cabal-commando's

Voor de dagelijkse ontwikkelcyclus gebruik je een beperkt aantal Cabal-commando's. Eerst werk je de lokale package index bij - dit is een snelle kopie van wat er op Hackage beschikbaar is. Daarna bouw je het project, draai je het of open je een interactieve sessie:

```bash
# Package index verversen (doe dit periodiek)
cabal update

# Project bouwen
cabal build

# Bouwen en uitvoeren
cabal run

# Interactieve GHCi-sessie met het project geladen
cabal repl
```

Een dependency toevoegen is eenvoudig: voeg de package name toe aan het `build-depends`-veld in je `.cabal`-bestand en voer dan `cabal build` uit. Cabal detecteert de nieuwe dependency, downloadt ze van Hackage en compileert ze vóór je eigen project. Als voorbeeld, om Yampa toe te voegen:

```cabal
build-depends:
    base   ^>=4.18
  , Yampa  ^>=0.14
```

Zoek je een pakket maar weet je de exacte naam niet, dan kun je in de terminal zoeken of de Hackage-website raadplegen:

```bash
cabal list yampa
cabal info Yampa
```

Tests uitvoeren doe je met:

```bash
cabal test
```

---

## Reproduceerbare builds met `cabal freeze`

Wanneer je een project deelt met anderen of automatisch laat bouwen op een CI-server, is het belangrijk dat iedereen exact dezelfde versies gebruikt. Het commando `cabal freeze` schrijft een `cabal.project.freeze`-bestand met de exacte versie van elke directe en transitieve dependency die door de solver gekozen werd:

```bash
cabal freeze
```

Commit dit bestand mee naar je versiebeheersysteem. Iedereen die daarna het project kloont en `cabal build` uitvoert, zal identieke dependencies gebruiken.

---

## Projectstructuur voor deze cursus

Een typisch Haskell-project dat Yampa gebruikt ziet er als volgt uit:

```
frp-spel/
  frp-spel.cabal
  cabal.project.freeze
  app/
    Main.hs           -- reactimate-lus, ingangspunt
  src/
    Spel/
      Logica.hs       -- signalfuncties
      Rendering.hs    -- uitvoer via IO
```

Het bijbehorende `.cabal`-bestand:

```cabal
cabal-version: 3.4
name:          frp-spel
version:       0.1.0.0
build-type:    Simple

executable frp-spel
  main-is:          Main.hs
  hs-source-dirs:   app, src
  build-depends:
      base   ^>=4.18
    , Yampa  ^>=0.14
    , time   ^>=1.12
  default-language:   Haskell2010
  default-extensions: OverloadedStrings
                       ScopedTypeVariables
                       Arrows
  ghc-options:        -Wall -O2
```

## Demo greeter app

In deze demo bouwen we stap voor stap een klein multi-package project. Het bestaat uit twee packages:

- **`greeter`** — een library met herbruikbare functies, plus een eigen executable
- **`greeter-web`** — een tweede executable die de `greeter`-library gebruikt

Zo zie je in de praktijk hoe je code verdeelt tussen packages via een local `cabal.project`.

---

### Stap 1: Projectstructuur aanmaken

Maak de volgende mappenstructuur aan:

```
greeter-demo/
  cabal.project
  greeter/
    greeter.cabal
    src/
      Hello.hs
      Greeter.hs
    exe/
      Main.hs
    test/
      Main.hs
      HelloTest.hs
  greeter-web/
    greeter-web.cabal
    src/
      Main.hs
```

---

### Stap 2: `cabal.project` — de overkoepelende configuratie

Maak `greeter-demo/cabal.project` aan. Dit bestand vertelt Cabal dat beide packages samen één project vormen. Zo kan `greeter-web` de lokale `greeter`-library gebruiken zonder dat die op Hackage gepubliceerd is.

```haskell
packages:
  greeter/
  greeter-web/
```

Voer alle `cabal`-commando's uit vanuit deze `greeter-demo/`-root.

---

### Stap 3: `greeter/greeter.cabal`

Dit `.cabal`-bestand declareert drie build-targets: een library, een executable en een test-suite. Let op de `common warnings`-stanza: daarmee definieer je gedeelde opties die je daarna met `import` hergebruikt in elk target.

```haskell
cabal-version: 3.0
name:          greeter
version:       0.1.0.0
license:       Apache-2.0
build-type:    Simple

common warnings
    ghc-options: -Wall

library
    import:           warnings
    exposed-modules:  Greeter, Hello
    build-depends:    base >=4.17 && <4.20
                    , titlecase
    hs-source-dirs:   src
    default-language: Haskell2010

executable greet
    build-depends:    base >=4.17 && <4.20
                    , greeter
    hs-source-dirs:   exe
    default-language: Haskell2010
    main-is:          Main.hs

test-suite hello-test
    type:             exitcode-stdio-1.0
    hs-source-dirs:   test
    other-modules:    HelloTest
    main-is:          Main.hs
    build-depends:    base, hspec, greeter
    default-language: Haskell2010
```

Merk op dat de library de externe package `titlecase` nodig heeft. Cabal haalt die automatisch op van Hackage bij `cabal build`.

---

### Stap 4: `greeter/src/Hello.hs`

De eenvoudigste module: één functie die een naam omsluit in een begroeting.

```haskell
module Hello where

hello :: String -> String
hello s = "hello, " <> s <> "!"
-- <> is de operator van de Semigroup-typeclass en is een generalisatie voor ++ voor lijsten.
```

---

### Stap 5: `greeter/src/Greeter.hs`

De `Greeter`-module importeert `Hello` en de externe `titlecase`-library om de begroeting in titelnotatie om te zetten. De functie `greet` is opgebouwd via functiecompositie met `.` - *(f . g) x = f (g x)*:

```haskell
module Greeter where

import Data.Text.Titlecase
import Hello

greet :: String -> String
greet = titlecase . hello
```

`titlecase . hello` betekent: pas eerst `hello` toe op de input, daarna `titlecase` op het resultaat. Dus `greet "alice"` geeft `"Hello, Alice!"`.

---

### Stap 6: `greeter/exe/Main.hs`

De executable leest commandoregelargumenten uit met `getArgs` en begroet elk argument op een aparte regel:

```haskell
module Main where

import System.Environment
import Greeter

main :: IO ()
main = mapM_ (putStrLn . greet) =<< getArgs
```

`=<< getArgs` haalt de lijst van argumenten op uit IO en geeft die door aan `mapM_`. Voor elk argument wordt `greet` toegepast en het resultaat afgedrukt.

Je kunt de executable starten met:

```bash
cabal run greet -- alice bob
```

{{% notice warning %}}
De `--` is verplicht om Cabal te vertellen dat alles wat erna komt als argumenten voor het programma bedoeld is, en niet als opties voor `cabal run` zelf. Zonder `--` zou Cabal `alice` proberen te interpreteren als een eigen vlag.
{{% /notice %}}

Verwachte uitvoer:
```
Hello, Alice!
Hello, Bob!
```

---

### Stap 7: Tests schrijven met hspec

#### `greeter/test/HelloTest.hs`

De eigenlijke testspecificatie. We testen de `hello`-functie op correcte begroeting en op een lege string:

```haskell
module HelloTest where

import Test.Hspec
import Hello (hello)

hello_test :: Spec
hello_test = describe "hello" $ do
  it "greets the name correctly" $ do
    hello "World" `shouldBe` "hello, World!"
    hello "Arne"  `shouldBe` "hello, Arne!"
  it "handles empty string correctly" $ do
    hello "" `shouldBe` "hello, !"
```

#### `greeter/test/Main.hs`

Het ingangspunt van de test-suite roept de `hello_test`-spec aan via `hspec`:

```haskell
module Main where

import Test.Hspec
import HelloTest (hello_test)

main :: IO ()
main = hspec hello_test
```

Voer de tests uit met:

```bash
cabal test
```

---

### Stap 8: `greeter-web/greeter-web.cabal`

Een tweede package in hetzelfde project. Deze executable hangt af van de lokale `greeter`-library. Omdat `cabal.project` beide packages kent, hoeft `greeter` niet op Hackage te staan.

```haskell
cabal-version: 3.0
name:          greeter-web
version:       0.1.0.0
license:       Apache-2.0
build-type:    Simple

common warnings
    ghc-options: -Wall

executable greeter-web
    import:           warnings
    main-is:          Main.hs
    build-depends:    base >=4.17 && <4.20
                    , greeter
    hs-source-dirs:   src
    default-language: Haskell2010
```

---

### Stap 9: `greeter-web/src/Main.hs`

Voorlopig een minimale main die de `Greeter`-module importeert. Je kunt hier later web-gerelateerde logica toevoegen:

```haskell
module Main where

import Greeter

main :: IO ()
main = putStrLn (greet "web")
```

---

### Stap 10: Alles bouwen en uitvoeren

Vanuit de `greeter-demo/`-root:

```bash
# Package index verversen (eerste keer of na lange tijd)
cabal update

# Beide packages in één keer bouwen
cabal build all

# De greeter-executable starten
cabal run greet -- alice bob

# De greeter-web-executable starten
cabal run greeter-web

# Alle tests uitvoeren
cabal test all
```

Cabal bouwt eerst de `greeter`-library (inclusief de `titlecase`-dependency van Hackage), daarna de executables en de test-suite van beide packages.

---

### Stap 11: De native executable rechtstreeks uitvoeren

`cabal run` is handig tijdens ontwikkeling, maar het gecompileerde binaire bestand kan je ook rechtstreeks vanuit de terminal aanroepen. Cabal plaatst het in een diep geneste map onder `dist-newstyle/`. In plaats van dat pad op te zoeken, gebruik je het commando `cabal exec` of vraag je het pad op met `cabal exec -- which greet`. De eenvoudigste manier op elk platform is:

```bash
cabal exec greet -- alice bob
```

Wil je het binaire bestand kopiëren naar een vaste locatie zodat je het overal kunt aanroepen, dan gebruik je `cabal install`:

```bash
cabal install greet --installdir=./bin --overwrite-policy=always
```

Daarna staat het executable in `./bin/greet` (of `./bin/greet.exe` op Windows) en kun je het direct uitvoeren:

```bash
./bin/greet alice bob
```

{{% notice warning %}}
Gebruik `cabal install` zonder `--installdir` niet zomaar in een projectdirectory: het installeert het binaire dan globaal in `~/.cabal/bin`, wat verwarring kan geven als je meerdere versies van hetzelfde project hebt.
{{% /notice %}}

