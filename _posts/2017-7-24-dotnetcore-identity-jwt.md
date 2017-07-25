---
layout: post
title: ASP.NET Core - Identity with JWT
---

## Overview
This post will go over the process of setting up an ASP.NET Core project to use the Identity providers with Json Web Tokens (JWT) instead of the default cookie based approach. It will also make use of a simple Claims Identity using a Role Claim. I'll assume some background knowledge on how ASP.NET Core and Identity works. Please see [the docs](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity) or [my previous blog post](https://sheelersoft.github.io/dotnetcore-identity/) for more information. For examples, check out the [Angular Template](https://github.com/sheelersoft/angulartemplate) code on github.com. The information in this post is pulled from that project.

## Nuget Packages
Below are the packages in my .csproj file. Some of them won't really be used directly in this post like AutoMapper and FluentValidation but if you have any reference issues you can refer to this list.

```xml
<PackageReference Include="Microsoft.AspNetCore" Version="1.1.0" />
<PackageReference Include="Microsoft.AspNetCore.Mvc" Version="1.1.1" />
<PackageReference Include="Microsoft.AspNetCore.SpaServices" Version="1.1.1" />
<PackageReference Include="Microsoft.AspNetCore.StaticFiles" Version="1.1.0" />
<PackageReference Include="Microsoft.Extensions.Logging.Debug" Version="1.1.0" />
<PackageReference Include="Microsoft.AspNetCore.Identity" Version="1.1.2" />
<PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="1.1.2" />
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="1.1.2" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="1.1.2" />
<PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="1.1.2" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="1.1.2" />
<PackageReference Include="AutoMapper" Version="6.0.2" />
<PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="2.0.1" />
<PackageReference Include="FluentValidation.AspNetCore" Version="6.4.0" />
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="1.1.1" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="1.0.0" />
<PackageReference Include="Microsoft.IdentityModel.Tokens" Version="5.1.3" />
<PackageReference Include="System.IdentityModel.Tokens.Jwt" Version="5.1.3" />
```

## App Settings
The first step is to add a configuration section to the appsettings.json file. These settings will be used be used by the JWT code to generate the token. You can customize the settings to fit your needs.

```json
  "Logging": {
        ...
    }
  },
  "JwtIssuerOptions": {
    "Issuer": "myproject",
    "Audience": "http://localhost:5000/"
  }
```

## JWT Factory
The JWT Factory will be used to generate the token and the claims identity. Eventually, this will be called from the Auth Controller.

The implementation for this class can be found on [github.](https://github.com/sheelersoft/AngularTemplate/blob/master/content/Service/jssAngular.Service/Auth/JwtFactory.cs) Here is the interface:

```c#
    public interface IJwtFactory
    {
        Task<string> GenerateEncodedToken(string userName, ClaimsIdentity identity);
        ClaimsIdentity GenerateClaimsIdentity(AppUser user);
    }
```

The logic for the JWT Factory is pretty straight forward. GenerateEncodedToken generates the token and GenerateClaimsIdentity generates the claims identity (duh). One feature of GenerateClaimsIdentity is that it adds a Claim called Role which gets set to the AppUsers role. This allows us to use the Authorize attribute on controllers with the role property set.

The AppUser is whatever user class you're using for Authentication / Authorization. You can learn more about setting up your Identity classes [here](http://sheelersoft.com/dotnetcore-identity/).


## Startup

### Configure Services

### Configure

## Auth Controller
