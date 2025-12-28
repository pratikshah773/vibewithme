# Project Architecture & Technical Specifications — Multi-Tenant SaaS Event Platform

This document provides comprehensive architecture, technology stack, and technical requirements for **EventHub**: a flexible multi-tenant SaaS platform for hosting events, selling tickets/products, managing fundraisers, and processing payments.

**Platform Purpose:**
- **Event Organizers** create and manage events (concerts, conferences, workshops, etc.)
- **Event Organizers** sell tickets, products, and track fundraising
- **Buyers** discover, purchase tickets/products, and manage their bookings
- **Platform** processes payments to a centralized merchant account and manages payouts to organizers based on commission rules

**Last Updated:** December 28, 2025

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Multi-Tenancy & Organizational Model](#multi-tenancy--organizational-model)
3. [Frontend Architecture](#frontend-architecture)
4. [Backend Architecture](#backend-architecture)
5. [GraphQL API](#graphql-api)
6. [Database Design](#database-design)
7. [Payment Processing & Ledger System](#payment-processing--ledger-system)
8. [Multi-Offering Strategy (Events, Products, Fundraising)](#multi-offering-strategy)
9. [Observability & Monitoring](#observability--monitoring)
10. [CI/CD & Deployment](#cicd--deployment)
11. [Docker & Containerization](#docker--containerization)
12. [AWS Infrastructure & Terraform](#aws-infrastructure--terraform)
13. [Security & Compliance](#security--compliance)
14. [Cross-Service Communication](#cross-service-communication)
15. [Development Workflows](#development-workflows)

---

## Project Overview

### Tech Stack Summary

| Component | Technology | Version |
|-----------|-----------|---------|
| **Frontend** | Next.js | Stable (LTS) |
| **Backend** | ASP.NET Core | Latest Stable |
| **API** | GraphQL (Hot Chocolate) | Latest |
| **Database** | SQL Server (MSSQL) | Local Dev & RDS for AWS |
| **Payment Gateway** | Stripe / Razorpay | Production APIs |
| **Auth** | JWT + OAuth 2.0 | OpenID Connect |
| **Observability** | OpenTelemetry + Prometheus-net + Serilog | Latest |
| **Visualization** | Grafana Cloud | SaaS |
| **Container Runtime** | Docker | Latest |
| **Infrastructure as Code** | Terraform | Latest |
| **Cloud Provider** | AWS | N/A |
| **CI/CD** | GitHub Actions | Native |

### Key Architectural Principles

- **Multi-Tenancy:** Each organization (event organizer, seller) is isolated; platform is SaaS (single codebase, multi-tenant data).
- **Flexible Offerings:** Support events (ticketed), products (for sale), fundraisers (with goals), and more via polymorphic design.
- **Payment Centralization:** Platform collects all payments to merchant account; organizers receive payouts based on commissions.
- **Ledger-Driven:** Every transaction (ticket sold, product purchased, refund, payout) is logged in ledger for audit trail.
- **Observable by default:** All services emit metrics, logs, and traces from inception.
- **Infrastructure as Code:** All AWS resources defined in Terraform; no manual AWS console changes.
- **Containerized deployment:** All services run in Docker containers orchestrated on AWS (ECS/EKS or similar).
- **Stateless services:** Backend should maintain no in-process state; use MSSQL for persistence.
- **Role-Based Access Control (RBAC):** Support Organizer, Buyer, Admin, Support roles with granular permissions.

---

## Multi-Tenancy & Organizational Model

### Organizational Structure

EventHub supports three primary user types:

1. **Event Organizers** — Create events, sell tickets/products, track sales, request payouts
2. **Buyers** — Browse events, purchase tickets, attend events, leave reviews
3. **Platform Admins** — Monitor platform health, manage commissions, handle disputes, process refunds

### Multi-Tenant Data Isolation

Each organization (event organizer) is a separate tenant:

```
EventHub Platform
├── Tenant A (Organizer: ACME Events)
│   ├── Events (owned by Organizer A)
│   ├── Tickets (sold by Organizer A)
│   ├── Products (sold by Organizer A)
│   ├── Transactions (payments for Organizer A's items)
│   └── Payouts (due to Organizer A)
├── Tenant B (Organizer: XYZ Corp)
│   ├── Events, Tickets, Products, Transactions, Payouts
│   └── (isolated from Tenant A)
└── Platform Admin (Global - sees all tenants for monitoring)
```

### Database-Level Tenant Isolation

- Every table has `OrganizationId` (tenant identifier).
- GraphQL resolvers filter by `organizationId` from JWT claims.
- API Gateway/Middleware enforces tenant context per request.
- Row-level security via views (optional, recommended for RLS at DB level).

### Multi-Tenancy Example in Schema

```sql
-- Organizations table
CREATE TABLE Organizations (
    OrganizationId INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(255) NOT NULL,
    Slug NVARCHAR(100) UNIQUE,
    OwnerUserId INT NOT NULL,
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    IsActive BIT DEFAULT 1,
    FOREIGN KEY (OwnerUserId) REFERENCES Users(UserId)
);

-- Events belong to organizations
CREATE TABLE Events (
    EventId INT PRIMARY KEY IDENTITY,
    OrganizationId INT NOT NULL,  -- Tenant identifier
    Title NVARCHAR(255) NOT NULL,
    Description NVARCHAR(MAX),
    EventDate DATETIME2 NOT NULL,
    Venue NVARCHAR(MAX),
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    Status NVARCHAR(50),  -- 'draft', 'published', 'ended', 'cancelled'
    FOREIGN KEY (OrganizationId) REFERENCES Organizations(OrganizationId)
);

-- Tickets belong to events
CREATE TABLE Tickets (
    TicketId INT PRIMARY KEY IDENTITY,
    EventId INT NOT NULL,
    OrganizationId INT NOT NULL,  -- Denormalized for query efficiency
    TicketType NVARCHAR(100),  -- 'VIP', 'General', 'Student'
    Price DECIMAL(10,2) NOT NULL,
    Quantity INT NOT NULL,
    QuantitySold INT DEFAULT 0,
    FOREIGN KEY (EventId) REFERENCES Events(EventId),
    FOREIGN KEY (OrganizationId) REFERENCES Organizations(OrganizationId)
);
```

### JWT Claims for Tenant Context

```json
{
  "sub": "user-123",
  "name": "John Organizer",
  "email": "john@acmeevents.com",
  "organizationId": "org-456",
  "organizationSlug": "acme-events",
  "roles": ["organizer", "event_creator"],
  "permissions": ["create:event", "view:sales", "request:payout"]
}
```

---

### Framework & Setup

- **Framework:** Next.js (Stable/LTS version)
- **Directory Structure:**
  ```
  frontend/
  ├── app/                    # Next.js App Router (Pages & Layouts)
  ├── components/             # Reusable UI components
  ├── lib/                    # Utilities, hooks, helpers
  ├── public/                 # Static assets
  ├── styles/                 # Global CSS/Tailwind config
  ├── .env.local              # Local environment variables (DO NOT commit)
  ├── next.config.js          # Next.js configuration
  ├── package.json            # Dependencies & scripts
  └── tsconfig.json           # TypeScript configuration
  ```

### Key Conventions

1. **Component Organization:**
   - Use `app/` directory for page routes and nested layouts.
   - Keep client components (`use client`) in `components/` or near routes.
   - Prefer server components; mark client-side interactivity only where needed.

2. **API Integration:**
   - Call backend via **GraphQL queries and mutations** (primary API).
   - Use Apollo Client for GraphQL state management.
   - All requests include correlation IDs in headers for tracing.
   - Implement retry logic and error handling for transient failures.
   - GraphQL playground available at `http://localhost:5000/graphql` (local dev).

3. **Environment Variables:**
   - `.env.local` (local development, never committed)
   - `.env.production` (production values, committed if safe)
   - Reference via `process.env.NEXT_PUBLIC_*` (client-side) or `process.env.*` (server-side).

4. **Styling:**
   - Use Tailwind CSS by default.
   - Keep theme config in `styles/tailwind.config.js` for consistency.

5. **Build & Run:**
   - Dev: `npm run dev` → http://localhost:3000
   - Build: `npm run build`
   - Start production: `npm start`

---

## Backend Architecture

### Framework & Setup

- **Framework:** ASP.NET Core (latest stable)
- **Language:** C#
- **Project Structure:**
  ```
  backend/
  ├── src/
  │   ├── Api/                        # Controllers, endpoints, middleware
  │   ├── Application/                # Business logic, services
  │   ├── Domain/                     # Entities, value objects, interfaces
  │   ├── Infrastructure/             # DynamoDB, external APIs, logging
  │   └── Program.cs                  # DI container setup, middleware pipeline
  ├── tests/
  │   ├── Unit/                       # Unit tests
  │   ├── Integration/                # Integration tests
  │   └── E2E/                        # End-to-end tests
  ├── appsettings.json                # Default config
  ├── appsettings.Development.json    # Dev overrides
  ├── appsettings.Production.json     # Prod overrides (secrets from env)
  ├── .csproj                         # Project file with dependencies
  └── Dockerfile                      # Container image definition
  ```

### Key Conventions

1. **Dependency Injection:**
   - All services registered in `Program.cs`.
   - Use constructor injection; no service locator pattern.
   - Keep DI configuration centralized and documented.

2. **API Routes & Controllers:**
   - **Note:** REST controllers are deprecated in favor of GraphQL.
   - Keep controllers minimal; move logic to GraphQL resolvers.
   - Controllers only used for non-GraphQL endpoints (webhooks, health checks, etc.)

3. **Data Access:**
   - Entity Framework Core abstraction in `Infrastructure/Data/`.
   - All queries/mutations should be traced (OpenTelemetry instrumentation).
   - Implement caching strategies where needed (in-memory cache with TTL or Redis if scale requires).

4. **Configuration:**
   - Read from `appsettings*.json` and environment variables.
   - Secrets (API keys, connection strings) must come from environment variables or AWS Secrets Manager.
   - Never hardcode credentials.

5. **Build & Run:**
   - Dev: `dotnet watch run` (watches for changes, auto-restarts)
   - Build: `dotnet build`
   - Test: `dotnet test`
   - Publish: `dotnet publish -c Release`

---

## GraphQL API

### Framework & Setup

- **Framework:** Hot Chocolate (ASP.NET Core GraphQL server)
- **NuGet Packages:**
  ```xml
  <PackageReference Include="HotChocolate.AspNetCore" Version="latest" />
  <PackageReference Include="HotChocolate.Types" Version="latest" />
  ```

- **Schema Location:** `src/GraphQL/Schema/` — organize by feature (e.g., `UserSchema.graphql`, `OrderSchema.graphql`)
- **Resolvers:** `src/GraphQL/Resolvers/` — C# resolver classes for each type

### GraphQL Project Structure

```
backend/
├── src/
│   ├── GraphQL/
│   │   ├── Resolvers/
│   │   │   ├── UserResolver.cs        # Query/Mutation resolvers for User
│   │   │   ├── OrderResolver.cs       # Resolvers for Order
│   │   │   └── FormResolver.cs        # Resolvers for Forms (JSON data)
│   │   ├── Types/
│   │   │   ├── UserType.cs            # GraphQL type definition for User
│   │   │   ├── OrderType.cs           # GraphQL type for Order
│   │   │   └── FormDataType.cs        # Custom scalar or type for JSON form data
│   │   └── Queries.cs                 # Root Query type
│   ├── Application/
│   ├── Domain/
│   └── Infrastructure/
└── Program.cs
```

### Setup in Program.cs

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddTypes()  // Auto-register all GraphQL types
    .AddAuthorization()  // JWT authorization
    .ModifyRequestOptions(opts => opts.Complexity = new QueryComplexityAnalyzer());

// Add resolvers to DI
builder.Services.AddScoped<UserResolver>();
builder.Services.AddScoped<OrderResolver>();
builder.Services.AddScoped<FormResolver>();

// Map GraphQL endpoint
app.MapGraphQL("/graphql");

// Optional: GraphQL UI (Banana Cake Pop or Apollo Sandbox)
// Available at http://localhost:5000/graphql
```

### Schema Definition & Resolvers

**Example: User Query with JSON Data**

```csharp
// src/GraphQL/Queries.cs
[QueryType]
public class Query
{
    public async Task<User?> GetUserAsync(
        int userId,
        UserResolver resolver,
        CancellationToken ct)
    {
        return await resolver.GetUserAsync(userId, ct);
    }

    public async Task<List<User>> GetUsersAsync(
        UserResolver resolver,
        CancellationToken ct)
    {
        return await resolver.GetUsersAsync(ct);
    }

    public async Task<List<UserForm>> GetUserFormsAsync(
        int userId,
        FormResolver resolver,
        CancellationToken ct)
    {
        return await resolver.GetUserFormsByUserIdAsync(userId, ct);
    }
}

// src/GraphQL/Types/UserType.cs
[ObjectType("User")]
public class UserType
{
    public int UserId { get; set; }
    public string Email { get; set; }
    public string FullName { get; set; }
    public DateTime CreatedAt { get; set; }

    [GraphQLType(typeof(StringType))]
    public string? UserPreferences { get; set; }  // JSON field

    // Resolve nested forms
    [GraphQLName("forms")]
    public async Task<List<UserForm>> GetFormsAsync(
        [Parent] User user,
        FormResolver resolver,
        CancellationToken ct)
    {
        return await resolver.GetUserFormsByUserIdAsync(user.UserId, ct);
    }
}

// src/GraphQL/Types/FormDataType.cs - Custom scalar for JSON
public class FormDataType : ScalarType<string>
{
    public FormDataType() : base("FormData")
    {
        Description = "JSON object for form data (name, age, preferences, etc.)";
    }

    public override IValueNode ParseValue(object? runtimeValue)
    {
        if (runtimeValue is string json)
            return new StringValueNode(json);
        throw new ArgumentException("FormData must be a valid JSON string");
    }

    public override object? SerializeValue(object? runtimeValue) => runtimeValue;

    public override object? ParseLiteral(IValueNode valueSyntax)
    {
        return valueSyntax is StringValueNode stringNode ? stringNode.Value : null;
    }
}

// src/GraphQL/Resolvers/UserResolver.cs
public class UserResolver
{
    private readonly ApplicationDbContext _context;

    public UserResolver(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<User?> GetUserAsync(int userId, CancellationToken ct)
    {
        return await _context.Users.FirstOrDefaultAsync(u => u.UserId == userId, ct);
    }

    public async Task<List<User>> GetUsersAsync(CancellationToken ct)
    {
        return await _context.Users.ToListAsync(ct);
    }
}

// src/GraphQL/Resolvers/FormResolver.cs
public class FormResolver
{
    private readonly ApplicationDbContext _context;

    public FormResolver(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<List<UserForm>> GetUserFormsByUserIdAsync(int userId, CancellationToken ct)
    {
        return await _context.UserForms
            .Where(f => f.UserId == userId)
            .ToListAsync(ct);
    }
}
```

### GraphQL Queries (Examples)

**Query User with Forms:**
```graphql
query GetUserWithForms($userId: Int!) {
  user(userId: $userId) {
    userId
    email
    fullName
    userPreferences  # JSON field
    forms {
      formId
      formType
      formData  # JSON field - returns as string
      createdAt
    }
  }
}
```

**Query with JSON Filtering (Backend-side):**
```csharp
// In FormResolver.cs - filter by JSON field
public async Task<List<UserForm>> GetUserFormsByTypeAsync(
    int userId, 
    string formType,
    CancellationToken ct)
{
    return await _context.UserForms
        .Where(f => f.UserId == userId && f.FormType == formType)
        .ToListAsync(ct);
}
```

**Mutation: Create User Form**
```csharp
// src/GraphQL/Mutations.cs
[MutationType]
public class Mutation
{
    public async Task<UserForm> CreateUserFormAsync(
        int userId,
        string formType,
        string formDataJson,  // JSON as string
        FormResolver resolver,
        CancellationToken ct)
    {
        var form = new UserForm
        {
            UserId = userId,
            FormType = formType,
            FormData = formDataJson,  // Store JSON as-is
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };

        resolver.CreateUserForm(form);
        return form;
    }
}
```

**Mutation call from frontend:**
```graphql
mutation CreateUserForm($userId: Int!, $formData: FormData!) {
  createUserForm(userId: $userId, formType: "preferences", formDataJson: $formData) {
    formId
    userId
    formType
    formData
    createdAt
  }
}
```

### Key Conventions

1. **Resolver Organization:**
   - One resolver class per feature (User, Order, Form, etc.).
   - Resolvers handle business logic and database access.
   - Keep resolvers focused and testable.

2. **Error Handling:**
   - Use custom GraphQL error types for domain errors.
   - Example:
     ```csharp
     throw new GraphQLException(
         new Error("User not found", "USER_NOT_FOUND"),
         statusCode: 404);
     ```

3. **Authorization:**
   - Use `[Authorize]` attribute on resolvers requiring authentication.
   - Extract JWT claims in resolvers via `IHttpContextAccessor` or direct parameter injection.
   - **Example:**
     ```csharp
     [Authorize]
     [GraphQLName("me")]
     public async Task<User> GetCurrentUserAsync(
         [CurrentUser] User currentUser,  // Injected from JWT
         CancellationToken ct)
     {
         return currentUser;
     }
     ```

4. **Performance:**
   - Use DataLoader for N+1 query prevention:
     ```csharp
     public async Task<List<Order>> GetOrdersAsync(
         [Parent] User user,
         IDataLoader<int, List<Order>> ordersByUserIdLoader,
         CancellationToken ct)
     {
         return await ordersByUserIdLoader.LoadAsync(user.UserId, ct);
     }
     ```
   - Add query complexity analysis to prevent malicious queries.
   - Index frequently queried JSON paths in MSSQL.

5. **Testing GraphQL:**
   - Test resolvers as unit tests (mock DbContext).
   - Integration tests query actual GraphQL server.
   - Use GraphQL playground to explore schema and test queries manually.

### Instrumentation

- All GraphQL operations automatically traced by OpenTelemetry.
- Hot Chocolate integrates with OpenTelemetry out-of-box.
- Track resolver execution time, database queries, and errors.

### Local Development

```bash
# GraphQL playground (Banana Cake Pop) available at:
# http://localhost:5000/graphql

# Test query:
query {
  users {
    userId
    email
    fullName
  }
}
```

---

## Database Design

### Current Strategy: Local MSSQL Development

**Status:** Currently optimized for local development using MSSQL (Developer Edition or Docker).

**Future Plan:** When ready for production, migrate to **Amazon RDS for SQL Server** (no code changes—just update connection string in Secrets Manager).

---

### MSSQL Overview

- **Service:** SQL Server (MSSQL) — runs locally for development
- **Future AWS:** Amazon RDS for SQL Server (managed, scalable, backup-enabled)
- **Data Storage Strategy:**
  - **Structured Data:** Traditional relational tables (Users, Orders, Payments, etc.)
  - **Unstructured Data:** JSON columns (`NVARCHAR(MAX)`) for forms, preferences, details, selections
- **Local Development:** Quick setup with Docker or free Developer Edition
- **Production Ready:** Configuration-based switch to RDS with zero code changes

### Table Design Principles

1. **Naming Conventions:**
   - Tables: PascalCase (e.g., `Users`, `Orders`, `UserForms`, `Preferences`)
   - Columns: PascalCase (e.g., `UserId`, `FirstName`, `CreatedAt`)
   - Foreign Keys: `FK_ParentTable_ChildTable`
   - Indexes: `IX_TableName_ColumnName`

2. **Schema Structure:**
   - Use `dbo` schema for all tables.
   - Organize related tables by feature (e.g., `Users`, `UserProfiles`, `UserForms`).

3. **Primary & Foreign Keys:**
   - **Primary Key:** Use `INT IDENTITY(1,1)` for synthetic keys or `UNIQUEIDENTIFIER` for GUIDs.
   - **Foreign Keys:** Reference parent tables; cascade deletes where appropriate.
   - **Example:**
     ```sql
     CREATE TABLE Users (
         UserId INT PRIMARY KEY IDENTITY(1,1),
         Email NVARCHAR(255) UNIQUE NOT NULL,
         FullName NVARCHAR(255) NOT NULL,
         CreatedAt DATETIME2 DEFAULT GETUTCDATE()
     );

     CREATE TABLE Orders (
         OrderId INT PRIMARY KEY IDENTITY(1,1),
         UserId INT NOT NULL,
         OrderAmount DECIMAL(10,2),
         CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
         FOREIGN KEY (UserId) REFERENCES Users(UserId)
     );
     ```

4. **JSON Storage in MSSQL (Unstructured Data):**
   - Use `NVARCHAR(MAX)` columns to store JSON documents for flexible/schema-less data.
   - **Use Cases:** User form responses, preferences, selections, personal details, metadata.
   - **Benefits:** Single table for both structured metadata + flexible JSON content.
   - **Example Table:**
     ```sql
     CREATE TABLE UserForms (
         FormId INT PRIMARY KEY IDENTITY(1,1),
         UserId INT NOT NULL,
         FormType NVARCHAR(100),  -- e.g., 'preferences', 'details', 'selections'
         FormData NVARCHAR(MAX) NOT NULL,  -- JSON: {"name":"Reenu","age":30,"preferences":["python","finance"]}
         CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
         UpdatedAt DATETIME2 DEFAULT GETUTCDATE(),
         FOREIGN KEY (UserId) REFERENCES Users(UserId)
     );

     CREATE TABLE UserPreferences (
         PrefId INT PRIMARY KEY IDENTITY(1,1),
         UserId INT NOT NULL,
         PreferenceData NVARCHAR(MAX) NOT NULL,  -- JSON: {"theme":"dark","notifications":true,"language":"en"}
         LastModified DATETIME2 DEFAULT GETUTCDATE(),
         FOREIGN KEY (UserId) REFERENCES Users(UserId)
     );
     ```

5. **Querying JSON in MSSQL:**

   **JSON_VALUE** — Extract scalar values:
   ```sql
   SELECT FormId, 
          JSON_VALUE(FormData, '$.name') AS UserName,
          JSON_VALUE(FormData, '$.age') AS UserAge,
          JSON_VALUE(FormData, '$.email') AS Email
   FROM UserForms;
   ```

   **JSON_QUERY** — Extract objects/arrays:
   ```sql
   SELECT FormId, 
          JSON_QUERY(FormData, '$.address') AS AddressObj,
          JSON_QUERY(FormData, '$.preferences') AS PreferencesArray
   FROM UserForms;
   ```

   **Filter by JSON content:**
   ```sql
   SELECT FormId, JSON_VALUE(FormData, '$.name') AS Name
   FROM UserForms
   WHERE CAST(JSON_VALUE(FormData, '$.age') AS INT) > 25;
   ```

   **OPENJSON** — Shred JSON into rows/columns for complex queries:
   ```sql
   -- Extract array elements
   SELECT FormId, value AS Preference
   FROM UserForms
   CROSS APPLY OPENJSON(FormData, '$.preferences') 
   WHERE value = 'python';

   -- Convert JSON object to table rows
   SELECT 
       FormId,
       JSON_VALUE(u.value, '$.firstName') AS FirstName,
       JSON_VALUE(u.value, '$.lastName') AS LastName,
       JSON_VALUE(u.value, '$.age') AS Age
   FROM UserForms f
   CROSS APPLY OPENJSON(f.FormData, '$.users') AS u;
   ```

6. **Indexes for JSON Performance:**
   - Index relational columns (PK, FK, common filters):
     ```sql
     CREATE INDEX IX_UserForms_UserId ON UserForms(UserId);
     CREATE INDEX IX_UserForms_FormType ON UserForms(FormType);
     CREATE INDEX IX_Users_Email ON Users(Email);
     ```
   - Index JSON paths if you filter frequently:
     ```sql
     CREATE INDEX IX_UserForms_Age 
     ON UserForms (CAST(JSON_VALUE(FormData, '$.age') AS INT))
     WHERE CAST(JSON_VALUE(FormData, '$.age') AS INT) IS NOT NULL;
     ```

7. **Constraints & Validation:**
   - Use `CHECK` constraints for business logic (e.g., `CHECK (Age >= 18)`).
   - Use `UNIQUE` constraints where needed (e.g., Email).
   - Prefer database constraints over application logic.
   - Validate JSON structure in application layer before insert/update.

### Data Access from Backend (Entity Framework Core)

- Use **Entity Framework Core (EF Core)** for ORM and relational access.
- Define `DbContext` in `Infrastructure/Data/ApplicationDbContext.cs`.
- **Example:**
  ```csharp
  public class ApplicationDbContext : DbContext
  {
      public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
          : base(options) { }

      public DbSet<User> Users { get; set; }
      public DbSet<Order> Orders { get; set; }
      public DbSet<UserForm> UserForms { get; set; }
      public DbSet<UserPreference> UserPreferences { get; set; }
  }

  public class UserForm
  {
      public int FormId { get; set; }
      public int UserId { get; set; }
      public string FormType { get; set; }
      public string FormData { get; set; }  // JSON as string
      public DateTime CreatedAt { get; set; }
      public DateTime UpdatedAt { get; set; }
  }
  ```

- **Connection String (appsettings.json):**
  ```json
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=MyAppDb;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true;"
  }
  ```

- **Migrations:**
  ```bash
  # Create new migration
  dotnet ef migrations add AddUserForms

  # Apply migrations
  dotnet ef database update
  ```

- **Handling JSON in EF Core:**
  ```csharp
  // Example: Query by JSON value
  var usersOver25 = await context.UserForms
      .Where(f => EF.Functions.JsonValue(f.FormData, "$.age") == "30")
      .ToListAsync();

  // Insert with JSON
  var form = new UserForm 
  { 
      UserId = 1, 
      FormType = "preferences",
      FormData = JsonConvert.SerializeObject(new { theme = "dark", notifications = true })
  };
  context.UserForms.Add(form);
  await context.SaveChangesAsync();
  ```

- **Instrumentation:** All DB operations wrapped with OpenTelemetry tracing (automatic with EF Core diagnostics).

---

## Observability & Monitoring

### Observability Stack

| Component | Purpose | Configuration |
|-----------|---------|----------------|
| **OpenTelemetry SDK** | Traces, metrics, logs instrumentation | Auto-instrumentation + custom spans |
| **Prometheus-net** | Metrics exposition | Exposes `/metrics` endpoint |
| **Serilog** | Structured logging | Loki sink for centralized logs |
| **Grafana Cloud** | Visualization & alerting | SaaS (metrics, logs, traces) |

### Setup in ASP.NET Core

1. **OpenTelemetry Configuration (Program.cs):**
   ```csharp
   builder.Services
       .AddOpenTelemetry()
       .WithTracing(traceBuilder => traceBuilder
           .AddAspNetCoreInstrumentation()
           .AddHttpClientInstrumentation()
           .AddSqlClientInstrumentation()
           .AddOtlpExporter(options => 
           {
               options.Endpoint = new Uri(builder.Configuration["Observability:OtlpEndpoint"]);
           }))
       .WithMetrics(metricsBuilder => metricsBuilder
           .AddAspNetCoreInstrumentation()
           .AddHttpClientInstrumentation()
           .AddProcessInstrumentation()
           .AddRuntimeInstrumentation()
           .AddPrometheusExporter())
       .WithLogging(loggingBuilder => loggingBuilder
           .AddOtlpExporter());
   ```

2. **Prometheus-net Middleware:**
   ```csharp
   app.UseHttpMetrics();
   app.MapMetrics();  // Exposes GET /metrics
   ```

3. **Serilog Configuration:**
   ```csharp
   Log.Logger = new LoggerConfiguration()
       .WriteTo.LokiHttp(credentials: new NoAuthCredentials 
       { 
           Uri = "https://<your-grafana-loki-endpoint>" 
       })
       .WriteTo.Console()
       .Enrich.FromLogContext()
       .CreateLogger();
   ```

4. **Correlation IDs:**
   - Add middleware to inject a unique ID per request.
   - Pass correlation ID to all outbound calls and logs.
   - Frontend should read `X-Correlation-ID` from response headers and include in next request.

### Key Metrics to Track

- **Latency:** Histogram of request duration by endpoint.
- **Error Rate:** Counter of 4xx / 5xx responses.
- **Throughput:** Requests per second.
- **Database:** Query latency, throttle events, item count.
- **Memory & CPU:** System resource usage (auto-instrumented).

### Grafana Cloud Integration

1. **Sign up** at grafana.com/cloud.
2. **Create data source** for Prometheus (metrics), Loki (logs), and Tempo (traces).
3. **Configure OTLP exporter** to send to Grafana's OTLP endpoint.
4. **Set up dashboards** for frontend (Next.js custom metrics), backend (ASP.NET Core), and system.
5. **Create alerts** for SLO violations (e.g., error rate > 5%, latency p99 > 1s).

---

## CI/CD & Deployment

### GitHub Actions Workflow

- **Location:** `.github/workflows/`
- **Triggers:** On push to `main`, `develop`, or pull requests.
- **Stages:** Build → Test → Security Scan → Docker Build → Deploy.

### Example Workflow Structure

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install dependencies
        run: npm ci
      - name: Build frontend
        run: npm run build
      - name: Run tests
        run: npm run test
      # ... more steps for backend, Docker build, push, deploy ...
```

### Key Principles

1. **Secrets Management:**
   - Store AWS credentials, Docker registry tokens, Grafana API keys in GitHub Secrets.
   - Reference as `${{ secrets.AWS_ACCESS_KEY_ID }}` in workflows.
   - Rotate secrets regularly.

2. **Approval Gates:**
   - Require approval before deploying to production.
   - Automated deployment to dev/staging; manual to prod.

3. **Build Caching:**
   - Cache npm/NuGet packages to speed up builds.
   - Cache Docker layers for faster image builds.

4. **Artifact Management:**
   - Push Docker images to ECR (AWS Elastic Container Registry).
   - Tag images with commit SHA and version.

---

## Docker & Containerization

### Multi-Stage Dockerfile Pattern

**Frontend (Next.js):**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY package*.json ./
RUN npm ci --only=production
EXPOSE 3000
CMD ["npm", "start"]
```

**Backend (.NET Core):**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
WORKDIR /src
COPY ["src/Api.csproj", "src/"]
RUN dotnet restore "src/Api.csproj"
COPY . .
RUN dotnet build "src/Api.csproj" -c Release -o /app/build

FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=builder /app/build .
EXPOSE 5000 5001
ENV ASPNETCORE_URLS=http://+:5000
ENTRYPOINT ["dotnet", "Api.dll"]
```

### Image Naming & Registry

- **Registry:** AWS ECR (Elastic Container Registry)
- **Image Name Format:** `<aws-account>.dkr.ecr.<region>.amazonaws.com/<service>:<tag>`
- **Tags:** Use semantic versioning + commit SHA (e.g., `v1.2.3-abc1234`).

### Local Development

```bash
# Build image
docker build -t my-service:latest .

# Run container
docker run -p 3000:3000 --env-file .env.local my-service:latest

# Inspect logs
docker logs <container-id>
```

---

## AWS Infrastructure & Terraform

**Note:** This section describes infrastructure for future **production deployment**. Currently, development uses local MSSQL.

### High-Level Architecture (Production - Future)

```
┌────────────────────────────────────────────────────────────────┐
│ AWS Account                                                    │
├────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐          ┌──────────────────────────┐   │
│  │ ECS/EKS Cluster  │          │ RDS SQL Server           │   │
│  │ (Frontend + API) │◄────────►│ - Users, Orders, etc     │   │
│  └──────────────────┘          │ - Forms (JSON), Prefs    │   │
│         ▲                       └──────────────────────────┘   │
│         │                                                      │
│    ┌────▼────┐     ┌──────────────┐    ┌──────────────────┐  │
│    │ ALB     │     │ ECR (images) │    │ Secrets Manager  │  │
│    └────┬────┘     └──────────────┘    └──────────────────┘  │
│         │                                                      │
│    ┌────▼───────────────────────────────┐                    │
│    │ CloudWatch Logs + Grafana Cloud    │                    │
│    │ (Metrics, Logs, Traces)            │                    │
│    └────────────────────────────────────┘                     │
│         │                                                      │
│    ┌────▼──────────┐                                          │
│    │ SNS / SQS     │ (async messaging, optional)              │
│    └───────────────┘                                          │
└────────────────────────────────────────────────────────────────┘
```

### Terraform Directory Structure

```
terraform/
├── main.tf                         # Main configuration
├── variables.tf                    # Input variables
├── outputs.tf                      # Output values
├── terraform.tfvars                # Variable values (DO NOT commit secrets)
├── terraform.auto.tfvars           # Auto-loaded variables
├── modules/
│   ├── ecs/                        # ECS task & service definitions
│   ├── rds/                        # RDS SQL Server database
│   ├── iam/                        # IAM roles & policies
│   ├── networking/                 # VPC, subnets, security groups
│   ├── ecr/                        # Container registry
│   └── monitoring/                 # CloudWatch alarms, SNS
├── environments/
│   ├── dev/terraform.tfvars        # Dev-specific variables
│   ├── staging/terraform.tfvars    # Staging variables
│   └── prod/terraform.tfvars       # Prod variables
└── .terraformrc                    # Terraform CLI config
```

### Key Terraform Modules

1. **Networking (`modules/networking/`):**
   - VPC, public/private subnets, NAT gateways, route tables.
   - Security groups for ALB, ECS tasks, and RDS.

2. **ECS (`modules/ecs/`):**
   - Cluster definition.
   - Task definitions for frontend and backend.
   - Service definitions (desired count, load balancing, auto-scaling).

3. **RDS SQL Server (`modules/rds/`):**
   - SQL Server database instance configuration.
   - Backup and multi-AZ settings (recommended for production).
   - DB subnet group and security group for secure access.
   - **Example Terraform:**
     ```hcl
     resource "aws_db_instance" "mssql" {
       identifier             = "myapp-db"
       engine                 = "sqlserver-se"  # Standard Edition
       engine_version         = "2022.04"
       instance_class         = "db.t3.medium"
       allocated_storage      = 100
       storage_type           = "gp3"
       username               = var.db_master_username
       password               = var.db_master_password
       db_subnet_group_name   = aws_db_subnet_group.default.name
       vpc_security_group_ids = [aws_security_group.db.id]
       multi_az               = true  # High availability
       backup_retention_period = 30
       skip_final_snapshot    = false
     }
     ```

4. **IAM (`modules/iam/`):**
   - Task execution role (pull images from ECR, write logs to CloudWatch).
   - Task role (RDS database access via Secrets Manager, CloudWatch Logs, X-Ray).

5. **ECR (`modules/ecr/`):**
   - Container registry for frontend and backend images.
   - Lifecycle policies (auto-cleanup old images to save costs).

6. **Monitoring (`modules/monitoring/`):**
   - CloudWatch log groups for application and database logs.
   - Alarms for CPU, memory, error rates, DB connection pool exhaustion.
   - SNS topics for alerting teams.

### Deployment Workflow

```bash
# 1. Init (first time only)
terraform init -backend-config=environments/prod/backend.tf

# 2. Plan (preview changes)
terraform plan -var-file=environments/prod/terraform.tfvars -out=tfplan

# 3. Apply (create/update resources)
terraform apply tfplan

# 4. Destroy (remove resources)
terraform destroy -var-file=environments/prod/terraform.tfvars
```

### State Management

- **State File Location:** AWS S3 bucket with versioning + DynamoDB lock table.
- **Access Control:** Restrict S3 bucket access to CI/CD role only.
- **Sensitive Data:** Use Terraform's `sensitive = true` on variables containing secrets.

---

## Payment Processing & Ledger System

### Payment Flow Architecture

**High-Level Payment Lifecycle:**

```
Buyer                  EventHub Platform              Payment Gateway           Organizer Account
│                           │                              │                           │
├─ Add to Cart              │                              │                           │
├─ Checkout ─────────────►  │                              │                           │
│                           ├─ Create Payment Intent      │                           │
│                           ├──────────────────────────►  │                           │
│                           │  (Stripe/Razorpay)          │                           │
│                           │◄──────────────────────────  │                           │
│                           │  (Returns client secret)    │                           │
│                           │                              │                           │
├─ Process Card ───────────►│                              │                           │
│                           ├─ Stripe/Razorpay JS SDK    │                           │
│                           │  (Tokenized payment)         │                           │
│◄─ Confirmation ───────────├◄──────────────────────────┤ (webhook)                 │
│                           │                              │                           │
│                           ├─ Record Transaction         │                           │
│                           │  in Ledger (T1)             │                           │
│                           ├─ Emit Event                 │                           │
│                           │  (OrderPaid)                │                           │
│                           │                              │                           │
│                           ├─ Calculate Commission       │                           │
│                           │  (e.g., 10% platform fee)   │                           │
│                           │                              │                           │
│                           ├─ Record Net Payout (T2)     │                           │
│                           │  (to organizer wallet)      │                           │
│                           │                              │                           │
│                           ├─ Schedule Payout            │                           │
│                           │  (daily/weekly settle)       │                           │
│                           │                              │                           │
│                           ├─ Transfer to Org Account ──────────────────────────────►│
│                           │  (via Stripe/Razorpay)      │                           │
```

### Ledger Schema

**Table 1: Transactions** (immutable ledger of all money movements)
```sql
CREATE TABLE Transactions (
    TransactionId BIGINT PRIMARY KEY IDENTITY,
    OrganizationId INT NOT NULL,
    OfferingId INT NOT NULL,           -- Foreign key to Events/Products/Fundraisers table
    BuyerId INT NOT NULL,              -- Who paid
    OrderId INT,                       -- Reference to Orders table
    TransactionType NVARCHAR(50),      -- 'ticket_purchase', 'product_purchase', 'donation', 'refund'
    AmountGross DECIMAL(12,2),         -- Total paid (e.g., 100.00 USD)
    PlatformFeePercent DECIMAL(5,2),   -- Commission rate (e.g., 10.0 for 10%)
    PlatformFeeAmount DECIMAL(12,2),   -- Calculated fee (e.g., 10.00)
    OrganizerNetAmount DECIMAL(12,2),  -- Amount organizer keeps (e.g., 90.00)
    Currency NVARCHAR(3),              -- 'USD', 'INR', etc.
    PaymentMethodId NVARCHAR(255),     -- Reference to payment source (tokenized)
    PaymentGateway NVARCHAR(50),       -- 'stripe', 'razorpay'
    ExternalTransactionId NVARCHAR(255), -- Stripe/Razorpay transaction ID
    Status NVARCHAR(50),               -- 'pending', 'completed', 'failed', 'refunded'
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    CompletedAt DATETIME2,
    RefundedAt DATETIME2,
    Metadata NVARCHAR(MAX),            -- JSON: additional context (coupon applied, etc.)
    FOREIGN KEY (OrganizationId) REFERENCES Organizations(OrganizationId),
    FOREIGN KEY (BuyerId) REFERENCES Users(UserId)
);

-- Index for quick lookups
CREATE INDEX idx_transactions_org_offering ON Transactions(OrganizationId, OfferingId);
CREATE INDEX idx_transactions_status ON Transactions(Status, CreatedAt DESC);
```

**Table 2: Payouts** (scheduled and completed payouts to organizers)
```sql
CREATE TABLE Payouts (
    PayoutId BIGINT PRIMARY KEY IDENTITY,
    OrganizationId INT NOT NULL,
    PayoutStatus NVARCHAR(50),         -- 'pending', 'processing', 'completed', 'failed'
    PayoutMethod NVARCHAR(50),         -- 'bank_transfer', 'stripe_connect', 'razorpay_contact'
    PayoutExternalId NVARCHAR(255),    -- Reference from payment gateway
    SettlementPeriodStart DATETIME2,   -- e.g., 2025-01-01
    SettlementPeriodEnd DATETIME2,     -- e.g., 2025-01-07
    TotalAmount DECIMAL(12,2),         -- Total payout amount
    TransactionCount INT,              -- Number of transactions in this payout
    ScheduledAt DATETIME2,
    ProcessedAt DATETIME2,
    FailureReason NVARCHAR(MAX),
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (OrganizationId) REFERENCES Organizations(OrganizationId)
);

CREATE INDEX idx_payouts_org_status ON Payouts(OrganizationId, PayoutStatus);
```

**Table 3: TransactionLog** (audit trail for compliance)
```sql
CREATE TABLE TransactionLog (
    LogId BIGINT PRIMARY KEY IDENTITY,
    TransactionId BIGINT,
    ChangeType NVARCHAR(50),           -- 'created', 'cancelled', 'refunded', 'disputed'
    OldValue NVARCHAR(MAX),            -- Previous state (JSON)
    NewValue NVARCHAR(MAX),            -- New state (JSON)
    ChangedBy INT,                     -- UserId who triggered change (admin/system)
    ChangedAt DATETIME2 DEFAULT GETUTCDATE(),
    Reason NVARCHAR(MAX),              -- Why was it changed? (e.g., customer request)
    FOREIGN KEY (TransactionId) REFERENCES Transactions(TransactionId),
    FOREIGN KEY (ChangedBy) REFERENCES Users(UserId)
);
```

### Commission Configuration

**Table: CommissionRules** (flexible commission rates by offering type, organizer tier, or global default)
```sql
CREATE TABLE CommissionRules (
    CommissionRuleId INT PRIMARY KEY IDENTITY,
    OrganizationId INT,                -- NULL = global default rule
    OfferingType NVARCHAR(50),         -- 'event', 'product', 'fundraiser', or NULL = all types
    FeePercent DECIMAL(5,2),           -- e.g., 10.0 for 10%
    MinimumFee DECIMAL(12,2),          -- Minimum absolute fee (e.g., 0.50)
    EffectiveFrom DATETIME2,
    EffectiveUntil DATETIME2,
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (OrganizationId) REFERENCES Organizations(OrganizationId)
);

-- Example data:
-- Global default: 10% on all transaction types
-- INSERT INTO CommissionRules (OrganizationId, OfferingType, FeePercent, EffectiveFrom)
-- VALUES (NULL, NULL, 10.0, GETUTCDATE());

-- Premium tier organizer (5% fee)
-- INSERT INTO CommissionRules (OrganizationId, OfferingType, FeePercent, EffectiveFrom)
-- VALUES (123, NULL, 5.0, GETUTCDATE());
```

### Payment Processing Logic (C# Service)

```csharp
// src/Application/Services/PaymentService.cs
public interface IPaymentService
{
    Task<PaymentIntentResponse> CreatePaymentIntentAsync(CreatePaymentRequest request, CancellationToken ct);
    Task<Transaction> ConfirmPaymentAsync(string paymentIntentId, CancellationToken ct);
    Task<bool> RefundTransactionAsync(long transactionId, string reason, CancellationToken ct);
    Task<Payout> SchedulePayoutAsync(int organizationId, DateTime settlementPeriodEnd, CancellationToken ct);
}

public class PaymentService : IPaymentService
{
    private readonly ApplicationDbContext _context;
    private readonly IStripeClient _stripeClient;  // Stripe or Razorpay adapter
    private readonly ILogger<PaymentService> _logger;

    public async Task<PaymentIntentResponse> CreatePaymentIntentAsync(
        CreatePaymentRequest request, 
        CancellationToken ct)
    {
        // Fetch offering details and calculate fees
        var offering = await _context.Offerings
            .FirstOrDefaultAsync(o => o.OfferingId == request.OfferingId, ct);

        var organization = await _context.Organizations
            .FirstOrDefaultAsync(o => o.OrganizationId == offering.OrganizationId, ct);

        var commissionRule = await GetApplicableCommissionRuleAsync(
            organization.OrganizationId, 
            offering.OfferingType, 
            ct);

        var platformFee = request.Amount * (commissionRule.FeePercent / 100m);
        var organizerNetAmount = request.Amount - platformFee;

        // Create payment intent with Stripe/Razorpay
        var paymentIntent = await _stripeClient.CreatePaymentIntentAsync(
            new PaymentIntentRequest
            {
                Amount = (long)(request.Amount * 100),  // Convert to cents
                Currency = request.Currency,
                Metadata = new Dictionary<string, string>
                {
                    { "organizationId", organization.OrganizationId.ToString() },
                    { "offeringId", offering.OfferingId.ToString() },
                    { "buyerId", request.BuyerId.ToString() }
                }
            },
            ct);

        _logger.LogInformation(
            "Created payment intent {PaymentIntentId} for offering {OfferingId}",
            paymentIntent.ClientSecret,
            offering.OfferingId);

        return new PaymentIntentResponse
        {
            ClientSecret = paymentIntent.ClientSecret,
            Amount = request.Amount,
            Currency = request.Currency
        };
    }

    public async Task<Transaction> ConfirmPaymentAsync(string paymentIntentId, CancellationToken ct)
    {
        // Retrieve payment result from Stripe/Razorpay
        var paymentResult = await _stripeClient.RetrievePaymentIntentAsync(paymentIntentId, ct);

        if (paymentResult.Status != "succeeded")
        {
            _logger.LogWarning("Payment intent {PaymentIntentId} failed with status {Status}",
                paymentIntentId, paymentResult.Status);
            return null;
        }

        // Extract metadata
        var organizationId = int.Parse(paymentResult.Metadata["organizationId"]);
        var offeringId = int.Parse(paymentResult.Metadata["offeringId"]);
        var buyerId = int.Parse(paymentResult.Metadata["buyerId"]);

        // Calculate fees
        var amountGross = paymentResult.Amount / 100m;  // Convert from cents
        var commissionRule = await GetApplicableCommissionRuleAsync(organizationId, null, ct);
        var platformFeeAmount = amountGross * (commissionRule.FeePercent / 100m);
        var organizerNetAmount = amountGross - platformFeeAmount;

        // Create transaction record (immutable ledger entry)
        var transaction = new Transaction
        {
            OrganizationId = organizationId,
            OfferingId = offeringId,
            BuyerId = buyerId,
            TransactionType = "ticket_purchase",
            AmountGross = amountGross,
            PlatformFeePercent = commissionRule.FeePercent,
            PlatformFeeAmount = platformFeeAmount,
            OrganizerNetAmount = organizerNetAmount,
            Currency = paymentResult.Currency,
            PaymentGateway = "stripe",  // or 'razorpay'
            ExternalTransactionId = paymentIntentId,
            Status = "completed",
            CompletedAt = DateTime.UtcNow
        };

        _context.Transactions.Add(transaction);

        // Create payout record (organizer's pending balance)
        var payout = await _context.Payouts
            .FirstOrDefaultAsync(p => 
                p.OrganizationId == organizationId &&
                p.PayoutStatus == "pending", 
            ct);

        if (payout == null)
        {
            payout = new Payout
            {
                OrganizationId = organizationId,
                PayoutStatus = "pending",
                SettlementPeriodStart = DateTime.UtcNow.StartOfDay(),
                SettlementPeriodEnd = DateTime.UtcNow.AddDays(7).EndOfDay(),
                TotalAmount = organizerNetAmount,
                TransactionCount = 1
            };
            _context.Payouts.Add(payout);
        }
        else
        {
            payout.TotalAmount += organizerNetAmount;
            payout.TransactionCount++;
        }

        await _context.SaveChangesAsync(ct);

        _logger.LogInformation(
            "Confirmed transaction {TransactionId} for organizer {OrgId}, net amount {NetAmount}",
            transaction.TransactionId, organizationId, organizerNetAmount);

        return transaction;
    }

    public async Task<bool> RefundTransactionAsync(long transactionId, string reason, CancellationToken ct)
    {
        var transaction = await _context.Transactions
            .FirstOrDefaultAsync(t => t.TransactionId == transactionId, ct);

        if (transaction == null)
            return false;

        // Refund via payment gateway
        var refundResult = await _stripeClient.RefundPaymentAsync(
            transaction.ExternalTransactionId, ct);

        if (!refundResult.Success)
            return false;

        // Update transaction status
        transaction.Status = "refunded";
        transaction.RefundedAt = DateTime.UtcNow;

        // Create audit log entry
        var logEntry = new TransactionLog
        {
            TransactionId = transactionId,
            ChangeType = "refunded",
            OldValue = JsonSerializer.Serialize(new { status = "completed" }),
            NewValue = JsonSerializer.Serialize(new { status = "refunded" }),
            ChangedBy = 0,  // System user
            Reason = reason
        };
        _context.TransactionLogs.Add(logEntry);

        await _context.SaveChangesAsync(ct);

        _logger.LogInformation("Refunded transaction {TransactionId}, reason: {Reason}",
            transactionId, reason);

        return true;
    }

    public async Task<Payout> SchedulePayoutAsync(int organizationId, DateTime settlementPeriodEnd, CancellationToken ct)
    {
        // Fetch pending payout
        var payout = await _context.Payouts
            .FirstOrDefaultAsync(p =>
                p.OrganizationId == organizationId &&
                p.PayoutStatus == "pending",
            ct);

        if (payout == null)
            return null;

        // Mark as processing
        payout.PayoutStatus = "processing";
        payout.ScheduledAt = DateTime.UtcNow;

        await _context.SaveChangesAsync(ct);

        // Async task: Transfer funds to organizer account (via payment gateway)
        _ = ProcessPayoutAsync(payout, ct);  // Fire and forget with proper logging

        return payout;
    }

    private async Task ProcessPayoutAsync(Payout payout, CancellationToken ct)
    {
        try
        {
            // Fetch organizer's payment account (bank details, Stripe Connect, etc.)
            var organization = await _context.Organizations
                .FirstOrDefaultAsync(o => o.OrganizationId == payout.OrganizationId, ct);

            // Call payment gateway to transfer
            var payoutResult = await _stripeClient.CreatePayoutAsync(
                new PayoutRequest
                {
                    Amount = (long)(payout.TotalAmount * 100),
                    Currency = "usd",
                    Destination = organization.PaymentAccountId  // Stripe Connect or bank account
                },
                ct);

            payout.PayoutExternalId = payoutResult.Id;
            payout.PayoutStatus = "completed";
            payout.ProcessedAt = DateTime.UtcNow;

            _logger.LogInformation(
                "Processed payout {PayoutId} to organization {OrgId}, amount {Amount}",
                payout.PayoutId, payout.OrganizationId, payout.TotalAmount);
        }
        catch (Exception ex)
        {
            payout.PayoutStatus = "failed";
            payout.FailureReason = ex.Message;

            _logger.LogError(ex, "Failed to process payout {PayoutId}", payout.PayoutId);
        }

        await _context.SaveChangesAsync(ct);
    }

    private async Task<CommissionRule> GetApplicableCommissionRuleAsync(
        int organizationId, 
        string offeringType, 
        CancellationToken ct)
    {
        // Try organization-specific rule first
        var rule = await _context.CommissionRules
            .Where(r =>
                r.OrganizationId == organizationId &&
                (r.OfferingType == null || r.OfferingType == offeringType) &&
                r.EffectiveFrom <= DateTime.UtcNow &&
                (r.EffectiveUntil == null || r.EffectiveUntil >= DateTime.UtcNow))
            .OrderByDescending(r => r.EffectiveFrom)
            .FirstOrDefaultAsync(ct);

        // Fall back to global rule
        if (rule == null)
        {
            rule = await _context.CommissionRules
                .Where(r =>
                    r.OrganizationId == null &&
                    (r.OfferingType == null || r.OfferingType == offeringType) &&
                    r.EffectiveFrom <= DateTime.UtcNow &&
                    (r.EffectiveUntil == null || r.EffectiveUntil >= DateTime.UtcNow))
                .OrderByDescending(r => r.EffectiveFrom)
                .FirstOrDefaultAsync(ct);
        }

        return rule ?? new CommissionRule { FeePercent = 10.0m };  // Default 10%
    }
}
```

### GraphQL Mutations for Payment & Payout

```graphql
# Payment-related mutations
mutation CreatePaymentIntent(
  $offeringId: Int!
  $amount: Float!
  $currency: String!
) {
  createPaymentIntent(
    input: {
      offeringId: $offeringId
      amount: $amount
      currency: $currency
    }
  ) {
    clientSecret
    amount
    currency
  }
}

mutation ConfirmPayment($paymentIntentId: String!) {
  confirmPayment(paymentIntentId: $paymentIntentId) {
    transactionId
    status
    amountGross
    organizerNetAmount
    completedAt
  }
}

mutation RefundTransaction($transactionId: Long!, $reason: String!) {
  refundTransaction(transactionId: $transactionId, reason: $reason) {
    success
    reason
  }
}

# Payout-related queries and mutations
query GetOrganizerBalance($organizationId: Int!) {
  organizerBalance(organizationId: $organizationId) {
    pendingAmount
    payoutSchedule {
      payoutId
      amount
      status
      scheduledDate
    }
  }
}

mutation RequestPayout($organizationId: Int!) {
  requestPayout(organizationId: $organizationId) {
    payoutId
    status
    amount
    scheduledAt
  }
}
```

---

## Multi-Offering Strategy (Events, Products, Fundraising)

### Overview

EventHub supports three primary offering types, with a flexible polymorphic design to accommodate future additions:

1. **Events** — Time-based, ticketed offerings (concerts, conferences, workshops)
2. **Products** — Inventory-based, non-time-bound offerings (merchandise, digital goods)
3. **Fundraisers** — Goal-based offerings with milestone unlocking (charity drives, campaigns)

All three share common attributes (organizer, revenue tracking, buyer visibility) but have unique properties (event dates, inventory levels, fundraising goals).

### Polymorphic Schema Design

**Base Offering Table:**
```sql
CREATE TABLE Offerings (
    OfferingId INT PRIMARY KEY IDENTITY,
    OrganizationId INT NOT NULL,
    OfferingType NVARCHAR(50) NOT NULL,  -- 'event', 'product', 'fundraiser'
    Title NVARCHAR(255) NOT NULL,
    Description NVARCHAR(MAX),
    ImageUrl NVARCHAR(MAX),
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    PublishedAt DATETIME2,
    Status NVARCHAR(50),                 -- 'draft', 'published', 'archived', 'cancelled'
    Metadata NVARCHAR(MAX),              -- JSON: common attributes (tags, categories, etc.)
    FOREIGN KEY (OrganizationId) REFERENCES Organizations(OrganizationId)
);

-- Type-specific tables with foreign key to Offerings
CREATE TABLE Events (
    EventId INT PRIMARY KEY IDENTITY,
    OfferingId INT UNIQUE NOT NULL,
    EventDate DATETIME2 NOT NULL,
    EndDate DATETIME2,
    Venue NVARCHAR(MAX),
    Capacity INT,
    AttendeesCount INT DEFAULT 0,
    IsRecurring BIT DEFAULT 0,
    RecurrencePattern NVARCHAR(MAX),    -- JSON: 'daily', 'weekly', 'monthly', etc.
    FOREIGN KEY (OfferingId) REFERENCES Offerings(OfferingId)
);

CREATE TABLE Products (
    ProductId INT PRIMARY KEY IDENTITY,
    OfferingId INT UNIQUE NOT NULL,
    SKU NVARCHAR(100) UNIQUE,
    Quantity INT NOT NULL,              -- Available inventory
    QuantitySold INT DEFAULT 0,
    IsPhysical BIT,                     -- Physical goods require shipping; digital do not
    ShippingWeight DECIMAL(8,2),        -- For physical products
    DigitalDownloadUrl NVARCHAR(MAX),   -- For digital products
    FOREIGN KEY (OfferingId) REFERENCES Offerings(OfferingId)
);

CREATE TABLE Fundraisers (
    FundraiserId INT PRIMARY KEY IDENTITY,
    OfferingId INT UNIQUE NOT NULL,
    GoalAmount DECIMAL(12,2) NOT NULL,
    CurrentAmount DECIMAL(12,2) DEFAULT 0,
    DeadlineDate DATETIME2,
    MinimumContribution DECIMAL(12,2),  -- Minimum donation per transaction
    Milestones NVARCHAR(MAX),          -- JSON array: [{amount: 1000, unlocks: "free_tshirt"}, ...]
    FOREIGN KEY (OfferingId) REFERENCES Offerings(OfferingId)
);

-- Tier/Pricing for products and fundraisers (optional, for flexibility)
CREATE TABLE OfferingTiers (
    TierId INT PRIMARY KEY IDENTITY,
    OfferingId INT NOT NULL,
    TierName NVARCHAR(100),
    Price DECIMAL(12,2),
    Description NVARCHAR(MAX),
    Quantity INT,                       -- For limited tiers
    Benefits NVARCHAR(MAX),             -- JSON: list of benefits/perks
    FOREIGN KEY (OfferingId) REFERENCES Offerings(OfferingId)
);
```

### Querying Mixed Offering Types

**GraphQL Schema:**
```graphql
type Offering {
  offeringId: Int!
  organizationId: Int!
  offeringType: OfferingType!  # ENUM: EVENT, PRODUCT, FUNDRAISER
  title: String!
  description: String
  imageUrl: String
  status: OfferingStatus!
  createdAt: DateTime!
  publishedAt: DateTime
  
  # Type-specific fields (union types)
  asEvent: Event
  asProduct: Product
  asFundraiser: Fundraiser
}

type Event {
  eventId: Int!
  offering: Offering!
  eventDate: DateTime!
  endDate: DateTime
  venue: String
  capacity: Int
  attendeesCount: Int!
  isRecurring: Boolean
}

type Product {
  productId: Int!
  offering: Offering!
  sku: String
  quantity: Int!
  quantitySold: Int!
  isPhysical: Boolean
  digitalDownloadUrl: String
}

type Fundraiser {
  fundraiserId: Int!
  offering: Offering!
  goalAmount: Float!
  currentAmount: Float!
  deadlineDate: DateTime
  minimumContribution: Float
  milestones: [Milestone!]!
  progressPercent: Float!
}

enum OfferingType {
  EVENT
  PRODUCT
  FUNDRAISER
}

# Query examples
query GetOrganizationOfferings($organizationId: Int!, $type: OfferingType) {
  offerings(organizationId: $organizationId, type: $type) {
    offeringId
    title
    status
    
    # Conditional fields based on type
    asEvent {
      eventDate
      venue
    }
    asProduct {
      quantity
      quantitySold
    }
    asFundraiser {
      goalAmount
      currentAmount
      progressPercent
    }
  }
}

query GetOfferingDetails($offeringId: Int!) {
  offering(offeringId: $offeringId) {
    offeringId
    title
    description
    offeringType
    
    # Load specific type details
    asEvent {
      eventDate
      endDate
      venue
      capacity
      attendeesCount
    }
  }
}
```

### C# Resolver for Polymorphic Offerings

```csharp
// src/GraphQL/Resolvers/OfferingResolver.cs
public class OfferingResolver
{
    private readonly ApplicationDbContext _context;

    public async Task<List<Offering>> GetOrganizationOfferingsAsync(
        int organizationId,
        string? offeringType,
        CancellationToken ct)
    {
        var query = _context.Offerings
            .Where(o => o.OrganizationId == organizationId);

        if (!string.IsNullOrEmpty(offeringType))
            query = query.Where(o => o.OfferingType == offeringType);

        return await query
            .OrderByDescending(o => o.PublishedAt ?? o.CreatedAt)
            .ToListAsync(ct);
    }

    public async Task<Offering?> GetOfferingWithDetailsAsync(int offeringId, CancellationToken ct)
    {
        return await _context.Offerings
            .Include(o => o.Events)  // Eager load if it's an event
            .Include(o => o.Products)  // Eager load if it's a product
            .Include(o => o.Fundraisers)  // Eager load if it's a fundraiser
            .FirstOrDefaultAsync(o => o.OfferingId == offeringId, ct);
    }

    public async Task<Event?> GetEventAsync([Parent] Offering offering, CancellationToken ct)
    {
        if (offering.OfferingType != "event")
            return null;

        return await _context.Events
            .FirstOrDefaultAsync(e => e.OfferingId == offering.OfferingId, ct);
    }

    public async Task<Product?> GetProductAsync([Parent] Offering offering, CancellationToken ct)
    {
        if (offering.OfferingType != "product")
            return null;

        return await _context.Products
            .FirstOrDefaultAsync(p => p.OfferingId == offering.OfferingId, ct);
    }

    public async Task<Fundraiser?> GetFundraiserAsync([Parent] Offering offering, CancellationToken ct)
    {
        if (offering.OfferingType != "fundraiser")
            return null;

        return await _context.Fundraisers
            .FirstOrDefaultAsync(f => f.OfferingId == offering.OfferingId, ct);
    }
}

// Resolver type for strong typing in GraphQL
[ObjectType("Event")]
public class EventType
{
    public int EventId { get; set; }
    public DateTime EventDate { get; set; }
    public DateTime? EndDate { get; set; }
    public string? Venue { get; set; }
    public int? Capacity { get; set; }
    public int AttendeesCount { get; set; }

    [GraphQLType(typeof(StringType))]
    public string? RecurrencePattern { get; set; }
}

[ObjectType("Product")]
public class ProductType
{
    public int ProductId { get; set; }
    public string? SKU { get; set; }
    public int Quantity { get; set; }
    public int QuantitySold { get; set; }
    public bool IsPhysical { get; set; }
    public string? DigitalDownloadUrl { get; set; }
}

[ObjectType("Fundraiser")]
public class FundraiserType
{
    public int FundraiserId { get; set; }
    public decimal GoalAmount { get; set; }
    public decimal CurrentAmount { get; set; }
    public DateTime? DeadlineDate { get; set; }
    public decimal? MinimumContribution { get; set; }

    [GraphQLName("progressPercent")]
    public double GetProgressPercent() => 
        GoalAmount > 0 ? (double)(CurrentAmount / GoalAmount * 100) : 0;
}
```

### Dashboard Views by Offering Type

**Organizer Dashboard (All Offerings):**
```sql
-- View: Summary of all offerings for an organizer
CREATE VIEW OrganizationOfferingsSummary AS
SELECT
    o.OfferingId,
    o.OrganizationId,
    o.OfferingType,
    o.Title,
    o.Status,
    ISNULL(e.EventDate, NULL) AS EventDate,
    ISNULL(p.Quantity - p.QuantitySold, NULL) AS InventoryRemaining,
    ISNULL(f.GoalAmount, 0) AS FundraiserGoal,
    ISNULL(f.CurrentAmount, 0) AS FundraiserCurrent,
    (SELECT ISNULL(SUM(AmountGross), 0)
     FROM Transactions
     WHERE OfferingId = o.OfferingId AND Status = 'completed') AS TotalRevenue,
    (SELECT COUNT(DISTINCT BuyerId)
     FROM Transactions
     WHERE OfferingId = o.OfferingId AND Status = 'completed') AS BuyerCount,
    o.CreatedAt,
    o.PublishedAt
FROM Offerings o
LEFT JOIN Events e ON o.OfferingId = e.OfferingId
LEFT JOIN Products p ON o.OfferingId = p.OfferingId
LEFT JOIN Fundraisers f ON o.OfferingId = f.OfferingId;
```

---

## Security & Compliance

### Data Isolation & Tenant Enforcement

1. **Application Layer (GraphQL Resolvers):**
   ```csharp
   // src/GraphQL/Resolvers/OfferingResolver.cs
   [Authorize]
   public async Task<Offering?> GetOfferingAsync(
       int offeringId,
       [Service] ApplicationDbContext context,
       ClaimsPrincipal user,  // Injected from JWT claims
       CancellationToken ct)
   {
       var userOrgId = int.Parse(user.FindFirst("organizationId")?.Value ?? "0");

       var offering = await context.Offerings
           .FirstOrDefaultAsync(o => o.OfferingId == offeringId, ct);

       // Enforce tenant isolation: only allow access if organizer owns the offering
       if (offering?.OrganizationId != userOrgId && !user.IsInRole("admin"))
           throw new UnauthorizedAccessException("Not authorized to access this offering.");

       return offering;
   }
   ```

2. **Database Layer (SQL Server Row-Level Security — Optional but Recommended):**
   ```sql
   -- Create security policy to enforce tenant isolation at DB level
   CREATE SCHEMA Security;

   CREATE FUNCTION Security.TenantAccessPredicate(@OrganizationId INT)
   RETURNS TABLE
   WITH SCHEMABINDING
   AS
   RETURN SELECT 1 AS AccessResult
   WHERE @OrganizationId = CAST(SESSION_CONTEXT(N'OrganizationId') AS INT)
       OR CAST(SESSION_CONTEXT(N'IsAdmin') AS BIT) = 1;

   -- Apply policy to Offerings table
   CREATE SECURITY POLICY OfferingsTenantFilter
   ADD FILTER PREDICATE Security.TenantAccessPredicate(OrganizationId)
   ON dbo.Offerings
   WITH (STATE = ON);

   -- On connection, set the tenant context
   -- EXECUTE sp_set_session_context @key=N'OrganizationId', @value=<user_org_id>;
   ```

### PCI Compliance for Payments

- **No raw credit card storage** — All card data tokenized via Stripe/Razorpay.
- **Webhook validation** — Verify webhook signatures from payment gateway before processing.
- **Encrypted storage** — Sensitive data (payment method IDs, refund reasons) encrypted at rest.
- **Audit logging** — Every transaction change logged with timestamp, actor, and reason.

### GDPR Compliance

1. **Data Export (Right to Data Portability):**
   ```graphql
   mutation RequestDataExport($userId: Int!) {
     requestDataExport(userId: $userId) {
       exportId
       status  # 'pending', 'processing', 'completed'
       expiresAt
       downloadUrl
     }
   }
   ```

2. **Account Deletion (Right to be Forgotten):**
   ```graphql
   mutation DeleteAccount($userId: Int!, $reason: String!) {
     deleteAccount(userId: $userId, reason: $reason) {
       success
       anonymizedAt
     }
   }
   ```
   - Buyer data: Anonymized (names, emails, addresses replaced with hashes).
   - Transaction ledger: Retained for audit trail but buyer identity removed.

3. **Consent Tracking:**
   ```sql
   CREATE TABLE UserConsents (
       ConsentId INT PRIMARY KEY IDENTITY,
       UserId INT NOT NULL,
       ConsentType NVARCHAR(100),  -- 'marketing', 'analytics', 'terms'
       IsGranted BIT,
       GrantedAt DATETIME2,
       RevokedAt DATETIME2,
       IpAddress NVARCHAR(50),
       FOREIGN KEY (UserId) REFERENCES Users(UserId)
   );
   ```

### Rate Limiting & DDoS Protection

```csharp
// Program.cs - Add rate limiting middleware
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter(
        policyName: "fixed",
        options => 
        {
            options.Window = TimeSpan.FromSeconds(60);
            options.PermitLimit = 100;  // 100 requests per minute
        });
});

app.UseRateLimiter();

// GraphQL endpoint with rate limiting
app.MapGraphQL("/graphql")
    .RequireRateLimiting("fixed");
```

### CSRF & CORS Configuration

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("GraphQL", policy =>
    {
        policy.WithOrigins("https://app.eventhub.com", "https://localhost:3000")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials()
              .WithExposedHeaders("X-Total-Count");  // For pagination
    });
});

app.UseCors("GraphQL");

// CSRF token handling (if using traditional forms, else not needed for SPA + JWT)
// For SPA with JWT: CSRF protection via Same-Site cookies
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    await next();
});
```

### Secrets Management

**AWS Secrets Manager Integration (Production):**
```csharp
// Program.cs
var builder = WebApplicationBuilder.CreateBuilder(args);

if (builder.Environment.IsProduction())
{
    var secretsManagerClient = new AmazonSecretsManagerClient(RegionEndpoint.USEast1);
    
    builder.Configuration.AddSecretsManager(
        client: secretsManagerClient,
        secretFilter: secret => secret.Name.StartsWith("prod/")
    );
}

// Access secrets
var stripeKey = builder.Configuration["prod/stripe/secret-key"];
var dbPassword = builder.Configuration["prod/database/password"];
```

**Environment Variables (Local Development — Never commit!):**
```bash
# .env.local (DO NOT commit)
STRIPE_SECRET_KEY=sk_test_xxx
RAZORPAY_KEY_SECRET=key_secret_xxx
DB_PASSWORD=YourPassword123!
JWT_SECRET=your-256-bit-secret-key
```

---

## Cross-Service Communication

### Frontend ↔ Backend (GraphQL)

1. **GraphQL Queries & Mutations:**
   - Frontend uses **Apollo Client** to send GraphQL queries/mutations.
   - All requests go to single endpoint: `POST /graphql`
   - Request format:
     ```json
     {
       "query": "query GetUser($id: Int!) { user(userId: $id) { userId email fullName } }",
       "variables": { "id": 1 }
     }
     ```
   - Response includes data and errors:
     ```json
     {
       "data": { "user": { "userId": 1, "email": "user@example.com", "fullName": "John Doe" } },
       "errors": null
     }
     ```

2. **Headers:**
   - All requests include:
     - `Content-Type: application/json`
     - `Authorization: Bearer <jwt-token>` (if protected)
     - `X-Correlation-ID: <uuid>` (for tracing)

3. **Error Handling:**
   - GraphQL errors returned in `errors` field:
     ```json
     {
       "errors": [
         {
           "message": "User not found",
           "extensions": {
             "code": "USER_NOT_FOUND",
             "statusCode": 404
           }
         }
       ]
     }
     ```
   - Frontend maps GraphQL error codes to user-friendly messages.

4. **Request/Response Tracing:**
   - Include correlation ID in every log entry and span.
   - Use W3C Trace Context headers (`traceparent`, `tracestate`).
   - GraphQL playground (`/graphql`) shows schema and allows query exploration.

### Backend ↔ Database

- **MSSQL Connection:** Entity Framework Core; credentials from appsettings or Secrets Manager.
- **Retry Strategy:** Exponential backoff for transient failures.
- **Connection Pooling:** EF Core manages connection pool automatically.
- **Circuit Breaker:** Implement health checks; fail fast if database unavailable.
- **JSON Queries:** Use EF Core's `EF.Functions.JsonValue()` to filter by JSON fields in resolvers.

### Backend ↔ External Services

- **Configuration:** Endpoint URLs and credentials in environment variables.
- **Timeouts:** Set reasonable timeouts (e.g., 30s for HTTP calls).
- **Logging:** Log request/response for debugging; mask sensitive data.

---

## Development Workflows

### Local Setup

1. **Prerequisites:**
   - Node.js 18+ (frontend)
   - .NET SDK 7+ (backend)
   - Docker Desktop
   - AWS CLI configured with dev credentials
   - Git

2. **Frontend Setup:**
   ```bash
   cd frontend
   npm install
   cp .env.example .env.local
   
   # Configure GraphQL endpoint
   # .env.local should include:
   # NEXT_PUBLIC_GRAPHQL_ENDPOINT=http://localhost:5000/graphql
   
   npm run dev
   # Visit http://localhost:3000
   ```

3. **Backend Setup:**
   ```bash
   cd backend
   dotnet restore
   cp appsettings.Development.example.json appsettings.Development.json
   dotnet watch run
   # GraphQL playground: http://localhost:5000/graphql
   # API GraphQL endpoint: http://localhost:5000/graphql (POST)
   ```

4. **MSSQL Setup (Local Development):**
   ```bash
   # Option 1: Docker (Recommended for quick start)
   docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=YourPassword123!" \
     -p 1433:1433 --name mssql-local \
     mcr.microsoft.com/mssql/server:2022-latest

   # Option 2: SQL Server Developer Edition (Free from Microsoft)
   # Download: https://www.microsoft.com/en-us/sql-server/sql-server-downloads
   # Install locally; runs on port 1433 by default

   # Option 3: Use Azure Data Studio or SQL Server Management Studio (SSMS) for GUI
   # Download: https://azure.microsoft.com/en-us/products/data-studio/
   ```

5. **Configure Connection String (Local):**
   ```json
   // appsettings.Development.json
   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=localhost,1433;Database=MyAppDb;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true;"
     }
   }
   ```

6. **Initialize Database:**
   ```bash
   # Apply EF Core migrations to create tables
   dotnet ef database update

   # Or manually run SQL scripts if needed
   sqlcmd -S localhost,1433 -U sa -P "YourPassword123!" -i schema.sql
   ```

7. **Verify Connections:**
   ```bash
   # Test MSSQL connection
   sqlcmd -S localhost,1433 -U sa -P "YourPassword123!" -Q "SELECT @@VERSION;"
   
   # Test GraphQL endpoint
   curl -X POST http://localhost:5000/graphql \
     -H "Content-Type: application/json" \
     -d '{"query":"{ users { userId email fullName } }"}'
   ```

8. **GraphQL Exploration:**
   - Open GraphQL playground: http://localhost:5000/graphql
   - Use the built-in IDE to explore schema, run queries, and test mutations
   - Write and test GraphQL queries before building frontend components

### Configuration for Local vs Cloud

Use **environment-based configuration** to switch between local and cloud databases:

**appsettings.json (Base settings - committed)**
```json
{
  "Database": {
    "Provider": "MSSQL",
    "ConnectionString": ""  // Override in environment-specific files
  }
}
```

**appsettings.Development.json (Local dev - DO NOT commit sensitive data)**
```json
{
  "Database": {
    "Provider": "MSSQL",
    "ConnectionString": "Server=localhost,1433;Database=MyAppDb;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true;"
  }
}
```

**appsettings.Production.json (Cloud RDS - use Secrets Manager in CI/CD)**
```json
{
  "Database": {
    "Provider": "MSSQL",
    "ConnectionString": "{{RESOLVED_FROM_SECRETS_MANAGER}}"
  }
}
```

**C# Configuration in Program.cs:**
```csharp
var builder = WebApplicationBuilder.CreateBuilder(args);

// Read environment-specific settings
var dbConnectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var dbProvider = builder.Configuration["Database:Provider"];

// Register DbContext with the connection string from appsettings
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    if (dbProvider == "MSSQL")
    {
        options.UseSqlServer(dbConnectionString);
    }
});

// In production, override with Secrets Manager
if (builder.Environment.IsProduction())
{
    var secretsManager = new AmazonSecretsManagerClient(RegionEndpoint.USEast1);
    var secret = await secretsManager.GetSecretValueAsync(new GetSecretValueRequest 
    { 
        SecretId = "prod/db/connection-string" 
    });
    var connectionString = secret.SecretString;
    // Rebuild DbContext with production connection string
}
```

### Migration Path: Local → Cloud (RDS)

When ready to move to production:

1. **Set up RDS instance** (via Terraform in `modules/rds/`).
2. **Store connection string** in AWS Secrets Manager.
3. **Update `appsettings.Production.json`** to reference Secrets Manager.
4. **Run migrations** against RDS: `dotnet ef database update`
5. **Deploy** backend to ECS with `ASPNETCORE_ENVIRONMENT=Production`.

**No code changes required**—just configuration!

### Branching Strategy

- **main:** Production-ready code.
- **develop:** Integration branch for features.
- **feature/xyz:** Individual feature branches off `develop`.
- **hotfix/xyz:** Critical fixes off `main`.

### Commit Conventions

```
<type>(<scope>): <subject>

<body>

<footer>
```

- **Types:** feat, fix, docs, style, refactor, test, chore.
- **Example:** `feat(auth): add JWT token refresh endpoint`.

### Testing

- **Frontend:** Jest + React Testing Library; run `npm run test`.
- **Backend:** xUnit; run `dotnet test`.
- **Coverage:** Aim for >80% unit test coverage.
- **CI:** Tests run automatically on every PR.

---

## Quick Reference Checklist for AI Agents

When working on this project, always:

- [ ] Check `.github/copilot-instructions.md` for general repo guidance.
- [ ] Reference this file for architecture and tech stack details.
- [ ] **Use local MSSQL** for all development (run Docker container or Developer Edition).
- [ ] Keep `appsettings.Development.json` with local connection strings (never commit credentials).
- [ ] **Use GraphQL** for all frontend-backend communication (endpoint: `/graphql`).
- [ ] Test GraphQL queries in the playground before building frontend components.
- [ ] Use Apollo Client on frontend to query/mutate GraphQL data.
- [ ] Organize resolvers by feature (UserResolver, OrderResolver, etc.).
- [ ] Add `[Authorize]` attribute to protected GraphQL resolvers.
- [ ] Use correlation IDs for any request tracing.
- [ ] Write unit tests for resolvers and mutations.
- [ ] Test all database queries locally before committing (especially JSON queries).
- [ ] Run `npm run lint` / `dotnet build` before committing.
- [ ] Include observability (logs, metrics) in new GraphQL resolvers via OpenTelemetry.
- [ ] Document new GraphQL queries/mutations in schema.
- [ ] For production migration: Store connection strings in AWS Secrets Manager (no code changes needed).
- [ ] Use Entity Framework Core for all MSSQL operations.

---

**Questions or Clarifications?**

If unclear on any section, refer to the team's documentation or ask in the project slack channel.
