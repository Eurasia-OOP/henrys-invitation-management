# Order Pipeline — C# / .NET / PostgreSQL / Docker Setup

 ## 1\. Project overview

 This document describes the development setup for the **Order Pipeline** project.

 Target environment:

 - Windows 11
- Intel x86\_64
- Docker Desktop
- WSL2
- Linux containers
- C#
- .NET 8
- ASP.NET Core
- Entity Framework Core
- PostgreSQL 16
- Docker Compose

 The target architecture is:

```
Windows 11
└── Docker Desktop
    └── WSL2 / Linux containers
        ├── order-pipeline
        │   └── ASP.NET Core / C#
        │       └── Entity Framework Core
        │
        └── postgres
            └── PostgreSQL 16
                └── postgres_data volume
```

---

 ## 2\. Prerequisites

 Verify WSL2:

```
wsl --status
wsl -l -v
```

 Verify Docker:

```
docker --version
docker compose version
docker info
```

 Docker Desktop should have WSL2 enabled and Ubuntu integrated.

---

 ## 3\. Project location

 From WSL:

```
mkdir -p ~/projects
cd ~/projects

mkdir order-pipeline
cd order-pipeline
```

 Recommended:

```
/home/<user>/projects/order-pipeline
```

---

 ## 4\. Create the .NET solution

```
dotnet new sln -n Order.Pipeline
```

 Create the Web API:

```
mkdir -p src
cd src

dotnet new webapi -n Order.Pipeline

cd ..
```

 Add it to the solution:

```
dotnet sln Order.Pipeline.sln add \
  src/Order.Pipeline/Order.Pipeline.csproj
```

---

 ## 5\. Install Entity Framework Core

```
dotnet add src/Order.Pipeline/ package Microsoft.EntityFrameworkCore
```

 Install PostgreSQL support:

```
dotnet add src/Order.Pipeline/ package Npgsql.EntityFrameworkCore.PostgreSQL
```

 Install EF tooling:

```
dotnet tool install --global dotnet-ef
```

 Or:

```
dotnet tool update --global dotnet-ef
```

 Verify:

```
dotnet ef --version
```

---

 ## 6\. Docker Compose

 Create:

```
docker-compose.yml
```

```
services:

  order-pipeline:
    container_name: order-pipeline

    build:
      context: .
      dockerfile: Dockerfile

    ports:
      - "8081:8080"

    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ASPNETCORE_HTTP_PORTS: 8080

      ConnectionStrings__DefaultConnection: "Host=postgres;Port=5432;Database=order_pipeline;Username=postgres;Password=postgres"

    depends_on:
      postgres:
        condition: service_healthy

    restart: unless-stopped

  postgres:
    container_name: order-pipeline-postgres

    image: postgres:16

    environment:
      POSTGRES_DB: order_pipeline
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres

    ports:
      - "5433:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U postgres -d order_pipeline"
        ]
      interval: 5s
      timeout: 5s
      retries: 10

    restart: unless-stopped

volumes:
  postgres_data:
```

 ### Port strategy

 The application uses:

```
localhost:8081
```

 PostgreSQL is exposed to the Windows/WSL host on:

```
localhost:5433
```

 Inside Docker, the application still connects using:

```
Host=postgres
Port=5432
```

---

 ## 7\. Dockerfile

 Create:

```
Dockerfile
```

```
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

COPY Order.Pipeline.sln ./
COPY src/Order.Pipeline/Order.Pipeline.csproj src/Order.Pipeline/

RUN dotnet restore "src/Order.Pipeline/Order.Pipeline.csproj"

COPY . .

WORKDIR /src/src/Order.Pipeline

RUN dotnet build "Order.Pipeline.csproj" \
    -c Release \
    --no-restore \
    -o /app/build

FROM build AS publish

RUN dotnet publish "Order.Pipeline.csproj" \
    -c Release \
    --no-restore \
    -o /app/publish \
    /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final

WORKDIR /app

COPY --from=publish /app/publish .

ENV ASPNETCORE_HTTP_PORTS=8080

EXPOSE 8080

ENTRYPOINT ["dotnet", "Order.Pipeline.dll"]
```

