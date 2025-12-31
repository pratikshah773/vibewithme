# Prerequisites for EventHub Development


## 1. Node.js (v18.19.0 LTS)
- Required for Next.js frontend and npx
- Download: https://nodejs.org/dist/v18.19.0/

## 2. npm (v10.2.4, comes with Node.js 18.19.0)
- Required for installing frontend dependencies

## 3. .NET SDK 7.0.404
- Required for ASP.NET Core backend
- Download: https://dotnet.microsoft.com/en-us/download/dotnet/7.0

## 4. Docker Desktop 4.26.1
- For running local MSSQL database
- Download: https://desktop.docker.com/win/main/amd64/104420/docker-desktop-4.26.1-amd64.exe

## 5. Git 2.43.0
- For version control
- Download: https://github.com/git-for-windows/git/releases/tag/v2.43.0.windows.1

## 6. MSSQL Client (optional)
- Azure Data Studio: https://azure.microsoft.com/en-us/products/data-studio/
- SQL Server Management Studio: https://aka.ms/ssms

## 7. Windows Package Manager (winget)
- For automated installs (comes with Windows 11/10)

---

# Installation Steps (Recommended)

1. **Install Node.js (includes npm)**
2. **Install .NET SDK 7+**
3. **Install Docker Desktop**
4. **Install Git**
5. (Optional) Install Azure Data Studio or SSMS

---


# Automated Install Commands (PowerShell)

```
# Node.js 18.19.0 LTS
winget install OpenJS.NodeJS.LTS --version 18.19.0.0 -h --accept-package-agreements --accept-source-agreements
# .NET SDK 7.0.404
winget install Microsoft.DotNet.SDK.7 --version 7.0.404 -h --accept-package-agreements --accept-source-agreements
# Docker Desktop 4.26.1
winget install Docker.DockerDesktop --version 4.26.1 -h --accept-package-agreements --accept-source-agreements
# Git 2.43.0
winget install Git.Git --version 2.43.0 -h --accept-package-agreements --accept-source-agreements
```

# After prerequisites are installed, continue with project setup.
