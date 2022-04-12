Create a folder `terraform-sample`.

Open windows terminal inside the new folder and run `dotnet new webapi -minimal -n weatherapi -o .`

Open VS Code and accept to add the assets for build and debug (will add folder `.vscode`).

In `Program.cs` comment `app.UseHttpsRedirection();`

> Note: feel free to try it. This is a crash intro to Azure DevOps and Terraform and we will avoid likely conflicts.

Run the app `dotnet run` to check every works nicely.

Add a `Dockerfile`to your project with the content below:

```
# https://hub.docker.com/_/microsoft-dotnet
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.csproj .
RUN dotnet restore

# copy everything else and build app
COPY . .
RUN dotnet publish -c release -o out

# final stage/image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
EXPOSE 80
COPY --from=build /app/out .
ENTRYPOINT ["dotnet", "weatherapi.dll"]
```

Create docker image with `docker build -t xgrois/weatherapi .`

Run a docker container using that image to test everything behaves as expected. Use `docker run --rm -p 8080:80 xgrois/weatherapi`

Push docker image to DockerHub `docker push xgrois/weatherapi`

Add `.gitignore` file with the Visual Studio template _and_ the next addition:

```
...
*.sln.docstates

# VS Code (just add this folder)
.vscode

# User-specific files (MonoDevelop/Xamarin Studio)
*.userprefs
...
```

Commit and push