---

 ## 8\. .dockerignore

```
**/bin/
**/obj/

.git/
.gitignore

.vs/
.vscode/
.idea/

.env
.env.*

TestResults/
coverage/

Dockerfile*
docker-compose*.yml
```

---

 ## 9\. DbContext

 Create:

```
src/Order.Pipeline/Data/ApplicationDbContext.cs
```

```
using Microsoft.EntityFrameworkCore;

namespace Order.Pipeline.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    // Add order pipeline entities here.
    // Example:
    //
    // public DbSet<Order> Orders => Set<Order>();
    // public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    // public DbSet<OrderStatus> OrderStatuses => Set<OrderStatus>();
}
```

---

 ## 10\. Configure PostgreSQL

 In `Program.cs`:

```
using Microsoft.EntityFrameworkCore;
using Order.Pipeline.Data;

var builder = WebApplication.CreateBuilder(args);

var connectionString =
    builder.Configuration.GetConnectionString("DefaultConnection");

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql(connectionString));

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

 Merge this configuration with any existing application configuration.

---

 ## 11\. Development connection string

 `appsettings.Development.json`:

```
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5433;Database=order_pipeline;Username=postgres;Password=postgres"
  }
}
```

 Docker Compose overrides this with:

```
Host=postgres;Port=5432
```

---

 ## 12\. Start PostgreSQL

```
docker compose up -d postgres
```

 Check:

```
docker compose ps
```

 Test:

```
docker exec -it order-pipeline-postgres \
  psql -U postgres -d order_pipeline
```

 Run:

```
SELECT version();
```

 Exit:

```
\q
```

---

 ## 13\. EF migrations

 After the domain entities are implemented:

```
dotnet ef migrations add InitialCreate \
  --project src/Order.Pipeline \
  --startup-project src/Order.Pipeline
```

 Apply:

```
dotnet ef database update \
  --project src/Order.Pipeline \
  --startup-project src/Order.Pipeline
```

---

 ## 14\. Build and start

```
dotnet restore
dotnet build
```

 Then:

```
docker compose build
docker compose up -d
```

 Check:

```
docker compose ps
```

---

 ## 15\. Application URL

 The application is available at:

```
http://localhost:8081
```

 Swagger, if enabled:

```
http://localhost:8081/swagger
```

---

 ## 16\. Logs

```
docker compose logs -f order-pipeline
```

 PostgreSQL:

```
docker compose logs -f postgres
```

 All:

```
docker compose logs -f
```

---

 ## 17\. Stop

```
docker compose down
```

 Do not use `docker compose down -v` unless you intentionally want to remove the local PostgreSQL data.

---

 ## 18\. Final structure

```
order-pipeline/
│
├── src/
│   └── Order.Pipeline/
│       ├── Controllers/
│       ├── Data/
│       │   └── ApplicationDbContext.cs
│       ├── Models/
│       ├── Services/
│       ├── Migrations/
│       ├── Program.cs
│       ├── appsettings.json
│       ├── appsettings.Development.json
│       └── Order.Pipeline.csproj
│
├── tests/
│
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .gitignore
└── Order.Pipeline.sln
```

---

 ## 19\. Normal development commands

```
docker compose up -d

docker compose up -d --build

docker compose ps

docker compose logs -f

docker compose down
```

 EF Core:

```
dotnet ef migrations add MigrationName \
  --project src/Order.Pipeline \
  --startup-project src/Order.Pipeline

dotnet ef database update \
  --project src/Order.Pipeline \
  --startup-project src/Order.Pipeline
```
 | Project | API | PostgreSQL host port | Compose service |
| --- | --- | --- | --- |
| Order Pipeline | `8081` | `5433` | `order-pipeline` |


