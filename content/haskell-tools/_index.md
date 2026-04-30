---
title: "Haskell Tools"
weight: 1
chapter: true
pre: "<b>1. </b>"
draft: false
---

# Haskell Tools

Dit hoofdstuk geeft een overzicht van de tools die je kan gebruiken om Haskell-projecten te ontwikkelen.

## GHCi

**GHCi** is de interactieve interpreter van GHC (Glasgow Haskell Compiler) en fungeert als een REPL (Read-Eval-Print Loop). Je gebruikt GHCi om snel code uit te proberen, types op te vragen met `:t` en definities te inspecteren met `:i`, zonder een volledig programma te compileren. GHCi is de snelste manier om met Haskell te experimenteren.

## Het `.ghci`-bestand

Wanneer GHCi opstart, zoekt het automatisch naar een configuratiebestand genaamd `.ghci`. Daarin kun je je persoonlijke instellingen vastleggen per directory: een aangepaste prompt, compiler warnings (`-Wall`), language extensions (zoals `-XArrows` voor Yampa) en modules die je altijd automatisch beschikbaar wil hebben. Zo vermijd je dat je bij elke sessie dezelfde commando's moet herhalen.

## Cabal

**Cabal** is het de-facto build-systeem en package manager voor Haskell. Je beschrijft je project in een `.cabal`-bestand: welke modules er zijn, welke libraries je nodig hebt en welke GHC-opties gelden. Cabal haalt dependencies automatisch op van [Hackage](https://hackage.haskell.org/) en berekent bij elke build een compatibele versiecombinatie via zijn dependency solver.

## Stack

**Stack** is een alternatieve build-tool die werkt met **snapshots**: vaste, vooraf geteste verzamelingen van packageversies die gegarandeerd compatibel zijn. Die snapshots worden beheerd door [Stackage](https://www.stackage.org/). Stack gebruikt intern nog steeds een `.cabal`-bestand voor de projectstructuur, maar voegt daar een `stack.yaml` aan toe dat het snapshot en de GHC-versie vastlegt. Hierdoor zijn builds automatisch reproduceerbaar zonder een apart freeze-bestand.

