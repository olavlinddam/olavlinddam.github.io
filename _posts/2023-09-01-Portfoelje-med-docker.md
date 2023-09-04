---
title: Portfølje i en Docker container
date: 01.09.2023 10:55
categories: [Generelt, Portfølje]
tags: [docker,portfølje,containerization,dockerfile,jekyll,"github pages"]
---

Efter lang tids problemer med hosting af min portfølje på GitHub har jeg valgt at containerize min portfølje med Docker. Det betyder, at hele det setup, der er nødvendigt for at køre et Jekyll-site på GitHub Pages, nu ligger i en Dockerfile. Jeg vil kort gennemgå processen.  

## Det nødvendige setup
For overhovedet at kunne bruge Jekyll er der nogle forudsætninger, man skal have styr på. Man skal have et operativsystem, programmeringssproget Ruby, pakkesystemet RubyGems, GNU Compiler Collection og Make. Alle disse ting skal man installere på sin computer. Man kan enten installere dem direkte på computeren, eller man kan installere dem i en Docker-container. Hvis man vælger den sidste mulighed, skal vi have et Docker-image.

## Dockerfile
Når man vil bygge et Docker-image, skal man bruge en Dockerfile. En Dockerfile er en tekstfil, der indeholder de instruktioner, der er nødvendige for at bygge et Docker-image. Det inkluderer specificering af, hvilket basisbillede man vil bruge, samt miljøvariabler, forskellige RUN-kommandoer osv. Man kan tilpasse dem, så de passer specifikt til ens formål, eller man kan finde dem på nettet, hvis nogen har lavet en, der passer til ens use-case. Min Dockerfile ser sådan ud:

```Dockerfile

# "#################################################"
# Dockerfile to build a GitHub Pages Jekyll site
# - Ubuntu 22.04
# - Ruby 3.1.2
# - Jekyll 3.9.3
# - GitHub Pages 288
#
# This code is from the following Gist:
# https://gist.github.com/BillRaymond/db761d6b53dc4a237b095819d33c7332#file-post-run-txt
#
# Instructions:
# 1. Copy all the text in this file
# 2. Create a file named Dockerfile and paste the code
# 3. Create the Docker image/container
# 4. Locate the shell file in this Gist file and run it in the local repo's root
# "#################################################"

FROM ubuntu:22.04

# "#################################################"
# "Get the latest APT packages"
# "apt-get update"

RUN apt-get update

# "#################################################"
# "Install Ubuntu prerequisites for Ruby and GitHub Pages (Jekyll)"
# "Partially based on https://gist.github.com/jhonnymoreira/777555ea809fd2f7c2ddf71540090526"

RUN apt-get -y install git \
curl \
autoconf \
bison \
build-essential \
libssl-dev \
libyaml-dev \
libreadline6-dev \
zlib1g-dev \
libncurses5-dev \
libffi-dev \
libgdbm6 \
libgdbm-dev \
libdb-dev \
apt-utils \
nano

# "#################################################"
# "GitHub Pages/Jekyll is based on Ruby. Set the version and path"
# "As of this writing, use Ruby 3.1.2
# "Based on: https://talk.jekyllrb.com/t/liquid-4-0-3-tainted/7946/12"
ENV RBENV_ROOT /usr/local/src/rbenv
ENV RUBY_VERSION 3.1.2
ENV PATH ${RBENV_ROOT}/bin:${RBENV_ROOT}/shims:$PATH

# "#################################################"
# "Install rbenv to manage Ruby versions"

RUN git clone https://github.com/rbenv/rbenv.git ${RBENV_ROOT} \
&& git clone https://github.com/rbenv/ruby-build.git \
${RBENV_ROOT}/plugins/ruby-build \
&& ${RBENV_ROOT}/plugins/ruby-build/install.sh \
&& echo 'eval "$(rbenv init -)"' >> /etc/profile.d/rbenv.sh

# "#################################################"
# "Install ruby and set the global version"

RUN rbenv install ${RUBY_VERSION} \
&& rbenv global ${RUBY_VERSION}
  
# "#################################################"
# "Install the version of Jekyll that GitHub Pages supports"
# "Based on: https://pages.github.com/versions/"
# "Note: If you always want the latest 3.9.x version,"
# " use this line instead:"
# " RUN gem install jekyll -v '~>3.9'"

# Copy the existing Jekyll project and Gemfile in to the container
COPY . /my-jekyll-site

# Set working directory
WORKDIR /my-jekyll-site

# Install bundle and run the install command
RUN gem install bundler
RUN bundle install

# Command to run when the container starts. This serves the jekyll site when we run the container.
CMD ["bundle", "exec", "jekyll", "serve", "--host", "0.0.0.0"]

```

