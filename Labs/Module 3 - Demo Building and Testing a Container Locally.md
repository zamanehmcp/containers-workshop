# Demonstration: Building and Testing a Container Locally

The .NET project is the standard To-Do app commonly used in Azure learning modules and repos. It is configured for ASP.NET Core and .NET Core 7.0.

The project structure will look as follows:

```console
- containers-workshop
  - todoapp
    - Controllers
    - Data
    - Models
    - Properties
    - Views
    - wwwroot
  - .gitignore
  - CONTRIBUTING.md
  - LICENSE
  - LICENSE.md
  - README.md
```

## Create a local database

Use the SQLite database for immediate testing of the code and any database schema changes.

Prepare the database by running the dotnet database migrations.

```console
dotnet ef database update
```

Run the app locally to test the code and database schema. 
```console
dotnet run
```

### Create an Azure Resource Group
Create a resource group in Azure to contain all of the services required to complete this demo.
```console
az group create --name rg-container-demo --location "East US"
```

### Create an Azure SQL database server
Create an Azure SQL database server to be used by the app when deployed in the cloud.
```console
az sql server create --name dbs-container-demo --resource-group rg-container-demo --location "East US" --admin-user <db admin username> --admin-password <admin password>
```

### Create the Azure SQL database
Create the SQL Server database on the SQL Server that was just create and display the connection string. Copy and save the connection string for later use in configuring the app and the cloud services.
```console
az sql db create --resource-group rg-container-demo --server dbs-container-demo --name todoDB --service-objective S0
az sql db show-connection-string --client ado.net --server dbs-container-demo --name todoDB
```

### Update the C# code to connect to the Azure SQL database
Update the database context in Startup.cs to connect to the Azure SQL database instead of the local SQLite database

`services.AddDbContext<MyDatabaseContext>(options => options.UseSqlServer(Configuration.GetConnectionString("MyDbConnection")));`

### Update the .NET Entity Framework code to access the Azure SQL database
Delete the database migrations associated with the SQLite database.
```console
rm -r Migrations
```

Recreate migrations for Azure SQL
```console
dotnet ef migrations add InitialCreate
```

Create an Azure SQL database connection string environment variable in Powershell. This env variable will be used by the .NET Entity Framework `dotnet ef` command to initialize the Azure SQL datbase schema for the app.
```console
$env:ConnectionStrings:MyDbConnection=<database connection string>
```

Run the .NET database migrations for Azure SQL database to create the database schema.
```console
dotnet ef database update
```

Run the web app locally with the Azure SQL database.
```console
dotnet run
```

## Build a container image for the app and store it in Azure Container Registry

### Create a container image for the app.
Create a Dockerfile that will be used to build the container image. The file should be located in the project root directory and include both the .NET restore and publish tasks, as show in the example below:

```
# todoapp Dockerfile example
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /source

# Copy csproj and restore the project as distinct layers
COPY todoapp/*.csproj .
RUN dotnet restore --use-current-runtime  

# Copy the remaining app files and build app
COPY todoapp/. .
RUN dotnet publish -c Release -o /app --self-contained

# Final stage: build the container image
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "todoapp.dll"]
```

### Create the container image using the Dockerfile.
The image will be stored in your local Docker Desktop image registry. Run the `docker build` command from the project root directory.
```console
docker build -t todoapp .
```

Run the containerized app locally in Docker Desktop to test the app. Use the `docker run` command to run the container.
```console
docker run -it --rm -p 8000:80 --name todoapp todoapp
```
