---
layout: post
title: How to create a dotnet web API using vertical slice - step by step
description: Step by step how to create a minimal API using vertical slices in a monolithic architecture
tags: dotnet webapi verticalslices architecture monolithics
minute: 3
---

## Vertical Slices Architecture

### 💡 The idea

+ A vertical slice represents a self-contained unit of functionality
+ Encapsulates all the code and components necessary to fulfill a specif feature

### Benefits

+ Improves cohesion
+ Reduced complexity
+ Focus on business logic / use case
+ Easier to navigate and to maintain

### 📌 Project Organization

```shell
mkdir Contacts-app && cd $_
mkdir tests
mkdir src && cd $_

dotnet new webapi --no-https -o Contacts.api;

dotnet new solution;
dotnet sln add ./Contacts.api/;

cd ..
dotnet new gitignore;
git init;
git add .;
git commit -m "first commit";
```

### 📌 API

#### 📁 ../Contacts-app/src/Contacts.api

```shell
dotnet add package Microsoft.EntityFrameworkCore;
dotnet add package Microsoft.EntityFrameworkCore.InMemory;
dotnet add package Swashbuckle.AspNetCore

mkdir Endpoints;
mkdir Database;
mkdir Entities;
mkdir Features && cd $_;
mkdir Contacts && cd $_;
```

#### ../Contacts-app/src/Contacts.api/Endpoints/IEndpoint.cs

```c#
namespace Contacts.Endpoints;

public interface IEndpoint
{
    void MapEndpoint(IEndpointRouteBuilder app);
}
```

#### ../Contacts-app/src/Contacts.api/Endpoints/EndpointExtensions.cs 

```c#
using System.Reflection;
using Microsoft.Extensions.DependencyInjection.Extensions;

namespace Contacts.Endpoints;

public static class EndpointExtensions
{
    public static IServiceCollection AddEndpoints(this IServiceCollection services)
    {
        services.AddEndpoints(Assembly.GetExecutingAssembly());

        return services;
    }

    public static IServiceCollection AddEndpoints(this IServiceCollection services, Assembly assembly)
    {
        ServiceDescriptor[] serviceDescriptors = assembly
            .DefinedTypes
            .Where(type => type is { IsAbstract: false, IsInterface: false} &&
                type.IsAssignableTo(typeof(IEndpoint)))
            .Select( type => ServiceDescriptor.Transient(typeof(IEndpoint), type))
            .ToArray();
        
        services.TryAddEnumerable(serviceDescriptors);

        return services;
    }

    public static IApplicationBuilder MapEndpoints(this WebApplication app, RouteGroupBuilder? routeGroupBuilder = null)
    {
        IEnumerable<IEndpoint> endpoints = app.Services.GetRequiredService<IEnumerable<IEndpoint>>();
        
        IEndpointRouteBuilder builder = routeGroupBuilder is null ? app : routeGroupBuilder;

        foreach(IEndpoint endpoint in endpoints)
        {
            endpoint.MapEndpoint(builder);
        }

        return app;
    }
} 
```

#### ../Contacts-app/src/Contacts.api/Entities/Contacts.cs

```c#
namespace Contacts.Entities;

public class Contact{
    public Guid Id {get; set;}
    public string Name {get; set;} = string.Empty;
    public string CountryCode{get; set;} = string.Empty;
    public string PhoneNumber {get; set;} = string.Empty;
}
```

#### ../Contacts-app/src/Contacts.api/Database/ContactsDbContext.cs

```c#
using Microsoft.EntityFrameworkCore;
using Contacts.Entities;

namespace Contacts.Database;

public class ContactsDbContext: DbContext
{
    public ContactsDbContext(DbContextOptions<ContactsDbContext> options): base(options)
    {
    }

    public DbSet<Contact> Contacts{get; init;}
}
```

#### ../Contacts-app/src/Contacts.api/Program.cs

```c#
using Contacts.Database;
using Contacts.Endpoints;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options => options.CustomSchemaIds(s=> s.FullName?.Replace('+','.')));

builder.Services.AddDbContext<ContactsDbContext>(o=> o.UseInMemoryDatabase("contactDB"));

builder.Services.AddEndpoints();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapEndpoints();

app.Run();
```

### 📌 Use Cases

#### ../Contacts-app/src/Contacts.api/Features/Contacts/CreateContact.cs

```c#
using Contacts.Database;
using Contacts.Endpoints;
using Contacts.Entities;

namespace Contacts.Features.Contacts;

public static class CreateContact
{
    public record Request(string Name, string CountryCode, string PhoneNumber);
    public record Response(Guid Id, string Name, string CountryCode, string PhoneNumber);

    public sealed class Endpoint : IEndpoint
    {
        public void MapEndpoint(IEndpointRouteBuilder app)
        {
            app.MapPost("contacts", Handler).WithTags("Contacts");
        }
    }

    public static async Task<IResult> Handler(Request request, ContactsDbContext context)
    {
        var contact = new Contact{Name=request.Name, CountryCode=request.CountryCode, PhoneNumber = request.PhoneNumber};
        
        context.Contacts.Add(contact);
        
        await context.SaveChangesAsync();

        return Results.Ok(new Response(contact.Id, contact.Name, contact.CountryCode, contact.PhoneNumber));
    }
}
```

