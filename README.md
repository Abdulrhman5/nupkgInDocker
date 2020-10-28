# This repository is to make the process of restoring nuget packages inside docker container faster if you have slow or expensive internet connection.
When building an image it used to take me more than 40 minutes to restore the nuget packages, Now it only take about 3 to 10 minutes.

The idea behind this is simple: 
1. Configure [Nuget.Server](https://docs.microsoft.com/en-us/nuget/hosting-packages/nuget-server) to use your host machine package cache, Build it, and Run it.
1. In the image you want to build, make the Nuget.Server a nuget package source.

### The code in this repository is just asp.net framework project using [Nuget.Server](https://docs.microsoft.com/en-us/nuget/hosting-packages/nuget-server).

## 1. Setup [Nuget.Server](https://docs.microsoft.com/en-us/nuget/hosting-packages/nuget-server) 
1. Clone this repository to your host machine, or create your [Nuget.Server](https://docs.microsoft.com/en-us/nuget/hosting-packages/nuget-server).
2. Configure it to use the package cache in your host machine.
    * the package cache is usually located at "C:\Users\UserName\\.nuget\packages", you don't need to change the structure of this directory, or move only *.nupkg files to another directory. it's fine as it is.
    * In the Web.config:
        ```xml
        <configuration>
            ...
            <appsetting>
                ...
                <add key="packagesPath" value="C:\Users\YourUserName\.nuget\packages" />
            </appsetting>
        </configuration>
        ```
3. Configure IIS Express to expose this application on all the end-points:
    * In applicationhost.config:
        ```xml
        <configuration>
            ...
            ...
            ...
            <system.applicationHost>
                ...
                ...
                ...
                <sites>
                    ...
                    <site name="NugetServer(1)" id="8">
                        <application path="/" applicationPool="Clr4IntegratedAppPool">
                            <virtualDirectory path="/" physicalPath="THE_PATH_TO_THE_PROJECT_PATH\src\NugetServer" />
                        </application>
                        <bindings>
                            <--- Change bindingInformation from "*:44353:localhost"  to "*:44353:*"--->
                            <binding protocol="http" bindingInformation="*:44353:*" />
                        </bindings>
                    </site>
        ```
4. To test if you can access the application from containers:
    
    In your host machine:
    * Get The ip address of the vEthernet(wsl) interface, In my case it was 172.20.112.1.
    * In the browser: http://172.20.112.1:44353 
            
        if you see something like this:
        <h2>You are running NuGet.Server v3.4.1.0</h2>
        You are good to go.


## 2. In The container, Make the NugetServer a nuget source.
1. In the project or the solution directory add NuGet.Config file:
    ```xml 
    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
        <packageSources>
            <add key="Host machine package source" value="http://host.docker.internal:44353/nuget" />    
        </packageSources>
    </configuration>
    ```

1. COPY this file when copying *.csproj and *.sln
    ```docker
        COPY ["MyProject.csproj", "/MyProject.csproj"]
        COPY ["docker-compose.dcproj", "docker-compose.dcproj"]
        COPY ["NuGet.Config","NuGet.Config"]
    ```

## And that's it. Now you can build your images faster.