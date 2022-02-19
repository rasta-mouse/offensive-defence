---
title: "Building an API Client from Swagger"
date: 2022-02-18T20:14:12Z
authors:
    - "RastaMouse"
draft: false
---

This post will demonstrate how to utiltise an OpenAPI definition from Swagger to build a .NET client.  We'll start off by throwing together a simple REST API in ASP.NET Core to create, get, update and delete "people".

## API Server

This is the model to represent a person object, which will sit at the domain layer.  Each person will have a unquie Id (think of this as a primary key in database parlance).  This Id will be used to find a person after they've been created.

```c#
namespace DemoAPI;

public class Person
{
    public Guid Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string EmailAddress { get; set; }

    public Person()
    {
        Id = Guid.NewGuid();
    }
}
```

We need a service that will handle managing our people.  The interface for which could look like this:


```c#
namespace DemoAPI;

public interface IPersonService
{
    /// <summary>
    /// Gets all people
    /// </summary>
    /// <returns>An IEnumerable of Person</returns>
    IEnumerable<Person> GetPeople();
    
    /// <summary>
    /// Adds a new person
    /// </summary>
    /// <param name="person"></param>
    /// <returns>True if successful</returns>
    bool AddPerson(Person person);
    
    /// <summary>
    /// Gets a single person by their Id
    /// </summary>
    /// <param name="id"></param>
    /// <returns>A person object</returns>
    Person GetPerson(Guid id);
    
    /// <summary>
    /// Removes a person
    /// </summary>
    /// <param name="person"></param>
    /// <returns>True if successful</returns>
    bool RemovePerson(Person person);
}
```

The interface will be dependancy injected into the API controller and represents a contract between the API and domain layers.  It should obviously expose methods sufficient to support the functionality we want to offer through the API.  As a side-note, putting the comments above the method definitions is useful because it populates intellisense when calling them.

![](/images/openapi-client/intellisense.png "Interface Intellisense")

My demo implementation of this service uses a simple in-memoy list to store the objects.  All our methods have to do is get, add, remove and update objects in this list:

```c#
namespace DemoAPI;

public class PersonService : IPersonService
{
    private readonly List<Person> _people = new();
    
    public IEnumerable<Person> GetPeople()
    {
        return _people;
    }

    public bool AddPerson(Person person)
    {
        if (_people.Contains(person))
        {
            return false;
        }
        
        _people.Add(person);
        return true;
    }

    public Person GetPerson(Guid id)
    {
        return _people.FirstOrDefault(p => p.Id.Equals(id));
    }

    public bool RemovePerson(Person person)
    {
        return _people.Remove(person);
    }
}
```

The API model to create a new person is as follows (this is the data a user will POST to the API).  When we create a new person, the caller should not need to specify an Id.  We rely on the server to generate it and tell us what it is.  This is one reason why we have different models for the API, domain and even storage layers.

```c#
namespace DemoAPI;

public class PersonRequest
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string EmailAddress { get; set; }
}
```

The API controller contains the actual HTTP endpoints.  We specify these as GET, POST, PUT and DELETE depending on the action that needs to be undertaken.  Within each method, we call the appropriate method on our interface.

```c#
using Microsoft.AspNetCore.Mvc;

namespace DemoAPI.Controllers;

[ApiController]
[Route("people")]
public class PeopleController : ControllerBase
{
    private readonly IPersonService _personService;

    public PeopleController(IPersonService personService)
    {
        _personService = personService;
    }

    [HttpGet (Name = "GetAllPeople")]
    public ActionResult<IEnumerable<Person>> GetPeople()
    {
        var people = _personService.GetPeople();
        return Ok(people);
    }

    [HttpGet("{id:guid}", Name = "GetPersonById")]
    public ActionResult<Person> GetPerson(Guid id)
    {
        var person = _personService.GetPerson(id);

        if (person is null)
        {
            return NotFound();
        }

        return Ok(person);
    }

    [HttpPost(Name = "CreatePerson")]
    public ActionResult<Person> AddPerson([FromBody] PersonRequest request)
    {
        var person = new Person
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            EmailAddress = request.EmailAddress
        };
        
        if (_personService.AddPerson(person))
        {
            var path = $"{Request.Scheme}://{Request.Host}{Request.Path.ToUriComponent()}/{person.Id}";
            return Created(path, person);
        }

        return BadRequest();
    }

    [HttpPut("{id:guid}", Name = "UpdatePersonById")]
    public IActionResult UpdatePerson(Guid id, [FromBody] PersonRequest request)
    {
        var person = _personService.GetPerson(id);

        if (person is null)
        {
            return NotFound();
        }

        person.FirstName = request.FirstName;
        person.LastName = request.LastName;
        person.EmailAddress = request.EmailAddress;

        if (_personService.RemovePerson(person))
        {
            if (_personService.AddPerson(person))
            {
                return NoContent();
            }    
        }

        return BadRequest();
    }

    [HttpDelete("{id:guid}", Name = "DeletePersonById")]
    public IActionResult DeletePerson(Guid id)
    {
        var person = _personService.GetPerson(id);

        if (person is null)
        {
            return NotFound();
        }

        if (_personService.RemovePerson(person))
        {
            return NoContent();
        }

        return BadRequest();
    }
}
```

