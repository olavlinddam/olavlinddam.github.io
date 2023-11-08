---
title: Forbindelse til SQL server i container
date: 08.11.2023 12:00
categories: [Nolek, Docker]
tags: [nolek,testobjectservice,docker,sql,compose,databaser]
---


Problemer med docker compose, specifikt hvordan får jeg forbundet til min sql server? Jeg har brug for at køre dotnet ef update database for at apply mine migrations til databasen når docker compose bliver kørt.

Nu er endnu en service - TestObject Service - kommet på plads. Den skulle i den forbindelse integreres i min ´compose.yaml´ fil, som anvendes til at opsætte alle containers i samme netværk. Det gav nogle udfordringer, i forhold til at få forbundet til SQL serveren og opdatering af databasen på den server.

### Forbindelsen til SQL serveren
Normalvis er der ikke de store udfordringer i at oprette forbindelse til en SQL server. Det er forholdsvist simpelt, med en connection string der kunne lige noget ala: `"DefaultConnection": "server=host,port;Initial Catalog=dbname;USER ID=sa;Password=Strong-Pa55w0rd;`. Når man så arbejder med docker, ændrer det sig en lille smule. Den host, som normalt bare er localhost hvis man udvikler lokalt, hedder ikke localhost, hvis serveren kører i en container. Der vil den i stedet hedde containerens navn, eller det hostname man måske har specificeret i konfigureringen af containeren. Dvs hvis containeren hedder `sql-server`, så er det nu hostname. Så langs så godt. Når så serveren kører i compose, får containeren typisk et andet navn. Det kunne f.eks. være `docker-sql-server-1`, hvis man ikke har specificeret noget yderligere. Forbinder man fra sin lokale maskine (ikke i en container) er det altså det hostname man skal bruge. I sidste ende ville jeg jo gerne have alle mine services og processer i containere, så da TestObjectService blev integreret i compose, var der pludselig ikke længere forbindelse til SQL serveren. Indenfor compose netværket har hver container nemlig et specifikt navn, som man definerer i `compose.yaml`. Det er nu det navn man skal bruge som hostname. Så, i sidste ende, hedder en connection string: `"DefaultConnection": "server=host,port;Initial Catalog=defineret-compose-service-navn;USER ID=sa;Password=Strong-Pa55w0rd;`

### Opdatering af database med nye migrations
Normalt, når man arbejder med EntityFramework, kan man køre migrations på databasen med CLI kommandoen `dotnet ef database update`. Det gør man bare fra sit udviklingsmiljø. Det skulle vise sig, at det også bare er sådan man gør det når man arbejder med Docker. Det troede jeg dog ikke, så jeg var både ude i at skrive entrypoint scripts til min Dockerfile, oprette services i TestObjectService der kunne udføre migrations ved runtime og alle mulige andre metoder. Intet af det virkede, så løsningen blev bare at starte sql serveren i compose, udskifte hostname i connection string med `docker-sql-1` og så køre migrations fra Rider. 
