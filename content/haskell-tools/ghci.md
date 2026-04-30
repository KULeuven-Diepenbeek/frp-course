---
title: "GHCi configureren met .ghci"
weight: 1
draft: false
---

## Wat is een `.ghci`-bestand?

GHCi is de interactieve interpreter van GHC en fungeert als een **REPL** — wat staat voor **Read-Eval-Print Loop**. Een REPL leest een expression die je intypt, evalueert die expression, drukt het resultaat af, en wacht vervolgens op de volgende invoer. Dit maakt GHCi ideaal om snel code uit te proberen, types op te vragen met `:t` en definities te inspecteren met `:i`, zonder telkens een volledig programma te compileren.

Wanneer je GHCi opstart, zoekt het naar een configuratiebestand met de naam `.ghci`. De inhoud van dat bestand wordt automatisch uitgevoerd vóór de interactieve prompt verschijnt, alsof je die commando's zelf had ingetypt. Dit mechanisme is de standaardmanier om je GHCi-sessie aan te passen aan je project of je persoonlijke voorkeur, zonder telkens opnieuw dezelfde opties op de commandoregel in te typen.

GHCi zoekt naar het configuratiebestand op twee plaatsen, in deze volgorde: eerst in de huidige werkdirectory als `./.ghci`, dan in je homedirectory als `~/.ghci`. Het projectbestand in de huidige directory heeft voorrang op het globale gebruikersbestand. Dat betekent dat je je persoonlijke standaardinstellingen in het gebruikersbestand kunt zetten, terwijl elk project zijn eigen specifieke instellingen krijgt in een lokaal `.ghci`-bestand.

Elke regel in een `.ghci`-bestand is ofwel een GHCi-metacommando dat begint met `:` (zoals `:set` of `:load`), ofwel een gewone Haskell expression die bij het opstarten geëvalueerd wordt, ofwel een commentaarregel die begint met `--`. Bij Haskell expressions kun je bijvoorbeeld standaardmodules importeren zodat ze altijd beschikbaar zijn in de interactieve sessie.

---

## De prompt aanpassen

De standaardprompt van GHCi is `Prelude>`, wat weinig informatief is zodra je meerdere modules geladen hebt. Met `:set prompt` kun je de prompt vervangen door een eigen tekst. Dit klinkt als een kleine aanpassing, maar in de praktijk helpt een duidelijke prompt je snel zien in welke context je werkt. Moderne terminals ondersteunen ook ANSI-kleurcodes, zodat je de prompt kleur kunt geven voor extra leesbaarheid.

```haskell
-- Eenvoudige aangepaste prompt
:set prompt "ghci> "

-- Prompt met ANSI-kleur (groen + reset)
:set prompt "\ESC[1;32mghci\ESC[0m> "
```

Naast de hoofdprompt is er ook een vervolgprompt die getoond wordt wanneer je een expression over meerdere regels intypt. Die stel je in met `:set prompt-cont`:

```haskell
:set prompt-cont "  | "
```

---

## Compiler warnings inschakelen

GHC beschikt over een uitgebreid systeem van warnings dat je helpt veelgemaakte fouten vroegtijdig te ontdekken. De vlag `-Wall` schakelt de meest nuttige warnings in één keer in: onvolledige patternmatching, ongebruikte bindings, ontbrekende typeannotations, enzovoort. Het is sterk aangeraden om `-Wall` standaard in te schakelen. Een warning is geen fout - de code compileert gewoon - maar ze wijzen op plekken in je code die mogelijk onbedoeld gedrag kunnen veroorzaken.

```haskell
-- Schakel alle gangbare warnings in
:set -Wall

-- Of schakel specifieke warnings in
:set -Wincomplete-patterns
:set -Wunused-imports
```

---

## Language extensions activeren

Haskell heeft een stabiele kern (Haskell 2010), maar GHC biedt tientallen optionele language extensions aan. Je kunt deze extensions per bestand activeren met een `{-# LANGUAGE ... #-}`-pragma bovenaan het bestand, maar je kunt ze ook globaal inschakelen voor je interactieve GHCi-sessie via `:set -X<ExtensienNaam>`. Dat is handig wanneer je snel iets wilt uitproberen in de REPL  zonder telkens een pragma toe te voegen.

Enkele veelgebruikte extensies die je in je `.ghci` zou kunnen opnemen:

```haskell
-- Maakt het mogelijk om string literals te gebruiken voor elk type dat IsString implementeert
:set -XOverloadedStrings

-- Maakt het mogelijk om typevariabelen expliciet te scopen binnen een functiedefinitie
:set -XScopedTypeVariables

-- Nodig voor de proc-notatie van arrow-programma's (o.a. Yampa)
:set -XArrows
```

Het is belangrijk om te begrijpen dat extensies in `.ghci` alleen gelden voor de interactieve sessie, niet voor de code die je compileert met Cabal. Voor compileerprojecten stel je extensies in via het `.cabal`-bestand (zie het Cabal-hoofdstuk).

---

## Modules automatisch laden

