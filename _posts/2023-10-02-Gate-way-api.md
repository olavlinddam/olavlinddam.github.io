---
title: API Gateway
date: 02.10.2023 08:00
categories: [Nolek, Microservices]
tags: [nolek,systemdokumentation,gateway,rabbitmq,microservices]
---

Jeg har været frem og tilbage omkring ideen om at implementere et API gateway, som kunne udgøre en ensartet interface 
for vores front end. Nu er jeg kommet til en beslutning om, at den skal med, hvilket også vil gøre hele opsætningen med 
RabbitMQ lettere. Dette gateway vil fungerer som mellemmand mellem de forskellige services, som beskrevet nedenfor:

#### API Gateway Service:
* Modtager Anmodninger fra Klient:
  * Modtager HTTP-anmodninger fra klienter.
  * Omsætter HTTP-anmodninger til beskeder, der sendes til de relevante køer for de underliggende services.
* Sender Svar til Klient:
  * Lytter på svar-køer fra de underliggende services.
  * Omsætter beskeder fra svar-køer til HTTP-responses, der sendes tilbage til klienten.


#### Workflow:
  1. Klient Sender Anmodning:
     * Klienten sender en HTTP-anmodning til API Gateway.
  2. API Gateway Omdirigerer Anmodning:
     * API Gateway modtager anmodningen og placerer en besked på den relevante kø baseret på anmodningens indhold (f.eks. en `leaktest-request kø for anmodninger relateret til lækagetest).
  3. Service Behandler Anmodning:
     * Den relevante service (f.eks. LeakTestService) lytter på sin anmodningskø, behandler anmodningen og sender et svar tilbage på en svar-kø (f.eks. `leaktest-responses).
  4. API Gateway Sender Svar:
     * API Gateway lytter på svar-køen, modtager svaret fra servicen, og sender det tilbage som en HTTP-response til klienten.
