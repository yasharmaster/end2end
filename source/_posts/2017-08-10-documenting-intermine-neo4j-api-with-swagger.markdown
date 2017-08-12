---
layout: post
title:  "Documenting InterMine-Neo4j API with Swagger UI"
date:   2017-08-10 08:00:00
comments: true
categories: gsoc
tags:
    - intermine
    - neo4j
    - gradle
    - gsoc
    - jersey
    - swagger
---

This post talks about InterMine-Neo4j API and its Swagger integration.

## Background

In the past blog posts I described that we have decided to query InterMine Neo4j Graph using Cypher and how I am going to convert PathQuery to Cypher. To expose the Neo4j Query Service, it was decided to keep the API endpoints and response *same as the InterMine API service* so that BlueGenes can be configured to work with it easily at later stages.

## Swagger UI

[Swagger UI](https://swagger.io/swagger-ui/) is a popular API documenting tool using which anyone can *visualize and interact with your API’s resources* without having any of the implementation logic in place. It’s *automatically generated from your Swagger specification*, with the visual documentation making it easy for back end implementation and client side consumption.

## InterMine-Neo4j API

The API is developed using [Jersey](https://jersey.github.io/) framework which provides support for JAX-RS APIs in Java. The code can be found is *intermine-neo4jwebapp* sub-project in the [intermine/neo4j](https://github.com/intermine/neo4j) repository. 

You can play with the InterMine-Neo4j API via Swagger UI at [intermine-neo4jwebapp.herokuapp.com](http://intermine-neo4jwebapp.herokuapp.com/). There are two endpoints in the API so far,

- `query/cypher` - This returns the Cypher equivalent of any Path Query.

- `query/results` - This endpoint returns the results of a Path Query by converting it into Cypher and running that Cypher against InterMine-Neo4j Graph.

## Integrating API with Swagger

Documenting InterMine-Neo4j API with Swagger involves the following steps.

#### Set up swagger-core in the project.

I used [Swagger Core Jersey 2.X Project Setup 1.5](https://github.com/swagger-api/swagger-core/wiki/Swagger-Core-Jersey-2.X-Project-Setup-1.5) guide to add swagger-core to the projects. The information given in this guide is as per Maven projects. As we use Gradle as the build tool at InterMine, I had to do add dependencies differently. The other steps remain more or less the same.

#### Add Annotations to the Code

The next step is to add annotations to the API code. At [Swagger-Core Annotations](https://github.com/swagger-api/swagger-core/wiki/Annotations-1.5.X) page you can find the documentation of all the annotations available with swagger-core.

#### Preview swagger.json

If the above two steps are executed properly, you should be able to see swagger.json generated for your code at `{swagger.api.basepath}/swagger.json`. For example, in our case, the swagger.json is generated at [http://intermine-neo4jwebapp.herokuapp.com/service/swagger.json](http://intermine-neo4jwebapp.herokuapp.com/service/swagger.json).

#### Add Swagger UI code to your project

The next step is to download the Swagger UI code from the *dist* folder in the [swagger/swagger-ui](https://github.com/swagger-api/swagger-ui) repository and place it in the `webapp` folder of the project.

#### Configuring Swagger UI

Among all the files that are downloaded in the previous step, we need to configure `index.html`. We need to update the `url` property in the `<script>` tag at the end of this file by pasting the link to our generated `swagger.json` file.

After following all the steps properly, you should be able to see the generated Swagger Documentation of the API. :)

Feel free to checkout Swagger UI of InterMine Neo4j API at [http://intermine-neo4jwebapp.herokuapp.com/](http://intermine-neo4jwebapp.herokuapp.com/) and stay tuned for more updates.