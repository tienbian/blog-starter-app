---
title: 'Build an Authentication and REST API application with Web API ASP.NET Core
        (Part1: Authentication API)'
excerpt: 'in a .NET series'
coverImage: '/assets/blog/preview/cover.jpg'
date: '2022-01-09T05:35:07.322Z'
author:
  name: TienTran
  picture: '/assets/blog/authors/jj.jpeg'
ogImage:
  url: '/assets/blog/hello-world/cover.jpg'
---
```sh
var user = new User();
user.fullName = 'Tran Vu Quoc Tien';
user.signature = 'from ruby with love';
_context.Users.Add(user);
await _context.SaveChangesAsync();
```
Since .NET Core has introduced along with ASP.NET Core, it provide a free, open-source, and cross-platform web framework for building cloud-based applications, such as web apps, IoT apps, and mobile backends. It is designed to run on the cloud as well as on-premises. As you are reading this post, I'm assuming that you are familiar with REST API, Postgresql, DB migrations, Docker.
Let's say that we want to create an API to support a fake web application based on Netflix where people who has account can upload movies. We will call it NextMovie.

# The setup 🛠
You need to have .NET 6.0 SDK installed.
Check your .NET SDK version by
```sh
dotnet --list-sdks
```
No more waiting. Let's start now.

This tutorial creates the following API:
```table
API                         Description               Request body        Response body
-----------------------------------------------------------------------------------------
POST /api/v1/register       Register a user           User info           None
POST /api/v1/login          Login an account          email and password  Access token
GET /api/v1/movies          List all movies           None                List movies
POST /api/v1/movie          Add a new movie           A movie             A movie
PUT /api/v1/movie/{id}      Update an existing movie  A movie             None
DELETE /api/movie/{id}      Delete a movie            None                None
```

# Project initialization
First of all, we need to create a new folder named *next_movie*.
From the terminal, getting into our new folder, we type

```sh
mkdir next_movie
cd next_movie
dotnet new webapi -f net6.0
```
This last command creates the files for a basic web API project that uses controllers, along with a C# sample project file named ContosoPizza.csproj that will return a list of weather forecasts.

From inside the next_movie directory, you can run the application with:
```sh
$ dotnet watch
```
Browser window open up and go to *https://localhost:7273/swagger* to find a Swagger UI listing our API’s endpoints with 7273 is the random port.

Then we need to install some packages to support PostgreSQL, CodeGenerator, JWT, Entity Framework.

```sh
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet tool install --global dotnet-aspnet-codegenerator
dotnet tool install --global dotnet-ef
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore
dotnet add package EFCore.NamingConventions
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Identity
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

# Add an Authentication operation

We will store user credentials in an PostgreSQL database and we will use Entity framework and ASP.NET Identity  for database operations.

Firstly, we should set up ASP.NET Identity for generating a membership system work like *Devise in Ruby on Rails*.
## Setup Database and Identity
We should connect to PostgreSQL database before do the migration and generating.
### Connecting to the database and performing initial app configuration
Create an *ApplicationUser* class inside a new folder *Authentication* which will inherit the IdentityUser class for adding custom user data to Identity. IdentityUser class is a part of Microsoft Identity framework. We will create all the authentication related files inside the *Authentication* folder.
#### Add custom user data
```sh
## ~/next_movie/Authentication/ApplicationUser.cs
using Microsoft.AspNetCore.Identity;

namespace nextMovie.Authentication
{
    public class ApplicationUser: IdentityUser
    {
      [PersonalData]
      public string ? Name { get; set; }
      [PersonalData]
      public DateTime DOB { get; set; }
    }
}
```
#### Add DBcontext
In order to connect to, query, and modify a database using EF Core, we need to create a DbContext. This is a class that serves as the entry point into the database. Create a new directory called Data in the project root and add this new ApplicationDbContext.cs file to it.
```sh
## ~/next_movie/Data/ApplicationDbContext.cs
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using nextMovie.Authentication;

