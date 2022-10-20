---
tags:
  - deploying
  - docker
---

# Docker Compose for ASP.Net Core with Postgres + S3 backups

_2021-07-12_

In this post, I will cover how I set up the following application structure, run from a single `docker-compose` file:

- ASP.Net Core MVC & Razor Web Application
- ASP.Net Core Entity Framework Migrations
- PostgreSQL
- SMTP Server
- S3 Backups for PostgreSQL
- S3 Restores for PostgreSQL

The directory structure, including pertinent files, looks like this.

```
Repository Root Folder
│   .dockerignore
│   database.env  
│   docker-compose.yml
│
└───ASP.Net Core Project Folder
│   │   Dockerfile
│   │   Migrations.Dockerfile
│   │   Setup.sh
│   
└───postgres-backup-s3
│    │   Dockerfile
│   
└───postgres-restore-s3
│    │   Dockerfile
```

Starting with an almost blank `docker-compose.yml` file, over the course of this post we'll add each service so that the entire infrastructure can be brought up with one single `docker-compose up` command.

```docker
version: '3.4'

services:
```

N.B. One optional thing not covered here is to include `restart: always` for each service in the `docker-compose.yml` file to ensure they come back online after the host machine reboots. I wouldn't recommend using this for anything but the core web app, database and mail server.

## ASP.Net Core Web Application

For this we want to specify a few key things:

- the name to give the container, to make it easier to work with than the docker auto-generated names
- the internal port to expose on the host machine's external port
- the Dockerfile to use to build the application
- the folder to map to make log files accessible from the host machine without having to use the terminal
- the environment variable to set the app into production mode instead of the default
- the other containers this one will depend on

```docker
version: '3.4'

services:
  aspprojectname:
    container_name: myaspprojectname
    ports:
      - "80:80"
    build:
      context: .
      dockerfile: MyAspProjectName/Dockerfile
    volumes:
      - ./MyAspProjectName/logs:/app/logs
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    depends_on:
      - db
      # - migrations
```

You will notice here that I am commenting out the migrations dependency. This is because with the S3 backup and restore images it's not really needed, and it requires more powerful hardware to run than any of the other images and is a large docker image, so worth avoiding if possible, or only using to set up the database initially, then deleting.

The Dockerfile itself is pretty standard for ASP.Net Core apps. In this instance, the app is running on dotnet 5. 

```docker
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src
COPY ["MyAspProjectName/MyAspProjectName.csproj", "MyAspProjectName/"]
RUN dotnet restore "MyAspProjectName/MyAspProjectName.csproj" --disable-parallel
COPY . .
WORKDIR "/src/MyAspProjectName"
RUN dotnet build "MyAspProjectName.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyAspProjectName.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyAspProjectName.dll"]
```

## ASP.Net Core Entity Framework Migrations

This one is pretty similar to the main web app in terms of what needs adding to add the service to the `docker-compose.yml` file.

```docker
  migrations:
    container_name: dbmigrations
    build: 
      context: .
      dockerfile: MyAspProjectName/Migrations.Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    depends_on: 
      - db
```

The Dockerfile itself is where the magic happens, building the web app, installing the global dotnet-ef tools needed and then running the migrations.

```docker
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build

WORKDIR /src
COPY ["MyAspProjectName/MyAspProjectName.csproj", "MyAspProjectName/"]
COPY ["MyAspProjectName/Setup.sh", "MyAspProjectName/"]

ENV PATH="${PATH}:/root/.dotnet/tools"
RUN dotnet tool install --global dotnet-ef

RUN dotnet restore "MyAspProjectName/MyAspProjectName.csproj" --disable-parallel
COPY . .
WORKDIR "/src/MyAspProjectName/."

RUN /root/.dotnet/tools/dotnet-ef migrations add InitialMigrations

RUN chmod +x ./Setup.sh
CMD /bin/bash ./Setup.**sh**
```

Personally, I don't use this unless I have to, for the reasons previously stated, but it's worth sharing in case it's useful to anyone else.

## PostgreSQL

There's two key parts to add to the `docker-compose.yml` file for this one. The service itself, with the port to expose and an `env` file to store sensitive information, and the volume mapped to a directory within the container.

```docker
services:
  db:
    container_name: myappdb
    image: "postgres"
    ports:
      - "5432:5432"
    env_file:
    - database.env # configure postgres
    volumes:
    - database-data:/var/lib/postgresql/data/ # persist data even if container shuts down

volumes:
    database-data: # named volumes can be managed easier using docker-compose
```

That `env` file is remarkably simple.

