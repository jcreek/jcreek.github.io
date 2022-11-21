---
tags:
  - development
  - nodejs
  - typescript
  - eslint
  - vscode
  - devcontainer
  - docker
---

# How to set up a Node.js Typescript project well

_2022-11-19_

This post covers how to set up a project with Node.js and Typescript using eslint and vscode devcontainers using Docker.

This project makes use of eslint with airbnb styling and typescript support.

All linting errors can be automatically fixed (usually) by running `npm run fix` before you commit your changes.

## Getting it running locally for development

### Devcontainer + vscode

If you have `Visual Studio Code` and `Docker` installed then you can open this project using the `Remote - Containers` extension (ms-vscode-remote.remote-containers). Click the button at the bottom left of vscode, then 'Reopen in container'. You can now run the application using the vscode debugger.

### Standard local install

Running `npm run dev` will build the development version in the `dist` folder and enable hot reloading. It will also open the application in a new browser tab automatically.

Running `npm run build` will create the production assets in the `dist` folder.

## Devcontainer

In the `.devcontainer` folder you'll need three files.

### `base.Dockerfile`

```dockerfile
# [Choice] Node.js version (use -bullseye variants on local arm64/Apple Silicon): 18, 16, 14, 18-bullseye, 16-bullseye, 14-bullseye, 18-buster, 16-buster, 14-buster
ARG VARIANT=16-bullseye
FROM mcr.microsoft.com/vscode/devcontainers/javascript-node:0-${VARIANT}

# Install tslint, typescript. eslint is installed by javascript image
ARG NODE_MODULES="tslint-to-eslint-config typescript"
COPY library-scripts/meta.env /usr/local/etc/vscode-dev-containers
RUN su node -c "umask 0002 && npm install -g ${NODE_MODULES}" \
    && npm cache clean --force > /dev/null 2>&1

# [Optional] Uncomment this section to install additional OS packages.
# RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
#     && apt-get -y install --no-install-recommends <your-package-list-here>

# [Optional] Uncomment if you want to install an additional version of node using nvm
# ARG EXTRA_NODE_VERSION=10
# RUN su node -c "source /usr/local/share/nvm/nvm.sh && nvm install ${EXTRA_NODE_VERSION}"

```

### `devcontainer.json`

```json linenums="1"
// For format details, see https://aka.ms/devcontainer.json. For config options, see the README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.238.0/containers/typescript-node
{
	"name": "Node.js & TypeScript",
	"build": {
		"dockerfile": "Dockerfile",
		// Update 'VARIANT' to pick a Node version: 18, 16, 14.
		// Append -bullseye or -buster to pin to an OS version.
		// Use -bullseye variants on local on arm64/Apple Silicon.
		"args": { 
			"VARIANT": "16-bullseye"
		}
	},

	// Configure tool-specific properties.
	"customizations": {
		// Configure properties specific to VS Code.
		"vscode": {
			// Add the IDs of extensions you want installed when the container is created.
			"extensions": [
				"dbaeumer.vscode-eslint"
			]
		}
	},

	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	"forwardPorts": [3000],

	// Use 'postCreateCommand' to run commands after the container is created.
	"postCreateCommand": "npm install",

	// Comment out to connect as root instead. More info: https://aka.ms/vscode-remote/containers/non-root.
	"remoteUser": "node"
}

```

### `Dockerfile`

One important thing to note in this file is the installation of `inotify-tools` to the underlying Debian OS, as this is required for nodemon to be able to poll and do cold reloading.

```dockerfile
# [Choice] Node.js version (use -bullseye variants on local arm64/Apple Silicon): 18, 16, 14, 18-bullseye, 16-bullseye, 14-bullseye, 18-buster, 16-buster, 14-buster
ARG VARIANT=16-bullseye
FROM mcr.microsoft.com/vscode/devcontainers/typescript-node:0-${VARIANT}

# [Optional] Uncomment this section to install additional OS packages.
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install --no-install-recommends inotify-tools

# [Optional] Uncomment if you want to install an additional version of node using nvm
# ARG EXTRA_NODE_VERSION=10
# RUN su node -c "source /usr/local/share/nvm/nvm.sh && nvm install ${EXTRA_NODE_VERSION}"

# [Optional] Uncomment if you want to install more global node packages
# RUN su node -c "npm install -g <your-package-list -here>"

```

