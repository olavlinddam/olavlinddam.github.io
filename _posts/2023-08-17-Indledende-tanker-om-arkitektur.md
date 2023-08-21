---
title: Indledende tanker om arkitektur
date: 17.08.2023 06:14
categories: [LeakMonitor, Systemdokumentation]
tags: [nolek,microservices,softwareudvikling,systemdesign,systemarkitektur,authenticationservice,leakstatusservice,loggingservice]
---

Jeg har valgt at døbe mit projekt LeakMonitor. Det virker som en passende beskrivelse af formålet, og jeg synes, det klinger meget godt. Det giver en følelse af at være kommet rigtigt i gang. Selvom backend er det, jeg virkelig synes er sjovt, kan jeg meget godt lide at arbejde med high-level systemdesign. Hvordan kan man opbygge systemet rent arkitektonisk, så man kan vinde nogle fordele og gøre det lettere for sig selv i implementeringen i koden? Hvordan kan man ved brug af designmønstre øge kvaliteten, holdbarheden, læsbarheden af ens kode samt gøre den lettere at vedligeholde? Selvom det nok er mere passende at starte med at udlede mere specifikke krav til systemet og så arbejde derfra, vil jeg i denne omgang fokusere lidt på arkitekturen. Det giver mening, fordi jeg netop har valgt at fokusere på microservices, hvilket i høj grad påvirker arkitekturen uanset systemkravene.

Ideen med microservices er, at man får nedbrudt systemet i mindre, mere håndterbare dele med klar adskillelse af ansvarsområder. Det har mange fordele, som f.eks.:

* **Skalering:** Det gør systemet nemmere at skalere, da man kan nøjes med at skalere de services, der oplever stor trafik. Især hvis man deployer i containere, f.eks. ved brug af Docker.
* **Fejlisolering:** Hvis en enkelt mikroservice fejler, påvirker det ikke nødvendigvis resten af systemet.
* **Sprog- og teknologiagnostisk:** Selvom jeg ikke på nuværende tidspunkt har planer om at anvende forskellige programmeringssprog, kan det nemt lade sig gøre med microservices, fordi alle services eksponerer et API. Det gør systemet fleksibelt i forhold til at anvende den bedste løsning til et givent problem.

Der er selvfølgelig også ulemper ved at implementere en microservices arkitektur. Det kan give øget kompleksitet i håndteringen af interaktionen mellem services. Synkronisering af data mellem services kan også blive udfordrende, da hver service har deres egen database, ofte af forskellige typer. Netop interaktionen mellem services, som nødvendigvis foregår over et netværk, kan give øget latenstid sammenlignet med et traditionelt monolitisk system.

Der er naturligvis andre fordele og ulemper ved microservices, men for at holde længden af indlægget nede stopper jeg med at liste dem her. I stedet vil jeg dreje fokuset over på nogle af de services, jeg forestiller mig, at LeakMonitor får brug for. Indtil videre har jeg identificeret tre services, som er nødvendige for at indfri de forretningsmæssige mål: Authentication Service, der skal stå for autentificering og autorisering af brugere, før der gives adgang til de andre services, Leak Status Service, som udgør kernefunktionaliteten i systemet. Det er her, alle brugerinputs går igennem, før de persisteres. Logging Service, der, som navnet antyder, skal stå for at persistere logdata om alle relevante interaktioner med systemet. Dette er en service, som jeg forventer vil opleve høj trafik, da mange andre services vil trigge logevents. De tre services og hvordan jeg ser programflowet er illustreret her:
<figure>
  <img src="/assets/images/overordnet_arkitektur.png" alt="Image should have been here.">
  <figcaption> <i>Fig.1 - LeakMonitor overordnet arkitektur.</i>  </figcaption>
</figure>

Det diagram bliver garanteret genstand for mange revisioner, og det er da også på et meget simpelt stadie i øjeblikket. I øjeblikket er det blot et API gateway (som også er en service for sig), de tre forretningsmæssigt nødvendige services og en message broker. Jeg overvejer en del andre services, som skal udgøre støttefunktionaliteter, såsom en notification service, configuration service og data backup service, hvor den sidste nok er den vigtigste. Men mange bække små, og jeg er nødt til at starte med kernefunktionaliteterne først.

Så, ville jeg have anvendt microservices arkitektur til et projekt som dette, hvis det bare handlede om at få lavet et godt produkt hurtigst muligt til en kunde? Nok ikke, da projektet er relativt simpelt, og det potentielt ville øge kompleksiteten til et unødigt punkt. Måske, hvis kunden udtrykte et ønske om at have en infrastruktur, der nemt kunne skaleres horisontalt med behov for at kunne tilføje mange nye funktionaliteter hen ad vejen. Heldigvis handler projektet først og fremmest om at lære en masse, og det er også Noleks holdning.
