# MyDemo - Dependency Injection Guide

## Table of Contents

1. [Introduction](#introduction)
2. [Step 1: Create the Project Structure](#step-1-create-the-project-structure)
3. [Step 2: Create Basic Models and Interfaces](#step-2-create-basic-models-and-interfaces)
4. [Step 3: Move Logic to a Service](#step-3-move-logic-to-a-service)
5. [Step 4: Wire Up Dependency Injection](#step-4-wire-up-dependency-injection)
6. [Step 5: Add Configuration (Why DI is Useful)](#step-5-add-configuration-why-di-is-useful)
7. [Why DI Makes Testing Easy](#why-di-makes-testing-easy-in-simple-terms)
8. [The Sandwich Shop Analogy](#the-sandwich-shop-analogy-for-beginners)
9. [The Pattern to Remember](#the-pattern-to-remember)

---

## Introduction

This guide demonstrates Dependency Injection (DI) in .NET using a simple Web API example. You'll learn how to structure your application with clean separation of concerns and why DI makes your code testable and maintainable.

---

## Step 1: Create the Project Structure

Create a solution with a web API project and a class library:

```bash
dotnet new sln -n MyDemo
dotnet new webapi -n MyDemo.Web -minimal
dotnet new classlib -n MyDemo.Library
dotnet sln add MyDemo.Web
dotnet sln add MyDemo.Library
```

Add a reference from the web project to the library:

```bash
cd MyDemo.Web
dotnet add reference ../MyDemo.Library
cd ..
```

Your structure should look like:

```
MyDemo/
├── MyDemo.sln
├── MyDemo.Web/
│   ├── Program.cs
│   └── appsettings.json
└── MyDemo.Library/
    ├── Models/
    ├── Interfaces/
    └── Services/
```

---

## Step 2: Create Basic Models and Interfaces

**MyDemo.Library/Models/User.cs**
```csharp
namespace MyDemo.Library.Models;

public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
}
```

**MyDemo.Library/Interfaces/IUserService.cs**
```csharp
using MyDemo.Library.Models;

namespace MyDemo.Library.Interfaces;

public interface IUserService
{
    List<User> GetAllUsers();
    User? GetUser(int id);
}
```

---

## Step 3: Move Logic to a Service

**About "Service":**
I can see that "service" could be misunderstood - I didn't mean a "Windows Service", just a service class by name.

**Example:**
When you have a button in your UI, you don't make a database connection and send emails directly from your UI code. Instead, you add that code in class libraries - in this case, it would be in `MyDemo.Library`.

**How to structure it:**

1. Create a "service" interface in `/Interfaces/IMessage.cs` for sending mail, Slack, Teams, etc.
2. Create the implementation in `/Services/Message.cs`
3. Don't forget to register it in your `Program.cs`, just like the others:

```csharp
builder.Services.AddScoped<IMessageService, MessageService>();
builder.Services.AddScoped<IAppSettingsService, AppSettingsService>();
```

**MyDemo.Library/Services/UserService.cs**
```csharp
using MyDemo.Library.Interfaces;
using MyDemo.Library.Models;

namespace MyDemo.Library.Services;

public class UserService : IUserService
{
    public List<User> GetAllUsers()
    {
        return new List<User>
        {
            new User { Id = 1, Name = "Alice", Email = "alice@example.com" },
            new User { Id = 2, Name = "Bob", Email = "bob@example.com" }
        };
    }

    public User? GetUser(int id)
    {
        var users = GetAllUsers();
        return users.FirstOrDefault(u => u.Id == id);
    }
}
```

---

## Step 4: Wire Up Dependency Injection

**MyDemo.Web/Program.cs**
```csharp
using MyDemo.Library.Interfaces;
using MyDemo.Library.Services;

var builder = WebApplication.CreateBuilder(args);

// Register the service with DI container
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

// Use the service via DI - it's automatically provided!
app.MapGet("/users", (IUserService userService) => userService.GetAllUsers());

app.MapGet("/user/{id}", (int id, IUserService userService) =>
{
    var user = userService.GetUser(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.Lifetime.ApplicationStarted.Register(() =>
{
    var url = app.Urls.FirstOrDefault() ?? "http://localhost:5000";
    
    Console.WriteLine("Endpoints:");
    Console.WriteLine($"  GET {url}/users");
    Console.WriteLine($"  GET {url}/user/2");
});

app.Run();
```

---

## Step 5: Add Configuration (Why DI is Useful)

Let's say we want a configurable welcome message.

### Update appsettings.json

**MyDemo.Web/appsettings.json**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "WelcomeMessage": "Welcome to our API"
}
```

### Create an interface and service to read the configuration

**MyDemo.Library/Interfaces/IAppSettingsService.cs**
```csharp
namespace MyDemo.Library.Interfaces;

public interface IAppSettingsService
{
    string GetWelcomeMessage();
}
```

### Implementation - Notice how it receives IConfiguration via DI

**MyDemo.Library/Services/AppSettingsService.cs**
```csharp
using Microsoft.Extensions.Configuration;
using MyDemo.Library.Interfaces;

namespace MyDemo.Library.Services;

public class AppSettingsService : IAppSettingsService
{
    private readonly IConfiguration _configuration;

    public AppSettingsService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GetWelcomeMessage()
    {
        return _configuration["WelcomeMessage"] ?? "Hello";
    }
}
```

**Key DI pattern here:** 
- The `private readonly IConfiguration _configuration` field stores the dependency
- The constructor receives it automatically from .NET
- You never use `new` to create dependencies yourself

### Add the NuGet package to MyDemo.Library

```bash
dotnet add MyDemo.Library package Microsoft.Extensions.Configuration.Abstractions
```

### Wire it all up in Program.cs

**MyDemo.Web/Program.cs** (complete version)
```csharp
using MyDemo.Library.Interfaces;
using MyDemo.Library.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IAppSettingsService, AppSettingsService>();

var app = builder.Build();

// Returns plain text
app.MapGet("/", (IAppSettingsService appSettings) =>
{
    return appSettings.GetWelcomeMessage();
});

// Returns JSON: { "message": "Welcome to our API" }
app.MapGet("/welcome", (IAppSettingsService appSettings) =>
{
    return Results.Json(new { message = appSettings.GetWelcomeMessage() });
});

app.MapGet("/users", (IUserService userService) => userService.GetAllUsers());

app.MapGet("/user/{id}", (int id, IUserService userService) =>
{
    var user = userService.GetUser(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.Lifetime.ApplicationStarted.Register(() =>
{
    var url = app.Urls.FirstOrDefault() ?? "http://localhost:5000";
    
    Console.WriteLine("Endpoints:");
    Console.WriteLine($"  GET {url}/");
    Console.WriteLine($"  GET {url}/welcome");
    Console.WriteLine($"  GET {url}/users");
    Console.WriteLine($"  GET {url}/user/2");
});

app.Run();
```

**Note:** The `/users` and `/user/{id}` endpoints already return JSON automatically - .NET serializes objects to JSON by default.

---

## Why DI Makes Testing Easy (In Simple Terms)

### The Problem Without DI

Imagine you have a service that sends emails:

```csharp
public class OrderService
{
    public void ProcessOrder(int orderId)
    {
        // ... process order logic ...
        
        var emailSender = new EmailSender(); // Creates real email sender!
        emailSender.Send("order@customer.com", "Order confirmed");
    }
}
```

**Problem when testing:** 
- Every time you test this code, it tries to send a REAL email
- You can't test without an email server
- Tests are slow (network calls)
- You might accidentally spam customers during testing!

### The Solution With DI

```csharp
public class OrderService
{
    private readonly IEmailSender _emailSender;
    
    public OrderService(IEmailSender emailSender) // Receives dependency
    {
        _emailSender = emailSender;
    }
    
    public void ProcessOrder(int orderId)
    {
        // ... process order logic ...
        
        _emailSender.Send("order@customer.com", "Order confirmed");
    }
}
```

**Now when testing:**

```csharp
// Create a test version that doesn't actually send emails
public class FakeEmailSender : IEmailSender
{
    public void Send(string to, string message)
    {
        // Do nothing - just pretend it worked
        Console.WriteLine($"Would send email to {to}: {message}");
    }
}

// In your test
var fakeEmailer = new FakeEmailSender();
var orderService = new OrderService(fakeEmailer); // Pass in fake!

orderService.ProcessOrder(123); // No real email sent!
```

**Benefits:**
- **Fast tests** - no network calls
- **Safe tests** - no real emails sent
- **Reliable tests** - no dependency on external services
- **Easy to verify** - you can check if the fake was called correctly

### Real-World Example

**Production (real app):**
```csharp
builder.Services.AddScoped<IEmailSender, RealEmailSender>(); // Sends real emails
```

**Testing:**
```csharp
var fakeEmailer = new FakeEmailSender();
var service = new OrderService(fakeEmailer); // Use fake instead
```

**Same code, different behavior!** That's the power of DI.

*(Note: In real testing, you'd use libraries like Moq or NSubstitute to create these test versions automatically, but the concept is the same)*

---

## The Sandwich Shop Analogy (For Beginners)

### Imagine You're Making a Sandwich Shop

#### The Old Way (Without DI)

You're making sandwiches, but every time you need bread, you have to:
1. Go to the store
2. Buy flour
3. Come back
4. Bake the bread yourself
5. Finally make the sandwich

**This is exhausting!** Every sandwich maker has to do ALL this work.

#### The New Way (With Dependency Injection)

Now imagine your sandwich shop has a **magic system**:

1. You write down: "I need bread" 
2. Someone automatically delivers fresh bread to your workstation
3. You just make the sandwich

**You don't care:**
- Where the bread comes from
- How it's made
- Who delivers it

**You just say "I need bread" and it appears!**

### In Programming Terms

**Your Service (Sandwich Maker):**
```csharp
public class SandwichMaker
{
    private readonly IBread _bread; // "I need bread"

    // Constructor - like filling out a request form
    public SandwichMaker(IBread bread)
    {
        _bread = bread; // Bread is delivered here!
    }

    public void MakeSandwich()
    {
        _bread.UseSlices(2); // Just use it!
    }
}
```

**The Registration (Setting Up Delivery):**
```csharp
builder.Services.AddScoped<IBread, WheatBread>();
```
This says: "When someone asks for bread, give them wheat bread"

### Why This is Amazing

**Without DI (Doing Everything Yourself):**
- Hard to test (what if the store is closed?)
- Hard to change (what if you want rye bread instead?)
- Lots of repeated code (every sandwich maker bakes bread)

**With DI (Magic Delivery System):**
- Easy to test (deliver fake bread for testing)
- Easy to change (just change the registration to `RyeBread`)
- Clean code (sandwich makers just make sandwiches)

### The Key Concept

**Dependency Injection = "Don't create things yourself - ask for them and they'll be provided"**

Instead of:
```csharp
var bread = new WheatBread(); // Making it yourself
```

You do:
```csharp
public SandwichMaker(IBread bread) // Asking for it
{
    _bread = bread; // Receiving it automatically
}
```

---

## The Pattern to Remember

Every service follows this pattern:

```csharp
public class MyService : IMyService
{
    private readonly ISomeDependency _dependency;

    // Constructor receives dependencies
    public MyService(ISomeDependency dependency)
    {
        _dependency = dependency;
    }

    // Methods use the dependency
    public void DoSomething()
    {
        _dependency.DoWork();
    }
}
```

**3 Steps:**
1. `private readonly` field to store dependency
2. Constructor receives dependency as parameter
3. .NET automatically provides it (after you register with `AddScoped`)

**See the pattern?** Every service gets its dependencies through the constructor with `private readonly` fields. .NET handles all the wiring via `builder.Services.AddScoped<>()`

---

## Summary

- **Dependency Injection** lets you request dependencies instead of creating them
- Services receive dependencies through **constructor injection**
- Register services in `Program.cs` with `builder.Services.AddScoped<>()`
- DI makes testing easy because you can provide test versions (fakes/mocks)
- Clean, maintainable code with clear separation of concerns

**Bottom line:** It's a good start - you've got something that works. The dependency injection part shouldn't be too difficult. You don't have to learn too many new tricks at the same time.