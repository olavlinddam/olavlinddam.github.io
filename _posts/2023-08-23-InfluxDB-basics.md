---
title: InfluxDB basics
date: 23.08.2023 19:48
categories: [Generelt, Faglig indsigt]
tags: [nolek,influxdb,databaser,systemdesign,databasedesign]
---
InfluxDB er en databasetype, der bruges til at gemme tidsserie data. Det største fokus for InfluxData, som er udviklerne af InfluxDb, er, at sikre enormt god performance i forhold til hastighed. Data lagres efter stigende tidsorden og der er strenge restriktioner i forhold til at opdatere og slette data. Generelt set slettes eller opdateres data ikke, men i stedet gemmes der blot et nyt "point" (se nedenfor). En måling har ikke et Id som man traditionel forstand med differentieres i stedet på tidsstempler.

## Dataelementer
InfluxDB arbejder med nogle data elementer som er beskrevet nedenfor.

* **Timestamp:** Alle data gemt i influx har en `_time` kolonne der gemmer tidsstempel.
* **Measurement:** Container for tags, fields og `_time` kolonnen. `_measurement` er beskrivelsen af det data der findes i containeren. Dvs. at hvis vi skriver "LeakTest" i `_measurement` kolonnen, så er det den overordnede beskrivelse af det data der følger i rækken.
* **Field Key**: En string der repræsenterer navnet på det field.
* **Field Value**: Værdien af et associeret field. Kan være strings, floats, ints og bools. Ikke indekseret - lang søgetid.
* **Field Set**: En samling af field key-value par associeret med et tidsstempel.
* **Tags**: Indekserede = hurtige forespørgelser. Der findes tag key, tag value og tag set som ved fields. Det kunne f.eks. sådan noget som `user`,`test_object` og `sniffing_point`. Bruges ofte til at gruppere og filtrere efter. Skal være string.
* **Point:** En måling. Det kan med nogle undtagelser oversættes til en række i en almindelig relationel database. Et point er altid unikt. Hvis man indsætter et point, med samme værdier som et allerede eksisterende point, overskrives det første point af det nye. Et point har et tidsstempel.
* **Bucket:** Al influx data gemmes i en bucket og en bucket er tilknyttet en organisation.
* **Organization**: I InfluxDB er en organisation en gruppe af brugere, lidt ligesom man kender det fra GitHub.


## Series and Series Keys
En series key er en samling af punkter som deler følgende egenskaber:
* \_measurement
* tag set
* \_field

```js
// series key template:
<_measurement>,<tag_key>=<tag_value> <field_value>

// series key example:
LeakTest,test_object=object1,status=ok user1
```
Ovenstående eksempel kunne give følgende serie:

| Timestamp      | Measurement | Tag Key (test_object) | Tag Key (status) | Field Key (user) | Field Key (sniffing_point) | Field Key (reason) |
|----------------|-------------|-----------------------|------------------|------------------|----------------------------|--------------------|
| 2023-08-23T10:00:00Z | LeakTest    | object1                     | OK               | user1            | point1                     |                    |
|2023-08-23T10:01:00Z|LeakTest|object1|OK|user1|point2|

Her er der altså to punkter der matcher vores series key.

### Hvorfor er serier vigtige?
1. **Ydelse:** Det er vigtigt at forstå hvordan serier virker for at kunne optimere forespørgelser.
2. **Kardinalitet**: Antallet af unikke serier i en InfluxDB kaldes kardinaliteten. Høj kardinalitet kan påvirke databasen ydelse. Man bør strukturere sine tags og målinger så man undgår for høj kardinalitet.
3. **Organisering:** At forstå serier hjælper på forståelsen af hvordan ens data er struktureret og lagret.

Serier er altså den måde InfluxDB opbevarer og strukturere data.

### At query InfluxDB
Når man vil query en InfulxDB kan man anvende data-sproget Flux. Det kan nedbrydes i følgende:


1. Definer bucket:
``` js
from(bucket:LeakMonitor)
```
2. Specificer tidsserien
```js
// relativ tidsserie uden stop.
from(bucket:LeakMonitor)
|> range(start:-1h)


// relativ tidsserie med start og stop.
from(bucket:LeakMonitor)
|> range(start:-1h, stop -10m)
```
Man kan ligeledes query for absolutte tidsrum ved at skrive det absolutte tidspunkt frem for et relativt.

