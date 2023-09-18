---
title: RESTful API
date: 12.09.2023 10:00
categories: [Nolek, Microservices]
tags: [nolek,microservices,leaktest,api]
---

Vi havde i fredags, d. 15 september et møde med Nolek, hvor jeg fremviste fremgangen med LeakTest servicen. De havde mange
gode pointer, bl.a. at jeg skulle overveje designprincipper for API'er. Efter at have set på det, kan jeg se at der er nogle
klare forbedringspunkter, herunder:

1. Brug af navneord fremfor udsagnsord til endpoints.
2. Metodenavne bør afspejle de nye endpoint navne.
3. Flere kommentarer som bedre forklarer hvad metoderne gør, hvilke parametre de tager og hvilken type respons de returnerer.
4. Responsformatet kan forbedres, så det returnerer en 201 created med en pointer til den lokation hvor ressourcen kan findes.
Det mindsker også dataoverførslen, da der så ikke skal returneres objekter men blot en pointer. 

Med ovenstående in mente har jeg revideret min controller fra:

Fra:

```c#
// usings...

namespace LeakTestService.Controllers;

[ApiController]
[Route("api/[controller]")]
public class LeakTestController : ControllerBase
{
    private readonly ILeakTestRepository _leakTestRepository;

    public LeakTestController(ILeakTestRepository leakTestRepository)
    {
        _leakTestRepository = leakTestRepository;
    }

    
    [HttpPost("AddRangeOfPointsAsync")] 
    public async Task<IActionResult> AddRangeOfPointsAsync([FromBody] List<LeakTest> leakTests)
    {
        try
        {
            if (leakTests == null || !leakTests.Any())
            {
                return BadRequest("The request body was null or empty.");
            }

            var validator = new LeakTestValidator();
            var validationErrors = new List<string>();

            foreach (var leakTest in leakTests)
            {
                var validationResult = await validator.ValidateAsync(leakTest);
                if (!validationResult.IsValid)
                {
                    validationErrors.AddRange(validationResult.Errors.Select(e => e.ErrorMessage));
                }
            }

            if (validationErrors.Any())
            {
                return BadRequest($"Some LeakTest objects could not be validated: {string.Join(", ", validationErrors)}");
            }

            await _leakTestRepository.AddRangeOfPointsAsync(leakTests);

            return Ok($"Data received successfully: {JsonConvert.SerializeObject(leakTests, Formatting.Indented)}");
        }
        catch (Exception e)
        {
            // Log the exception here
            return BadRequest($"The request could not be processed due to: {e.Message}");
        }
    }

    [HttpPost("AddSinglePointAsync")]
    public async Task<IActionResult> AddSinglePointAsync([FromBody] LeakTest leakTest)
    {
        try
        {
            // Creating the validator and validating the LeakTest object.
            var validator = new LeakTestValidator();
            var validationResult = await validator.ValidateAsync(leakTest);
            
            if (!validationResult.IsValid)
            {
                return BadRequest($"LeakTest object could not be validated: {string.Join(", ", validationResult.Errors.Select(e => e.ErrorMessage))}");
            }
            
            // Posting the LeakTest object as a point in the database. 
            await _leakTestRepository.AddSinglePointAsync(leakTest);
            
            return Ok($"{JsonConvert.SerializeObject(leakTest, Formatting.Indented)}");
        }
        catch (Exception e)
        {
            // Log the exception here
            return BadRequest($"The request could not be processed due to: {e.Message}");
        }
    }
    
    
    [HttpGet("GetAllPointsAsync")]
    public async Task<IActionResult> GetAllPointsAsync()
    {
        try
        {
            // Create and fill the list of LeakTest objects.  
            var leakTests = await _leakTestRepository.GetAllPointsAsync() as List<LeakTest>;
            
            // Check if there are any objects in the LeakTest list and if not, return NoContent error message. 
            if (leakTests == null) return NoContent();

            // Validate the objects
            var validator = new LeakTestValidator();
            
            foreach (var leakTest in leakTests)
            {
                var validationResult = await validator.ValidateAsync(leakTest);
                if (!validationResult.IsValid)
                {
                    return BadRequest(
                        $"LeakTest object could not be validated: {string.Join(", ", validationResult.Errors.Select(e => e.ErrorMessage))}");
                }
            }

            // Convert to JSON format
            var jsonPayload = JsonConvert.SerializeObject(leakTests, Formatting.Indented);

            // Return the JSON payload
            
            return Ok(jsonPayload);
        }
        catch (Exception e)
        {
            return StatusCode(500, $"Internal server error: {e.Message}");
        }
    }

    [HttpGet("GetPointsWithinTimeRangeAsync")] // Start and stop values must be valid DateTimes, input as strings.
    public async Task<IActionResult> GetPointsWithinTimeRangeAsync(DateTime start, DateTime? stop)
    {
        try
        {
            // Create TimeRange object
            var timeRange = new TimeRange(start, stop);
            
            // Validate the TimeRange object
            var timeRangeValidator = new TimeRangeValidator();
            var timeRangeValidationResult = await timeRangeValidator.ValidateAsync(timeRange);

            if (!timeRangeValidationResult.IsValid)
            {
                return BadRequest(
                    $"Time range could not be validated: {string.Join(", ", timeRangeValidationResult.Errors.Select(e => e.ErrorMessage))}");
            }
            
            // Create and fill the list of LeakTest objects.  
            var leakTests = await _leakTestRepository.GetPointsWithinTimeRangeAsync(timeRange) as List<LeakTest>;
            
            // Check if there are any objects in the LeakTest list and if not, return NoContent error message. 
            if (leakTests == null) return NoContent();

            // Validate the objects
            var leakTestValidator = new LeakTestValidator();
            
            foreach (var leakTest in leakTests)
            {
                var leakTestValidationResult = await leakTestValidator.ValidateAsync(leakTest);
                if (!leakTestValidationResult.IsValid)
                {
                    return BadRequest(
                        $"LeakTest object could not be validated: {string.Join(", ", leakTestValidationResult.Errors.Select(e => e.ErrorMessage))}");
                }
            }
            // Convert to JSON format
            var jsonPayload = JsonConvert.SerializeObject(leakTests, Formatting.Indented);

            // Return the JSON payload
            return Ok(jsonPayload);
        }
        catch (Exception e)
        {
            return StatusCode(500, $"Internal server error: {e.Message}");
        }
    }

    [HttpGet("GetPointsByTagAsync")] // Queries the db to return where a tag is equal to a certain value
    public async Task<IActionResult> GetPointsByTagAsync(string tag, string? value)
    {
        var leakTests = await _leakTestRepository.GetPointsByTag(tag, value);
        var jsonPayload = JsonConvert.SerializeObject(leakTests, Formatting.Indented);

        return Ok(jsonPayload);
    }
}
```
til:
```c#
// usings

namespace LeakTestService.Controllers;

[ApiController]
[Route("api/LeakTests")]
public class LeakTestController : ControllerBase
{
    private readonly ILeakTestRepository _leakTestRepository;

    public LeakTestController(ILeakTestRepository leakTestRepository)
    {
        _leakTestRepository = leakTestRepository;
    }

    
    [HttpPost("Batch")] 
    public async Task<IActionResult> AddBatchAsync([FromBody] List<LeakTest> leakTests)
    {
        try
        {
            if (leakTests == null || !leakTests.Any())
            {
                return BadRequest("The request body was null or empty.");
            }

            // making sure the value of user and status are upper case and that the leakTestId is set to a new guid.
            foreach (var leakTest in leakTests)
            {
                leakTest.User = leakTest.User.ToUpper();
                leakTest.Status = leakTest.Status.ToUpper();

                // setting the id of the leaktest. 
                leakTest.LeakTestId = Guid.NewGuid();
            }

            var validator = new LeakTestValidator();
            var validationErrors = new List<string>();

            foreach (var leakTest in leakTests)
            {

                var validationResult = await validator.ValidateAsync(leakTest);
                if (!validationResult.IsValid)
                {
                    validationErrors.AddRange(validationResult.Errors.Select(e => e.ErrorMessage));
                }
            }

            if (validationErrors.Any())
            {
                return BadRequest(
                    $"Some LeakTest objects could not be validated: {string.Join(", ", validationErrors)}");
            }

            await _leakTestRepository.AddBatchAsync(leakTests);

            return Ok($"Data received successfully: {JsonConvert.SerializeObject(leakTests, Formatting.Indented)}");

        }
        // return CreatedAtAction("Task<IActionResult> GetAllAsync()", null);        }
        catch (Exception e)
        {
            // Log the exception here
            return BadRequest($"The request could not be processed due to: {e.Message}");
        }
    }

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

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id)
    {
        try
        {
            // Getting the LeakTest from the database by id. 
            var leakTest = await _leakTestRepository.GetByIdAsync(id);
            
            // Creating the validator and validating the LeakTest object.
            var validator = new LeakTestValidator();
            var validationResult = await validator.ValidateAsync(leakTest);
            
            if (!validationResult.IsValid)
            {
                return BadRequest($"LeakTest object could not be validated: {string.Join(", ", validationResult.Errors.Select(e => e.ErrorMessage))}");
            }

            return Ok(JsonConvert.SerializeObject(leakTest, Formatting.Indented));
        }
        catch (Exception e)
        {
            // Log the exception here
            return BadRequest($"The request could not be processed due to: {e.Message}");
        }
    }
    
    [HttpGet]
    public async Task<IActionResult> GetAllAsync()
    {
        try
        {
            // Create and fill the list of LeakTest objects.  
            var leakTests = await _leakTestRepository.GetAllAsync() as List<LeakTest>;
            
            // Check if there are any objects in the LeakTest list and if not, return NoContent error message. 
            if (leakTests == null) return NoContent();

            // Validate the objects
            var validator = new LeakTestValidator();
            
            foreach (var leakTest in leakTests)
            {
                var validationResult = await validator.ValidateAsync(leakTest);
                if (!validationResult.IsValid)
                {
                    return BadRequest(
                        $"LeakTest object could not be validated: {string.Join(", ", validationResult.Errors.Select(e => e.ErrorMessage))}");
                }
            }

            // Add HATEOAS links
            var baseUrl = $"{Request.Scheme}://{Request.Host}{Request.PathBase}";
            leakTests.ForEach(leakTest =>
            {
                leakTest.Links = new Dictionary<string, string>
                {
                    { "self", $"{baseUrl}/api/LeakTests/{leakTest.LeakTestId}" }
                };
            });

            // Convert to JSON format
            var jsonPayload = JsonConvert.SerializeObject(leakTests, Formatting.Indented);

            // Return the JSON payload
            return Ok(jsonPayload);
        }
        catch (Exception e)
        {
            // Log the exception here
            return StatusCode(500, $"Internal server error: {e.Message}");
        }
    }
    
    [HttpGet("TimeRange")] // Start and stop values must be valid DateTimes, input as strings.
    public async Task<IActionResult> GetWithinTimeRangeAsync(DateTime start, DateTime? stop)
    {
        try
        {
            // Create TimeRange object
            var timeRange = new TimeRange(start, stop);
            
            // Validate the TimeRange object
            var timeRangeValidator = new TimeRangeValidator();
            var timeRangeValidationResult = await timeRangeValidator.ValidateAsync(timeRange);

            if (!timeRangeValidationResult.IsValid)
            {
                return BadRequest(
                    $"Time range could not be validated: {string.Join(", ", timeRangeValidationResult.Errors.Select(e => e.ErrorMessage))}");
            }
            
            // Create and fill the list of LeakTest objects.  
            var leakTests = await _leakTestRepository.GetWithinTimeRangeAsync(timeRange) as List<LeakTest>;
            
            // Check if there are any objects in the LeakTest list and if not, return NoContent error message. 
            if (leakTests == null) return NoContent();

            // Validate the objects
            var leakTestValidator = new LeakTestValidator();
            
            foreach (var leakTest in leakTests)
            {
                var leakTestValidationResult = await leakTestValidator.ValidateAsync(leakTest);
                if (!leakTestValidationResult.IsValid)
                {
                    return BadRequest(
                        $"LeakTest object could not be validated: {string.Join(", ", leakTestValidationResult.Errors.Select(e => e.ErrorMessage))}");
                }
            }
            // Convert to JSON format
            var jsonPayload = JsonConvert.SerializeObject(leakTests, Formatting.Indented);

            // Return the JSON payload
            return Ok(jsonPayload);
        }
        catch (Exception e)
        {
            // Log the exception here
            return StatusCode(500, $"Internal server error: {e.Message}");
        }
    }

    [HttpGet("Tag")] // Queries the db to return where a tag is equal to a certain value
    public async Task<IActionResult> GetByTagAsync(string tag, string value)
    {
        try
        {
            // Check if 'tag' is a valid property of the LeakTest class. 
            var match = false;
            var culture = CultureInfo.InvariantCulture;
            foreach (var property in typeof(LeakTest).GetProperties())
            {
                // returns true if the tag matches a property name.
                var tagMatchesProperty = culture.CompareInfo.IndexOf(property.Name, tag, CompareOptions.IgnoreCase) >= 0;

                if (tagMatchesProperty)
                {
                    match = true;
                }
            }
            if (!match)
            {
                throw new NoMatchingDataException($"The specified tag '{tag}' does not exist.");
            }

            if (tag.ToLower() == "status" || tag.ToLower() == "user")
            {
                value = value.ToUpper();
            }

            tag = StringExtensions.FirstCharToUpper(tag);
            
            var leakTests = await _leakTestRepository.GetByTagAsync(tag, value);

            if (!leakTests.Any())
            {
                throw new NoMatchingDataException("No test results match the specified tag key-value pair.");
            }
            
            var jsonPayload = JsonConvert.SerializeObject(leakTests, Formatting.Indented);

            return Ok(jsonPayload);
        }
        catch (NoMatchingDataException noMatchingDataException)
        {
            // Log the exception 
            
            // Return a BadRequest status code along with the exception message
            return NotFound(noMatchingDataException.Message);
        }
        catch (InfluxException influxException)
        {
            // Log the exception 
            
            // Return a BadRequest status code along with the exception message
            return BadRequest(influxException.Message);
        }
    }
    
    [HttpGet("Field")] // Queries the db to return where a tag is equal to a certain value
    public async Task<IActionResult> GetByFieldAsync(string field, string value)
    {
        try
        {
            // Check if 'field' is a valid property of the LeakTest class, by comparing the names of the LeakTest properties
            // with the value of the 'field' input parameter
            
            var match = false;
            var culture = CultureInfo.InvariantCulture;
            foreach (var property in typeof(LeakTest).GetProperties())
            {
                // returns true if the tag matches a property name.
                var tagMatchesProperty = culture.CompareInfo.IndexOf(property.Name, field, CompareOptions.IgnoreCase) >= 0;

                if (tagMatchesProperty)
                {
                    match = true;
                }
            }
            if (!match)
            {
                throw new NoMatchingDataException($"The specified field '{field}' does not exist.");
            }

            // Making sure that the value of the input param 'field' has the correct letters Upper Cased. 
            field = StringExtensions.NormalizeField(field);
            
            var leakTests = await _leakTestRepository.GetByFieldAsync(field, value);

            if (!leakTests.Any())
            {
                throw new NoMatchingDataException("No test results match the specified tag key-value pair.");
            }
            
            var jsonPayload = JsonConvert.SerializeObject(leakTests, Formatting.Indented);

            return Ok(jsonPayload);
        }
        catch (NoMatchingDataException noMatchingDataException)
        {
            // Log the exception 
            
            // Return a BadRequest status code along with the exception message
            return NotFound(noMatchingDataException.Message);
        }
        catch (InfluxException influxException)
        {
            // Log the exception 
        
            // Return a BadRequest status code along with the exception message
            return BadRequest(influxException.Message);
        }
    }
}
```

Det inkluderer bedre fejlhåndtering, bedre navngivning af endpoints og metoder, bedre og flere kommentarer, forbedring 
af responsformater samt lidt HATEAOS.
