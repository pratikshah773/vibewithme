# EventHub Backend

This is the ASP.NET Core backend for the EventHub SaaS platform.

## Tech Stack
- ASP.NET Core (.NET 7+)
- Hot Chocolate (GraphQL)
- Entity Framework Core (MSSQL)
- JWT Auth (placeholder)
- OpenTelemetry, Serilog

## Getting Started

1. Install dependencies:
   ```bash
   dotnet restore
   ```
2. Copy `appsettings.Development.example.json` to `appsettings.Development.json` and update values.
3. Run the development server:
   ```bash
   dotnet watch run
   ```

## Environment Variables
See `appsettings.Development.example.json` for required variables.
