---
title: LeakTest Service statusopdatering
date: 12.09.2023 10:00
categories: [Nolek, Microservices]
tags: [nolek,microservices,server,leaktest,hosting,infrastruktur,influxdb]
---
Nu har jeg efterhånden arbejdet på LeakTest service i et par uger, og det viser sig at være en stor mundfuld.
Jeg er blevet overrasket over, hvor lang tid det tager at udvikle og teste en service på egen hånd. Der er mange 
overvejelser der går ind i det og det er bl.a. nogle af dem jeg vil dokumentere her.

## Design
Som beskrevet i [dette post](https://olavlinddam.github.io/posts/Design-af-leaktest-service/), har jeg valgt en arkitektur
med controller, repository og model. Det giver en klar adskillelse af ansvar og gør det lettere at vedligeholde koden.

### Modellen og valideringsklasse
Modellen er skabt ud fra vores [domænemodel](https://olavlinddam.github.io/posts/Systemdesign-overordnet/), hvor det er 
entiteten "Testresultat" som her er repræsenteret. 

```c#
using InfluxDB.Client.Core;

namespace LeakTestService.Models;

[Measurement("LeakTest")]
public class LeakTest
{
    [Column(IsTimestamp = true)] public DateTime? TimeStamp { get; set; }

    // tags 
    [Column("test_object", IsTag = true)] public string TestObjectId { get; set; }
    [Column("status", IsTag = true)] public string Status { get; set; }
    [Column("machine_id", IsTag = true)] public string MachineId { get; set;  }
    [Column("test_object_type", IsTag = true)] public string TestObjectType { get; set; }

    // fields
    [Column("user")] public string User { get; set; }
    [Column("sniffing_point")] public string SniffingPoint { get; set; }
    [Column("reason")] public string? Reason { get; set; }

    public string? Measurement { get; set; }
}
```
Til at validere modellen bruger jeg Fluent Validation:

```c#
public class LeakTestValidator : AbstractValidator<LeakTest>
{
    public LeakTestValidator()
    {
        RuleFor(x => x.TimeStamp)
            .NotNull().WithMessage("TimeStamp is required and can not be null.")
            .Must(x => x <= DateTime.Now).WithMessage("TimeStamp cannot be a future date.");


        RuleFor(x => x.TestObjectId)
            .NotEmpty().WithMessage("TestObject is empty")
            .Must(IsValidGuid).WithMessage("TestObject must be a valid GUID");
        
        
        RuleFor(x => x.Status)
            .NotEmpty().WithMessage("Status can not be empty.")
            .Matches("^(OK|NOK)$").WithMessage("Status must be either OK or NOK");
        
        RuleFor(x => x.User)
            .NotEmpty().WithMessage("There must be a user associated with a LeakTest.")
            .Must(IsValidGuid).WithMessage("User must be a valid GUID.");


        RuleFor(x => x.SniffingPoint)
            .NotEmpty().WithMessage("SniffingPoint cannot be empty.")
            .NotNull().WithMessage("SniffingPoint cannot be null.")
            .Custom((value, context) =>
            {
                if (string.IsNullOrWhiteSpace(value))
                {
                    context.AddFailure("SniffingPoint should not be whitespace.");
                }
                else if (!Regex.IsMatch(value, @"^[a-zA-Z0-9-_]+$"))
                {
                    context.AddFailure("SniffingPoint can only contain alphanumeric characters, hyphens, and underscores.");
                }
            })
            .Length(1, 999).WithMessage("SniffingPoint must have a length between 1 and 999 characters.");

        RuleFor(x => x.Measurement)
            .Equal("LeakTest").WithMessage("The measurement for LeakTest objects must be 'LeakTest'.");
    }
    
    
    private bool IsValidGuid(string value)
    {
        // Checking if the GUID is valid and dashed ("-")
        return Guid.TryParse(value, out var guid) && value == guid.ToString();    }
    }
}
```
Det er en forholdsvist omfattende validering, men det er nødvendigt da der er strenge krav til datatyper i InfluxDB.
Her kan tags f.eks. kun være strings, hvilket også er derfor de er repræsenteret strings i entiteten. De skal så valideres
så vi er sikre på at de rent faktisk er valide GUIDs. 

### LeakTestController
I controlleren findes de endpoints som resten af systemet kan forbinde til.

<img src="/assets/images/LeakTestController-180923.png" alt="image should have been here">


De er alle opbygget med try-catch statements, hvor modellen valideres, enten når den kommer ind i controlleren via kaldet 
fra klienten eller fra repositoriet. Dette gøres for at sikre at det som vi skriver til databaen eller til klienten overholder
de nødvendige krav, både ift. overhovedet at kunne skrive det til databasen og ift. at klienten får præcis det de forventer.
Der er naturligvis også de sikkerhedsmæssige overvejelser ifm. validering. Her er der et eksempel på en af metoderne:

```c#
[HttpPost]
public async Task<IActionResult> AddSingleAsync([FromBody] LeakTest leakTest)
{
    try
    {
        leakTest.User = leakTest.User.ToUpper();
        leakTest.Status = leakTest.Status.ToUpper();
        
        // Creating the validator and validating the LeakTest object.
        var validator = new LeakTestValidator();
        var validationResult = await validator.ValidateAsync(leakTest);
        
        // setting the id of the leaktest. 
        leakTest.LeakTestId = Guid.NewGuid();
        
        if (!validationResult.IsValid)
        {
            return BadRequest($"LeakTest object could not be validated: {string.Join(", ", validationResult.Errors.Select(e => e.ErrorMessage))}");
        }
        
        // Posting the LeakTest object as a point in the database. 
        await _leakTestRepository.AddSingleAsync(leakTest);
        
        // Returner en 201 Created statuskode og en Location header
        return CreatedAtAction(nameof(GetById), new { id = leakTest.LeakTestId }, null);
    }
    catch (Exception e)
    {
        // Log the exception here
        return BadRequest($"The request could not be processed due to: {e.Message}");
    }
}
```

Logging er endnu ikke implementeret i metoderne. Det kommer når jeg har opbygget LoggingService.

### Repository & Converter
Jeg har kun et enkelt repository i denne service, men nu er mønsteret på plads, hvilket gør det lettere at udvide 
koden med flere hvis det bliver nødvendigt. 

```c#
public async Task AddSingleAsync(LeakTest leakTest)
{
    try
    {
        await _client.GetWriteApiAsync().WriteMeasurementAsync(leakTest);
    }
    
    catch (InfluxException influxException)
    {
        throw new InfluxException("Could not save data", influxException);
    }
    catch (Exception ex)
    {
        throw new LeakTestRepositoryException("An unexpected error occured", ex);
    }
}
```

Ovenstående er den videre implementering af "AddSinglePointAsync" endpointet fra controlleren. Når man skriver til en
InfluxDb server kan man bruge "WriteMeasuermentApi" endpoint som InfluxDb eksponerer. Når man skriver et query for at hente 
data kan man enten gøre det med et Flux query eller ved brug af LINQ som herunder:

```c#
public async Task<IEnumerable<LeakTest>> GetAllAsync()
{
    try
    {
        // Creating a Task so we can run the method async.
        return await Task.Run(() =>
        {
            // Init the LeakTestConverter, which implements IDomainObjectMapper, and using it as input for GetQueryApi
            var converter = new LeakTestConverter();
        
            // using var client = _client;
            var queryApi = _client.GetQueryApiSync(converter);
    
            // Creating an instance of QueryableOptimizerSettings to enable Measurement Column
            var optimizerSettings = new QueryableOptimizerSettings()
            {
                DropMeasurementColumn = false
            };
    
            // Creating the query to pull all points from the specified bucket and mapping each to a LeakTest objects
            var query = from t in InfluxDBQueryable<LeakTest>
                    .Queryable(_config.Bucket, _config.Org, queryApi, converter, optimizerSettings)
                select t;
        
            var leakTests = query.ToList();

            return leakTests;
        });
    }
    catch (Exception e)
    {
        throw new BadHttpRequestException($"The request could not be processed. {e.Message}");
    }
    }
```

Det gør det ret intuitivt at arbejde med InfluxDb hvis man er vant til LINQ i forvejen. Det kræver dog at man opretter 
en converter klasse som håndterer mapping fra FluxRecord til entitet og omvendt.

```c#
using System.Reflection;
using InfluxDB.Client;
using InfluxDB.Client.Api.Domain;
using InfluxDB.Client.Core.Flux.Domain;
using InfluxDB.Client.Linq;
using InfluxDB.Client.Writes;
using LeakTestService.Models;

namespace LeakTestService.Converters;

public class LeakTestConverter : IDomainObjectMapper, IMemberNameResolver
{
    /// <summary>
    /// This class is used to map from a LeakTest object to a Point Data representation of that object, or to map
    /// from a Flux Record to a LeakTest object. 
    /// </summary>
    /// <param name="fluxRecord">A representation of a point in an InfluxDb. This representation must contain all the
    /// required information to populate a LeakTest object</param>
    /// <typeparam name="T">An object of type LeakTest</typeparam>
    /// <returns>Either a LeakTest object when mapping from a Flux Record, or a Point Data representation of a LeakTest
    /// object when mapping from a LeakTest object.</returns>
    public T ConvertToEntity<T>(FluxRecord fluxRecord)
    {
        return (T)ConvertToEntity(fluxRecord, typeof(T));
    }

    public object ConvertToEntity(FluxRecord fluxRecord, Type type)
    {
        if (fluxRecord == null)
        {
            throw new ArgumentNullException(nameof(fluxRecord));
        }

        if (type != typeof(LeakTest))
        {
            throw new NotSupportedException($"This converter doesn't support: {type}");
        }
        
        try
        {
            var leakTest = new LeakTest();

            leakTest.TimeStamp = fluxRecord.GetTime().GetValueOrDefault().ToDateTimeUtc().ToLocalTime();
            leakTest.Measurement = fluxRecord.GetValueByKey("_measurement")?.ToString();
            leakTest.MachineId = Guid.Parse(fluxRecord.GetValueByKey("MachineId")?.ToString());
            leakTest.Status = fluxRecord.GetValueByKey("Status")?.ToString();
            leakTest.TestObjectId = Guid.Parse(fluxRecord.GetValueByKey("TestObjectId")?.ToString());
            leakTest.TestObjectType = fluxRecord.GetValueByKey("TestObjectType")?.ToString();
            leakTest.LeakTestId = Guid.Parse(fluxRecord.GetValueByKey("LeakTestId").ToString());
            leakTest.SniffingPoint = fluxRecord.GetValueByKey("SniffingPoint")?.ToString();
            leakTest.User = fluxRecord.GetValueByKey("User")?.ToString();
            leakTest.Reason = fluxRecord.GetValueByKey("Reason")?.ToString() ?? null;
            
            

            return Convert.ChangeType(leakTest, type);
        }
        catch (Exception e)
        {
            // throw new Exception(
            //     $"There was an error converting the record with timestamp: {fluxRecord.GetTime()} - {e.Message}");
            throw new Exception(e.Message);
        }
    }


    public PointData ConvertToPointData<T>(T entity, WritePrecision precision)
    {
            if (!(entity is LeakTest ce))
            {
                throw new NotSupportedException($"This converter doesn't support: {typeof(T)}");
            }
            try
            {
                var point = PointData
                    .Measurement(ce.Measurement)
                    .Tag("TestObjectId", ce.TestObjectId.ToString())
                    .Tag("Status", ce.Status)
                    .Tag("MachineId", ce.MachineId.ToString())
                    .Tag("TestObjectType", ce.TestObjectType)
                    .Tag("User", ce.User.ToString())
                    .Field("SniffingPoint", ce.SniffingPoint)
                    .Field("Reason", ce.Reason ?? null)
                    .Field("LeakTestId", ce.LeakTestId.ToString())
                    .Timestamp(ce.TimeStamp, precision);

                return point;
            }
            catch (Exception e)
            {
                // Log here if necessary
                throw new Exception(e.Message);
            }
    }
    

    public MemberType ResolveMemberType(MemberInfo memberInfo)
    {
        return memberInfo.Name switch
        {
            "TimeStamp" => MemberType.Timestamp,
            "TestObjectId" => MemberType.Tag,
            "Status" => MemberType.Tag,
            "MachineId" => MemberType.Tag,
            "TestObjectType" => MemberType.Tag,
            "User" => MemberType.Tag,
            _ => MemberType.Field
        };
    }

    public string GetColumnName(MemberInfo memberInfo)
    {
        return memberInfo.Name.ToLower();
    }

    public string GetNamedFieldName(MemberInfo memberInfo, object value)
    {
        return memberInfo.Name.ToLower();
    }
    
    public List<PointData> ConvertLeakTestsToPoints(IEnumerable<LeakTest> entities, WritePrecision precision)
    {
        return entities.Select(entity => ConvertToPointData(entity, precision)).ToList();
    }
}
```
Klassen er større men det giver en god ide om hvordan den overordnet opererer.

### Tidszoner
InfluxDb gemmer som standard tidsstempler som UTC, derfor er jeg nødt til at kalde `.ToDateTimeUtc()` på tidsstemplet i 
metoden ovenfor. Da vi ikke befinder os indenfor den tidszone omskriver jeg den så igen til lokal tid, så brugeren
får tidspunktet i et format der passer med vedkommenes egen tidszone, så længe brugeren befinder sig i samme tidszone som 
serveren.

### Test
Indtil videre har jeg skrevet Unit Tests for converteren, valideringslogikken og mappingen til og fra JSON.

<img src="/assets/images/LeakTestUnitTests-140923.png">

Min plan er at mock mit repo og teste metoderne på den måde, ved at bruge NSubstitute. I forhold til controlleren og endpoints
findes der testværktøjer som f.eks. "TestServer", men jeg har ikke sat mig ind i det endnu. 
Mine endpoints og repo metoder er indtil videre kun testest manuelt via Swagger.

### Dokumentation
Jeg vil dokumentere med postman eller swagger. Det er under udarbejdelse lige i øjeblikket.
