# Module 3 - Lab 1: Building and Testing a Container Locally

The sample .NET project is a standard To-Do app commonly used in Azure learning modules and repos. It is configured for ASP.NET Core and .NET Core 7.0. The application currently uses the SQLite desktop database. In this demonstration, the app will be refctored to use Azure SQL in preparation to be migrated to the cloud.

In this lab, you will create the Azure services required to support the application, containerize the application, and run it locally on your desktop.

The project structure will look as follows:

```console
- containers-workshop
  - App
    - Starter
      - todoapp
        - Controllers
        - Data
        - Models
        - Properties
        - Views
        - wwwroot
    -Working
  - Labs
  - Presentations
  - .gitignore
  - CONTRIBUTING.md
  - LICENSE
  - LICENSE.md
  - README.md
```

## Prepare the app for refactoring

### Prepare the code baseline for refactoring and containerization

Copy the app code to the working directory that will be used to complete all labs. You will complete all of the labs with the .NET project in the `Working` folder.

Powershell

```console
PS C:\Repos\containers-workshop\App> Copy-Item -Path '.\App\Starter\todoapp' -Recurse -Destination '.\App\Working\todoapp'
```

## Provision the Azure services required by the application

### Set environment variables

```console
$env:RESOURCE_GROUP="rg-containers-workshop"
$env:LOCATION="eastus"
$env:CLUSTER_NAME="aks-containers-workshop"
$env:ACR_NAME="acrcontainersworkshop"
$env:AS_DBSRV_NAME="as-dbs-containers-workshop"
$env:AS_DBSRV_SKU="S0"
```

### Create an Azure Resource Group

```console
az group create --name rg-containers-workshop --location "East US"
```

### Create an Azure SQL database server

Azure SQL database is used by the app. You must create the database server and then create the app database.

```console
az sql server create --name $env:AS_DBSRV_NAME --resource-group $env:RESOURCE_GROUP --location $env:LOCATION --admin-user <db admin username> --admin-password <admin password>
```

Create the database on the Azure SQL server that was just created and display the connection string. Copy and save the connection string for later use in configuring the app and the cloud services.

```console
az sql db create --resource-group $env:RESOURCE_GROUP --server $env:AS_DBSRV_NAME --name todoDB --service-objective $env:AS_DBSRV_SKU
az sql db show-connection-string --client ado.net --server $env:AS_DBSRV_NAME --name todoDB
```

## Modify and run the app locally

### Run the web app natively on your desktop with the Azure SQL database

Populate the MyDbConnection configuration item displayed by the last command in the appsettings.json file with connection string displayed for the Azure SQL database. Be sure to replace the admin username and password with the username and password used when creating the database.

Your appsettings.json file should look as follows:

```console
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "MyDbConnection": \
      "Server=tcp:as-dbs-containers-workshop.database.windows.net,1433; \
      Initial Catalog=todoDB; \
      Persist Security Info=False; \
      User ID=<admin user>; \
      Password=<admin pwd>; \
      MultipleActiveResultSets=False; \
      Encrypt=true; \
      TrustServerCertificate=False; \
      Connection Timeout=30;"
  }
}
```

Create migrations for Azure SQL

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

Run the app.

```console
dotnet run
```

## Build a container image for the app and store it in Azure Container Registry

### Create a container image for the app

Create a Dockerfile that will be used to build the container image. The file should be located in the project root directory and include both the .NET restore and publish tasks, as show in the example below:

```console
# todoapp Dockerfile example
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /source

# Copy csproj and restore the project as distinct layers
COPY *.csproj .
RUN dotnet restore --use-current-runtime  

# Copy the remaining app files and build app
COPY . .
RUN dotnet publish -c Release -o /app --self-contained

# Final stage: build the container image
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "todoapp.dll"]
```

### Create the container image using the Dockerfile

The image will be stored in your local Docker Desktop image registry. Run the `docker build` command from the project root directory. The build command caches the content of build layers. To completely rebuild the image, use the --no-cache switch.

```console
docker build -t todoapp .

OR

docker build --no-cache -t todoapp .
```

Run the containerized app locally in Docker Desktop to test the app. Use the `docker run` command to run the container.

```console
docker run -it --rm -p 8000:80 --name todoapp todoapp
```

In your web browser, navigate to http://localhost:8000 to test the app.