Når man har sin bucket og sit tidsrum kan man filtrere og gruppere det data, så man kun får det ønskede. Det netop hentede data kan bruges som input i `filter()` metoden, som kun har parameteren `fn`. `fn` forventer en [[predicate function]] og evaluere hver rækkes kolonneværdier. Hver række bliver så et input i `filter()` metoden som `r`  hvor det evalueres med predicate expressions. Rækker som evalueres til `false` droppes fra det data som bliver til vores output og rækker som evalueres til `true` beholdes.

```js
// det mønster som anvendes:
(r) => (r.recordProperty comparisonOperator comparisonExpression)

// Example with single filter
(r) => (r._measurement == "LeakStatus")

// Example with multiple filters
(r) => r._measurement == "LeakStatus" and r.test_object == "test_object1")
```

I dette eksempel filtrerer vi på LeakStatus som værende det vi måler på og vi vil gerne finde de målinger der findes for test_object1 hvor statussen for et sniffing point var "NOK". Vi definerer et tidsrum på den seneste uge.

```js
from(bucket:"example-bucket")
  |> range(start: -7d)
  |> filter(fn: (r) =>
    r._measurement == "LeakStatus" and
    r.test_object == "test_object1" and
    r.status == "NOK"
  )yield()
```

Influx kalder automatisk funktionen `yield()` som outputter resultatet af forespørgelsen. Hvis man laver flere forespørgelser i samme Flux query skal man kalde `yield()`explicit til sidst i ens query.
at ud fra seriekardinalitet er det ovenstående skema godt.
### Data layout og databaseskemadesign best practices
Når det kommer til at optimere ressourceforbruget af ens InfluxDB, og til at forbedre ingestion rate - hvor hurtigt data kan gemmes i databasen - er det nødvendigt at overveje ens databaseskema. Influxdata anbefaler følgende retningslinjer:

1. Metadata bør kodes som tags, især metadata som man ofte vil query, da tags er indekseret.
2. Begræns antal serier og seriers kardinalitet.
3. Hold bucket og measurement navne korte og simple.
4. Data skal ikke kodes ind i measurement navne.
5. Separer data i forskellige buckets hvis der skal være forskellige "retention policies", som refererer til hvor længe data gemmes før det slettes, på forskelligt data.

##### Serie kardinalitet
Hvis vi starter med at se på seriekardinalitet, så refererer det til antallet af unikke kombinationer af bucket+measurement+tag sets+field keys, der eksisterer i en organisation. Selvom InfluxDB er designet til at håndtere høj seriekardinalitet, er det stadig et sted hvor man kan forbedre ydeevnen. Hvordan beregner man så seriekardinaliteten? Lad os tage et eksempel med et data layout som det kunne se ud for LeakStatus servicen(link til artikel) i LeakMonitor projektet.

|Bucket|LeakStatus|
|-------|-----------|
|Measurement|LeakTest|
|Tag Keys|status, test_object |
|Field Keys|user, sniffing_point, reason|

I projektet med Nolek ved vi, at vi altid kun vil have 2 statusser, men antallet af testobjekter kan stige uendeligt højt. Lad os antage følgende:

* **Status:** 2 værdier
* **Testobject:** 100 værdier


I forhold til InfluxDB's kapacitet er 600 ikke en høj kardinalitet. Der er dog nogle ting som kan påvirke kardinaliteten. Hvad skal der hvis vi tilføjer 200 testobjekter mere? Så ender kardinaliteten på 1800. Tilføjer vi 10.000 testobjekter ender vi på 60.000. Selvom det er 100 gange flere end den oprindelige antagelse, er vi stadig et godt stykke inden for rammen af hvad InfluxDB kan håndtere. Jeg skal ikke kunne sige hvor mange forskellige testobjekter en virksomhed kan komme op på, men jeg tillader mig den antagelse, at 10.000 er et højt tal. Ovenstående databaseskema kan altså anvendes uden bekymring for høj seriekardinalitet.

Man bør også se på ingest rate, altså hvor mange målinger pr. sekund databasen kan håndtere. Antal målinger pr. sekund afhænger af størrelsen af hver måling. Derfor bør man ikke inkludere redundant information i sine buckets. Der er f.eks. ikke nogen grund til at inkludere brugeren email hver gang der skrives en måling. I projektet med Nolek, hvor det er en bruger der manuelt skal eksekvere dataskrivelsen, kommer vi dog aldrig i nærheden af InfluxDB's kapacitet.

På [linket her](https://docs.influxdata.com/influxdb/v2.7) kan man finde meget mere information om InfluxDB.
