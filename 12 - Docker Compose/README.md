# Dockerising the MathStack Solution

## Objective

In this guide, you will containerise the full MathStack solution so that:

- the SQL Server database runs in Docker
- the API runs in Docker
- the client runs in Docker
- the database schema is created from an SQL script
- environment variables are stored in a `.env` file
- the full stack can be started and stopped using Docker Compose

Repo to refer to: https://github.com/PROG7311-2026-EMWVL/MathStack

## Overall Project Structure

### File Structure

Your solution should look similar to this:

```text
MathStack/
│
├── .env
├── compose.yml
├── db/
│   └── init.sql
│
├── MathAPI/
│   ├── Dockerfile
│   ├── Program.cs
│   └── ...
│
└── MathAPIClient/
    ├── Dockerfile
    └── ...
```

### Setting up sub-folders 
1. Create a new folder called MathStack
1. Clone/Copy your MathAPI into this folder
1. Clone/Copy your MathAPIClient into this folder
1. Ensure that your new copy of MathAPI and MathAPIClient do not contain the following as you won't need them:
    - .git folders
    - /Guides
    - README.md
1. Run both locally to see that they are working correctly. If all runs as expected, move to the next steps.

### Initialise the git repo
1. Initialise a git repo in MathStack - we are now moving to a mono-repo structure.
1. Run `git add .` to add both projects and do an initial commit.

### Adding a .env file
1. Add a .env file with these variables - change to match your own
    ```
    SA_PASSWORD=your_db_pass_here
    FirebaseMathApp=your_firebase_api_key_here
    MathAppJwtKey=your_long_random_jwt_secret_here
    ```
1. Add a .gitignore file with at least the following:
    ```
    .env
    .vs/
    ```

### Add the database initialisation script

Since we will be running our DB in its own container, we need to configure the schema when that container starts up.

Create a folder named `db`, and inside it create a file named `init.sql`.

```sql
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = 'MathDB')
BEGIN
    CREATE DATABASE MathDB;
END
GO

USE MathDB;
GO

IF NOT EXISTS (
    SELECT * FROM sysobjects
    WHERE name = 'MathCalculations' AND xtype = 'U'
)
BEGIN
    CREATE TABLE MathCalculations
    (
        CalculationID INT IDENTITY(1,1) PRIMARY KEY,
        FirstNumber DECIMAL(18,2) NULL,
        SecondNumber DECIMAL(18,2) NULL,
        Operation INT NULL,
        Result DECIMAL(18,2) NULL,
        FirebaseUUID VARCHAR(512) NULL
    );
END
GO
```

#### What this does
- creates the `MathDB` database if it does not already exist
- switches into that database
- creates the `MathCalculations` table if it does not already exist

This follows a database-first approach using SQL scripts instead of EF Core migrations.

## API Changes

### Add a Dockerfile to the API project
Create a file named `Dockerfile` inside the `MathAPI` folder.

```
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base

USER root
RUN mkdir /keys && chown app:app /keys
RUN apt-get update && apt-get install -y libgssapi-krb5-2
USER app

WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

COPY . .
RUN dotnet restore "./MathAPI.csproj"
RUN dotnet publish "./MathAPI.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MathAPI.dll"]
```

#### What this does
- uses the .NET SDK image to build the API
- publishes the API into a release folder
- uses the smaller ASP.NET runtime image to run the API
- exposes port `8080` inside the container

### Update Program.cs
Update `Program.cs` to match the below to make sure the API reads its connection string and keys from configuration.

```
using MathAPI.Models;
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

var connectionString = builder.Configuration["Math_DB"];
var jwtKey = builder.Configuration["MathAppJwtKey"];
var firebaseApiKey = builder.Configuration["FirebaseMathApp"];

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddDbContext<MathDbContext>(options =>
    options.UseSqlServer(connectionString));

var key = Encoding.ASCII.GetBytes(jwtKey);

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.RequireHttpsMetadata = false;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(key),
        ValidateIssuer = false,
        ValidateAudience = false
    };
});


var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### Update AuthController.cs

Update the first several lines of `AuthController.cs` to use the configuration values by injecting `IConfiguration`.

```csharp
using Firebase.Auth;
using Microsoft.AspNetCore.Mvc;
using System.Text;

