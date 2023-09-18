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