namespace nextMovie.Data
{
    public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options)
        {

        }
        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);
        }
    }
}
```

#### Set environment variables
Firstly, we shoud register some environment variables related to DB and JWT.
In development environment, we will use [Secret Manager](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-6.0&tabs=linux) tool for set some environment variables.
In production environment, we will use [Azure Key Vault](https://docs.microsoft.com/en-us/aspnet/core/security/key-vault-configuration?view=aspnetcore-6.0) for store secrets.

The Secret Manager tool includes an init command. To use user secrets, run the following command in the project directory:

```sh
dotnet user-secrets init
```
Let's set some secrets
```sh
dotnet user-secrets set "Host" "localhost"
dotnet user-secrets set "Database" "NextMovie"
dotnet user-secrets set "Username" "mba0313"
dotnet user-secrets set "Password" ""
dotnet user-secrets set "JWT:ValidAudience" "http://localhost:4200"
dotnet user-secrets set "JWT:ValidIssuer" "http://localhost:61955"
dotnet user-secrets set "JWT:Secret" "ByYM000OLlMQG6VVVp1OH7Xzyr7gHuw1qvUC5dcGt3SNM"
```
Check list secrets
```sh
dotnet user-secrets list
```
#### Register the database context and register Identity
Update Program.cs with the following code:
For full configuration Identity see [here](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity-configuration?view=aspnetcore-6.0).
```sh
## ~/next_movie/Program.cs
using nextMovie.Data;
using Microsoft.EntityFrameworkCore;
using nextMovie.Authentication;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
var config     = builder.Configuration;
var DBHost     = config["Host"];
var DBUserName = config["Username"];
var DBDatabase = config["Database"];
var DBPassword = config["Password"];

var connectionString = $"Host={DBHost};Database={DBDatabase};Username={DBUserName};Password={DBPassword};Include Error Detail=True";

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql(connectionString)
           .UseSnakeCaseNamingConvention()
           .UseLoggerFactory(LoggerFactory.Create(builder => builder.AddConsole()))
           .EnableSensitiveDataLogging()
  );
builder.Services.AddDatabaseDeveloperPageExceptionFilter();
builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
                .AddEntityFrameworkStores<ApplicationDbContext>()
                .AddDefaultTokenProviders();

builder.Services.AddAuthentication(options =>
                  {
                      options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
                      options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
                      options.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
                  })
                .AddJwtBearer(options =>
                  {
                      options.SaveToken = true;
                      options.RequireHttpsMetadata = false;
                      options.TokenValidationParameters = new TokenValidationParameters()
                      {
                          ValidateIssuer = true,
                          ValidateAudience = true,
                          ValidAudience = config["JWT:ValidAudience"],
                          ValidIssuer = config["JWT:ValidIssuer"],
                          IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["JWT:Secret"]))
                      };
                  });

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```
### Running DB migration
We must create a database and required tables before running the application. As we are using entity framework, we can use below database migration command with package manger console to create a migration script.
```sh
dotnet ef migrations add InitIdentity --context ApplicationDbContext
```
Now, to actually run the migration script and apply the changes to the database, we do:

```sh
dotnet ef database update
```
That command inspects our project to find any migrations that haven’t been run yet, and applies them. In this case, we only have one, so that’s what it runs. 
It's will create some tables
- AspNetRoleClaims
- AspNetRoles
- AspNetUserClaims
- AspNetUserLogins
- AspNetUserRoles
- AspNetUsers
- AspNetUserTokens

Now we should implement the authentication process.
## Authentication API
Firstly we should setting the roles and model for register and login action
### Add user roles
We will have 2 roles *Admin* and *User*.
Create a static class *UserRoles* inside the *Authentication* folder and add below values.
```sh
## ~/next_movie/Authentication/UserRoles.cs
namespace nextMovie.Authentication  
{  
    public static class UserRoles  
    {  
        public const string Admin = "Admin";  
        public const string User = "User";  
    }  
}
```
### Add models login and RegisterAccount class
A model is a set of classes that represent the data that the app manages.
Login class will be used for Login API.
RegisterAccount class will be used for Register API.
Create a new directory called Models in the project root and add this new Login.cs file to it.
Validation *Required* is getting from [DataAnnotations](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0)
```sh
## ~/next_movie/Models/Login.cs
using System.ComponentModel.DataAnnotations;

namespace nextMovie.Authentication
{
    public class Login
    {
        [Required(ErrorMessage = "User Name is required")]
        public string Username { get; set; }

