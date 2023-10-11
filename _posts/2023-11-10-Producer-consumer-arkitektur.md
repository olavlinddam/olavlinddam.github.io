---
title: LeakTest Service Refaktorering
date: 11.10.2023 11:00
categories:
  - Nolek
  - Microservices
tags:
  - microservices
  - nolek
  - leakstatusservice
---
Beslutningen om at implementere RabbitMQ som message broker i systemet, ændrer behovene ift. hvilke typer applikationer systemet skal bestå af. Hvor vi før havde et behov for at udvikle et API for hver service, kan vi nu overgå til andre typer, såsom worker services eller konsol applikationer hvor det giver mening, da det eneste offentlige API vil være GatewayService. Resten af systemet vil så kommunikere via message brokeren i en Producer/Consumer arkitektur. Det giver mening af flere årsager, men især muligheden for smertefri skalering og sikring af asynkron kommunikation springer i øjnene. 

### Producer/consumer designmønstret
I producer/consumer designmønstret, står produceren for at producerer eller sende beskeder til en kø, som en consumer lytter på og tager og behandler beskederne. Det kan illustreres således:

<img src="/assets/images/producer-consumer-simple.png" alt="image should have been here">

Det som er smart er, at man kan have lige så mange producere/consumers som man vil på en given kø. En service kan også være producer eller consumer på flere forskellige køer eller på forskellige køer på samme tid, som illustreret her:

<img src="/assets/images/producer-consumer-morph.png" alt="image should have been here">


Hvis vi skal overføre det til vores system, får vi et system som ser sådan ud:
<img src="/assets/images/producer-consumer-complex.png" alt="image should have been here">


Ovenstående giver et godt billede af hvordan kommunikationen flyder i systemet. Det er stadig et simplificeret diagram, da det ville fuldstændigt uoverskueligt at inkluderer alle kommunikationsruter. Det vigtige takeaway er, at services der har brug for at kommunikere kan gøre det via de forskellige queues, og når en service er klar til at modtage et nyt request, samler den blot det næste i køen op. På den måde bliver vores services uafhængige af kendskabet til hinanden, da de blot skal kende til de køer de har relationer til. Hvis trafikken på en bestemt kø bliver stor, kan vi nemt spin en ny instans af en given service op, som så strakt begynder at behandle requests hvilket letter trafikken:

<img src="/assets/images/producer-manyconsumers.png" alt="image should have been here">

Med den beskrevne arkitektur vil vi altså få skabt et system som er mere robust og fleksibelt i forhold til belastning. Integreringen med Docker og brug af load balancing i Docker Swarm kan vores system skaleres smidigt efter behov. 


### Påvirkning af de andre gruppemedlemmer
Introduktionen af RabbitMQ og producer/consumer arkitekturen er et valg jeg har truffet, da det bidrager til at give mit emne (Microservices) mere fylde. Det er overkill i forhold til vores use case, selvom man kan fremsætte det argument, at det altid er fornuftigt at opbygge systemer så de er fremtidssikret. Når det er sagt, så er det også vigtigt at huske, at de andre medlemmer af projektgruppen ikke har meget erfaring med designmønstret. Vi kender mest til API'er fra skolen og det er der de er komfortable. Der ligger altså en ekstra arbejdebyrde hos mig i at få introduceret resten af gruppen til mønstret, og i at hjælpe dem med at få implementeret det, så det ikke går ud over deres egne projektdele, at de skal omstille sig på baggrund af mine valg. 
