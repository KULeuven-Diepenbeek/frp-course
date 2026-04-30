---
title: "Stack: Build-tool en Dependency Manager"
weight: 3
draft: false
---

## Wat is Stack?

**Stack** is een alternatieve build-tool en dependency manager voor Haskell. Waar Cabal een **dependency solver** gebruikt die bij elke build de beste versiecombinatie opnieuw uitrekent, werkt Stack met **snapshots**: vaste, vooraf geteste verzamelingen van packageversies die gegarandeerd compatibel zijn met elkaar. Die snapshots worden beheerd door [Stackage](https://www.stackage.org/), een community-project dat regelmatig nieuwe releases uitbrengt nadat gecontroleerd is dat alle packages in de snapshot foutloos сompileren.

Het gevolg is dat Stack-builds van nature reproduceerbaar zijn: iedereen die hetzelfde `stack.yaml`-bestand gebruikt, bouwt met exact dezelfde GHC-versie en exact dezelfde packageversies, zonder dat er een afzonderlijk freeze-bestand nodig is. Dit maakt Stack populair in grotere teams en in situaties waar reproduceerbare CI-builds cruciaal zijn.

Stack en Cabal sluiten elkaar niet uit. Stack gebruikt intern nog steeds het `.cabal`-bestand om de structuur van je project te beschrijven (modules, dependencies, build-targets). Stack voegt daar bovenop een eigen `stack.yaml` toe dat bepaalt welk snapshot en welke GHC-versie gebruikt worden.

---

## Stack installeren

Stack wordt ondersteund door [GHCup](https://www.haskell.org/ghcup/), de aanbevolen toolchain-installer voor Haskell. Installeer Stack via:

```bash
ghcup install stack latest
```

Controleer of de installatie gelukt is:

```bash
stack --version
```

Dit zou iets moeten uitvoeren als `Version 2.15.x, ...`. Stack beheert zijn eigen GHC-installaties los van GHCup: wanneer je voor het eerst een project bouwt, downloadt Stack automatisch de GHC-versie die bij het gekozen snapshot past.

---

## Een nieuw project aanmaken

Stack biedt sjablonen om snel een projectstructuur op te zetten. Het eenvoudigste sjabloon is `simple`:

```bash
stack new mijn-project simple
cd mijn-project
```

Na dit commando heb je de volgende structuur:

```
mijn-project/
  stack.yaml            -- Stack-configuratie: snapshot en extra instellingen
  stack.yaml.lock       -- automatisch gegenereerde vergrendeling van exacte versies
  mijn-project.cabal    -- projectbeschrijving (modules, dependencies, build-targets)
  app/
    Main.hs             -- ingangspunt
  src/                  -- broncode voor de library (indien aanwezig)
  test/
    Spec.hs             -- testingangspunt
  README.md
```

Stack genereert ook automatisch een `.cabal`-bestand. Je bewerkt dat bestand op dezelfde manier als bij een puur Cabal-project: je voegt er modules, dependencies en language extensions aan toe. Stack leest dat bestand en combineert het met de informatie uit `stack.yaml`.

---

## Het `stack.yaml`-bestand

Het `stack.yaml`-bestand is de eigenlijke Stack-configuratie. Het bevat minimaal de **resolver**: de verwijzing naar het Stackage-snapshot dat Stack gebruikt om packageversies op te zoeken.

```yaml
resolver: lts-23.8

packages:
  - .
```

Het veld `packages: [.]` betekent dat Stack het project in de huidige directory moet bouwen. Je kunt hier ook paden naar andere lokale projecten opgeven als je een multi-package workspace hebt.

### Resolvers: LTS en Nightly

Stackage biedt twee typen snapshots:

- **LTS** (Long Term Support): stabiele snapshots die maandenlang onderhouden worden en alleen bugfixes ontvangen. De naam `lts-23.8` betekent LTS-serie 23, revisie 8. Elke LTS-serie is gekoppeld aan een specifieke GHC-versie.
- **Nightly**: dagelijks bijgewerkte snapshots met de nieuwste packageversies. Minder stabiel, maar nuttig als je de allernieuwste versie van een package wil gebruiken.

### Extra dependencies toevoegen

Niet elk package staat in het Stackage-snapshot. Packages die je nodig hebt maar die niet in het snapshot zitten, voeg je toe via `extra-deps`. Je specificeert dan de exacte versie:

```yaml
resolver: lts-23.8

packages:
  - .

extra-deps:
  - some-package-1.2.3
```

Voor Yampa geldt dat het wél in recente LTS-snapshots staat, dus je hoeft het doorgaans niet als `extra-dep` op te geven — het volstaat het aan `build-depends` in je `.cabal`-bestand toe te voegen.

---

## Het `.cabal`-bestand in een Stack-project

Het `.cabal`-bestand in een Stack-project heeft dezelfde structuur als bij een puur Cabal-project. Je declareert er je build-targets, modules, dependencies en language extensions. Stack leest dit bestand en gebruikt het om te weten wat er gebouwd moet worden; het snapshot bepaalt vervolgens welke versies van de opgegeven packages gebruikt worden.

Een typisch `.cabal`-bestand voor een FRP-project met Yampa:

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
  default-language:   Haskell2010
  default-extensions:
      OverloadedStrings
    , ScopedTypeVariables
    , Arrows
  ghc-options: -Wall -O2
```

Voeg een nieuwe dependency toe door de naam aan `build-depends` toe te voegen en daarna `stack build` uit te voeren. Stack zoekt de package op in het snapshot en downloadt hem automatisch.

---

## Veelgebruikte Stack-commando's

De dagelijkse ontwikkelcyclus met Stack vraagt slechts een handvol commando's:

```bash
# Project bouwen
stack build

# Bouwen en het uitvoerbare programma starten
stack run

# Interactieve GHCi-sessie met het project geladen
stack repl

# Tests uitvoeren
stack test

# Een willekeurig commando uitvoeren in de Stack-omgeving (bijv. een gegenereerd binair)
stack exec mijn-project
```

Bij het eerste `stack build` in een nieuw project kan het enige minuten duren: Stack moet mogelijk GHC downloaden en alle dependencies compileren. Daarna zijn de resultaten gecached en zijn volgende builds snel.

### GHCi-instellingen meegeven

Je kunt GHCi-opties doorgeven via `--ghci-options`. Dat is handig als je tijdelijk een language extension wil inschakelen zonder je `.ghci`-bestand aan te passen:

```bash
stack repl --ghci-options="-XArrows"
```

---

## Reproduceerbare builds met `stack.yaml.lock`

Stack genereert automatisch een `stack.yaml.lock`-bestand naast je `stack.yaml`. Dit bestand bevat de exacte commit-hashes en versiegegevens van het snapshot en alle extra-deps. Je hoeft dit bestand zelf nooit aan te passen — Stack houdt het bij.

Commit het `stack.yaml.lock`-bestand mee naar je versiebeheersysteem. Zo bouwt iedereen die het project kloont met exacte dezelfde toolketen, ook als het Stackage-snapshot later bijgewerkt zou worden aan de Stack-servers.

---

## Stack versus Cabal: wanneer welke tool?

Beide tools zijn volledig ondersteund en actief onderhouden. De keuze hangt grotendeels af van de context:

| Aspect | Stack | Cabal |
|---|---|---|
| Reproduceerbaar zonder extra stap | Ja, via snapshot | Nee, vereist `cabal freeze` |
| GHC-versiebeheer | Automatisch via snapshot | Handmatig via GHCup |
| Flexibiliteit in versiekeuze | Beperkt tot snapshot | Volledige controle via solver |
| Configuratiebestand | `stack.yaml` + `.cabal` | Enkel `.cabal` |
| Aanbevolen voor | Reproduceerbare projecten, teams | Bibliotheekontwerp, Hackage-uploads |

---

## Projectstructuur voor deze cursus

Een typisch Stack-project voor FRP-experimenten met Yampa ziet er als volgt uit:

```
frp-spel/
  stack.yaml
  stack.yaml.lock
  frp-spel.cabal
  app/
    Main.hs           -- reactimate-lus, ingangspunt
  src/
    Spel/
      Logica.hs       -- signalfuncties
      Rendering.hs    -- uitvoer via IO
  test/
    Spec.hs
```

Het bijbehorende `stack.yaml`:

```yaml
resolver: lts-23.8

packages:
  - .
```

Start een interactieve sessie met:

```bash
stack repl
```

Stack laadt dan automatisch alle modules uit je project en alle dependencies uit het snapshot, inclusief Yampa als je dat aan `build-depends` hebt toegevoegd.

---

## Overzicht van veelgebruikte commando's

| Commando | Beschrijving |
|---|---|
| `stack new <naam> simple` | Nieuw project aanmaken met het simple-sjabloon |
| `stack build` | Project bouwen |
| `stack run` | Bouwen en uitvoeren |
| `stack repl` | GHCi-sessie met project geladen |
| `stack test` | Tests uitvoeren |
| `stack exec <naam>` | Gegenereerd binair uitvoeren |
| `stack update` | Lokale Stackage-index verversen |
| `stack ls snapshots` | Beschikbare lokale snapshots tonen |
