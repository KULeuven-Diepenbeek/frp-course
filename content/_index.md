---
title: "Index"
---

# Functional Reactive Programming

Academiejaar 2025–2026.

## Overzicht

Deze cursus bouwt verder op je bestaande kennis van Haskell en functioneel programmeren. We vertrekken vanuit de veronderstelling dat je de basisconcepten van Haskell al beheerst — syntaxis, types, typeklassen en hogere-ordefuncties — en verdiepen ons in de meer praktische en geavanceerde kant van het werken met Haskell, en vervolgens in een krachtig programmeerparadigma voor het modelleren van time-dependent systemen (Functional Reactive Programming met Yampa).

De cursus is opgedeeld in vier grote onderdelen. In het eerste deel leer je werken met de Haskell-ontwikkelomgeving: hoe je GHCi configureert via een `.ghci`-bestand, en hoe je met Cabal aan de slag gaat als build-tool en dependency manager. In het tweede deel leer je hoe je invoer/uitvoer, mutable state en concurrentie aanpakt in Haskell — concepten die in puur functionele talen bijzonder zijn georganiseerd. Het derde deel introduceert het paradigma van Functioneel Reactief Programmeren: wat het is, waar het vandaan komt, en welk probleem het oplost. Daarna duiken we diep in **Yampa**, een volwassen FRP-library voor Haskell, van de eerste stappen tot geavanceerde switching-patronen.

## Vereiste voorkennis

Je wordt verwacht al vertrouwd te zijn met:

- Haskell-syntaxis, types, typeklassen en hogere-ordefuncties.
- Basisconcepten van functioneel programmeren zoals `map`, `filter`, `fold`, curryen en patroonmatching.

Als je je kennis wilt opfrissen, vind je nuttige bronnen in het Extra-onderdeel van deze cursus.

## Cursusindeling

| # | Onderdeel | Onderwerpen |
|---|---------|--------|
| 1 | Haskell Tools | `.ghci`-configuratie, Cabal-projecten |
| 2 | I/O & State | `IO`-monade, `IORef`, `MVar`, `TQueue` |
| 3 | FRP-concepten | Tijdsvariërende waarden, eventstromen, het FRP-paradigma |
| 4 | Yampa — Basisbegrippen | Signalfuncties, `SF a b`, `arr`, `>>>` |
| 5 | Yampa — Events | `Event`, `edge`, `never`, `now`, `after` |
| 6 | Yampa — Uitvoering | `reactimate`, game loops, koppeling met I/O |
| 7 | Yampa — Switches | `switch`, `dSwitch`, `rSwitch`, `kSwitch` |
| 8 | Yampa — Geavanceerd | State machines, `dpSwitch`, parallelle netwerken |
