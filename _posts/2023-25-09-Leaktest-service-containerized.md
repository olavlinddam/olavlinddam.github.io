---
title: LeakTest Service containerized
date: 25.09.2023 10:00
categories: [Nolek, Docker]
tags: [nolek,docker,leaktest]
---

LeakTest Service er lige ved at være helt færdig. Der mangler bare de sidste tests. Derfor har jeg nu rettet fokus mod 
at få den containerized med Docker. Det er gjort via følgende trin:

### 1. LeakTestService Dockerfile
Det første der skal være styr på, er den Dockerfile, som det image der definerer LeakTest Service bygges ud fra. Når man 
bygger en ASP.NET Web API applikation, kan man vælge at få autogenereret en Dockerfile, som nogenlunde er brugbar.
Der skulle kun laves nogle smårettelser ift. stier.

```dockerfile
# Using the official .net 7.0 image
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
# Ports to expose
EXPOSE 80

# Using the official .net 7.0 sdk image
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["LeakTestService/LeakTestService.csproj", "LeakTestService/"]
RUN dotnet restore "LeakTestService/LeakTestService.csproj"
COPY . .
WORKDIR "/src/LeakTestService"

# Publish the application
FROM build AS publish
RUN dotnet publish "LeakTestService.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Run the application
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "LeakTestService.dll"]
```
En hurtig gennemgang af Dockerfilen: Først hentes det officielle .net 7 image, derefter bruges `EXPOSE` til at dokumentere
den port som tjenesten er tilgængelig på. Herefter bruges `FROM` igen til at hente .NET 7.0 SDK. `WORKDIR` sættes først til /src,
som angiver arbejdsmappen inde i containeren. Herefter kopieres LeakTestService.csproj til /src/LeakTestService i
containeren. 

Så kører `RUN dotnet restore`, som gendanner afhængigheder og værktøjer i projektet. `COPY . .` kopiere alle
filer over i containeren, hvorefter arbejdsmappen sættes til /src/LeakTestService. `FROM build AS publish` angiver, at 
vi bygger et nyt lag kaldet`publish` ovenpå det tidligere lag kaldet `build`. Så kører `RUN dotnet publish`, som 
kompilerer og kopierer outputtet til en ny mappe, `/app/publish` i containeren. Den sidste parameter bestemmer, at der ikke 
skal genereres en eksekverbar fil. Den skal vi ikke bruge når det kører i en container.

Den sidste blok er ansvarlig for at køre programmet. Her bygges der et nyt lag på, hvor vi sætter arbejdsmappen til /app. 
Vi kopierer fra /app/publish/ (den /app/publish/ som findes i "publish" laget), til /app i "final laget". Til sidst 
specificeres `ENTRYPOINT`, hvor `"dotnet", "LeakTestService.dll"`] kører kommandoen dotnet LeakTestService.dll i containeren,
hvilket starter applikationen.

### Docker Compose
LeakTestService anvender en InfluxDB og for at holde en klar opdeling af processer, har jeg valgt, at de skal køre i hver 
deres container. For at de to containere kan kommunikere med hinanden bruger jeg docker compose til at opsætte de to 
containers. Det sparer mig for en masse arbejde ift. IP-adresser der ændrer sig når containers stopper og starter, 
portbinding og serviceopdagelse.

#### compose.yaml
Til det formål bruger jeg filen compose.yaml:

```yaml
version: '3'
services:
  leaktestservice: 
    build:
      context: .
      dockerfile: ./LeakTestService/Dockerfile
    ports:
      - "5000:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=docker
    depends_on:
      - influxdb
      
  influxdb:
    image: "influxdb:latest"
    ports:
    - "8086:8086"
```

hvor jeg definerer de services som kører i det "bridge-network" docker compose sætter op. Først opsætter jeg betingelserne 
for LeakTest Service, herunder konteksten, hvilke porte der skal forbindes, miljøvariable og afhængigheder. Derefter defineres
servicen "influxdb", hvor det specificeres hvilket image der skal bruges og port mapping. 

Nu kan kommandoen `docker compose up` bruges til at opsætte og køre de to containere. 

Her er vi i mappen som vi har defineret som vores context:
```bash
olav@olav-laptop:~/Documents/Code/Projects/LeakMonitor/LeakTestService$ ls
compose.yaml     LeakTestService.sln  run-leaktestcontainer.sh
LeakTestService  README.md
```

som giver outputtet:

```bash
olav@olav-laptop:~/Documents/Code/Projects/LeakMonitor/LeakTestService$ docker compose up
[+] Running 11/11
 ✔ influxdb 10 layers [⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿]      0B/0B      Pulled                  3.0s 
   ✔ 7dbc1adf280e Already exists                                           0.0s 
   ✔ a07f6b4debcb Already exists                                           0.0s 
   ✔ c5bae45a22f0 Already exists                                           0.0s 
   ✔ 86650b4f28b9 Already exists                                           0.0s 
   ✔ 9093105ab21f Already exists                                           0.0s 
   ✔ 5534dc93d456 Already exists                                           0.0s 
   ✔ 386201384d0e Already exists                                           0.0s 
   ✔ e7698891e7f5 Already exists                                           0.0s 
   ✔ b5f337d6afbd Already exists                                           0.0s 
   ✔ 35cdf65f7c31 Already exists                                           0.0s 
[+] Building 0.0s (17/17) FINISHED                               docker:default
 => [leaktestservice internal] load build definition from Dockerfile       0.0s
 => => transferring dockerfile: 684B                                       0.0s
 => [leaktestservice internal] load .dockerignore                          0.0s
 => => transferring context: 380B                                          0.0s
 => [leaktestservice internal] load metadata for mcr.microsoft.com/dotnet  0.0s
 => [leaktestservice internal] load metadata for mcr.microsoft.com/dotnet  0.0s
 => [leaktestservice base 1/2] FROM mcr.microsoft.com/dotnet/aspnet:7.0    0.0s
 => [leaktestservice build 1/6] FROM mcr.microsoft.com/dotnet/sdk:7.0      0.0s
 => [leaktestservice internal] load build context                          0.0s
 => => transferring context: 2.01kB                                        0.0s
 => CACHED [leaktestservice base 2/2] WORKDIR /app                         0.0s
 => CACHED [leaktestservice final 1/2] WORKDIR /app                        0.0s
 => CACHED [leaktestservice build 2/6] WORKDIR /src                        0.0s
 => CACHED [leaktestservice build 3/6] COPY [LeakTestService/LeakTestServ  0.0s
 => CACHED [leaktestservice build 4/6] RUN dotnet restore "LeakTestServic  0.0s
 => CACHED [leaktestservice build 5/6] COPY . .                            0.0s
 => CACHED [leaktestservice build 6/6] WORKDIR /src/LeakTestService        0.0s
 => CACHED [leaktestservice publish 1/1] RUN dotnet publish "LeakTestServ  0.0s
 => CACHED [leaktestservice final 2/2] COPY --from=publish /app/publish .  0.0s
 => [leaktestservice] exporting to image                                   0.0s
 => => exporting layers                                                    0.0s
 => => writing image sha256:6f92991e2f8583dcfdd095a8b69d22bb262d6e47bdb7a  0.0s
 => => naming to docker.io/library/leaktestservice-leaktestservice         0.0s
[+] Running 2/0
 ✔ Container leaktestservice-influxdb-1         Created                    0.0s 
 ✔ Container leaktestservice-leaktestservice-1  Created                    0.0s 
```

Vi kan her se at de to containers er blevet oprettet. 
