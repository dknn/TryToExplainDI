I hope this helps, and you will see C# isn't that hard. DI is just another way to think, there is some unlearning to do. But it's not hard.

(It has been years since I used React and only for building some SharePoint webparts)

I hope this will get the wheels turning.

**DI – "don't create it, ask for it"** (like React context/props) (I will repeat later)

---

## Step 1: Start with the simplest possible API

```bash
mkdir MyDemo
cd MyDemo
dotnet new webapi -n MyDemo.Web --no-openapi
dotnet new sln -n MyDemo
dotnet sln add MyDemo.Web
```

Replace `MyDemo.Web/Program.cs` with:

```csharp
// MyDemo.Web/Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World");

app.Run();
```

Run it:

```bash
dotnet run --project MyDemo.Web
```

That's a working API. No controllers, no abstract classes, no magic.

---

## Step 2: Add a model

We want to return a User. Let's keep models in their own project (so we can reuse them later):

```bash
dotnet new classlib -n MyDemo.Models
dotnet sln add MyDemo.Models
dotnet add MyDemo.Web reference MyDemo.Models
Remove-Item MyDemo.Models/Class1.cs
```

Create the model:

```csharp
// MyDemo.Models/User.cs
namespace MyDemo.Models;

public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
}
```

Now update Program.cs to return a user:

```csharp
// MyDemo.Web/Program.cs
using MyDemo.Models;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/user/{id}", (int id) =>
{
    if (id == 1)
    {
        return Results.Ok(new User { Id = 1, Name = "Joe", Email = "joe@test.com" });
    }
    return Results.NotFound();
});

app.Run();
```

Still simple. But now our API endpoint contains business logic. In React terms, it's like putting fetch calls and data transformation directly in your JSX.

---

## Step 3: Move logic to a service

Let's keep our API clean and move the logic somewhere else:

```bash
dotnet new classlib -n MyDemo.Library
dotnet sln add MyDemo.Library
dotnet add MyDemo.Library reference MyDemo.Models
dotnet add MyDemo.Web reference MyDemo.Library
Remove-Item MyDemo.Library/Class1.cs
mkdir MyDemo.Library/Interfaces
mkdir MyDemo.Library/Services
```

First, **an interface** (think of it as a TypeScript type for a class):

```csharp
// MyDemo.Library/Interfaces/IUserService.cs
using MyDemo.Models;

namespace MyDemo.Library.Interfaces;

public interface IUserService
{
    User? GetUser(int id);
    List<User> GetAllUsers();
}
```

Then **the implementation** (notice the **`: IUserService`** – this means "I implement that interface"):

```csharp
// MyDemo.Library/Services/UserService.cs
using MyDemo.Library.Interfaces;
using MyDemo.Models;

namespace MyDemo.Library.Services;

public class UserService : IUserService
{
    public User? GetUser(int id)
    {
        return GetAllUsers().FirstOrDefault(u => u.Id == id);
    }

    public List<User> GetAllUsers()
    {
        // Fake data for now
        return new List<User>
        {
            new User { Id = 1, Name = "Joe", Email = "joe@test.com" },
            new User { Id = 2, Name = "Jane", Email = "jane@test.com" }
        };
    }
}
```

---

## Step 4: Now we need Dependency Injection

Here's where DI comes in. We *could* do this: (But we are not going to do that!!!)

```csharp
app.MapGet("/user/{id}", (int id) =>
{
    var service = new UserService(); // Bad: we create it ourselves
    return service.GetUser(id);
});
```

But that's like hardcoding a fetch URL in every React component. What if we want to swap **UserService** for a **FakeUserService** in tests? Or what if **UserService** needs configuration?

**DI means: "Don't create it yourself, ask for it."**

```csharp
// MyDemo.Web/Program.cs
using MyDemo.Library.Interfaces;
using MyDemo.Library.Services;

var builder = WebApplication.CreateBuilder(args);

// Register: "When someone asks for IUserService, give them a UserService"
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

// Now we just ask for it:
app.MapGet("/users", (IUserService userService) =>
{
    return userService.GetAllUsers();
});

app.MapGet("/user/{id}", (int id, IUserService userService) =>
{
    var user = userService.GetUser(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.Run();
```

**.NET sees `IUserService` in the parameter and automatically provides it.** That's the "magic" – but now you know it's just one line doing the work: **`builder.Services.AddScoped<IUserService, UserService>()`**

---

## Step 5: Add configuration (to show why DI is useful)

Let's say we want a configurable welcome message. Update `appsettings.json`:

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

Create **an interface** and **service** to read the configuration:

```csharp
// MyDemo.Library/Interfaces/IAppSettingsService.cs
namespace MyDemo.Library.Interfaces;

public interface IAppSettingsService
{
    string GetWelcomeMessage();
}
```

Now **the implementation** – notice how it receives **`IConfiguration`** via DI:

```csharp
// MyDemo.Library/Services/AppSettingsService.cs
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

**Key DI pattern here:** The **`private readonly IConfiguration _configuration`** field stores the dependency, and the **constructor** receives it automatically from .NET.

Add the NuGet package to MyDemo.Library:

```bash
dotnet add MyDemo.Library package Microsoft.Extensions.Configuration.Abstractions
```

Now wire it all up:

```csharp
// MyDemo.Web/Program.cs
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

**Note:** The `/users` and `/user/{id}` endpoints already return JSON automatically – .NET serializes objects to JSON by default.

**See the pattern?** Every service gets its dependencies through the constructor with **`private readonly`** fields. .NET handles all the wiring via **`builder.Services.AddScoped<>()`**.

---

## That's it.

You now understand:

- **Models** – just classes with properties (like TypeScript interfaces)
- **Services** – where your logic lives (like custom hooks in React)
- **Interfaces** – contracts that say "this service can do X" (like TypeScript types)
- **DI** – "don't create it yourself, ask for it" (like React context/props)

The complexity you see in enterprise code? Most of it is optional patterns stacked on top of these basics. Start here, add more only when you feel the pain of not having it.

Next step? Add some logging, and then code some more. My tip for logging? NLog 6 (Yes I have used Serilog for the last 3 years but...)
Now the important stuff: just set your mindset to "it's not hard". Don't overcomplicate things. Don't spend your time constantly refactoring. Keep the layers clean. Use a simple test framework
I use xUnit, and have used NUnit. Again, keep it simple. Don't do TDD, you will use too much time fiddling with your tests.
Last time: keep it simple. Happy coding!