Jeg ville gerne tage æren for Dockerfilen, men det er ikke mig, der har skrevet størstedelen af den. En fyr ved navn Bill Raymond har lavet en god [guide](https://www.youtube.com/watch?v=zijOXpZzdvs&t=1901s) på YouTube, og det meste af Dockerfilen er hans. Han har gjort en del forarbejde med at undersøge, hvilken version af Ubuntu og Ruby der er bedst til at køre Jekyll på, samt hvilke afhængigheder Ubuntu-operativsystemet skal have. Det er kun de sidste fem kommandoer, jeg har modificeret eller tilføjet. I Bill Raymonds guide laver han et nyt projekt. Jeg skulle have mit gamle med over, så derfor kopierer jeg de gamle filer med over i containeren, specificerer arbejdsområdet og installerer bundler, som skal bruges til at kompilere alle mine pakker. Den sidste kommando er den kommando, der skal køres, når containeren er oppe at køre.

## Fra Dockerfile til container - at bygge og køre et jekyll site lokalt. 
Når man har sin Dockerfile på plads, kan man bygge sit Image ved at køre kommandoen:

```bash
docker build -t <IMAGE_ID_OR_NAME]> . 
```

Og for at køre en container, med specifik port-mapping og et volume til at persistere ændringer, baseret på det image:

```bash
docker run -d --name [NEW_CONTAINER_NAME] -p [PORT:PORT] --user [UID]:[GID] -v $(pwd):/my-jekyll-site [IMAGE_ID_OR_NAME]
```

I mit tilfælde vil jeg gerne skrive til en mappe, der kræver rwx-rettigheder, og det er derfor, jeg også inkluderer --user [UID]:[GID] i kommandoen. Hvis man ikke skal skrive til en mappe, der kræver disse rettigheder, kan man udelade den del.

Når det er gjort, har man faktisk en kørende container. Den container, man så har lavet, kan man stoppe med docker stop [CONTAINER_NAME_OR_ID] og starte med docker start [CONTAINER_NAME_OR_ID], uden at man behøver at specificere volume og porte igen. Hvis man sletter sin container og vil starte en ny, skal man køre hele den lange kommando igen. Det næste, der skal ske, er, at man skal "ind i" containeren, så man kan arbejde på projektet derfra.

Det gøres således i en terminal:

```bash
docker exec -it <containerID> /bin/bash
```

Hvor "-d" gør, at processen kører i baggrunden, "-it" gør sessionen interaktiv, og "/bin/bash" er den shell, vi vil bruge. Nu er vi inde i containeren. Vi kan tjekke det ved at se, om brugernavnet i vores shell har ændret sig.

```bash
[root@username ~]$
root@container_id:~#
```

Hvor den første linje er den normale bash, og linje to er containerens bash. Nu kan vi skrive, slette eller redigere de markdown-filer, som bliver til indlæg. Vi kan også åbne containeren i VS Code ved at bruge Devcontainers og Docker-udvidelser. Ændringer skrives automatisk til min lokale mappe med git-konfigurationen til mit GitHub Pages-repository, så jeg kan arbejde med git fra min lokale maskine og skubbe alle ændringer til mit repository på en sikker måde.

## Det smarte i at køre det hele i en container

De fleste af os installerer bare alle de ting, vi skal bruge, direkte på vores computer. Det er det nemmeste og ofte det hurtigste. Men når man over tid arbejder på forskellige projekter, får man hurtigt hentet en masse forskellige sprog, værktøjer, afhængigheder og alt muligt andet ned. Det kan på et tidspunkt blive svært at vedligeholde en computer, der er lige så hurtig, som den en gang var, og det kan blive irriterende i længden at arbejde på en sløv maskine. Docker-containere kan afhjælpe dette problem ved at lade dig installere alle disse ting i en container i stedet for direkte på din computer. På den måde kan man have flere forskellige udviklingsmiljøer liggende, uafhængigt af hinanden, i forskellige containere. Jeg kunne faktisk have hele det udviklingsmiljø, der skal til for at udvikle semesterprojektet, liggende i en container. Det er smart, og jeg overvejer at gøre det, for så kan jeg være sikker på, at jeg har det nøjagtigt samme miljø på alle maskiner, så længe de kan køre Docker.