# Point of Sale and Stock — C# / .NET / PostgreSQL / Docker Setup

 ## 1\. Project overview

 This document describes the development setup for the **Point of Sale and Stock** project.

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
        ├── pos-stock
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

 Docker Desktop should have:

```
Use the WSL 2 based engine
```

 enabled.

 Ubuntu should be enabled under:

```
Docker Desktop
└── Settings
    └── Resources
        └── WSL Integration
```

---

 ## 3\. Project location

 From WSL:

```
mkdir -p ~/projects
cd ~/projects

mkdir pos-stock
cd pos-stock
```

 Use the Linux filesystem:

```
/home/<user>/projects/pos-stock
```

 rather than:

```
/mnt/c/Users/...
```

 for the development workspace.

---

 ## 4\. Create the .NET solution

```
dotnet new sln -n Pos.Stock
```

 Create the Web API:

```
mkdir -p src
cd src

dotnet new webapi -n Pos.Stock

cd ..
```

 Add it to the solution:

```
dotnet sln Pos.Stock.sln add \
  src/Pos.Stock/Pos.Stock.csproj
```

---

 ## 5\. Install Entity Framework Core

```
dotnet add src/Pos.Stock/ package Microsoft.EntityFrameworkCore
```

 Install PostgreSQL:

```
dotnet add src/Pos.Stock/ package Npgsql.EntityFrameworkCore.PostgreSQL
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

  pos-stock:
    container_name: pos-stock

    build:
      context: .
      dockerfile: Dockerfile

    ports:
      - "8082:8080"

    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ASPNETCORE_HTTP_PORTS: 8080

      ConnectionStrings__DefaultConnection: "Host=postgres;Port=5432;Database=pos_stock;Username=postgres;Password=postgres"

    depends_on:
      postgres:
        condition: service_healthy

    restart: unless-stopped

  postgres:
    container_name: pos-stock-postgres

    image: postgres:16

    environment:
      POSTGRES_DB: pos_stock
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres

    ports:
      - "5434:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U postgres -d pos_stock"
        ]
      interval: 5s
      timeout: 5s
      retries: 10

    restart: unless-stopped

volumes:
  postgres_data:
```

 ### Port strategy

 Application:

```
http://localhost:8082
```

 PostgreSQL from Windows/WSL:

```
localhost:5434
```

 PostgreSQL from the application container:

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

COPY Pos.Stock.sln ./
COPY src/Pos.Stock/Pos.Stock.csproj src/Pos.Stock/

RUN dotnet restore "src/Pos.Stock/Pos.Stock.csproj"

COPY . .

WORKDIR /src/src/Pos.Stock

RUN dotnet build "Pos.Stock.csproj" \
    -c Release \
    --no-restore \
    -o /app/build

FROM build AS publish

RUN dotnet publish "Pos.Stock.csproj" \
    -c Release \
    --no-restore \
    -o /app/publish \
    /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final

WORKDIR /app

COPY --from=publish /app/publish .

ENV ASPNETCORE_HTTP_PORTS=8080

EXPOSE 8080

ENTRYPOINT ["dotnet", "Pos.Stock.dll"]
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

 ## 9\. Database context

 Create:

```
src/Pos.Stock/Data/ApplicationDbContext.cs
```

```
using Microsoft.EntityFrameworkCore;

namespace Pos.Stock.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    // Add POS and stock entities here.
    //
    // Example:
    //
    // public DbSet<Product> Products => Set<Product>();
    // public DbSet<StockItem> StockItems => Set<StockItem>();
    // public DbSet<Sale> Sales => Set<Sale>();
    // public DbSet<SaleItem> SaleItems => Set<SaleItem>();
}
```

---

 ## 10\. Suggested domain areas

 The exact model should follow the project's requirements, but the POS/stock application will typically contain concepts such as:

```
Product
ProductCategory
StockItem
StockMovement
Warehouse
Register
Sale
SaleItem
Payment
Customer
```

 Do not create all of these automatically if they are not part of the application's requirements.

---

 ## 11\. Configure PostgreSQL

 In `Program.cs`:

```
using Microsoft.EntityFrameworkCore;
using Pos.Stock.Data;

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

 Merge this into the existing application startup configuration if the project already has one.

---

 ## 12\. Development connection string

 Create/update:

```
src/Pos.Stock/appsettings.Development.json
```

```
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5434;Database=pos_stock;Username=postgres;Password=postgres"
  }
}
```

 Docker Compose overrides the connection string with:

```
Host=postgres;Port=5432
```

---

 ## 13\. Start PostgreSQL

```
docker compose up -d postgres
```

 Check:

```
docker compose ps
```

 Expected:

```
pos-stock-postgres    Up (healthy)
```

 Connect:

```
docker exec -it pos-stock-postgres \
  psql -U postgres -d pos_stock
```

 Test:

```
SELECT version();
```

 Exit:

```
\q
```

---

 ## 14\. Create EF migration

 After implementing the actual POS/stock entities:

```
dotnet ef migrations add InitialCreate \
  --project src/Pos.Stock \
  --startup-project src/Pos.Stock
```

 Apply:

```
dotnet ef database update \
  --project src/Pos.Stock \
  --startup-project src/Pos.Stock
```

---

 ## 15\. Build the application

 Verify the application:

```
dotnet restore
dotnet build
```

 Build Docker:

```
docker compose build
```

 Start:

```
docker compose up -d
```

 Check:

```
docker compose ps
```

---

 ## 16\. Application URL

 The application is available at:

```
http://localhost:8082
```

 Swagger, if enabled:

```
http://localhost:8082/swagger
```

---

 ## 17\. Logs

 Application:

```
docker compose logs -f pos-stock
```

 PostgreSQL:

```
docker compose logs -f postgres
```

 All services:

```
docker compose logs -f
```

---

 ## 18\. Stop the project

```
docker compose down
```

 Keep the database volume by using the command above.

 Do not use:

```
docker compose down -v
```

 unless you intentionally want to delete the local PostgreSQL database.

---

 ## 19\. Final structure

```
pos-stock/
│
├── src/
│   └── Pos.Stock/
│       ├── Controllers/
│       ├── Data/
│       │   └── ApplicationDbContext.cs
│       ├── Models/
│       ├── Services/
│       ├── Migrations/
│       ├── Program.cs
│       ├── appsettings.json
│       ├── appsettings.Development.json
│       └── Pos.Stock.csproj
│
├── tests/
│
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .gitignore
└── Pos.Stock.sln
```

---

 ## 20\. Normal development commands

 Start:

```
docker compose up -d
```

 Rebuild:

```
docker compose up -d --build
```

 Status:

```
docker compose ps
```

 Logs:

```
docker compose logs -f
```

 Stop:

```
docker compose down
```

 Create migration:

```
dotnet ef migrations add MigrationName \
  --project src/Pos.Stock \
  --startup-project src/Pos.Stock
```

 Apply migration:

```
dotnet ef database update \
  --project src/Pos.Stock \
  --startup-project src/Pos.Stock
```

 These three projects use **different host ports** so you can run all three simultaneously:

 | Project | API | PostgreSQL host port | Compose service |
| --- | --- | --- | --- |
| Point of Sale & Stock | `8082` | `5434` | `pos-stock` |

