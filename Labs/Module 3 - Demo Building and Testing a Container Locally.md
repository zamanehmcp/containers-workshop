# Demonstration: Building and Testing a Container Locally

The .NET project is the standard To-Do app commonly used in Azure learning modules and repos. It is configured for ASP.NET Core and .NET Core 7.0. It assumes the existence of an Azure SQL database.

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

Run the web app locally with the Azure SQL database.

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
