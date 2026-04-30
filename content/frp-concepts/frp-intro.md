---
title: "Wat is Functioneel Reactief Programmeren?"
weight: 1
draft: false
---

## Het probleem: state en time

De meeste programma's moeten omgaan met de wereld zoals die continu verandert: een muis die beweegt, sensoren die waarden doorgeven, spelletjesfiguren die bewegen, netwerkcommunicatie die binnenkomt. De traditionele manier om daarmee om te gaan is **state mutation**: je houdt variabelen bij die de huidige state van het systeem beschrijven, je polt regelmatig de invoer, je past de variabelen aan, en je geeft de nieuwe state weer. Dit werkt, maar het leidt snel tot code die moeilijk te redeneren valt: de state verspreid zich over het hele programma, de volgorde van updates is subtiel, en het wordt steeds moeilijker om na te gaan waarom het systeem zich in een bepaalde state bevindt.


*Functional Reactive Programming (FRP)* biedt een alternatief: in plaats van expliciete state updates beschrijf je **hoe de uitvoer continu afhangt van de invoer als functie van de time**. Het systeem is geen reeks van stap-voor-stap mutaties, maar een declaratieve beschrijving van relaties. Je zegt niet "update positie x met delta", maar "de positie is de integraal van de snelheid". Dit is fundamenteel anders en brengt krachtige abstracties mee.

---

## Behaviours en Events

De oorspronkelijke FRP-theorie van Conal Elliott en Paul Hudak (1997, "Functional Reactive Animation") introduceert twee kernconcepten:

Een **Behaviour** is een waarde die continu varieert in de tijd. Je kunt het formeel zien als een functie `Time -> a`. De positie van de muis is een `Behaviour (Double, Double)`. De huidige time zelf is een `Behaviour Double`. Het volume van een microfoon is een `Behaviour Double`. Behaviours zijn altijd gedefinieerd: op elk moment in de time heeft een behaviour een waarde.

Een **Event** is een stroom van discrete gebeurtenissen, elk op een specifiek time point. Een toetsaanslag is een event. Het klikken van een muisknop is een event. Het aflopen van een timer is een event. Events zijn **niet** continu: ze zijn er soms wel, soms niet. Formeel is een event een (mogelijkerwijs oneindige) lijst van timestamps met bijbehorende waarden: `[(Time, a)]`.

De kracht van FRP zit in de combinatoren: je kunt behaviours samenvoegen, transformeren, integreren en differentiëren. Je kunt events filteren, vertragen, samenvoegen. En je kunt behaviours veranderen op het moment dat een event optreedt. Dit geeft een rijke algebra om reactieve systemen te beschrijven zonder expliciete state mutation.

{{< mermaid align=left >}}
graph LR
    subgraph B["Behaviour — altijd een waarde"]
        direction LR
        b0["t=0\n0.0"] --- b1["t=1\n0.3"] --- b2["t=2\n0.7"] --- b3["t=3\n1.0"]
    end
    subgraph E["Event — discrete voorvallen"]
        direction LR
        e0["t=0\n—"] --- e1["t=1\n—"] --- e2["t=2\nKlik!"] --- e3["t=3\n—"]
    end
{{< /mermaid >}}

---

## Push-based vs. pull-based FRP

Er zijn twee implementatiestrategieën voor FRP. Bij **pull-based** FRP vraagt het systeem bij elke update-stap actief de huidige waarde van elk behaviour op: het "trekt" waarden uit het systeem. Dit is eenvoudig te implementeren maar kan inefficiënt zijn als veel behaviours worden berekend terwijl ze nauwelijks veranderen.

Bij **push-based** FRP worden waarden automatisch doorgegeven wanneer een invoer verandert: het systeem "duwt" updates door het netwerk van afhankelijkheden. Dit is efficiënter maar complexer te implementeren, en geeft aanleiding tot problemen zoals **glitches** (tijdelijk inconsistente state als niet alle afhankelijkheden tegelijkertijd geüpdate worden).

Moderne FRP-libraries gebruiken vaak hybride benaderingen om de voor- en nadelen te balanceren.

{{< mermaid align=left >}}
graph LR
    subgraph Pull["Pull-based"]
        direction LR
        Sys1[Systeem] -->|"vraag waarde op"| Beh[Behaviour]
        Beh -->|"huidige waarde"| Sys1
    end
    subgraph Push["Push-based"]
        direction LR
        Inv[Invoer] -->|"wijziging"| A[Node A]
        A -->|"update"| B[Node B]
        B -->|"update"| Uit[Uitvoer]
    end
{{< /mermaid >}}

---

## Problemen met klassieke FRP en de komst van Arrowized FRP

De originele "behaviours en events"-stijl van FRP heeft in de praktijk problemen opgeleverd. Het voornaamste probleem is **spaceleaks**: door behaviours als functies van de tijd te modelleren, moeten ze de volledige geschiedenis van hun invoer kunnen raadplegen, wat geheugengebruik doet groeien naarmate de tijd verstrijkt. Bovendien zijn higher-order behaviours (behaviours die zelf behaviours produceren) conceptueel krachtig maar kunnen ze tot onvoorspelbaar geheugengebruik leiden.

**Arrowized FRP** lost dit op door het concept van een *signal function* te introduceren. Een signal function is een transformatie van een signaalstroom naar een andere signaalstroom: `SF a b` is een functie die loopt over signalen van type `a` en signalen van type `b` produceert. Signal functions hebben **geen directe toegang tot verleden of toekomst** van het signaal: ze zijn causaal (ze kunnen alleen de huidige waarde en geïntegreerde waarden gebruiken). Dit garandeert dat het geheugengebruik beheersbaar blijft.

Yampa is de meest gebruikte implementatie van Arrowized FRP in Haskell en is het onderwerp van de rest van deze cursus.

{{< mermaid align=left >}}
graph LR
    IN["Invoersignaal&nbsp;(a)"] --> SF["SF a b"]
    SF --> OUT["Uitvoersignaal&nbsp;(b)"]
{{< /mermaid >}}

---

## Waarom Yampa?

Yampa is een stabiele Haskell library voor Arrowized FRP die al meer dan twintig jaar actief gebruikt en onderhouden wordt. Ze is ontworpen voor realtime toepassingen zoals simulaties en games. Ze is gebaseerd op het type `SF a b` ("signal function from `a` to `b`") en gebruikt Haskell's **arrow-notatie** (`proc`/`-<`) als syntactic sugar voor het samenstellen van signal functions.

Yampa heeft een kleine maar expressieve kern. De meeste signal functions zijn combinatoren die andere signal functions samenstellen: serieel (`>>>`), parallel (`***`), of conditioneel via switches. Deze compositionele aanpak maakt het gemakkelijk om complexe reactieve systemen op te bouwen uit eenvoudige building blocks. Bovendien is Yampa puur functioneel: er is geen verborgen global state, geen mutatie achter de schermen. Dit maakt Yampa-code goed testbaar en redeneerbaar.

{{< mermaid align=left >}}
graph LR
    subgraph S["Serieel&nbsp;&nbsp; sf1 >>> sf2"]
        direction LR
        i1["a"] --> s1["SF a b"] --> s2["SF b c"] --> o1["c"]
    end
    subgraph P["Parallel&nbsp;&nbsp; sf1 *** sf2"]
        direction TB
        ia["a"] --> pa["SF a b"] --> oa["b"]
        ib["c"] --> pb["SF c d"] --> ob["d"]
    end
{{< /mermaid >}}

Het volgende hoofdstuk leidt je stap voor stap door de kernconcepten van Yampa: signal functions, events, reactimate, switches, en geavanceerde technieken.
