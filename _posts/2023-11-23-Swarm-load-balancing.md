---
title: Swarm load balance og ingress network
date: 22.11.2023 08:00
categories: [Nolek, docker]
tags: [nolek,docker,swarm, "load balance", "ingress network"]
---

Docker har en indbygget mekanisme til at distribuere indkommende trafik fra eksterne forbindelser (ingress traffic) til 
den rette service. Det tjener både det formål at trafik lander de rigtige steder og at belastningen fordeles jævnt over 
ens cluster. Docker anvender et ingress routing mesh til at sikre, at services eksponeres konsistent på den samme port, 
uanset hvilken node der kører en instans af den service. Så, når der kommer et request til Service A, så ved Docker 
Swarm, at den kan sende trafikken til en hvilken som helst Node som kører Service A, på den samme port uanset hvor i 
clusteren service instansen kører.

Dette ingress routing netværk spreder sig over hele ens cluster, hvilket tillader, at uanset hvilken node der modtager 
trafikken, kan de fordeles til den rette task.

<img src="/assets/images/swarm-ingress-mesh.png.png" alt="image should have been here">

Load balancing opnås ved, at routing netværket fordeler trafik ligeligt mellem tilgængelige tasks via et cirkulært round 
robin princip.

Service Discovery sikres ved, at Swarm Manager holder styr på samtlige IP adresser på de tasks der kører i ens cluster. 
Når ingress trafik lander ved ens node, ved manageren præcic hvilken node og hvilken task trafikken skal føres til.

<img src="/assets/images/swarm-load-balancing.png" alt="image should have been here">

Diagramemt viser, hvordan trafik til port 5001 lander ved den eksponerede node (webdock server), men da den node kører
en container med den service som eksponerer port 5001, sendes trafikken videre til enten Node 2 eller Node 3. 
Valget af hvilken node modtager trafikken afhænger af den nuværende belastning på de to nodes. 

Hvis en task fejler eller en node går ned, muliggør dette routing mesh, at manageren kan distribuere opgaverne og 
trafikken til andre aktive tasks.

Ingress i Docker Swarm er et overlay network som Swarm Manager anvender til at eksponerer service instanser for 
indkommende trafik. 

