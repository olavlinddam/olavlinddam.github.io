---
title: Docker stacks og netværk
date: 22.11.2023 08:50
categories: [Nolek, docker]
tags: [nolek,docker,network,stacks]
---

Jeg vil kort gennemgå to principper i Docker og hvordan jeg har anvendt dem praktisk i projektet.

## Network
I Docker referer "networking" til containeres mulighed for at kommunikere med hinanden eller med ikke-docker systemer.
En container kender ikke til andre enheder på netværket, det kender blot et netværksinterface med en IP adresse, gateway,
routing table, DNS service osv. Det betyder, at så længe en enhed er koblet på det samme netværk som Docker containeren,
så kan de, fra containerens perspektiv, kommunikere. I Docker kan man opsætte `overlay-networks`, og forbinde containers
til flere forskellige netværk. Mere om det senere.

### Ports
Når man opsætter en container, kan man specificere hvilke porte den skal eksponere til udefrakommende trafik.
Det bruge flaget `-p` eller`--publish` til. Det oprettet en regel i værtens firewall, som tillader trafik på den 
angivne port. Jeg benytter denne funktion flere steder, bl.a. til at eksponere et interface til InfluxDB, RabbitMQ og
til GatewayService, så clients kan forbinde til den. Nedenfor er et eksempel på hvordan man kan åbne ports i en 
`compose.yaml` fil.

```yaml
services:
  gatewayservice:
    anden-config: ..
    ports:
      - "5001:80"
    anden-config: ..
```
Når flaget er sat, kan alle kalde GatewayService på `<host-ip>:5001`, hvilket så mappes til port 80 inde i containeren.

### Custom netværk
I Docker kan man oprette custom netværk, som især bliver anvendeligt når man arbejder med Swarm, som ofte er 
distribueret over forskellige hosts. For at oprette et netværk kan man bruge følgende kommando:
`docker network create [OPTIONS] NETWORK`. Jeg har oprettet et netværk til GatewayService, som fungerer som indgangspunkt
for udefrakommende trafik: `docker network create --scope=swarm --attachable --driver overlay gw-net`, hvor `scope` er sat til,
at være et swarm net, hvilket vil sige det kun er for containers der deltager i hostens swarm. `attachable` specificerer, 
at containers udefra kan kobles på, og `driver` indikerer hvilken netværksdriver der skal anvendes. I dette tilfælde 
benytter jeg "overlay", da overlay er et netværk der kan eksistere på tværs af flere Docker hosts, altså flere maskiner.
Det er nødvendigt for at en swarm kan kommunikere. Den anden mulighed er "bridge", som er et isoleret netværk på hosten.
En container kan forbindes til flere netværk, og får en IP adresse for hvert netværk de er forbundet til. Hvis vi tager 
GatewayService som eksempel, er den configureret således:

```yaml
services:
  gatewayservice:
    anden-config: ..
    ports:
      - "5001:80"
    networks:
      - gw-net
      - msg-net
      
anden-config: ..

networks:
  gw-net:
    external:
      name: gw-net
  msg-net:
    external:
      name: msg-net
```
Den er altså en del af gw-net og msg-net, et andet custom netværk jeg har sat op. Det smarte ved, at gw-net er attachable
er som nævnt, at man kan tilkoble containers der ikke indgår i Swarm. Det giver mulighed for, at have en ekstern 
load-balancer, som kan redirect trafikken videre til Swarm. På den måde, behøver man ikke direkte at eksponere IP adresser
på de hosts som indgår i Swarm.

## Stacks
Docker Stacks er et værktøj, som fungerer på et højere niveau end containers. Man kan oprette stacks til konfigurering
af flere containers, hvilket jeg har brugt til at oprette logiske grupperinger af services. Stacks indgår som en del af
Docker Swarm, og når man deployere en stack er det også som en del af en Swarm. Man kan anvende compose filer til at
initere stacks og det er på mange måder det samme som at benytte `docker compose up`. Der er dog nogle ting man kan
udelade fra ens compose fil hvis man anvender stacks, herunder restart policies, da en Swarm automatisk forsøger at 
genoprette det specificerede antal containers. Når man vil deploy en stack bruges kommandoen: 
`docker stack deploy [OPTIONS] STACK`. Til at deploy den stack hvori GatewayService findes har jeg anvendt:
`docker stack deploy --with-registry-auth -c gw-stack.yaml gw`. Det opretter en stack ved navn "gw", med konfigurationer
fra filen "gw-stack.yaml". `--with-registry-auth` tillader, at alle hosts i Swarm kan anvende de login credentials den
host som kalder denne kommando har anvendt til at logge ind til Docker Hub. Det er en nemmere løsning end at logge ind
på Docker Hub på alle hosts i Swarm. Det er nødvendigt, da de images jeg bruger til at oprette mine services ligger der.

## Samling af de to
Nedenstående diagram viser hvordan jeg har anvendt de to begreber i praksis:

<img src="/assets/images/stacks-networks.png" alt="image should have been here">

Der indgår altså fire stacks i denne Swarm: gw stack, broker stack, service stack og db stack, med tre tilhørende netværk:
db-net, msg-net og gw-net. De stacks der krydser grænsnerne indgår i flere netværk, hvilket er det som tillader 
kommunikationen mellem komponenterne i systemet. Denne isolering sikrer, at kommunikation kun kan flyde via de 
kanaler jeg ønsker. Der er ingen vej fra clienten til databasen, som ikke går via gateway-broker-service-db. Dette 
tilbyder et ekstra lag af sikkerhed samt en logisk afgrænsning af systemets komponenter. I diagrammet er en proxy 
inlkuderet, men i praksis er denne ikke implementeret som en del af netværket, da den ikke er containerized og er hosted
eksternt. Den kunne dog nemt integreres og fungerer som den tidligere nævnte eksterne load-balancer, der stadig kørers
i en container, men ikke som en del af Swarm. 
