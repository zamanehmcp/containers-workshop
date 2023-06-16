# Module 3 - Demonstration: Building and Testing a Container Locally

In this lab, you will create the Azure services required to support the application, containerize the application, and run it locally on your desktop.

The sample .NET project is a standard To-Do app commonly used in Azure learning modules and repos. It is configured for ASP.NET Core and .NET Core 7.0. It assumes the existence of an Azure SQL database.

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

## Provision the Azure services required to complete the lab

### Create an Azure Resource Group

```console
az group create --name rg-containers-workshop --location "East US"
```

### Create an Azure SQL database server

Azure SQL database is used by the app. You must create the database server and then create the app database.

```console
az sql server create --name dbs-container-demo --resource-group rg-container-demo --location "East US" --admin-user <db admin username> --admin-password <admin password>
```

Create the database on the Azure SQL server that was just created and display the connection string. Copy and save the connection string for later use in configuring the app and the cloud services.

```console
az sql db create --resource-group rg-container-demo --server dbs-container-demo --name todoDB --service-objective S0
az sql db show-connection-string --client ado.net --server dbs-container-demo --name todoDB
```

## Modify and run the app locally

### Run the web app locally with the Azure SQL database

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
        "Server=tcp:dbs-sql-containers-workshop.database.windows.net,1433; \
        Initial Catalog=tododb; \
        Persist Security Info=False; \
        User ID={serverlogin_name}; \
        Password={your_password}; \
        MultipleActiveResultSets=False; \
        Encrypt=True; \
        TrustServerCertificate=False; \
        Connection Timeout=30;"
  }
}
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