## Visual Studio Code Debugging

Within the `.vscode` folder you'll need to create this `launch.json` file:

```json linenums="1"
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch via NPM",
            "request": "launch",
            "runtimeArgs": [
                "run",
                "dev"
            ],
            "runtimeExecutable": "npm",
            "skipFiles": [
                "<node_internals>/**"
            ],
            "type": "pwa-node",
            "stopOnEntry": true,
        }
    ]
}
```

## Eslint

Create the following files:

### `.eslintignore`

Assuming you're using git, this will be the same as your `.gitignore` file.

```text
dist
node_modules
```

### `.eslintrc`

```json linenums="1"
{
    "root": true,
    "parser": "@typescript-eslint/parser",
    "parserOptions": {
        "ecmaVersion": "latest",
        "sourceType": "module"
    },
    "env": {
        "browser": true,
        "node": true
    },
    "plugins": [
        "@typescript-eslint"
    ],
    "extends": [
        "airbnb-base",
        "eslint:recommended",
        "plugin:@typescript-eslint/eslint-recommended",
        "plugin:@typescript-eslint/recommended",
        "plugin:import/errors",
        "plugin:import/warnings",
        "plugin:import/typescript"
    ],
    "rules": {
        "import/extensions": [
            "error",
            "ignorePackages",
            {
              "js": "never",
              "jsx": "never",
              "ts": "never",
              "tsx": "never"
            }
        ]
    }
}

```

## TS support, plus cold reloading

The structure for this application is to have a `src` folder that contains all `.ts` files. The files will be compiled to a `dist` folder for deploying to a server.

Create the following files:

### `nodemon.json`

```json linenums="1"
{
    "watch": ["src"],
    "ext": ".ts,.js",
    "ignore": [],
    "exec": "ts-node .src/index.ts"
}
```

### `tsconfig.json`

```json linenums="1"
{
    "compilerOptions": {
      "target": "es2016",
      "lib": ["es6"],
      "module": "commonjs",
      "rootDir": "src",
      "resolveJsonModule": true,
      "allowJs": false,
      "outDir": "dist",
      "esModuleInterop": true,
      "forceConsistentCasingInFileNames": true,
      "strict": true,
      "noImplicitAny": true,
      "strictNullChecks": true,
      "strictFunctionTypes": true,
      "strictPropertyInitialization": true,
      "noImplicitThis": true,
      "noUnusedLocals": true,
      "noUnusedParameters": true,
      "noImplicitReturns": true,
      "noFallthroughCasesInSwitch": true,
      "noImplicitOverride": true,
      "skipLibCheck": true
    }
}
```

### `package.json`

```json linenums="1"
{
  "name": "your app name",
  "type": "commonjs",
  "version": "1.0.0",
  "description": "",
  "scripts": {
    "dev": "nodemon",
    "lint": "eslint . --ext .ts",
    "fix": "eslint . --ext .ts --fix",
    "build": "rimraf ./dist && tsc",
    "start": "npm run build && node dist/index.js"
  },
  "repository": {
    "type": "git",
    "url": ""
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@types/node": "^18.11.9",
    "@typescript-eslint/eslint-plugin": "^5.43.0",
    "@typescript-eslint/parser": "^5.43.0",
    "eslint": "^8.28.0",
    "eslint-config-airbnb-base": "^15.0.0",
    "eslint-plugin-import": "^2.26.0",
    "nodemon": "^2.0.20",
    "rimraf": "^3.0.2",
    "ts-node": "10.9.1",
    "typescript": "^4.9.3"
  }
}
```
