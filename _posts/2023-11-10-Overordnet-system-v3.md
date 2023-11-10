---
title: Overordnet systemdiagram v3
date: 10.11.2023 11:00
categories: [Nolek, Systemdokumentation]
tags: [nolek,systemdokumentation,systemdesign,infrastruktur]
---


Det er ved at være længe siden, jeg har opdateret om det samlede system, så det er vidst ved at være på tiden. Det er ikke fordi, det har ændret sig forfærdeligt meget siden sidst, men denne udgave giver nok et forbedret overblik over sammenhænge.

<img src="/assets/images/systemdiagram-v3.png" alt="image should have been here.">

Vi har stadig et system, som er opbygget af følgende komponenter:

* Frontend, som består af en Single Page Application samt en Android-app. 
  * Appen har en onboard MongoDB. Denne database synkroniseres automatisk med den serverbaserede MongoDB. 
* Reverse Proxy, der frasorterer potentielt ondsindede HTTP-requests baseret på header og body. 
* API Gateway, der står for message routing og data aggregation.
  RabbitMQ message broker til message queueing. 
* 3 services:
  * LeakTest Service, som arbejder op imod en InfluxDB. Den behandler requests om lækagedata. 
  * TestObject Service, som arbejder op imod en SqlDB. Den behandler requests om de objekter, tests foretages på. 
  * TestResultService, som arbejder op imod en MongoDB. Den udfører funktionelt den samme funktion som LeakTest Service; den er dog ikke integreret i RabbitMQ/Docker-systemet, men fungerer i stedet som et API, der kan modtage HTTP-requests. 
* Docker Host, hvori Gateway, LeakTestService, InfluxDb, Testobject Service, SqlDB og RabbitMQ er deployed.

Denne arkitektur giver os fleksibilitet, da clients kan sende requests til både LeakTest Service og TestObject Service, selvom de ikke kører på det tidspunkt, så længe Gateway og RabbitMQ kører. Det sikrer, at produktionen kan fortsætte, da man stadig kan teste for lækager, selvom den service, som gemmer data, er nede. Det kunne f.eks. være en serverfejl, opdatering eller lignende, der havde lagt servicen ned. Da vi også har TestResult Service, som ligger udenfor Docker/RabbitMQ-systemet, er det faktisk muligt stadig at indsende data om testresultater, selvom RabbitMQ er nede. I en situation, hvor det er Docker, som fejler, har brugere stadig en onboard database på mobile enheder, der kan synkroniseres med den serverbaserede MongoDB. Det giver systemet enormt god redundans i forhold til at sikre, at virksomheden kan fortsætte produktionen på trods af flere forskellige potentielle systemnedbrud. I et sådant tilfælde kan den applikationsbaserede frontend, når systemet igen er oppe at køre, indlæse tilstanden af den serverbaserede MongoDB og derfra opdatere InfluxDB med den nye tilstand. Omvendt kan den serverbaserede MongoDB opdateres med InfluxDB's tilstand via den mobile frontend. Der er på nuværende tidspunkt ikke implementeret direkte synkronisering mellem InfluxDB og serverbaseret MongoDB. Alle POST-endpoints til håndtering af lækagetests vil dog blive sat op, således at requests omfordeles til både LeakTestService og TestResultService.
