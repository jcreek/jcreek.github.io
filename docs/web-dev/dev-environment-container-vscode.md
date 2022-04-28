---
layout: post
parent: Web Development
nav_order: 3
title:  "Setting up remote containers for IAC dev environments in dotnet 6 with postgres and SMTP"
date:   2022-04-20 15:51:03 +0000
categories: dotnet core asp vscode remote-container remote container iac postgress smtp dev environment
---

# {{page.title}}

_{{page.date}}_

![](/assets/devcontainer.gif)

In this post I will go through how to set up remote containers for a dotnet 6 API project using PostgreSQL and an SMTP server, so that any new devs picking up the project only have to clone down the git repo and open it in Visual Studio code to get started debugging it.

No dependencies need to be installed onto developer machines for each project they work on, other than:

- Docker
- Visual Studio Code

Yes, that's all they need. They don't even need to install any extensions - the only one they need will be prompted the first time they open the project in vscode. Any extensions they need for the project will be loaded in a vscode server instance within the docker container, and then their local vscode will allow access to the vscode server as if they're using it natively, right down to providing terminal access within the container.

## How to set up the project

You will need to ensure that you've created a `.gitattributes` file in the root folder if you don't have one already, and specified `* text=auto eol=lf` to default all the line endings to LF, if you're on Windows and find that swapping between the host and the container changes the line endings in all your files.

In the root of the repo create a folder called `.devcontainer`. Within this you need to create three files:

1. `devcontainer.json`
2. `docker-compose.yml`
3. `Dockerfile`

### `devcontainer.json`

