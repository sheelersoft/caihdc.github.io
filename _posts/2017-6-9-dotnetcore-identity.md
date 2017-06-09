---
layout: post
title: ASP.NET Core - Identity
---

## Overview
This post will go over the process of setting up an ASP.NET Core Web Api project to use the built in Identity solution. It will show how to set up the code and give examples of it working in action.

## Installation
To follow along, you'll need to have .NET Core installed. I'll also be using Visual Studio Code as my editor and terminal which I recommend for anyone trying this out. Finally, I'll be using Postman to test the api after it is set up.

## Create ASP.NET Core project
To create the project, first you'll need to navigate into the folder you want to create it in. For me, this was /coreapitest/server/coreapitest.api

```
> cd coreapitest/server/coreapitest.api
```

Then you should execute the command to create a new ASP.NET Core project with the web api template. This will give us some basic set up and a values controller we'll use for testing later.

```
coreapitest/server/coreapitest.api> dotnet new webapi
```

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

```
coreapitest/server/coreapitest.api> dotnet restore
```

## IdentityContext
The first thing to do is create the IdentityContext class. This an EF Core db context that will be used to set up the Identity providers in the Startup class. Don't worry about creating a database as you can use the In Memory Database Provider for testing. Normally, the context classes would be seperated into some sort of domain layer but for this example just create the IdentityContext.cs file in the root of the project. The class is simple and it's using all build in classes like IdentityUser.

```c#
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using System;

namespace yournamespace
{
    public class IdentityContext : IdentityDbContext<IdentityUser>
    {
        public IdentityContext(DbContextOptions<IdentityContext> options)
            : base(options)
        {
        }
    }
}
```

## IdentityData
After IdentityContext is created, the next thing to do is create the IdentityData class. This class is not actually required to use Identity but it is useful for testing once everything is finished. All this class is really doing is getting the IdentityContext, creating a user, and assigning it a role.

```c#
using System;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace yournamespace
{
    public class IdentityData
    {
        public static void Initialize(IServiceProvider serviceProvider)
        {
            var context = serviceProvider.GetService<IdentityContext>();

            string[] roles = new string[] { "Administrator", "Customer" };

            foreach (string role in roles)
            {
                var roleStore = new RoleStore<IdentityRole>(context);

                if (!context.Roles.Any(r => r.Name == role))
                {
                    roleStore.CreateAsync(new IdentityRole(role) 
                    { 
                        NormalizedName = role.ToUpper() 
                    }).Wait();
                }
            }

            var user = new IdentityUser
            {
                Email = "test@test.com",
                NormalizedEmail = "TEST@TEST.COM",
                UserName = "Admin",
                NormalizedUserName = "ADMIN",
                EmailConfirmed = true,
                PhoneNumberConfirmed = true,
                SecurityStamp = Guid.NewGuid().ToString("D")
            };


            if (!context.Users.Any(u => u.UserName == user.UserName))
            {
                var password = new PasswordHasher<IdentityUser>();
                var hashed = password.HashPassword(user,"password");
                user.PasswordHash = hashed;

                var userStore = new UserStore<IdentityUser>(context);
                userStore.CreateAsync(user).Wait();
            }

            context.SaveChanges();

            AssignRoles(serviceProvider, user.Email, new string[] { "Administrator" }).Wait();

            context.SaveChanges();
        }

        public static async Task<IdentityResult> AssignRoles(
            IServiceProvider services,
             string email, 
             string[] roles)
        {
            UserManager<IdentityUser> _userManager = services.GetService<UserManager<IdentityUser>>();
            IdentityUser user = await _userManager.FindByEmailAsync(email);
            var result = await _userManager.AddToRolesAsync(user, roles);

            return result;
        }
    }
}
```

## Startup
Now, time for the important stuff. Startup is where all the services and providers for the app get configured along with the request pipeline. The first thing to do is update the ConfigureServices method to configure the IdentityContext and to add the IdentityProviders. Note: there are other things added here that might not necessarily be needed for this example. Note 2: The AuthorizationPolicyBuilder is just setting up our api so that each api method requires Authorize by default and to allow anonymous the AllowAnonymousAttribute needs to be added.

```c#
    // This method gets called by the runtime. Use this method to add services to the container.
    public void ConfigureServices(IServiceCollection services)
    {
        // Add framework services.
        services.AddLogging();       
        // Add the Identity Context
        services.AddDbContext<IdentityContext>(opt => opt.UseInMemoryDatabase());
        services.AddSingleton<IConfigurationRoot>(Configuration);
        // Add the Identity providers
        services.AddIdentity<IdentityUser, IdentityRole>()
            .AddEntityFrameworkStores<IdentityContext>()
            .AddDefaultTokenProviders();
        services.AddMvc(config => 
        {
            var policy = new AuthorizationPolicyBuilder()
                .RequireAuthenticatedUser()
                .Build();
            config.Filters.Add(new AuthorizeFilter(policy));
        });
    }
``` 

After that, the Congifure method needs to be updated to initialize our IdentityData class and to tell the app to actually use Identity.

```c#
    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        loggerFactory.AddConsole(Configuration.GetSection("Logging"));
        loggerFactory.AddDebug();
        IdentityData.Initialize(app.ApplicationServices);
        app.UseIdentity();
        app.UseMvc();
    }
```

Here is the code for the whole Startup.cs file:

