# Stage 1: Build the application using the SDK
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Build context is src/backend, so all paths are relative to that folder:
#   docker build -f src/backend/PatientRepo.Api/Dockerfile src/backend

# Copy and restore dependencies as distinct layers
# (add each referenced project's .csproj here as the solution grows)
COPY PatientRepo.Api/PatientRepo.Api.csproj PatientRepo.Api/
RUN dotnet restore PatientRepo.Api/PatientRepo.Api.csproj

# Copy everything else and publish the application
COPY . .
RUN dotnet publish PatientRepo.Api/PatientRepo.Api.csproj -c Release -o /app --no-restore

# Stage 2: Build the runtime image
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "PatientRepo.Api.dll"]