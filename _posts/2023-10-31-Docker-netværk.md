---

---

## hvad er docker netværk

## kommando til at bygge image uden cache. 
docker build -f TestObjectService/Dockerfile --force-rm --no-cache -t testobjectservice:1-standalone .

her specificerer vi hvor dockerfilen er og bruger --force-rm 

## kommando til at starte en container der kobler sig på et docker netværk.
docker run -d --name testobjectservice-standalone -e ASPNETCORE_ENVIRONMENT=docker --network=leak-monitor  testobjectservice:1-standalone