Je kunt GHCi opdragen om bij het opstarten automatisch een bepaalde module te laden. Daarvoor gebruik je `:load`. Dit is bijzonder handig in een project waarbij je steeds met dezelfde module wil beginnen werken:

```haskell
:load Main
```

Je kunt ook modules importeren zodat hun functies direct beschikbaar zijn in de REPL, zonder dat je ze telkens opnieuw moet importeren:

```haskell
import Data.ByteString.Char8 (pack, unpack)
import Data.List (sort)
import Data.Maybe (fromMaybe, mapMaybe)
```

---

## Packages beschikbaar stellen met `:set -package`

Standaard laadt GHCi buiten een Cabal-project alleen de packages die altijd aanwezig zijn (zoals `base` en `containers`). Wil je in een losse REPL-sessie een package zoals `Yampa` gebruiken zonder een volledig Cabal-project op te zetten, dan maak je het package expliciet beschikbaar met `:set -package`:

```haskell
:set -package Yampa
```

GHCi laadt dan de gecompileerde library van dat package en stelt alle modules ervan beschikbaar voor import. Dit werkt alleen als het package geïnstalleerd is in je globale Cabal store (via `cabal install --lib Yampa`). Binnen een Cabal-project (`cabal repl`) is dit niet nodig: Cabal regelt zelf welke packages beschikbaar zijn op basis van je `.cabal`-bestand.

Je kunt dit ook in je `.ghci`-bestand zetten zodat het package automatisch beschikbaar is bij elke sessie in die directory:

```haskell
-- .ghci
:set -package Yampa
:set -package stm
```

---

## Modules importeren met `:module`

Nadat een package geladen is, gebruik je gewone `import`-statements om specifieke modules in scope te brengen. Een alternatief is het `:module`-commando (afgekorte versie: `:m`). Met `:module + ...` voeg je modules toe aan de huidige scope zonder de bestaande imports te wissen. Met `:module - ...` verwijder je ze weer:

```haskell
-- Voeg FRP.Yampa toe aan de huidige scope
:module + FRP.Yampa

-- Equivalent met import (meer Haskell-idiomatisch)
import FRP.Yampa

-- Meerdere modules tegelijk toevoegen
:module + FRP.Yampa Control.Concurrent.STM.TQueue

-- Alle zelfgevoegde modules verwijderen en terugkeren naar Prelude
:module
```

Het verschil tussen `:module + M` en `import M` in GHCi is subtiel: `import M` gedraagt zich als een gewone Haskell-import en wordt meegenomen bij `:reload`, terwijl `:module` de scope direct aanpast en niet persistent is na een reload. In de praktijk gebruik je `import` voor alles wat je in je `.ghci` of interactief wil vasthouden, en `:module` voor snelle tijdelijke aanpassingen.

Een typische interactieve sessie voor deze cursus ziet er zo uit:

```haskell
:set -package Yampa
import FRP.Yampa
import Control.Concurrent.STM.TQueue

-- Nu kun je direct signal functions uitproberen:
embed (arr (*2)) (1.0, [(0.1, Nothing), (0.1, Just 3.0)])
```

---

## Een praktisch projectbestand

Hieronder vind je een realistisch `.ghci`-bestand dat je kunt gebruiken als startpunt voor een Haskell-project:

```haskell
-- .ghci --- projectinstellingen

-- Aangepaste prompt met kleur
:set prompt "\ESC[34m[ghci]\ESC[0m \ESC[33m%s\ESC[0m> "
:set prompt-cont "      | "

-- Waarschuwingen
:set -Wall
:set -Wno-name-shadowing

-- Language extensions die handig zijn tijdens interactief werken
:set -XOverloadedStrings
:set -XScopedTypeVariables
:set -XArrows

-- Packages beschikbaar stellen (buiten cabal repl)
:set -package Yampa
:set -package stm

-- Veelgebruikte modules in scope brengen
import FRP.Yampa
import Control.Concurrent.STM.TQueue

-- Laad de hoofdmodule van het project
:load Main
```

Sla dit bestand op als `.ghci` in de root van je projectdirectory en start dan gewoon `ghci` op. GHCi pikt het bestand automatisch op en past alle instellingen toe vóór de prompt verschijnt.

---

## Beveiligingsopmerking

Omdat GHCi het `.ghci`-bestand automatisch uitvoert, is het belangrijk te beseffen dat dit bestand arbitraire Haskell IO-acties kan uitvoeren. Vertrouw nooit een `.ghci`-bestand van een onbekende bron. Wanneer je code downloadt van het internet of van iemand anders ontvangt, bekijk je altijd eerst het `.ghci`-bestand voordat je `ghci` in die directory opstart.

---

## Overzicht van veelgebruikte instellingen

| Instelling | Voorbeeld |
|---|---|
| Aangepaste prompt | `:set prompt "λ> "` |
| Vervolgprompt | `:set prompt-cont " \| "` |
| Alle warnings | `:set -Wall` |
| Language extension | `:set -XOverloadedStrings` |
| Package beschikbaar stellen | `:set -package Yampa` |
| Module toevoegen aan scope | `:module + FRP.Yampa` |
| Module laden | `:load Main` |
| Module importeren | `import Data.List (sort)` |