```text
POSTGRES_USER=postgres
POSTGRES_PASSWORD=passwordgoeshere
POSTGRES_DB=yourdbname
```

## SMTP Server

The service for the SMTP server is super simple.

```docker
mail:
    image: bytemark/smtp
```

## S3 Backups for PostgreSQL

The `docker-compose.yml` for this is as below, which sets a daily schedule and connection details for both the Postgres database and S3. Note that the Postgres host uses the internal Docker hostname for the database container.

```docker
pgbackups3:
    build:
      context: .
      dockerfile: postgres-backup-s3/Dockerfile
    links:
      - db
    environment:
      SCHEDULE: '@daily'
      S3_REGION: eu-west-2
      S3_ACCESS_KEY_ID: keygoeshere
      S3_SECRET_ACCESS_KEY: secretkeygoeshere
      S3_BUCKET: yourapp-backups
      S3_PREFIX: backup
      POSTGRES_HOST: db
      POSTGRES_DATABASE: yourdbname
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: passwordgoeshere
      POSTGRES_EXTRA_OPTS: '--schema=public --blobs'
```

I won't go into how this application works here, that'll be in another post.

## S3 Restores for PostgreSQL

The `docker-compose.yml` for this is as below, which sets  connection details for both the Postgres database and S3. Note that the Postgres host uses the internal Docker hostname for the database container.

This container should be run at setup time then stopped and commented out of the `docker-compose.yml` file or removed entirely, to prevent it from being accidentally run and restoring unintentionally, overwriting the database (notice the drop public option - this will wipe everything in the database before restoring).

```docker
pgrestores3:
    build:
      context: .
      dockerfile: postgres-restore-s3/Dockerfile
    links:
      - db
    environment:
      S3_ACCESS_KEY_ID: keygoeshere
      S3_SECRET_ACCESS_KEY: secretkeygoeshere
      S3_BUCKET: yourapp-backups
      S3_PREFIX: backup
      POSTGRES_HOST: db
      POSTGRES_DATABASE: yourdbname
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: passwordgoeshere
      DROP_PUBLIC: 'yes'
```

I won't go into how this application works here, that'll be in another post.

## Putting it all together

The complete `docker-compose.yml` file, after the initial `docker-compose up` command has been run, looks like this. Note that migrations is commented out (we don't need it as we have the S3 backup and restore) and that the S3 restore is commented out to prevent accidental restores.

```docker
version: '3.4'

services:
  aspprojectname:
    container_name: myaspprojectname
    ports:
      - "80:80"
    build:
      context: .
      dockerfile: MyAspProjectName/Dockerfile
    volumes:
      - ./MyAspProjectName/logs:/app/logs
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    depends_on:
      - db
      # - migrations
  # migrations:
  # container_name: dbmigrations
  # build: 
  #     context: .
  #     dockerfile: MyAspProjectName/Migrations.Dockerfile
  # environment:
  #     - ASPNETCORE_ENVIRONMENT=Production
  # depends_on: 
  #     - db
  db:
      container_name: myappdb
      image: "postgres"
      ports:
          - "5432:5432"
      env_file:
      - database.env # configure postgres
      volumes:
      - database-data:/var/lib/postgresql/data/ # persist data even if container shuts down
  mail:
      image: bytemark/smtp
  pgbackups3:
      build:
          context: .
          dockerfile: postgres-backup-s3/Dockerfile
      links:
          - db
      environment:
          SCHEDULE: '@daily'
          S3_REGION: eu-west-2
          S3_ACCESS_KEY_ID: keygoeshere
          S3_SECRET_ACCESS_KEY: secretkeygoeshere
          S3_BUCKET: yourapp-backups
          S3_PREFIX: backup
          POSTGRES_HOST: db
          POSTGRES_DATABASE: yourdbname
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: passwordgoeshere
          POSTGRES_EXTRA_OPTS: '--schema=public --blobs'    
  # pgrestores3:
  #     build:
  #         context: .
  #         dockerfile: postgres-restore-s3/Dockerfile
  #     links:
  #         - db
  #     environment:
  #         S3_ACCESS_KEY_ID: keygoeshere
  #         S3_SECRET_ACCESS_KEY: secretkeygoeshere
  #         S3_BUCKET: yourapp-backups
  #         S3_PREFIX: backup
  #         POSTGRES_HOST: db
  #         POSTGRES_DATABASE: yourdbname
  #         POSTGRES_USER: postgres
  #         POSTGRES_PASSWORD: passwordgoeshere
  #         DROP_PUBLIC: 'yes'

volumes:
    database-data: # named volumes can be managed easier using docker-compose
```
