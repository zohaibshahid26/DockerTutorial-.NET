# Dockerized .NET Core Application with SQL Server

This project demonstrates how to dockerize a .NET Core application with SQL Server integration. Follow the steps below to set up, build, and run the application.

---

## Prerequisites
- Docker installed ([Get Docker](https://www.docker.com/products/docker-desktop))
- SQL Server Management Studio (SSMS)
- .NET SDK 9.0 or later

---

## Steps to Set Up

### Step 1: Backup the Database
1. Open **SQL Server Management Studio (SSMS)** and connect to your SQL Server instance.
2. Right-click on the database you want to back up, select **Tasks**, and then **Back Up...**
3. Configure the backup settings and specify the destination path for the backup file (e.g., `C:/Program Files/Microsoft SQL Server/MSSQL16.SQLEXPRESS/MSSQL/Backup/DockerTutorial.bak`).
4. Click **OK** to create the backup.

### Step 2: Create `docker-compose.yml`
Create a file named `docker-compose.yml` with the following content:

```yaml
docker-compose.yml
services:
  DockerTutorialDb:
    container_name: DockerTutorialDb
    image: custom-sqlserver
    build:
      context: .
      dockerfile: Dockerfile.sqlserver
    ports:
      - 1433:1433
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=Dotnet@123
    networks:
      - DockerTutorial
    volumes:
      - sqlserver_data:/var/opt/mssql
      - C:/Program Files/Microsoft SQL Server/MSSQL16.SQLEXPRESS/MSSQL/Backup:/var/opt/mssql/backup

  web:
    container_name: web
    ports:
      - 5000:8080
    image: ${DOCKER_REGISTRY-}web
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
      - DockerTutorialDb
    networks:
      - DockerTutorial

networks:
  DockerTutorial:
    driver: bridge

volumes:
  sqlserver_data:
```

### Step 3: Create `Dockerfile.sqlserver`
Create a file named `Dockerfile.sqlserver` with the following content:

```dockerfile
# Use the official SQL Server image as a base
FROM mcr.microsoft.com/mssql/server:2022-latest

# Switch to root user to install mssql-tools
USER root

# Install required dependencies and mssql-tools
RUN apt-get update && \
    apt-get install -y curl gnupg && \
    curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
    curl https://packages.microsoft.com/config/ubuntu/22.04/prod.list | tee /etc/apt/sources.list.d/msprod.list && \
    apt-get update && \
    ACCEPT_EULA=Y apt-get install -y mssql-tools unixodbc-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Switch back to the default user
USER mssql
```

### Step 4: Create `Dockerfile`
Create a file named `Dockerfile` with the following content:

```dockerfile
# See https://aka.ms/customizecontainer for details.

# This stage is used when running from VS in fast mode
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# This stage is used to build the service project
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["DockerTutorial.csproj", "."]
RUN dotnet restore "./DockerTutorial.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./DockerTutorial.csproj" -c $BUILD_CONFIGURATION -o /app/build

# This stage is used to publish the service project
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./DockerTutorial.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# This stage is used in production
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DockerTutorial.dll"]
```

### Step 5: Update `appsettings.json`
Ensure the connection string in `appsettings.json` is correct:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=DockerTutorialDb;Initial Catalog=DockerTutorial;User ID=sa; Password=Dotnet@123;Encrypt=False;TrustServerCertificate=True;"
  }
}
```

### Step 6: Build and Run the Containers
Run the following command to build and start the containers:

```bash
docker-compose up --build -d
```

### Step 7: Restore the Database
Restore the database from the backup file using the following command:

```bash
docker exec -it DockerTutorialDb /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P Dotnet@123 -Q "RESTORE DATABASE [DockerTutorial] FROM DISK = '/var/opt/mssql/backup/DockerTutorial.bak' WITH MOVE 'DockerTutorial' TO '/var/opt/mssql/data/DockerTutorial.mdf', MOVE 'DockerTutorial_log' TO '/var/opt/mssql/data/DockerTutorial_log.ldf'"
```

### Step 8: Verify the Database Restoration
To verify that the database has been restored successfully:
1. Connect to the `DockerTutorialDb` container:
   ```bash
   docker exec -it DockerTutorialDb /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P Dotnet@123
   ```
2. Execute the following command in the SQL prompt to list available databases:
   ```sql
   SELECT name FROM sys.databases;
   GO
   ```
Ensure `DockerTutorial` is listed among the databases.

### Step 9: Access the Application
The application will be available at:

[http://localhost:5000](http://localhost:5000)

---

## Notes
- Ensure that the database backup path in `docker-compose.yml` matches the backup file path created in Step 1.
- Use a strong password for `MSSQL_SA_PASSWORD` in production environments.

---

## Troubleshooting
- If the containers fail to start, check the logs using:
  ```bash
  docker logs <container_name>
  ```
- Verify that port `5000` for the application and port `1433` for SQL Server are not in use on your system.
