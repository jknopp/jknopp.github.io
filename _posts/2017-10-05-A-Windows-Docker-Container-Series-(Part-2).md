---
# published: true
# layout: post
# date: 2017-10-06T00:00:00.000Z
# comments: true
description: A Windows Docker Container Series (Part 2)
category: docker
tags:
  - docker
  - windows
  - visual studio
title: A Windows Docker Container Series (Part 2)
---

#### Goals
After [Part 1](https://laughlin.github.io/A-Windows-Docker-Container-Series-(Part-1)) where we created a build template in Packer for a Docker compatible image of Windows Server, capable of running containers, I was inspired by the [West Wind Album Viewer](https://github.com/dockersamples/dotnet-album-viewer) sample project, and decided it was time to start developing a containerized app for the server to run. With [Visual Studio 2017](https://docs.microsoft.com/en-us/aspnet/core/publishing/visual-studio-tools-for-docker) and .NET Framework 4.7, there is native support for running containerized applications with Docker. 

From the [Docker for Windows](https://docs.docker.com/docker-for-windows/install/#download-docker-for-windows) website: "Docker for Windows requires 64bit Windows 10 Pro with Hyper-V available", but I opted for Enterprise which worked fine also. The trick for me was enabling Hyper-V nested virtualization since my Windows 10E image is running in a VM, and also needs the "Virtualization", and "Containerization" Windows Features enabled, which I will cover more on later. Visual Studio 2017 also contains a native template for .NET Core 2.0 Web API Projects with Docker support via Docker Compose. The goal is to combine this with their Angular Template and a Docker SQL Server container to create a 3 container app debuggable from Visual Studio and SSMS.

#### An Introduction to Visual Studio with Docker
In order to run Docker and Visual Studio together you must have the following prerequisites: Install Microsoft Visual Studio 2017 with the .NET Core workload and setup [Shared Drives](https://docs.docker.com/docker-for-windows/#shared-drives) in Docker for Windows. With a new project, there are 2 ways to add docker support to you .NET Core project: 1) Using Visual Studio, create a new ASP.NET Core Web Application. When the application is loaded, either select Add Docker Support from the Project Menu or 2) right-click the project from the Solution Explorer and select Add > Docker Support. In this solution, we will use both ways.
#### Enabling support for Docker inside a Hyper-V VM
One of the tricks I face in getting docker working with Visual Studio was compatibility for Docker for Windows inside a Hyper-V VM. Since I wanted to create as much of this project in an automated format as I could, I opted to use fresh VMs for the different steps of the project. This allows me the reparability and reliance we discussed in Part 1. With that comes a catch, Docker for Windows requires Hyper-V Virtualization in order to run containers, which means Nested Virtualization for me since my Windows 10E instance is already running inside a Hyper-V VM. Luckily, I was able to find this [post](http://techgenix.com/docker-and-containers-part5/) which had the following solution to enabling Nested Virtualization in Hyper-V:
```powershell
Set-VMProcessor -VMName ContainerVM01 -ExposeVirtualizationExtensions $true
```

### How I want to setup the project
As mentioned above, the goal I set out to achieve with this solution was 2 different projects, 1 angular front-end container, and 1 back-end API container that would be networked together to allow for communication. Finally I wanted to run [SQL Server](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker) as a database solution inside a third container which would be networked with the API container. Since Visual Studio [natively uses Docker Compose](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/multi-container-applications-docker-compose) for multi-container apps, I also wanted to craft my solution in such a way that all the Visual Studio Debugging tools and SQL Server Management functions still worked liked a traditional .NET application.

### Walkthrough
#### Setting up the Projects
To start of I created the Web API project using the Visual Studio Project Menu. From the menu, I used option #1 from above to enable support for Docker directly when the project was created. What this does is creates a Dockerfile and a docker-compose.yml file which can be used with the Docker engine to run your application inside a container. For the Angular project which is presented in the Project Menu the Docker Support menu is not enabled so I started off creating this project without it. I then used option #2 from above to enable the Docker Support later. More on that, and the SQL Server container to come. 
Breakdown of tiers:

| Tier              |           Visual Studio Docker Support            |          Container Support       |
| ----------------- | :-----------------------------------------------: | :------------------------------: |
| Angular           |              Yes, but not natively                |                Yes               |
| Web API           |                  Yes, natively                    |                Yes               |
| SQL Server        |                  Not Available                    |                Yes               |

As noted by the chart, the SQL Server container will not have a Visual Studio project but will be contained within the Visual Studio solutions docker-compose.yml file so that it is run with the solution when debugging.

#### Should I run Linux Containers or Windows Containers?
Windows supports the ability to develop and run both Linux-based and Windows-based containers, however, I don't believe there is a way to develop for both simultaneously in the same solution. When starting off I chose Windows-based containers however, I quickly decided this wasn't a good choice and started the solution over as Linux-based. The reason I chose this was because my front-end container is being created in Angular and could benefit from having NodeJS running. Also, being a backend developer by trade, I like the idea that the frontend project would be completely changed, and modified (even ported to another language) without affecting any of the API logic. Trying to install NodeJS fully from Powershell in a Dockerfile was proving extremely cumbersome so I opted for Linux in favor of a simple Bash install:
```
RUN apt-get update  
RUN apt-get -f install  
RUN apt-get install -y wget gnupg
RUN wget -qO- https://deb.nodesource.com/setup_8.x | bash -  
RUN apt-get install -y build-essential nodejs
```

SQL Server is available as a [Linux container](https://hub.docker.com/r/microsoft/mssql-server-linux/) and .NET Core 2.0 is also compatible on Linux so for this solution it seemed like the right fit. 

#### The API Project
In the name of keeping things simple and straightforward at this point in the tutorial, I have the left the API project very close to what was auto-generated, with a few exceptions:
* In the sampledatacontroller.cs class I have the attributes:
 * [EnableCors("CorsPolicy")] - needed to allow calls from another container
 * [Route("api/[controller]")] - setup our API prefix
* In the startup.cs file I added:
```csharp
services.AddCors(options =>
{
	options.AddPolicy("CorsPolicy",
    	builder => builder
        			.AllowAnyOrigin()
                    .AllowAnyMethod()
                    .AllowAnyHeader()
                    .AllowCredentials());
});
```

#### The Angular Project
This project was the trickiest to get running in a Docker container. As mentioned above, I had to add the bash script lines to the generate Dockerfile. I also had to update the ClientApp\app\components\fetchdata\fetchdata.component.ts file to account for the API call instead of a local call. In addition to updating the call I also added an app.config.ts file to ClientApp\app\ which held the variable 'apiBaseUrl' in a class, so that the "http://localhost:PORT/ value could be configurable and global. The reason this project is separated is because it is still served by .NET Core, meaning it uses Startup.cs, Program.cs and a HomeContoller to bootstrap the routing of the Angular SPA, similar to how Express might be used in NodeJS. 

As previously mentioned, in theory this project could be moved entirely out of .NET Core, and run solely on a NodeJS web server which could then communicate to the existing API back-end. In this scenario, I could see the Front-end Angular project being excluded from the Visual Studio solution, and possibly developed in a lighter-weight editor with better NodeJS support, like Sublime. The final trick was making sure that the port was predictable for the API container so that the Angular calls could reach, which I will discuss in the next section.

#### The Docker Compose files
The setup of the docker-compose.yml was pretty straight-forward based on the generated file with a few [modifications](https://docs.docker.com/compose/aspnet-mssql-compose/):

```yaml
services:
  angular.web:
    image: angular.web
    build:
      context: ./Angular
      dockerfile: Dockerfile
    depends_on:
      - angular.api

  angular.api:
    image: angular.api
    build:
      context: ./Angular.API
      dockerfile: Dockerfile
    depends_on:
      - db

  db:
    image: "microsoft/mssql-server-linux:2017-latest"
    environment:
        SA_PASSWORD: "SuperSecretPassword"
        ACCEPT_EULA: "Y"
```

I combined all 3 containers into 1 file and added the depends_on sections, which helps ensure the containers are launched in the proper order.

The other file of importance that is included is the docker-compose.override.yml file. This allows you to set some development overrides. Following the template that was auto-generated I ended up with:
```yaml
services:
  angular.web:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "80"

  angular.api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - 4828:80 # Explicitly map this port because Angular.Web project expects it 

  db:
    ports:
      - 1433:1433 # Explicitly map this port because SSMS expects it 
```

Here is where I was able to explicitly define both the API port, and the DB port so that it would be predictable and constant, not auto-assigned by Docker.

#### SQL Server Container
Getting the initial SQL Server container up and running was pretty straight-forward thanks to the surprisingly comprehensive documentation and examples from both [Microsoft](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker) and Docker. However, in my initial attempt to get the container running I ran into a bug which I could not get around and [finally assumed must have been in the Docker for Windows platform or SQL Container](https://github.com/Microsoft/mssql-docker/issues/158). It centered around the Memory requirement (4GB RAM) for the SQL Server container which I clearly met. I was able to figure out the problem thanks to [rabih](https://github.com/rabih-harb) who [posted the code](https://github.com/Microsoft/mssql-docker/issues/20#issuecomment-296226637) to print the stdout messages in the powershell console:
```powershell
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=abc@123" -p 1433:1433 -a stdin -a stdout -a stderr microsoft/mssql-server-linux:2017-latest
```
After letting a few versions pass, I updated all my clients and all was well and running.

### Next Steps
The next steps I would like to take with the projects in this solution are (in no particular order):
* add full database support to the API so data can be pulled out of tables
* continue to improve upon the front-end project by adding other NodeJS libraries and tools
* look into swapping the Angular frontend for React frontend based on some feedback from a front-end team-member



Hope you learned something with me; till next time,

Jon Knopp
