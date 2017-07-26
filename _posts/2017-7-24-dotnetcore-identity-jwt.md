---
layout: post
title: ASP.NET Core - Identity with JWT
author: Ben Sheeler
---

## Overview
This post will go over the process of setting up an ASP.NET Core project to use the Identity providers with Json Web Tokens (JWT) instead of the default cookie based approach. It will also make use of Claims Identity and allow for role based authorization. I'll assume some background knowledge on how ASP.NET Core and Identity works. Please see [the docs](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity) or [my previous blog post](https://sheelersoft.github.io/dotnetcore-identity/) for more information. For examples, check out the [Angular Template](https://github.com/sheelersoft/angulartemplate) code on github.com. The information in this post is pulled from that project.

## Nuget Packages
Below are the packages in my .csproj file. Some of them might not be used directly in this post but if you have any reference issues you can refer to this list.

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
The first thing to do is to add a configuration section to the appsettings.json file. These settings will be used be used by the JWT code to generate the token. These settings can be customized to fit the needs of your project.

```json
"Logging": {

},
"JwtIssuerOptions": {
"Issuer": "myproject",
"Audience": "http://localhost:5000/"
}
```

## JWT Factory
The JWT Factory will be used to generate the token and the claims identity. Eventually, this will be called from the Auth Controller during login.

The implementation for this class can be found on [github.](https://github.com/sheelersoft/AngularTemplate/blob/master/content/Service/jssAngular.Service/Auth/JwtFactory.cs) Here is the interface:

```c#
public interface IJwtFactory
{
    Task<string> GenerateEncodedToken(string userName, ClaimsIdentity identity);
    ClaimsIdentity GenerateClaimsIdentity(AppUser user);
}
```

The logic for the JWT Factory is pretty straight forward. GenerateEncodedToken generates the token and GenerateClaimsIdentity generates the claims identity (duh). One feature of GenerateClaimsIdentity is that it adds a Claim called Role which gets set as the AppUsers role. This, in combination with an Authorization Policy that will be set up later, allows the Authorize attribute to be used with roles on controllers.

The AppUser is whatever user class you're using for Authentication / Authorization. You can learn more about setting up your Identity classes [here](http://sheelersoft.com/dotnetcore-identity/).


## Startup
Next, we'll have to wire up the services and tell the app to use the JWT Authentication. This all happens in [Startup.cs.](https://github.com/sheelersoft/AngularTemplate/blob/master/content/Web/jssAngular/Startup.cs) The first thing to do here is setup the signing key for the token. Include the following code snippet in the Startup class to get started. This is not secure so be sure not to deploy this to production.

```c#
private const string SecretKey = "iNivDmHLpUA223sqsfhqGbMRdRj1PVkH"; // todo: get this from somewhere secure
private readonly SymmetricSecurityKey _signingKey = new SymmetricSecurityKey(Encoding.ASCII.GetBytes(SecretKey));
```

### Configure Services
Configure Services is where the services get configured for Dependency Injection. It's also where the Admin Authorize Policy is added tell the app to look for the Admin Role claim. This is what my method looks like once everything is set up.

```c#
public void ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddLogging();       

    services.AddDbContext<ApplicationContext>(opt => opt.UseInMemoryDatabase());

    services.AddSingleton<IJwtFactory, JwtFactory>();

    // jwt wire up
    // Get options from app settings
    var jwtAppSettingOptions = Configuration.GetSection(nameof(JwtIssuerOptions));

    // Configure JwtIssuerOptions
    services.Configure<JwtIssuerOptions>(options =>
    {
        options.Issuer = jwtAppSettingOptions[nameof(JwtIssuerOptions.Issuer)];
        options.Audience = jwtAppSettingOptions[nameof(JwtIssuerOptions.Audience)];
        options.SigningCredentials = new SigningCredentials(_signingKey, SecurityAlgorithms.HmacSha256);
    });

    // api user claim policy
    services.AddAuthorization(options =>
    {
        options.AddPolicy("Admin", policy => 
            policy.RequireClaim(Constants.JwtClaimIdentifiers.Role, Constants.JwtClaimValues.Admin));
    });

    services.AddSingleton(Configuration);

    services.AddIdentity<AppUser, IdentityRole>()
        .AddEntityFrameworkStores<ApplicationContext>()
        .AddDefaultTokenProviders();

    services.AddMvc();
}
```

### Configure
The Configure method is used to configure the HTTP request pipeline. This is the app get configured to authenticate users using the JWT. Here's the method with everything wired up.

```c#
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    loggerFactory.AddConsole(Configuration.GetSection("Logging"));
    loggerFactory.AddDebug();

    // adds a test user and role
    IdentityData.Initialize(app.ApplicationServices);

    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
        app.UseWebpackDevMiddleware(new WebpackDevMiddlewareOptions {
            HotModuleReplacement = true
        });
    }
    else
    {
        app.UseExceptionHandler("/Home/Error");
    }

    var jwtAppSettingOptions = Configuration.GetSection(nameof(JwtIssuerOptions));
    var tokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = jwtAppSettingOptions[nameof(JwtIssuerOptions.Issuer)],

        ValidateAudience = true,
        ValidAudience = jwtAppSettingOptions[nameof(JwtIssuerOptions.Audience)],

        ValidateIssuerSigningKey = true,
        IssuerSigningKey = _signingKey,

        RequireExpirationTime = false,
        ValidateLifetime = false,
        ClockSkew = TimeSpan.Zero
    };

    app.UseJwtBearerAuthentication(new JwtBearerOptions
    {
        AutomaticAuthenticate = true,
        AutomaticChallenge = true,
        TokenValidationParameters = tokenValidationParameters
    });

    app.UseStaticFiles();

    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");

        routes.MapSpaFallbackRoute(
            name: "spa-fallback",
            defaults: new { controller = "Home", action = "Index" });
    });
}
```

## Auth Controller
Now that everything is set up, a login method can be addded to the Auth Controller that will check a users credentials and generate a token based on the Claims Identity it creates. You can see the entire AuthController.cs file [here.](https://github.com/sheelersoft/AngularTemplate/blob/master/content/Web/jssAngular/Controllers/AuthController.cs)

## Testing
To test we can just add an Authorize attribute to a controller and use the Admin policy that was set up in Startup.cs. For an example, look [here.](https://github.com/sheelersoft/AngularTemplate/blob/master/content/Web/jssAngular/Controllers/SampleDataController.cs) The attribute will look like this.

```c#
[Authorize(Policy = "Admin")]
```

Now, you can use Postman or something similar to make http calls to your app. You'll need to call the login controller first to get your token. Once you have that you can include the token in requests to actions that have the Authorize attribute on them.