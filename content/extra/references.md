---
title: "Referenties en Verder Lezen"
weight: 1
draft: false
---

## Officiële documentatie

- **Yampa op Hackage**: [https://hackage.haskell.org/package/Yampa](https://hackage.haskell.org/package/Yampa)  
  De volledige API-documentatie van de Yampa-bibliotheek, inclusief alle signal functions, events, switches en hulpfuncties.

- **GHC User's Guide**: [https://downloads.haskell.org/ghc/latest/docs/users_guide/](https://downloads.haskell.org/ghc/latest/docs/users_guide/)  
  Uitgebreide documentatie over GHC, language extensions, runtime-opties en de profiler. Bijzonder nuttig voor geavanceerde gebruikers.

- **Cabal User Guide**: [https://cabal.readthedocs.io/en/stable/](https://cabal.readthedocs.io/en/stable/)  
  Officiële documentatie voor Cabal: projectstructuur, dependencies, versionering, meerdere componenten (library, executables, tests).

- **Haskell Report 2010**: [https://www.haskell.org/onlinereport/haskell2010/](https://www.haskell.org/onlinereport/haskell2010/)  
  De formele taalspecificatie van Haskell 2010. Nuttig als naslag voor taalsemantieken.

---

## Academische artikelen

- **Hudak, P., Courtney, A., Nilsson, H., Peterson, J. (2003)**: *Arrows, Robots, and Functional Reactive Programming*.  
  Het oorspronkelijke artikel dat Yampa introduceert als een arrow-gebaseerde FRP-bibliotheek. Beschikbaar via [http://www.cs.yale.edu/homes/hudak/CS429F04/AFPLectureNotes.pdf](http://www.cs.yale.edu/homes/hudak/CS429F04/AFPLectureNotes.pdf)

- **Elliott, C., Hudak, P. (1997)**: *Functional Reactive Animation*.  
  Het baanbrekende artikel dat FRP introduceerde. Presenteert behaviours en events als de twee basisbouwstenen van reactieve systemen.

- **Nilsson, H., Courtney, A., Peterson, J. (2002)**: *Functional Reactive Programming, Continued*.  
  Behandelt de evolutie van FRP naar een arrow-gebaseerde aanpak en introduceert pijlnotatie voor signal functions.

---

## Boeken en cursussen

- **"Learn You a Haskell for Great Good!"**: [http://learnyouahaskell.com/](http://learnyouahaskell.com/)  
  Gratis online Haskell-inleiding met toegankelijke uitleg van types, typeclasses, monads en meer.

- **"Real World Haskell"**: [http://book.realworldhaskell.org/](http://book.realworldhaskell.org/)  
  Gratis online boek gericht op praktisch Haskell: I/O, concurrency, parsing, testen. Bijzonder nuttig als aanvulling bij de I/O-hoofdstukken van deze cursus.

- **"Programming in Haskell" (Graham Hutton, 2e editie)**  
  Beknopt academisch leerboek met formele behandeling van types, klassen, monads en parser-combinatoren.

- **Haskell MOOC (University of Helsinki)**: [https://haskell.mooc.fi/](https://haskell.mooc.fi/)  
  Gratis online cursus in het Engels met interactieve oefeningen, van basis tot gevorderd niveau.

---

## Handige Hackage packages

| Pakket | Beschrijving |
|---|---|
| `Yampa` | De FRP-bibliotheek die in deze cursus centraal staat |
| `stm` | Software Transactional Memory: `TVar`, `TMVar`, `TQueue` |
| `async` | Hogerniveau-concurrentie: `async`, `wait`, `race`, `concurrently` |
| `time` | Tijdsmeting: `getCurrentTime`, `diffUTCTime`, `UTCTime` |
| `containers` | `Map`, `Set`, `Seq` — essentiële datastructuren |
| `vector` | Efficiënte arrays voor performantie-kritische code |
| `QuickCheck` | Property-based testen, ideaal voor het testen van signal functions via `embed` |
| `HUnit` | Unit tests in Haskell |
| `SDL2` / `OpenGL` | Graphics en invoer voor games (veelgebruikt met Yampa) |

---

## Community en hulpbronnen

- **Haskell Discourse**: [https://discourse.haskell.org/](https://discourse.haskell.org/)  
  Het officiële discussieforum van de Haskell-gemeenschap.

- **r/haskell op Reddit**: Actieve community voor vragen, nieuws en discussies.

- **Haskell IRC / Matrix**: Realtime hulp in `#haskell` op Libera.Chat of via Matrix.

- **Hoogle**: [https://hoogle.haskell.org/](https://hoogle.haskell.org/)  
  Zoek Haskell-functies op naam of type. Onmisbaar hulpmiddel bij het werken met Yampa en andere libraries.
