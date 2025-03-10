# 1 - Base image to work from. SDK for .NET
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build-stage
# 2 - Set working directory
WORKDIR /app
# 3 - Copy source files to the image
COPY . ./
# 4 - Run 'dotnet restore' to install missing dependencies
RUN dotnet restore 
# 5 - Publish our application to the image
RUN dotnet publish -c Release -o out

# 6 - Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0
# 7 - re-establish the working directory
WORKDIR /app
# 8 - Copy over the published files (the rest will be discarded)
COPY --from=build-stage /app/out .
# 9 - Configure how the application is run
ENTRYPOINT ["dotnet", "weatherstationCICD.dll"]