#### ../Contacts-app/src/Contacts.api/Features/Contacts/GetContact.cs

```c#
using Contacts.Database;
using Contacts.Endpoints;
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.EntityFrameworkCore;

namespace Contacts.Features.Contacts;

public static class GetContact
{
    public record Response(Guid Id, string Name, string CountryCode, string PhoneNumber);

    public sealed class Endpoint : IEndpoint
    {
        public void MapEndpoint(IEndpointRouteBuilder app)
        {
            app.MapGet("contacts/{id}", Handler).WithTags("Contacts");
        }
    }

    public static async Task<Results<Ok<Response>, NotFound>> Handler(Guid id, ContactsDbContext context)
    {
        var contact = await context.Contacts.AsNoTracking().FirstOrDefaultAsync(c=> c.Id == id);
        
        if (contact is null)
        {
            return TypedResults.NotFound();
        }

        return TypedResults.Ok(new Response(contact.Id, contact.Name, contact.CountryCode, contact.PhoneNumber));
    }
}
```

#### ../Contacts-app/src/Contacts.api/Features/Contacts/GetContacts.cs

```c#
using Contacts.Database;
using Contacts.Endpoints;
using Microsoft.EntityFrameworkCore;

namespace Contacts.Features.Contacts;

public static class GetContacts
{
    public record Response(Guid Id, string Name, string CountryCode, string PhoneNumber);

    public sealed class Endpoint : IEndpoint
    {
        public void MapEndpoint(IEndpointRouteBuilder app)
        {
            app.MapGet("contacts", Handler).WithTags("Contacts");
        }
    }

    public static async Task<IResult> Handler(ContactsDbContext context)
    {
        var contacts = await context.Contacts.AsNoTracking().ToListAsync();
        
        var responses = contacts.Select(c => new Response(c.Id, c.Name, c.CountryCode, c.PhoneNumber));

        return TypedResults.Ok(responses);
    }
}
```

#### ../Contacts-app/src/Contacts.api/Features/Contacts/RemoveContact.cs

```c#
using Contacts.Database;
using Contacts.Endpoints;

namespace Contacts.Features.Contacts;

public static class RemoveContact
{
    public sealed class Endpoint : IEndpoint
    {
        public void MapEndpoint(IEndpointRouteBuilder app)
        {
            app.MapDelete("contacts/{id}", Handler).WithTags("Contacts");
        }
    }

    public static async Task<IResult> Handler(Guid id, ContactsDbContext context)
    {
        var contact = await context.Contacts.FindAsync(id);
        
        if (contact is null)
        {
            return TypedResults.NotFound();
        }

        context.Remove(contact);

        await context.SaveChangesAsync();

        return Results.NoContent();
    }
}
```

#### ../Contacts-app/src/Contacts.api/Features/Contacts/UpdateContact.cs

```c#
using Contacts.Database;
using Contacts.Endpoints;
using Microsoft.AspNetCore.Http.HttpResults;

namespace Contacts.Features.Contacts;

public static class UpdateContact
{
    public record Request(string Name, string CountryCode, string PhoneNumber);
    public record Response(Guid Id, string Name, string CountryCode, string PhoneNumber);

    public sealed class Endpoint : IEndpoint
    {
        public void MapEndpoint(IEndpointRouteBuilder app)
        {
            app.MapPut("contacts/{id}", Handler).WithTags("Contacts");
        }
    }

    public static async Task<Results<Ok<Response>, NotFound>> Handler(Guid id, Request request, ContactsDbContext context)
    {
        var contact = await context.Contacts.FindAsync(id);
        
        if (contact is null)
        {
            return TypedResults.NotFound();
        }

        contact.Name = request.Name;
        contact.CountryCode = request.CountryCode;
        contact.PhoneNumber = request.PhoneNumber;

        await context.SaveChangesAsync();

        return TypedResults.Ok(new Response(contact.Id, contact.Name, contact.CountryCode, contact.PhoneNumber));
    }
}
```

### Docker File

```
# https://hub.docker.com/_/microsoft-dotnet

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

WORKDIR /source

  

# copy csproj and restore as distinct layers

COPY *.csproj ./

RUN dotnet restore

  

# copy everything else and build app

COPY . ./

RUN dotnet publish -c release -o /out

  

# final stage/image

FROM mcr.microsoft.com/dotnet/aspnet:9.0

WORKDIR /app

COPY --from=build /out ./

ENTRYPOINT ["dotnet", "Contacts.api.dll"]
```
