---
layout: post
parent: Development Teams
nav_order: 2
title:  "How to write an excellent readme file"
date:   2022-03-16 19:01:03 +0000
categories: git readme
---

# {{page.title}}

_{{page.date}}_

All git projects need a readme file, to explain what it is, why it exists, how it works and how to use it. A good starting point with guidance can be found below.

```md
# Project title

A little info about the application and an overview that explains **what** the application is for.

## Motivation

A short description of the motivation behind the creation and maintenance of the application. This should explain **why** the application exists.

## How the dev/staging/prod versions are built

This should be a brief explanation of how the application gets to where it's meant to be used. Typically this will involve linking to build pipelines in Jenkins/Azure/etc.

## Code style

Usually this would just link to the documentation for your established code style. In the event that the application uses additional or different styles or linting tooling this can be detailed here.

## Screenshots

Include some logos and demos screenshots to briefly give a rough idea of the application.

## Tech/framework used

Hopefully this will be identical across most of your organisation's applications, but the value of this section really becomes clear when an application does differ, and developers can see this at a glance here, without having to dig into the code or run the project locally to work it out.

## Features

What are the key features of this application? A few bullet points are ideal here.

## Getting it running locally for development

Provide a step by step series of examples and explanations about how to get a development environment running. If it is dependent on other local environmental factors, for example a specific database, this should also be covered here. This should be a single source of truth for getting this application running, without being dependent on any external sources of knowledge, even if that means there is some duplication across projects within this readme file.

## API Reference

If it's appropriate, a link should be included here to the API documentation, for example OpenAPI docs or a Swagger file.

## Tests

Describe and show how to run the tests with basic examples.

## How to use as a user

A brief guide on how to use the application AS A USER should be included here. A developer picking up the application should be able to find their way around it quickly as a user based on the guidance included here, to be able to sufficiently understand how most parts of the application are used.

## Anything else that seems useful

If there is anything else that seems useful to include in the readme file, this should be included here. Documentation of specific parts of the code SHOULD NOT live in this readme file, instead they should be in external documentation such as Confluence or in the code itself, whichever is more appropriate. References to such documentation could be included in this readme file for significant examples, but largely is not necessary.
```
