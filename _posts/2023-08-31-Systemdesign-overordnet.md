---
title: Overordnet systemdesign
date: 31.08.2023 10:37
categories: [LeakMonitor, Systemdokumentation]
tags: [nolek,systemdesign,softwareudvikling,microservices,systemarkitektur]
---

Vores design ser ud som følger.

## Domænemodel
##### Klasser og properties
* **Testobjekt**
	* Layout 
	* Navn
	* Beskrivelse
	  
* **Sniffing Point** 
	* Navn
	  
* **Begrundelse**
	* Tekst
	* Ændringstidspunkt
	  
* **Testresultat**
	* Status
	* Tidspunkt
	  
* **Bruger**
	* Brugernavn
	* Password
	* Email
	* Rolle


  <img src="/assets/images/domainmodel.png" alt="Image should have been here.">


## Overordnet systemdiagram

Her illustreres snitfladerne mellem de forskellige systemer gruppen har i spil. Jeg arbejder selv på den microservice cluster som ses i nederste venstre del af billedet og vores den infrastruktur som systemet skal køre på. Det gælder vores API gateway og den messagebroker som skal håndtere kommunikationen mellem alle de services vi ender med at have. 

  <img src="/assets/images/overordnet_systemdiagram.png" alt="Image should have been here.">

Det samlede system indeholder altså:
* Single Page Application der skal fungere som administratorenes side.
* App som er den del brugeren anvender.
* Firewall/proxy, som bliver et beskyttelseslag mellem brugergrænseflader og vores services. 
* Security prediction service. En machine learning model der kan opsnappe potentielle sikkerhedstrusler.
* Leak prediction service. En machine learning model der kan forudsige hvilke testobjekter/sniffing points der har høj risiko for at have svage punkter.
* Microservice cluster, som er min del og som indledningsvist vil bestå af forskellige services, herunder:
	* Lækagetest service, til håndtering af information om alle de tests brugerne udfører.
	* Logging service, til håndtering af log info.
	* Data backup service. 
* Messagebroker og API gateway, som også er min del og skal stå for at udjævne kommunikationen mellem de forskellige services og brugergrænsefladerne. 
