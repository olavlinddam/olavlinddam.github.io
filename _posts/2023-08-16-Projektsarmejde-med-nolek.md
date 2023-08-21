---
title: Projektsamarbejdet med Nolek
date: 16.08.2023 12:23
categories: [LeakMonitor, Systemdokumentation]
tags: [nolek,microservices,containerization,softwareudvikling,brugergrænseflade,rabbitmq,api-gateway]
---

Jeg vil blot bruge et øjeblik på at gennemgå det overordnede mål for det projekt, der nu starter i samarbejde med Nolek. 
Nolek, som specialiserer sig i lækagetest, har præsenteret os for en udfordring, der ligner et softwareprojekt, de selv 
kunne påtage sig. Selvom opgaven er i den simplere ende - for at sikre, at vi som studerende har en realistisk chance 
for at fuldføre det - er der stadig mulighed for at udvide, hvis vi ønsker en større udfordring.

Nolek har leveret en prototype af en brugergrænseflade, som klart illustrerer problemet.

<img src="/assets/images/nolek_product_example.png" alt="Image should have been here.">


Billedet viser, fra venstre mod højre, først en menu, hvor brugeren kan vælge et ud af flere billeder. I midten ser vi 
et diagram over "sniffing points", som er identificeret som potentielle lækagepunkter. Dette skema kunne eksempelvis 
repræsentere et rørsystem i et køleskab. Til højre skal brugeren angive, om et punkt er tæt. Der vil være et element for
hvert sniffing point. I praksis måler brugeren fysisk på det specifikke punkt og indtaster derefter, om det er "OK" 
eller "NOT OK". Hvis brugeren ønsker at ændre fra "NOK" til "OK", skal der gives en grund. I menuen til venstre, 
navngivet "pictures", kan brugeren vælge det system, han/hun ønsker at teste.

Dette er min nuværende forståelse af problemstillingen. Det er nogle måneder siden, vi sidst talte med Nolek, så nogle
detaljer kan være blevet glemt eller kan kræve justering, men dette er essensen.

Da jeg vil fokusere på microservices og containerization, er min første tanke at identificere de nødvendige 
services for et sådant system. Jeg planlægger at implementere hver service som en isoleret enhed med sin egen 
database. De vil kommunikere via en message broker som RabbitMQ. Kommunikationen til brugergrænsefladen og de 
øvrige gruppemedlemmers systemer vil ske gennem et API gateway for at sikre en ensartet kommunikationsflade. 
Jeg vil deployere hver service i sin egen container, hvilket muliggør skalering baseret på trafik. Der er mange flere 
fordele ved containerization og microservice-arkitekturen, men det vil jeg dykke dybere ned i i fremtidige indlæg.

