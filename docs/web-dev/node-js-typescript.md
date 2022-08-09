---
layout: post
parent: Web Development
nav_order: 4
title:  "How to set up a Node.js Typescript project well"
date:   2022-06-15 19:26:00 +0000
categories: nodejs typescript eslint webpack babel sourcemaps scss vscode devcontainer docker
---

# {{page.title}}

_{{page.date}}_

This post covers how to set up a project with Node.js and Typescript using eslint, webpack, babel, source maps, SCSS and vscode devcontainers using Docker.

This project makes use of eslint with airbnb styling and typescript support.

All linting errors can be automatically fixed (usually) by running `npm run lint -- --fix` before you commit your changes.

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

```json
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

One important thing to note in this file is the installation of `inotify-tools` to the underlying Debian OS, as this is required for webpack to be able to poll and do hot reloading.

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

```json
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

### `.eslintrc.json`

```json
{
    "env": {
        "browser": true,
        "es2021": true
    },
    "extends": [
        "airbnb-base",
        "plugin:import/errors",
        "plugin:import/warnings",
        "plugin:import/typescript"
    ],
    "parser": "@typescript-eslint/parser",
    "parserOptions": {
        "ecmaVersion": "latest",
        "sourceType": "module"
    },
    "plugins": [
        "@typescript-eslint"
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

## Webpack with SCSS and TS support, plus hot reloading

The structure for this application is to have a `src` folder that contains: all `.html` and `.ts`; `assets` folder for images; `styles` folder for `.scss` files. The files will be compiled to a `dist` folder for deploying to a web server.

Create the following files:

### `custom.d.ts`

```ts
declare module '*.svg' {
    const content: any;
    export default content;
}
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "preserveConstEnums": true
  },
  "include": ["src/**/*", "src/custom.d.ts"],
  "exclude": ["node_modules", "**/*.spec.ts"]
}
```

### `webpack.config.js`

```js
const path = require('path');

module.exports = {
  mode: 'development',
  entry: {
    bundle: path.resolve(__dirname, 'src/index.ts'),
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
    clean: true,
    assetModuleFilename: '[name].[contenthash][ext]',
  },
  devtool: 'source-map',
  devServer: {
    static: {
      directory: path.resolve(__dirname, 'dist'),
    },
    port: 3000,
    open: true,
    hot: true,
    compress: true,
    historyApiFallback: true,
  },
  watchOptions: { poll: true },
  module: {
    rules: [
      {
        // Use these loaders for any matching scss file types
        test: /\.scss$/,
        use: [
          'style-loader',
          'css-loader',
          'sass-loader',
        ],
      },
      {
        // Add backwards compatibility
        test: /\.(js|ts)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env', '@babel/preset-typescript'],
          },
        },
      },
      {
        // Add support for images
        test: /\.(png|svg|jpg|jpeg|gif)$/i,
        type: 'asset/resource',
      },
    ],
  },
  resolve: {
    // Specify the order in which to resolve files by their extension
    extensions: ['*', '.js', '.ts'],
  },
};
```

### `package.json`

```json
{
  "name": "your app name",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "build": "webpack",
    "dev": "webpack serve",
    "lint": "eslint --ignore-path .eslintignore --ext .js,.ts ."
  },
  "repository": {
    "type": "git",
    "url": ""
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@babel/core": "^7.18.5",
    "@babel/preset-env": "^7.18.2",
    "@babel/preset-typescript": "^7.17.12",
    "@typescript-eslint/eslint-plugin": "^5.28.0",
    "@typescript-eslint/parser": "^5.28.0",
    "babel-loader": "^8.2.5",
    "css-loader": "^6.7.1",
    "eslint": "^8.17.0",
    "eslint-config-airbnb-base": "^15.0.0",
    "eslint-plugin-import": "^2.26.0",
    "html-webpack-plugin": "^5.5.0",
    "sass": "^1.52.3",
    "sass-loader": "^13.0.0",
    "style-loader": "^3.3.1",
    "typescript": "^4.7.3",
    "webpack": "^5.73.0",
    "webpack-cli": "^4.10.0",
    "webpack-dev-server": "^4.9.2"
  }
}
```