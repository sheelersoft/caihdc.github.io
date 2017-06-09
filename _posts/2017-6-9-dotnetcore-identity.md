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

```
...> cd coreapitest/server
```

Then you should execute the command to create a new ASP.NET Core project with the web api template. This will give us some basic set up and a values controller we'll use for testing later.

```
coreapitest/server> dotnet new webapi
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
coreapitest/server> dotnet restore
```

## IdentityContext
The first thing to do is create the IdentityContext class. This an EF Core db context that will be used to set up the Identity providers in the Startup class. Don't worry about creating a database as you can use the In Memory Database Provider for testing. Normally, the context classes would be seperated into some sort of domain layer but for this example just create the IdentityContext.cs file in the root of the project. The class is simple and it's using all build in classes like IdentityUser.

```c#
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using System;

namespace coreapitest.api
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

namespace coreapitest.api
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
                    roleStore.CreateAsync(new IdentityRole(role) { NormalizedName = role.ToUpper() }).Wait();
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

        public static async Task<IdentityResult> AssignRoles(IServiceProvider services, string email, string[] roles)
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

## AuthController

## ValuesController

## Testing