```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.Authorization;

namespace yournamespace
{
    public class Startup
    {
        public Startup(IHostingEnvironment env)
        {
            var builder = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                .AddEnvironmentVariables();
            Configuration = builder.Build();
        }

        public IConfigurationRoot Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            // Add framework services.
            services.AddLogging();       
            services.AddDbContext<IdentityContext>(opt => opt.UseInMemoryDatabase());     
            services.AddSingleton<IConfigurationRoot>(Configuration);
            services.AddIdentity<IdentityUser, IdentityRole>()
                .AddEntityFrameworkStores<IdentityContext>()
                .AddDefaultTokenProviders();
            services.AddMvc(config => 
            {
                var policy = new AuthorizationPolicyBuilder()
                    .RequireAuthenticatedUser()
                    .Build();
                config.Filters.Add(new AuthorizeFilter(policy));
            });
        }

        // This method gets called by the runtime. 
        // Use this method to configure the HTTP request pipeline.
        public void Configure(
            IApplicationBuilder app, 
            IHostingEnvironment env, 
            ILoggerFactory loggerFactory)
        {
            loggerFactory.AddConsole(Configuration.GetSection("Logging"));
            loggerFactory.AddDebug();
            IdentityData.Initialize(app.ApplicationServices);
            app.UseIdentity();
            app.UseMvc();
        }
    }
}
```

## AuthController
The AuthController will be the api controller that allows a user to login and logout from the client. It will have the Identity providers injected into the contstructors which will then be used to perform the login / logout logic. Since earlier I configured every controller to use Authorize by default, I had to AllowAnonymous on the Login method. It's pretty straightforward so here is the code.

```c#
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;

namespace yournamespace.Controllers
{
    [Route("api/[controller]")]
    public class AuthController : Controller
    {
        private readonly SignInManager<IdentityUser> _signInManager;
        private UserManager<IdentityUser> _userManager;

        public AuthController(UserManager<IdentityUser> userManager, SignInManager<IdentityUser> signInManager)
        {
            _userManager = userManager;
            _signInManager = signInManager;
        }

        [HttpPost("{username}/{password}/{rememberMe?}")]
        [AllowAnonymous]
        public async Task<IActionResult> Login(string username, string password, bool rememberMe = false)
        {
            var user = await _userManager.FindByNameAsync(username);
            if (user == null)
            {
                return NotFound(username);
            }

            var signInResult = await this._signInManager.PasswordSignInAsync(user, password, rememberMe, lockoutOnFailure: false);
            
            if (signInResult.Succeeded)
            {
                return Ok();
            }

            return Unauthorized();
        }

        [HttpGet("logout")]        
        public async Task<IActionResult> Logout()
        {
            await this._signInManager.SignOutAsync();

            return Ok();
        }
    }
}
```

## ValuesController
The ValuesController that is created with the Web Api template will be used to test the Identity providers and the login / logout methods. The only thing to change here is to either have a method or two allow anonymous or to add the Authorize attribute if you removed the code that configured the controllers to have Authorize on by default. Here is what my controller looks like and is what the Testing section assumes.

```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Configuration;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;

namespace yournamespace.Controllers
{
    [Route("api/[controller]")]
    public class ValuesController : Controller
    {
        private UserManager<IdentityUser> _userManager;
        public ValuesController(UserManager<IdentityUser> userManager)
        {
            _userManager = userManager;
        }

        // GET api/values
        [HttpGet]
        [AllowAnonymous]
        public IEnumerable<string> Get()
        {
            return new string[] { "value1", "value2" };
        }

        // GET api/values/5
        [HttpGet("{id}")]
        public string Get(int id)
        {
            return "value";
        }

        // POST api/values
        [HttpPost, Authorize]
        public void Post([FromBody]string value)
        {
            var user = _userManager.GetUserAsync(User);
        }

        // PUT api/values/5
        [HttpPut("{id}")]
        public void Put(int id, [FromBody]string value)
        {
        }

        // DELETE api/values/5
        [HttpDelete("{id}")]
        public void Delete(int id)
        {
        }
    }
}
```

## Testing
Using Postman, the api can be tested by sending requests to the end points. The first thing to do is run the app.

```
coreapitest/server/coreapitest.api> dotnet run
```

This should start the app and be listening on localhost:5000. Now that the app is running requests can be sent to the endpoints.

#### First, send a request to an endpoint that does not requre auth.

```
Get: localhost:5000/api/values
```

The response should be a 200 with some data.

#### Next, try sending a request to an endpoint that does require auth.

```
Post: localhost:5000/api/values
```

```
Body: { "value": "woooooooooooooooooo" }
```

This request should fail, possibly with a 404.

#### Now send a login request. The response to this, if successful, will contain a cookie which will be used to authenticate later. This also uses the test user we created earlier. Note: In a real app you probably would not want the password in the url.

```
Post: localhost:5000/api/auth/admin/password
```

The reponse should be a 200.

#### Try the endpoint that requres auth again.

```
Post: localhost:5000/api/values
```

```
Body: { "value": "woooooooooooooooooo" }
```

#### This time it should succeed with a 200, showing that the login worked.

Now logout

```
Get: localhost:5000/api/auth/logout
```

The reponse should be a 200.

### Try the endpoint that requres auth again.

```
Post: localhost:5000/api/values
```

```
Body: { "value": "woooooooooooooooooo" }
```

This time it should fail, showing that the logout worked.

Note: Postman included the cookies returned by the server automatically in the later requests which is similar to what a web browser does. If your login does not seem to work or the requests after login don't work, make sure to check the cookies passed with the request.