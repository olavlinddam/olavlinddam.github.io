---
title: Hosting af LeakMonitor infrastruktur
date: 31.08.2023 12:54
categories: [LeakMonitor, Produkt]
tags: [nolek,systemarkitektur,hosting,devops,infrastruktur]
---
Jeg har gjort mig flere forskellige tanker omkring hosting af infrastrukturen. Der er nogle forskellige muligheder, herunder Azure, AWS, UCL, Webdock.io og selvfølgelig lokal hosting af en eller anden art. Der er fordele og ulemper ved dem alle, så valget er ikke helt nemt.

## Azure og AWS

Azure og AWS er oplagte valg, da de begge er meget anvendte i erhvervslivet. Det ville være fedt at lære at bruge dem, mens jeg stadig læser. Jeg ved, at de har alt, hvad jeg overhovedet kunne have brug for, og de har brugergrænseflader, der gør processen nemmere. Desværre er begge løsninger dyre og uden for budget i øjeblikket.

## Webdock.io

Webdock.io er en dansk virksomhed, der udlejer serverkapacitet, og det er et af de billigste alternativer. Kenneth fra UCL har anbefalet mig at bruge dem, da han allerede kender dem og ved, at de er gode til at supportere og hjælpe til. De har også praktikanter fra UCL. Det ville være fedt at stifte bekendtskab med en dansk virksomhed og ad den vej måske udvide mit netværk lidt. Igen må jeg desværre erkende, at det hurtigt bliver meget dyrt. Det afhænger dog af de specifikke krav, jeg ender med at have, f.eks. hvor mange kerner jeg har brug for, at min server har. Det ved jeg ikke nok om endnu til at retfærdiggøre at bruge mange penge på det.

## UCL's servere

Kenneth har været så venlig at stille nogle servere til rådighed for mig. Jeg kan få adgang til 16 kerner, hvilket jeg er helt sikker på ville være nok. Problemet med denne løsning er, at jeg ikke kan logge remote ind på serveren. Jeg er nødt til at være fysisk til stede på UCL, i det laboratorium hvor serveren er, samtidig med at Kenneth er der. Det vil sige, at jeg skal planlægge fuldstændigt efter hans skema, og det bliver nok besværligt for mig, da jeg også arbejder på projektet nogle aftener og nogle gange stopper tidligt om dagen for at hente min datter i vuggestue.

## Lokal hosting

Den sidste løsning er at hoste systemet lokalt. Jeg har en god stationær computer med 6 kerner, 32 GB RAM og en stor harddisk. Det tror jeg er rigeligt til formålet. Jeg kunne godt dedikere en partition på harddisken til at installere en server på. Problemet er, at jeg så kun kan tilgå mine data, når min computer kører herhjemme. Det er lidt bøvlet, hvis jeg f.eks. er i Kolding ved Nolek, men alt i alt en okay fleksibel måde at gøre det på. Problemerne opstår først, når resten af gruppen skal kobles på infrastrukturen. Jeg ved ikke, om det er så smart, hvis de bliver afhængige af, at min computer kører.

## Min beslutning

Først og fremmest må jeg hoste lokalt. Jeg skal have lavet et proof of concept, og jeg skal have lidt flere services op at køre, før jeg begynder at betale for noget. Vi er nødt til at se, hvordan det udvikler sig, også i forhold til hardwarekrav, før jeg ved noget om, præcis hvor dyrt det bliver at hoste. Hvis jeg skal ud og betale for noget, bliver det nok Webdock.io.