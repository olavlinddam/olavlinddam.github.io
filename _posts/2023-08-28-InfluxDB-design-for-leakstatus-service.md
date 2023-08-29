---
title: InfluxDB design for LeakStatus Service
date: 28.08.2023 10.11
categories: [LeakMonitor, Systemdokumentation]
tags: [nolek,influxdb,databaser,systemdesign,databasedesign,leakstatusservice]
---

I dette post vil jeg gennemgå mit indledende design, af den InfluxDB der skal opbevare informationen om hver lækagetest.
Databasen er en del af den microservice der hedder LeakStatus-Service, som hvis hovedformål er, at modtage brugerinput
og gemme det i en InfluxDB, samt hente information om lækagetests til visning for brugeren.
Læs min [grundlæggende introduktion til InfluxDB](https://olavlinddam.github.io/posts/InfluxDB-basics/)  inden dette 
post, hvis du ikke kender InfluxDB i forvejen.

##### Tags:
* **test_object**: Selve det object der testes på. Der vil være mange forespørgelser på dette og det kan tage mange 
forskellige værdier. 
* **status**: Ofte forespørge på status så derfor vil vi have den indekseret.

##### Fields:
* **user**: Sjælendt forespørgelser på dette, potentielt mange forskellige værdier. Egner sig ikke som tag, for at undgå
høj kardinalitet (at man har mange unikke værdier i kolonnen).
* **sniffing_points:** Vi forespørger sjælendt på dette, da det er relateret til et test_object som vi i stedet vil 
forespørge på. Der kan også være mange unikke værdier her, men det afhænger af Noleks navngivningskonventionerne for 
sniffing points.
* **reason:** Vi vil nok sjældent forespørge på reason.

#### Bucket eksempel

| _time | _measurement | test_object | status | user  | sniffing_point | reason      |
|-------|--------------|-------------|--------|-------|----------------|-------------|
| tid   | LeakTest     | object1     | OK     | user1 | point1         |             |
| tid   | LeakTest     | object2     | NOK    | user2 | point2         |             |
| tid   | LeakTest     | object2     | OK     | user2 | point2         | reason here |

#### Bucket skema

| name           | type      | data_type |
|----------------|-----------|-----------|
| time           | timestamp |           |
| test_object    | tag       | string    |
| status         | tag       | string    |
| user           | field     | string    |
| sniffing_point | field     | string    |
| reason         | field     | string    |
