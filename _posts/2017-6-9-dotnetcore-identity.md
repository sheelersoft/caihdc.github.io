---
layout: post
title: ASP.NET Core - Identity
---

## Overview
This post will go over the process of setting up an ASP.NET Core Web Api project to use the built in Identity solution. It will show how to set up the code and give examples of it working in action.

## Installation
To follow along, you'll need to have .NET Core installed. I'll also be using Visual Studio Code as my editor and terminal which I recommend for anyone trying this out. Finally, I'll be using Postman to test the api after it is set up.

## Create ASP.NET Core project
To create the project, first you'll need to navigate into the folder you want to create it in. For me, this was /coreapitest/server

`...> cd coreapitest/server`

Then you should execute the command to create a new ASP.NET Core project with the web api template. This will give us some basic set up and a values controller we'll use for testing later.

`coreapitest/server> dotnet new webapi`

After this is finished you should see some scaffolding in your root folder. Double check to make sure a ValuesController.cs file was created in the Controllers folder and the Startup.cs file was also created.

## Nuget Packages
Before any coding is done, there are a few nuget packages that need to be added. Open your .csproj file and make sure the Package Reference Item Group has the following packages.

```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore" Version="1.1.2" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc" Version="1.1.3" />
    <PackageReference Include="Microsoft.Extensions.Logging.Debug" Version="1.1.2" />
    <PackageReference Include="Microsoft.AspNetCore.Identity" Version="1.1.2" />
    <PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="1.1.2" />
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="1.1.2" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="1.1.2" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="1.1.2" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="1.1.2" />
  </ItemGroup>
```
After this is added, save the file, and run dotnet restore on the project. This pulls down any new packages you added.

`coreapitest/server> dotnet restore`

## IdentityContext and IdentityData

## Startup

## AuthController

## ValuesController

## Testing