namespace MathAPI.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class AuthController : Controller
    {
        private readonly IConfiguration _configuration;
        private readonly FirebaseAuthProvider _auth;
        private readonly byte[] _key;

        public AuthController(IConfiguration configuration)
        {
            _configuration = configuration;

            var firebaseApiKey = _configuration["FirebaseMathApp"];
            var jwtKey = _configuration["MathAppJwtKey"];

            _auth = new FirebaseAuthProvider(new FirebaseConfig(firebaseApiKey));
            _key = Encoding.ASCII.GetBytes(jwtKey);
        }
    }
}
```

#### Why this matters
- Without dependency injection, the `configuration` object will be `null`, causing a `NullReferenceException`. 

## Client Changes

### Add a Dockerfile to the client project
Create a file named `Dockerfile` inside the `MathAPIClient` folder.

```
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base

USER root
RUN mkdir /keys && chown app:app /keys
RUN apt-get update && apt-get install -y libgssapi-krb5-2
USER app

WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

COPY . .
RUN dotnet restore "./MathAPIClient.csproj"
RUN dotnet publish "./MathAPIClient.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MathAPIClient.dll"]
```

### Update Program.cs

Update Program.cs to use a single base URL and to use the configuration settings.

```
using Microsoft.Extensions.Options;
using Microsoft.AspNetCore.DataProtection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));

builder.Services.AddHttpClient("MathAPI", (sp, client) =>
{
    var options = sp.GetRequiredService<IOptions<ApiSettings>>().Value;
    client.BaseAddress = new Uri(options.BaseUrl);
});

builder.Services.AddControllersWithViews();

builder.Services.AddDistributedMemoryCache();

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});

builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/keys"))
    .SetApplicationName("MathAPIClient");

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseSession();

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();

public class ApiSettings
{
    public string BaseUrl { get; set; } = string.Empty;
}
```


### Update AuthController.cs and MathController.cs
Update the `AuthController.cs` to use the HTTP client that gets the base URL from configuration. Also some changes to avoid conflicts and session issues in containerized environments.

```
using System.Text;
using MathAPIClient.Models;
using Microsoft.AspNetCore.Mvc;
using Newtonsoft.Json;

namespace MathAPIClient.Controllers
{
    public class AuthController : Controller
    {
        private static HttpClient? httpClient;
        public AuthController(IConfiguration configuration)
        {
            if (httpClient == null)
            {
                var baseUrl = configuration["ApiSettings:BaseUrl"];

                if (string.IsNullOrWhiteSpace(baseUrl))
                {
                    throw new InvalidOperationException("ApiSettings:BaseUrl is missing.");
                }

                httpClient = new HttpClient
                {
                    BaseAddress = new Uri(baseUrl)
                };
            }
        }

        [HttpGet]
        public IActionResult Register()
        {
            return View();
        }

        [HttpPost]
        public async Task<IActionResult> Register(LoginModel login)
        {
            StringContent jsonContent = new(
                JsonConvert.SerializeObject(login),
                Encoding.UTF8,
                "application/json"
            );

            HttpResponseMessage response = await httpClient!.PostAsync("api/Auth/Register", jsonContent);

            if (response.IsSuccessStatusCode)
            {
                var jsonResponse = await response.Content.ReadAsStringAsync();
                AuthResponse? deserialisedResponse = JsonConvert.DeserializeObject<AuthResponse>(jsonResponse);

                if (deserialisedResponse == null)
                {
                    ViewBag.Result = "Invalid response from server.";
                    return View();
                }

                HttpContext.Session.SetString("currentUser", deserialisedResponse.UserId);
                HttpContext.Session.SetString("MathJWT", deserialisedResponse.Token);

                return RedirectToAction("Calculate", "Math");
            }
            else
            {
                ViewBag.Result = await response.Content.ReadAsStringAsync();
                return View();
            }
        }

        [HttpGet]
        public IActionResult Login()
        {
            return View();
        }

        [HttpPost]
        public async Task<IActionResult> Login(LoginModel login)
        {
            StringContent jsonContent = new(
                JsonConvert.SerializeObject(login),
                Encoding.UTF8,
                "application/json"
            );

            HttpResponseMessage response = await httpClient!.PostAsync("api/Auth/Login", jsonContent);

            if (response.IsSuccessStatusCode)
            {
                var jsonResponse = await response.Content.ReadAsStringAsync();
                AuthResponse? deserialisedResponse = JsonConvert.DeserializeObject<AuthResponse>(jsonResponse);

                if (deserialisedResponse == null)
                {
                    ViewBag.Result = "Invalid response from server.";
                    return View();
                }

                HttpContext.Session.SetString("currentUser", deserialisedResponse.UserId);
                HttpContext.Session.SetString("MathJWT", deserialisedResponse.Token);

                return RedirectToAction("Calculate", "Math");
            }
            else
            {
                ViewBag.Result = await response.Content.ReadAsStringAsync();
                return View();
            }
        }

