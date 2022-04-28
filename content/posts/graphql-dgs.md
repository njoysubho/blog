---
title: "Graphql Dgs"
date: 2022-03-16T21:13:55+01:00
draft: true
---

In this blog we will see how to use Netflix DGS library with spring boot to create a GraphQL server.
Graphql has gained popularity in recent years and with that we have now several libraries that has been built following the specifications.
Netlfix DGS library provides a comprehensive set of features to create a GraphQL server.

## Use Case
In our use case we have two objects `Author` and `Book`. The Graphql schema looks like below-

![graphql-dgs](/static/graphqlschema.png)

We have two queries
* bookById - returns a book by its id
* books - returns all books