There are two important aspects here to hightlight.

First - some of these have a return type of `ActionResult<T>` and some just an `IActionResult`.  `IActionResult` tells the controller that we're returning an HTTP status (200, 404, 500 etc), but not specifically what data those responses will contain; whereas `ActionResult<T>` tells the controller exactly what data type will be returned.  This is important for the OpenAPI definition so that consumers know exactly what data to expect from each endpoint.

Second - I've given each endpoint a `Name`, such as `Name = "GetAllPeople"`.  These will be used by the auto-code generator to give the methods in our C# code user-friendly names.  Otherwise, it would default to `PeopleAsync`, `People2Async`, `People3Async` etc.

Running the server, Swagger will list each API endpoint we've defined.

![](/images/openapi-client/swagger.png "Swagger")

Furthermore, if we expand the POST endpoint, it knows that we provide a `PersonRequest` object in the request body and that it returns a `Person` object.

![](/images/openapi-client/swagger-post.png "Swagger POST")

We can test the API in Swagger by creating a person:

```json
{
  "firstName": "Robin",
  "lastName": "Hood",
  "emailAddress": "rhood@sherwood.forest"
}
```

The response we get is:

```json
{
  "id": "5d4764ec-8859-46bf-a29d-f2717e4cc077",
  "firstName": "Robin",
  "lastName": "Hood",
  "emailAddress": "rhood@sherwood.forest"
}
```

As well as the `Person` object, the response header contains a `Location`:  `location: https://localhost:7014/people/5d4764ec-8859-46bf-a29d-f2717e4cc077`.  We can GET this URL to get the same person back again.

```text
$ curl -k -s https://localhost:7014/people/5d4764ec-8859-46bf-a29d-f2717e4cc077 | jq
{
  "id": "5d4764ec-8859-46bf-a29d-f2717e4cc077",
  "firstName": "Robin",
  "lastName": "Hood",
  "emailAddress": "rhood@sherwood.forest"
}
```

Have a play with the API to make sure it all works...

## API Client

This is the really easy part.

Swagger allow us to download the OpenAPI definition in JSON format.  For me, the URL was: *https://localhost:7014/swagger/v1/swagger.json*.  Create any .NET client app you want (Console, WPF, etc).  Then in Visual Studio, right-click that project and select **Add > Service Reference**.  Select OpenAPI and provide it the Swagger JSON file.

![](/images/openapi-client/service-reference.png "Add Service Reference")

When you click Finish, VS will install the necessary NuGet packages and auto-generate the necessary C# code.  It will save to `obj\swaggerClient.cs`.

Contruct the `DemoApi` client by providing the base address of the API and a new `HttpClient`.  Then it's simply a case of calling the methods you want.  In this example, I create a new person and print the output to the console.


```c#
namespace DemoClient;

internal static class Program
{
    public static async Task Main(string[] args)
    {
        // create new instance of the API Client
        var client = new DemoApi("https://localhost:7014/", new HttpClient());

        // build the request
        var request = new PersonRequest
        {
            FirstName = "Maid",
            LastName = "Marian",
            EmailAddress = "mmarian@sherwood.forest"
        };

        // post to the server
        var person = await client.CreatePersonAsync(request);

        // print person to the console
        Console.WriteLine($"Id: {person.Id}");
        Console.WriteLine($"First Name: {person.FirstName}");
        Console.WriteLine($"Last Name: {person.LastName}");
        Console.WriteLine($"Email: {person.EmailAddress}");
    }
}
```

```text
Id: 24d745ef-0af6-4e47-9fbc-9a8908a8ca40
First Name: Maid
Last Name: Marian
Email: mmarian@sherwood.forest
```