        [HttpGet]
        public IActionResult LogOut()
        {
            HttpContext.Session.Clear();

            if (httpClient != null)
            {
                httpClient.DefaultRequestHeaders.Authorization = null;
            }

            Response.Headers["Cache-Control"] = "no-cache, no-store, must-revalidate";
            Response.Headers["Pragma"] = "no-cache";
            Response.Headers["Expires"] = "0";

            return RedirectToAction("Login", "Auth");
        }
    }
}
```

Also do the same for `MathController.cs`

```
using System.Net.Http.Headers;
using System.Text;
using MathAPIClient.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using Newtonsoft.Json;

namespace MathAPIClient.Controllers
{
    public class MathController : Controller
    {
        private static HttpClient? httpClient;
        public MathController(IConfiguration configuration)
        {
            if (httpClient == null)
            {
                var baseUrl = configuration["ApiSettings:BaseUrl"];

                if (string.IsNullOrWhiteSpace(baseUrl))
                {
                    throw new InvalidOperationException("ApiSettings:BaseUrl is missing.");
                }

                httpClient = new HttpClient
                {
                    BaseAddress = new Uri(baseUrl)
                };
            }
        }

        [ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
        public IActionResult Calculate()
        {
            var token = HttpContext.Session.GetString("MathJWT");

            if (token == null)
            {
                return RedirectToAction("Login", "Auth");
            }

            ViewBag.Operations = GetOperations();
            return View();
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        [ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
        public async Task<IActionResult> Calculate(decimal? FirstNumber, decimal? SecondNumber, int Operation)
        {
            var token = HttpContext.Session.GetString("MathJWT");
            if (token == null)
            {
                return RedirectToAction("Login", "Auth");
            }

            var currentUser = HttpContext.Session.GetString("currentUser");
            decimal? result = 0;
            MathCalculation mathCalculation;

            try
            {
                mathCalculation = MathCalculation.Create(FirstNumber, SecondNumber, Operation, result, currentUser);
            }
            catch (Exception ex)
            {
                ViewBag.Error = ex.Message;
                ViewBag.Operations = GetOperations();
                return View();
            }

            StringContent jsonContent = new(
                JsonConvert.SerializeObject(mathCalculation),
                Encoding.UTF8,
                "application/json"
            );

            var request = new HttpRequestMessage(HttpMethod.Post, "api/Math/PostCalculate");
            request.Content = jsonContent;
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

            HttpResponseMessage response = await httpClient!.SendAsync(request);

            if (response.IsSuccessStatusCode)
            {
                var jsonResponse = await response.Content.ReadAsStringAsync();
                MathCalculation? deserialisedResponse = JsonConvert.DeserializeObject<MathCalculation>(jsonResponse);

                if (deserialisedResponse == null)
                {
                    ViewBag.Result = "Invalid response from server.";
                    ViewBag.Operations = GetOperations();
                    return View();
                }

                ViewBag.Result = deserialisedResponse.Result;
                ViewBag.Operations = GetOperations();
                return View();
            }
            else
            {
                ViewBag.Result = await response.Content.ReadAsStringAsync();
                ViewBag.Operations = GetOperations();
                return View();
            }
        }

        [ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
        public async Task<IActionResult> History()
        {
            var token = HttpContext.Session.GetString("MathJWT");
            if (token == null)
            {
                return RedirectToAction("Login", "Auth");
            }

            var request = new HttpRequestMessage(HttpMethod.Get, "api/Math/GetHistory");
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

            HttpResponseMessage response = await httpClient!.SendAsync(request);

            if (response.IsSuccessStatusCode)
            {
                var jsonResponse = await response.Content.ReadAsStringAsync();
                List<MathCalculation>? deserialisedResponse =
                    JsonConvert.DeserializeObject<List<MathCalculation>>(jsonResponse);

                if (deserialisedResponse == null || deserialisedResponse.Count == 0)
                {
                    ViewBag.HistoryMessage = "No history exists";
                    return View(new List<MathCalculation>());
                }

                return View(deserialisedResponse);
            }
            else
            {
                ViewBag.HistoryMessage = "No history to show";
                return View(new List<MathCalculation>());
            }
        }

        [ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
        public async Task<IActionResult> Clear()
        {
            var token = HttpContext.Session.GetString("MathJWT");
            if (token == null)
            {
                return RedirectToAction("Login", "Auth");
            }

            var request = new HttpRequestMessage(HttpMethod.Delete, "api/Math/DeleteHistory");
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

            HttpResponseMessage response = await httpClient!.SendAsync(request);

            if (!response.IsSuccessStatusCode)
            {
                TempData["HistoryMessage"] = "Failed to clear history.";
            }

            return RedirectToAction("History");
        }

        private static List<SelectListItem> GetOperations()
        {
            return new List<SelectListItem>
            {
                new SelectListItem { Value = "1", Text = "+" },
                new SelectListItem { Value = "2", Text = "-" },
                new SelectListItem { Value = "3", Text = "*" },
                new SelectListItem { Value = "4", Text = "/" }
            };
        }
    }
}
```

### Update appsettings.json
Update `appsettings.json` to fall back to localhost to enable local (outside container) execution.
```
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ApiSettings": {
    "BaseUrl": "https://localhost:5001"
  }
}

```

## Docker Compose

### Add Docker Compose file
Add a `compose.yml' file to your project:
```
services:
  math-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: math-db
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "${SA_PASSWORD}"
    ports:
      - "1433:1433"
    volumes:
      - mathdb_data:/var/opt/mssql
    healthcheck:
      test: ["CMD-SHELL", "bash -c '</dev/tcp/localhost/1433'"]
      interval: 10s
      timeout: 5s
      retries: 20
      start_period: 20s

  db-init:
    image: mcr.microsoft.com/mssql-tools
    container_name: db-init
    depends_on:
      math-db:
        condition: service_healthy
    volumes:
      - ./db/init.sql:/init.sql
    entrypoint: >
      /bin/bash -c "
      /opt/mssql-tools/bin/sqlcmd -S math-db -U sa -P ${SA_PASSWORD} -Q 'SELECT 1' ;
      until [ $$? -eq 0 ]; do
        echo 'Waiting for SQL Server...';
        sleep 2;
        /opt/mssql-tools/bin/sqlcmd -S math-db -U sa -P ${SA_PASSWORD} -Q 'SELECT 1' ;
      done;
      echo 'Running init.sql...';
      /opt/mssql-tools/bin/sqlcmd -S math-db -U sa -P ${SA_PASSWORD} -i /init.sql;
      echo 'Database initialized';
      "
    restart: "no"

  math-api:
    build:
      context: ./MathAPI
      dockerfile: Dockerfile
    container_name: math-api
    depends_on:
      db-init:
        condition: service_completed_successfully
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      FirebaseMathApp: "${FirebaseMathApp}"
      MathAppJwtKey: "${MathAppJwtKey}"
      Math_DB: "Server=math-db,1433;Database=MathDB;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=True;"
    ports:
      - "8081:8080"

  math-client:
    build:
      context: ./MathAPIClient
      dockerfile: Dockerfile
    container_name: math-client
    depends_on:
      - math-api
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ApiSettings__BaseUrl: "http://math-api:8080"
    ports:
      - "8082:8080"
    volumes:
      - client-dataprotection-keys:/keys

volumes:
  mathdb_data:
  client-dataprotection-keys:
```

### What does this file do?

- Overall
    - Runs the full stack using Docker Compose
    - Includes database, setup step, API, and client
    - Ensures things start in the correct order
- math-db (SQL Server)
    - Runs SQL Server in a container
    - Uses SA_PASSWORD from .env for the sa user
    - Exposes database on localhost:1433
    - Stores data in a volume (mathdb_data) so it is not lost
    - Has a healthcheck to know when SQL Server is ready
- db-init (Database setup)
    - Waits for SQL Server to be ready
    - Runs init.sql
        - creates database (MathDB)
        - creates tables (e.g. MathCalculations)
    - Runs once and then stops
    - Ensures database exists before API starts
- math-api (Backend)
    - Builds and runs the API from MathAPI
    - Waits for db-init to finish successfully
    - Uses environment variables:
        - database connection string
        - Firebase key
        - JWT key
    - Connects to SQL Server using math-db
    - Exposed on http://localhost:8081
- math-client (Frontend)
    - Builds and runs the client from MathAPIClient
    - Waits for API to start
    - Calls API using http://math-api:8080 (container name)
    - Exposed on http://localhost:8082
- Networking
    - All containers are on the same network
    - They talk to each other using service names:
        - API → math-db
        - Client → math-api
        - etc.
- Volume
    - mathdb_data
        - stores database data permanently
        - data persists even if containers stop
    - client-dataprotection-keys
        - stores the antiforgery tokens needed by the frontend between runs

- Startup order
    - SQL Server starts
    - DB script runs
    - API starts
    - Client starts

### Commands to use
* Start everything:
    ```
    docker compose up --build
    ```
* Stop everything:
    ```
    docker compose down
    ```
* Reset everything (including database):
    ```
    docker compose down -v
    ```