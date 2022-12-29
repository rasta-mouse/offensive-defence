---
title: "Custom Implicit & Explicit Conversions in C#"
date: 2022-12-29T19:22:25Z
draft: false
authors:
    - RastaMouse
---

Implicit and explicited operators are provided as a means of converting one datatype to another.

```c#
// this is an implicit conversion from an int to a double
int i = 8;
double d = i;

// this is an explicit conversion from a double to an int
double d = 8.8;
int i = (int) d;
```

I recently learned that you can implement your own implicit/explicit operators to convert custom classes, which is useful when needing to convert between API, domain and storage objects in an application.

Take the followng `Person` class as an example.

```c#
public sealed class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }

    public string FullName
        => $"{FirstName} {LastName}";
    
    public DateOnly DateOfBirth { get; set; }
    
    // obviously not accurate, just go with it
    public int Age
        => DateOnly.FromDateTime(DateTime.Today).Year - DateOfBirth.Year;
}
```

If we create a new instance of a `Person` and use sqlite-net-pcl to write it into a SQLite database, we'll see that it fails because SQLite does not understand the `DateOnly` type.  There are also plenty of other real-life examples as to why we want to use different classes in each part of our application.

So instead we'd want a `PersonDto` class, which could look something like this:

```c#
[Table("people")]
public sealed class PersonDto
{
    [Column("first_name")]
    public string FirstName { get; set; }
    
    [Column("last_name")]
    public string LastName { get; set; }
    
    [Column("dob")]
    public int DateOfBirth { get; set; }
}
```

So first we create a `Person`, convert it to a `PersonDto`, and then write it to the database.  The long way could be:

```c#
var path = Path.Combine(Directory.GetCurrentDirectory(), "people.db");

var conn = new SQLiteAsyncConnection(path);
await conn.CreateTableAsync<PersonDto>();

var person = new Person
{
    FirstName = "Charles",
    LastName = "Dickens",
    DateOfBirth = new DateOnly(1812, 2, 7)
};

var dto = new PersonDto
{
    FirstName = person.FirstName,
    LastName = person.LastName,
    DateOfBirth = person.DateOfBirth.DayNumber
};

await conn.InsertAsync(person);
```

We could then read the `PersonDto` back from the database, convert it back to a `Person`, and carry out any business operations on it.

```c#
dto = await conn.Table<PersonDto>().FirstAsync();

person = new Person
{
    FirstName = dto.FirstName,
    LastName = dto.LastName,
    DateOfBirth = DateOnly.FromDayNumber(dto.DateOfBirth)
};

Console.WriteLine($"{person.FullName} is {person.Age} years old.");
```
```text
Charles Dickens is 210 years old.
```

Obviously, manually mapping between `Person` and `PersonDto` every time is a tiresome process.  Some projects, such as [AutoMapper](https://automapper.org/) try to solve this for you, but sometimes you don't need or want more external packages.  This is where the implicit/explicit operators can help you out.

Here's an example of using an implicit operator to convert a `Person` to a `PersonDto`.

```c#
[Table("people")]
public sealed class PersonDto
{
    ...

    public static implicit operator PersonDto(Person person)
    {
        return new PersonDto
        {
            FirstName = person.FirstName,
            LastName = person.LastName,
            DateOfBirth = person.DateOfBirth.DayNumber
        };
    }
}
```

This would allow us to convert a `Person` to a `PersonDto` by simply changing the variable type, like so:

```c#
var person = new Person
{
    FirstName = "Charles",
    LastName = "Dickens",
    DateOfBirth = new DateOnly(1812, 2, 7)
};

PersonDto dto = person;
```

Here's an example of using an implicit operator to convert a `PersonDto` back to a `Person`.

```c#
[Table("people")]
public sealed record PersonDto
{
    ...

    public static explicit operator Person(PersonDto dto)
    {
        return new Person
        {
            FirstName = dto.FirstName,
            LastName = dto.LastName,
            DateOfBirth = DateOnly.FromDayNumber(dto.DateOfBirth)
        };
    }
}
```

This would allow us to convert a `PersonDto` to a `Person` using a cast.

```c#
dto = await conn.Table<PersonDto>().FirstAsync();
person = (Person) dto;
```

This makes your domain-level code much more readable and easier to work with.  You can use multiple converstions in your class as long as they have different method signatures.  For instance, you wouldn't be able to declare both an implicit and explicit operator like this:

```c#
public static implicit operator PersonDto(Person person)
{
    ...
}

public static explicit operator PersonDto(Person person)
{
    ...
}
```

What's the difference between implicit and explicit, and where should you use each one?  At the beginning of this blog, we had the following two examples:

```c#
// implicit
int i = 8;
double d = i;

// explicit
double d = 8.8;
int i = (int) d;
```

The only real difference has to do with whether or not there's a loss of data in the conversion.

Since an `int` is less accurate than an a `double`, there is no problem with converting it.  `8` as an `int` is also just `8` when converted to a double.  However, the reverse is not true.  `8.8` as a `double` becomes only `8` when converted to an `int`, so we lose the `.8` of precision.  The compiler requires us to do the cast to make us damn aware that this is happening.

You should always stick to this rule when implementing these conversions.  In my scenario above, it does not make sense to use explicit converstions since no data loss is occuring.  In which case, the following would work just fine:

```c#
[Table("people")]
public sealed class PersonDto
{
    [Column("first_name")]
    public string FirstName { get; set; }
    
    [Column("last_name")]
    public string LastName { get; set; }
    
    [Column("dob")]
    public int DateOfBirth { get; set; }

    public static implicit operator PersonDto(Person person)
    {
        return new PersonDto
        {
            FirstName = person.FirstName,
            LastName = person.LastName,
            DateOfBirth = person.DateOfBirth.DayNumber
        };
    }

    public static implicit operator Person(PersonDto dto)
    {
        return new Person
        {
            FirstName = dto.FirstName,
            LastName = dto.LastName,
            DateOfBirth = DateOnly.FromDayNumber(dto.DateOfBirth)
        };
    }
}
```