        [Required(ErrorMessage = "Password is required")]
        public string Password { get; set; }
    }
}
```
Add this new RegisterAccount.cs file to Models folder.
```sh
## ~/next_movie/Models/RegisterAccount.cs
using System.ComponentModel.DataAnnotations;

namespace nextMovie.Authentication
{
    public class RegisterAccount
    {
        [Required(ErrorMessage = "User Name is required")]
        public string Username { get; set; }

        [EmailAddress]
        [Required(ErrorMessage = "Email is required")]
        public string Email { get; set; }

        [Required(ErrorMessage = "Password is required")]
        public string Password { get; set; }

    }
}
```
We can create a class “Response” for returning the response value after user registration and user login. It will also return error messages, if the request fails.
### Add custom response format
```sh
## ~/next_movie/Authentication/Response.cs
namespace nextMovie.Authentication
{
    public class Response
    {
        public string Status { get; set; }
        public string Message { get; set; }
    }
}
```
We can create an API controller *AuthenticateController* inside the *Controllers* folder and add below code.
### Add Authentication controller

```sh
## ~/next_movie/Controllers/AuthenticateController.cs
using nextMovie.Authentication;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

namespace nextMovie.Controllers
{
    [Route("api/v1/[controller]")]
    [ApiController]
    public class AuthenticateController : ControllerBase
    {
        private readonly UserManager<ApplicationUser> userManager;
        private readonly RoleManager<IdentityRole> roleManager;
        private readonly IConfiguration _configuration;

        public AuthenticateController(UserManager<ApplicationUser> userManager, RoleManager<IdentityRole> roleManager, IConfiguration configuration)
        {
            this.userManager = userManager;
            this.roleManager = roleManager;
            _configuration = configuration;
        }

        [HttpPost]
        [Route("login")]
        public async Task<IActionResult> Login([FromBody] Login model)
        {
            var user = await userManager.FindByNameAsync(model.Username);
            if (user != null && await userManager.CheckPasswordAsync(user, model.Password))
            {
                var userRoles = await userManager.GetRolesAsync(user);

                var authClaims = new List<Claim>
                {
                    new Claim(ClaimTypes.Name, user.UserName),
                    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
                };

                foreach (var userRole in userRoles)
                {
                    authClaims.Add(new Claim(ClaimTypes.Role, userRole));
                }

                var authSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["JWT:Secret"]));

                var token = new JwtSecurityToken(
                    issuer: _configuration["JWT:ValidIssuer"],
                    audience: _configuration["JWT:ValidAudience"],
                    expires: DateTime.Now.AddHours(3),
                    claims: authClaims,
                    signingCredentials: new SigningCredentials(authSigningKey, SecurityAlgorithms.HmacSha256)
                    );

                return Ok(new
                {
                    token = new JwtSecurityTokenHandler().WriteToken(token),
                    expiration = token.ValidTo
                });
            }
            return Unauthorized();
        }

        [HttpPost]
        [Route("register")]
        public async Task<IActionResult> Register([FromBody] RegisterAccount model)
        {
            var userExists = await userManager.FindByNameAsync(model.Username);
            if (userExists != null)
                return Conflict();

            ApplicationUser user = new ApplicationUser()
            {
                Email = model.Email,
                SecurityStamp = Guid.NewGuid().ToString(),
                UserName = model.Username
            };
            var result = await userManager.CreateAsync(user, model.Password);
            if (!result.Succeeded)
                return StatusCode(StatusCodes.Status500InternalServerError, new Response { Status = "Error", Message = result.Errors.ToString() });

            return Ok();
        }
    }
}
```
## Test Authenticate API
You can test Register API with this call
```sh
curl --location --request POST 'https://localhost:7273/api/v1/authenticate/register' \
--header 'Content-Type: application/json' \
--header 'Cookie: _session_id=7b5bae51b89cb6efcf736635e3edf715' \
--data-raw '{
    "username": "tien1",
    "email": "tientranvu@gmail.com",
    "password": "Abc123!!!"

}'
```
And test Login API with this call
```sh
curl --location --request POST 'https://localhost:7273/api/v1/authenticate/login' \
--header 'Content-Type: application/json' \
--header 'Cookie: _session_id=7b5bae51b89cb6efcf736635e3edf715' \
--data-raw '{
    "username": "tien1",
    "password": "Abc123!!!"
}'
```
Finish Authenticate API, we will do the REST APIs which requires authentication to action.