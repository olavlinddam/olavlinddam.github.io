---
title: Læringsplan 09.10.23 - 30.10.23
date: 10.10.2023 10:00
categories: [Nolek, Læringsplan- og mål]
tags: [nolek,læringsplan,projektstyring,delmål]
---
Hermed den femte læringsplan. 

| Periode             | Delmål i denne cyklus                                                                                                                                  | Opfyldelseskriterie                                                                                                                                                                                                                                                                                                                                                                                              | Status | Refleksion          | Evaluering        |
|:--------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-------|:--------------------|:------------------|
| 09.10.23 - 30.10.23 | 1. LeakTestService omskrevet til consumer/producer arkitektur.<br> 2. LeakTestService, InfluxDB, RabbitMQ og GatewayService kommunikerer i containers. | 1.1: Alle endpoints omskrevet til consumers.<br>1.2: Alle consumers lytter på de rigtige routing keys<br> og modtager de rigtige beskeder.<br> 1.3: Alle producers er sat op til at give response på requests.<br> 2.1: Alle dependencies mellem services er sat korrekt op. <br> 2.2: Alle services lytter på de korrekte porte. <br> 2.3: Kommunikation testet fra gateway -> leaktest -> influxdb og tilbage. |    1.1<br>1.2<br>1.3<br>2.1<br>2.2<br>2.3    |  Det store issue i denne omgang, har været at få al<br> kommunikation til at foregå indenfor et HTTP request.<br> For at opnå dette har jeg implementeret et mønster som kaldes<br> "Request-Reply Pattern".<br> | For en gangs skyld nåede jeg i mål<br> med alt hvad jeg havde sat mig for. Tidsestimeringen har <br> fungeret bedre denne gang og der har generelt været færre problemer med udviklingen.  |  

Mit kanban ved begyndelsen af perioden:

<img src="/assets/images/kanban_1010.png" alt="image should have been here">