This is customised from the default Microsoft file. You can change the name, the vscode settings (although if you're working with newer dotnet projects I'd recommend leaving `omnisharp.useModernNet` set to `true`), add any extensions you want to use in vscode for this project (find the ID for an extension by installing it in your local vscode, then right clicking it and clicking `Copy Extension ID` - I've included some of my favourites for aspdotnet projects but the only one you *need* is `ms-dotnettools.csharp`), the post-create commands and the remote user.

There is an issue where Omnisharp, the underlying technology in the C# vscode plugin doesn't work properly unless `dotnet restore` is run at the point of setting up the dev container, so do not remove that command.

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.231.6/containers/dotnet-postgres
{
    "name": "Dotnet 6 API, Postgres & SMTP",
    "dockerComposeFile": "docker-compose.yml",
    "service": "app",
    "workspaceFolder": "/workspace",

    // Set *default* container specific settings.json values on container create.
    "settings": {
		"omnisharp.useModernNet": true //Use the build of OmniSharp that runs on the .NET 6 SDK. This build requires that the .NET 6 SDK be installed and does not use Visual Studio MSBuild tools or Mono. It only supports newer SDK-style projects that are buildable with dotnet build. Unity projects and other Full Framework projects are not supported. 
	},

    // Add the IDs of extensions you want installed when the container is created.
    "extensions": [
		"formulahendry.auto-rename-tag",
		"ms-dotnettools.csharp",
		"EditorConfig.EditorConfig",
		"dbaeumer.vscode-eslint",
		"xabikos.JavaScriptSnippets",
		"PKief.material-icon-theme",
		"eg2.vscode-npm-script",
		"christian-kohler.path-intellisense",
		"esbenp.prettier-vscode",
		"gencer.html-slim-scss-css-class-completion"
	],

    // Use 'forwardPorts' to make a list of ports inside the container available locally.
    // "forwardPorts": [5000, 5001],

	// [Optional] To reuse of your local HTTPS dev cert:
	//
	// 1. Export it locally using this command:
	//    * Windows PowerShell:
	//        dotnet dev-certs https --trust; dotnet dev-certs https -ep "$env:USERPROFILE/.aspnet/https/aspnetapp.pfx" -p "SecurePwdGoesHere"
	//    * macOS/Linux terminal:
	//        dotnet dev-certs https --trust; dotnet dev-certs https -ep "${HOME}/.aspnet/https/aspnetapp.pfx" -p "SecurePwdGoesHere"
	// 
	// 2. Uncomment these 'remoteEnv' lines:
	//    "remoteEnv": {
	// 	      "ASPNETCORE_Kestrel__Certificates__Default__Password": "SecurePwdGoesHere",
	//        "ASPNETCORE_Kestrel__Certificates__Default__Path": "/home/vscode/.aspnet/https/aspnetapp.pfx",
	//    },
	//
	// 3. Next, copy your certificate into the container:
	//      1. Start the container
	//      2. Drag ~/.aspnet/https/aspnetapp.pfx into the root of the file explorer
	//      3. Open a terminal in VS Code and run "mkdir -p /home/vscode/.aspnet/https && mv aspnetapp.pfx /home/vscode/.aspnet/https"

    // Use 'postCreateCommand' to run commands after the container is created.
    "postCreateCommand": "dotnet restore", // had to use otherwise omnisharp wouldn't work properly, so no intellisense in vscode

    // Comment out to connect as root instead. More info: https://aka.ms/vscode-remote/containers/non-root.
    "remoteUser": "vscode"
}
```

### `docker-compose.yml`

The docker compose file sets up the three components of this example project:

1. The dotnet 6 API
2. The PostgreSQL database
3. The SMTP server

Take note of the name of the SMTP server container as this will be needed later, as will the environment variables for the database. These should be changed from `postgres`.

```yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        # Update 'VARIANT' to pick a version of .NET: 3.1, 5.0, 6.0
        VARIANT: "6.0"
        # Optional version of Node.js
        NODE_VERSION: "lts/*"

    volumes:
      - ..:/workspace:cached

    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity

    # Runs app on the same network as the database container, allows "forwardPorts" in devcontainer.json function.
    network_mode: service:db
    
    # Uncomment the next line to use a non-root user for all processes.
    # user: vscode

    # Use "forwardPorts" in **devcontainer.json** to forward an app port locally. 
    # (Adding the "ports" property to this file will not forward from a Codespace.)

  db:
    image: postgres:14.1
    restart: unless-stopped
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: postgres
      
    # Add "forwardPorts": ["5432"] to **devcontainer.json** to forward PostgreSQL locally.
    # (Adding the "ports" property to this file will not forward from a Codespace.)
  
  mail:
    image: bytemark/smtp
    restart: unless-stopped

volumes:
  postgres-data:

```

### `Dockerfile`

This comes directly from Microsoft, I've made no modifications to it.

```dockerfile
# [Choice] .NET version: 6.0, 5.0, 3.1, 6.0-bullseye, 5.0-bullseye, 3.1-bullseye, 6.0-focal, 5.0-focal, 3.1-focal
ARG VARIANT="6.0"
FROM mcr.microsoft.com/vscode/devcontainers/dotnet:0-${VARIANT}

# [Choice] Node.js version: none, lts/*, 16, 14, 12, 10
ARG NODE_VERSION="none"
RUN if [ "${NODE_VERSION}" != "none" ]; then su vscode -c "umask 0002 && . /usr/local/share/nvm/nvm.sh && nvm install ${NODE_VERSION} 2>&1"; fi

# [Optional] Uncomment this section to install additional OS packages.
# RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
#     && apt-get -y install --no-install-recommends <your-package-list-here>

# [Optional] Uncomment this line to install global node packages.
# RUN su vscode -c "source /usr/local/share/nvm/nvm.sh && npm install -g <your-package-here>" 2>&1
```

### `appsettings.json` and `appsettings.Development.json`

To get the SMTP working, use these keys. If you're doing anything more fancy with `bytemark/smtp` than just the default functionality you may need to change the SMTP port.

```json
"SmtpHost": "mail",
"SmtpPort": 25,
```

To connect to the Postgres database set your connection string to `"User ID=postgres;Password=postgres;Host=db;Port=5432;Database=postgres;Pooling=true;"`, substituting in your chosen username, password and database name.
