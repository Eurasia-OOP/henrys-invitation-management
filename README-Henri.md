# Invitations and RSVP — C# / .NET / PostgreSQL / Docker Setup

## 1. Project overview

This document describes the development setup for the **Invitations and RSVP** project.

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
        ├── henrys-invitation
        │   └── ASP.NET Core / C#
        │       └── Entity Framework Core
        │
        └── postgres
            └── PostgreSQL 16
                └── postgres_data volume
```

---

## 2. Prerequisites

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

## 3. Project location

From WSL:

```
mkdir -p ~/projects
cd ~/projects

mkdir henrys-invitation
cd henrys-invitation
```

Use the Linux filesystem:

```
/home/<user>/projects/henrys-invitation
```

rather than:

```
/mnt/c/Users/...
```

for the development workspace.

---

## 4. Create the .NET solution

```
dotnet new sln -n Henrys.Invitation
```

Create the Web API:

```
mkdir -p src
cd src

dotnet new webapi -n Henrys.Invitation

cd ..
```

Add it to the solution:

```
dotnet sln Henrys.Invitation.sln add \
  src/Henrys.Invitation/Henrys.Invitation.csproj
```

---

## 5. Install Entity Framework Core

```
dotnet add src/Henrys.Invitation/ package Microsoft.EntityFrameworkCore
```

Install PostgreSQL:

```
dotnet add src/Henrys.Invitation/ package Npgsql.EntityFrameworkCore.PostgreSQL
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

## 6. Docker Compose

Create:

```
docker-compose.yml
```

```yaml
services:

  henrys-invitation:
    container_name: henrys-invitation

    build:
      context: .
      dockerfile: Dockerfile

    ports:
      - "8084:8080"

    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ASPNETCORE_HTTP_PORTS: 8080

      ConnectionStrings__DefaultConnection: "Host=postgres;Port=5432;Database=henrys_invitation;Username=postgres;Password=postgres"

    depends_on:
      postgres:
        condition: service_healthy

    restart: unless-stopped

  postgres:
    container_name: henrys-invitation-postgres

    image: postgres:16

    environment:
      POSTGRES_DB: henrys_invitation
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres

    ports:
      - "5436:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U postgres -d henrys_invitation"
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
http://localhost:8084
```

PostgreSQL from Windows/WSL:

```
localhost:5436
```

PostgreSQL from the application container:

```
Host=postgres
Port=5432
```

---

## 7. Dockerfile

Create:

```
Dockerfile
```

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

COPY Henrys.Invitation.sln ./
COPY src/Henrys.Invitation/Henrys.Invitation.csproj src/Henrys.Invitation/

RUN dotnet restore "src/Henrys.Invitation/Henrys.Invitation.csproj"

COPY . .

WORKDIR /src/src/Henrys.Invitation

RUN dotnet build "Henrys.Invitation.csproj" \
    -c Release \
    --no-restore \
    -o /app/build

FROM build AS publish

RUN dotnet publish "Henrys.Invitation.csproj" \
    -c Release \
    --no-restore \
    -o /app/publish \
    /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final

WORKDIR /app

COPY --from=publish /app/publish .

ENV ASPNETCORE_HTTP_PORTS=8080

EXPOSE 8080

ENTRYPOINT ["dotnet", "Henrys.Invitation.dll"]
```

---

## 8. .dockerignore

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

## 9. Database context

Create:

```
src/Henrys.Invitation/Data/ApplicationDbContext.cs
```

```csharp
using Microsoft.EntityFrameworkCore;

namespace Henrys.Invitation.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    // Add invitation and RSVP entities here.
    //
    // Example:
    //
    // public DbSet<Event> Events => Set<Event>();
    // public DbSet<Invitation> Invitations => Set<Invitation>();
    // public DbSet<Guest> Guests => Set<Guest>();
    // public DbSet<Rsvp> Rsvps => Set<Rsvp>();
}
```

---

## 10. Suggested domain areas

The exact model should follow the project's requirements, but the Invitations and RSVP application will typically contain concepts such as:

```
Event
EventHost
Invitation
InvitationStatus
Guest
GuestGroup
Rsvp
RsvpResponse
Reminder
Template
```

Do not create all of these automatically if they are not part of the application's requirements.

---

## 11. Configure PostgreSQL

In `Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Henrys.Invitation.Data;

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

## 12. Development connection string

Create/update:

```
src/Henrys.Invitation/appsettings.Development.json
```

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5436;Database=henrys_invitation;Username=postgres;Password=postgres"
  }
}
```

Docker Compose overrides the connection string with:

```
Host=postgres;Port=5432
```

---

## 13. Start PostgreSQL

```
docker compose up -d postgres
```

Check:

```
docker compose ps
```

Expected:

```
henrys-invitation-postgres    Up (healthy)
```

Connect:

```
docker exec -it henrys-invitation-postgres \
  psql -U postgres -d henrys_invitation
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

## 14. Create EF migration

After implementing the actual invitation/RSVP entities:

```
dotnet ef migrations add InitialCreate \
  --project src/Henrys.Invitation \
  --startup-project src/Henrys.Invitation
```

Apply:

```
dotnet ef database update \
  --project src/Henrys.Invitation \
  --startup-project src/Henrys.Invitation
```

---

## 15. Build the application

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

## 16. Application URL

The application is available at:

```
http://localhost:8084
```

Swagger, if enabled:

```
http://localhost:8084/swagger
```

---

## 17. Logs

Application:

```
docker compose logs -f henrys-invitation
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

## 18. Stop the project

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

## 19. Final structure

```
henrys-invitation/
│
├── src/
│   └── Henrys.Invitation/
│       ├── Controllers/
│       ├── Data/
│       │   └── ApplicationDbContext.cs
│       ├── Models/
│       ├── Services/
│       ├── Migrations/
│       ├── Program.cs
│       ├── appsettings.json
│       ├── appsettings.Development.json
│       └── Henrys.Invitation.csproj
│
├── tests/
│
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .gitignore
└── Henrys.Invitation.sln
```

---

## 20. Normal development commands

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
  --project src/Henrys.Invitation \
  --startup-project src/Henrys.Invitation
```

Apply migration:

```
dotnet ef database update \
  --project src/Henrys.Invitation \
  --startup-project src/Henrys.Invitation
```

These three projects use **different host ports** so you can run all three simultaneously:

| Project | API | PostgreSQL host port | Compose service |
| --- | --- | --- | --- |
| Point of Sale & Stock (DHI) | `8082` | `5434` | `pos-stock` |
| Order Pipeline (JAM) | `8083` | `5435` | `jam-marketplace` |
| Invitations and RSVP (HAK) | `8084` | `5436` | `henrys-invitation` |
