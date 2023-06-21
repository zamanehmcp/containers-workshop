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