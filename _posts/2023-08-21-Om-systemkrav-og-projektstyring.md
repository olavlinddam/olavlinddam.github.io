---
title: Om systemkrav og projektstyring
date: 18.08.2023 18:33
categories: [Nolek, Systemdokumentation]
tags: [nolek,systemkrav,projektstyring,kanban]
---
I dette post vil jeg gennemgå de systemkrav, jeg har identificeret på baggrund af det oplæg, vi fik af Nolek i starten af
sommeren. De er ikke blevet revideret eller prioriteret af PO endnu, så jeg forventer, at de skal justeres, og at der vil
komme tilføjelser efter vores næste møde. Derudover vil jeg beskrive mine overordnede tanker om projektstyringen.

### Krav til det samlede system
* **Indtastning af testresultater:** Appen skal tillade brugere at indtaste "OK" eller "NOT OK" for hver lækagetest.
* **Visning af lækagepunkter:** Appen skal vise et layout (f.eks. et rørsystem), der markerer potentielle lækagepunkter
  (svejsninger eller andre svage punkter). Brugeren skal kunne vælge mellem forskellige layouts, der repræsentere
  forskellige testobjekter.
* **Ændring af testresultater:** Hvis en bruger ønsker at ændre et testresultat fra "NOT OK" til "OK", skal han/hun give
  en begrundelse for denne ændring. Systemet skal ikke tillade ændringen at blive gemt, medmindre der er givet en
  begrundelse.
* **Gemme data:** Alle testresultater og eventuelle ændringer skal gemmes i en database. Systemet skal gemme tidspunktet
  for hver test og hver ændring.

### Krav til min del af systemet
* Systemet skal kunne lagre tidsserie data (OK/NOT OK status over tid) for hver lækagepunkt i en influxDB.
* Systemet skal kunne opdatere status for et lækagepunkt i influxDB'en.
* Systemet skal kunne hente status for alle lækagepunkter eller et specifikt lækagepunkt fra influxDB'en.
* Systemet skal kunne håndtere forespørgsler til databasen på en effektiv måde for at sikre hurtig respons tid.
* Systemet skal have data backup implementeret i en eller anden form.
* Systemet skal kunne autentificere brugere for at sikre, at kun autoriserede personer kan opdatere statussen for
  lækagepunkter.
* Systemet skal automatisk logge aktiviteter (som ændringer i lækagepunktsstatus) for at holde styr på, hvem der gjorde
  hvad og hvornår.
* Systemet skal ved efterspørgsel kunne levere logdata.
* Systemet skal kunne køre på en hvilken som helst maskine der kan køre Docker.
* Systemet skal eksponere et ensartet API som resten af produktgruppen kan interagere med.

### Styring af arbejdsprocessen
For at styre min arbejdsprocess effektivt, anvender jeg et KanBan board som illustreret nedenfor.

<figure>
  <img src="/assets/images/kanban_2108.png" alt="Image should have been here.">
  <figcaption> <i>Hvert kort i backlog repræsenterer et krav eller en oversættelse af et krav. Nogle kort kan 
repræsentere flere krav. Tasks er en nedbrydelse af de items, der ligger i min sprint backlog. Jeg skriver dem i takt
med at de falder mig ind, eller som del af analysen af et sprint backlog objekt, når jeg begynder arbejdet på det.
</i>  </figcaption>
</figure>

KanBan-boardet er designet til at give mig et klart visuelt overblik over projektets progression og opdeling. Jeg
arbejder med både en product backlog og en sprint backlog, som begge udelukkende repræsenterer min specifikke del af det
samlede projekt. Disse kolonner er inspireret af SCRUM-metodologien, og jeg forsøger efter at implementere en
SCRUM-lignende arbejdsstruktur med to ugers sprints. Ved afslutningen af hver sprint planlægger jeg at gennemføre et
retrospektiv for at evaluere og forbedre min arbejdsproces. Vi har fastlagt ugentlige møder med Nolek, der begynder
den 1. oktober. Jeg forventer at anvende hvert andet af disse møder som et sprint review. Det indledende møde vil være
en mulighed for at finjustere min backlog og for at gennemføre for det første review.
