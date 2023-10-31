---
title: Test Object Service
date: 31.10.2023 07:00
categories: [Nolek, Microservices]
tags: [nolek,testobjectservice,systemdokumenation,systemdesign,gateway,databaser]
---

Indtil videre har vi i gruppen arbejdet individuelt på vores dele af projektet. Tiden er nu kommet til at David og jeg begynder at integrere vores delelementer. David arbejder på en Single Page Application, som skal fungere som en af to front ends, hvori data om testobjekter og testresultater kan indtastes og indlæses. Det skal sammenkobles med Gateway Service og LeakTest Service, og så skal vi have udviklet en mindre service som skal håndtere data om testobjekter. Tanken er, at når der oprettes et testobjekt, så gemmes der informationen herom i en SQL database, samtidig med at der sendes "pre data" om sniffing points til LeakTest Service, så der ligger testresultater i databasen inden brugeren første gang skal indlæse testobjektet. Det er efter anmodning fra David, for at gøre det lettere for ham ift. front end. Jeg overvejer, om man ikke kan undlade dette, og opsætte en logik der siger, at hvis der ikke er nogle testresultater, så returner at alle sniffing points er OK. 


### Dokumentation på TestObject Service
Nedenfor gennemgår jeg forskellige diagrammer for at give en bedre ide om hvordan kommunikationen i systemet foregår, og hvilke elementer jeg forestiller mig TestObject Service vil indeholde.

Det første er et simpelt diagram, som viser hvordan kommunikationen går i oprettelsen af et test objekt, hvis vi isoleret set ser på TestObject Service.

<img src="/assets/images/testobj_dataflow.png" alt="image should have been here">

Herunder visualiseres de to klasser som udgør modellaget i servicen. Der er testobjekt og sniffing point. Der eksisterer en én-til-mange relation hvor et testobjekt indeholder en liste af sniffing points, som er assicieret med testobjektet. Denne simple relation kan direkte oversættes til vores databasedesign. Det er oplagt i dette tilfælde at anvende en relationel SQL database, da vi vil gerne vil benytte det, for os, simpleste værktøj til at opbevare information om relationen. Der kan ikke eksistere et sniffing point som ikke har en relation til et testobjekt og et testobjekt skal have sniffing points associeret med sig. Derudover er implementeringen af logikken til at gemme i en SQL server lige til, da vi kan anvende Entity Framework til scaffolding. 
<img src="/assets/images/testobj_entity_relations.png" alt="image should have been here">

Nedenfor ses de klasser, samt deres relationer, jeg forventer at TestObject Service skal have. Vi har altså et antal "consumers", som udgør kernen i applikationen. Consumers kan sammenlignes med controllers, hvor de modtager information om testobjekter, og gemmer dem i SQL databasen. Valideringen af testobjekterne foregår i "validation" klasser. Vi går på kompromis med Single Responsibility Princippet, idet vores consumers får flere ansvarsområder. De skal både oprette objekterne og gemme dem i databasen. Dette er bevidst, da det vil være en overkomplicering af systemet at begynde at implementere repositories og lign. for at adskille logikken. Derudover er der et behov for hurtig udvikling, da vi begynder at mærke tidspresset. 
<img src="/assets/images/testobj_createobj.png" alt="image should have been here">

Nu hvor vi har styr på klasserne internt i TestObject Service, vil jeg kort vise et sekvensdiagram over processen at gemme et nyt testobjekt i databasen. Vi vælger at gemme billedet af testobjektet på disken, da det både er mere overskueligt ift. hurtigt at se billedet selv som udvikler, det sparer os for at encode det når det skal gemmes, det mindsker størrelsen af databasen og det er generelt hurtigere at indlæse det sådan fremfor fra SQL databasen.
<img src="/assets/images/testobj_add_test_obj_sequence.png" alt="image should have been here">

Diagrammet herunder viser, hvordan de samlede system kan indhente oplysninger om et testobjekt samt associerede sniffing points. Der sendes et request fra klienten, som Gateway fordeler ud til både LeakTestService og TestObject Service. LeakTest Service indhenter alle testresultater associeret med det givne TestObject og  TestObject Service indhenter information om objektet, herunder layout og placering af sniffing points. 
<img src="/assets/images/testobj_get_testobj_with_test_results.png" alt="image should have been here">
 

Dette diagram viser det samme som ovenstående i sekvensformat. Det går i dybden i begge services og viser hvordan kommunikationen flyder. 
<img src="/assets/images/testobj_create_sequence.png" alt="image should have been here">




