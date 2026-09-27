---
title: "Spring Flyway"
date: 2022-04-30T16:37:58+02:00
draft: false
categories: ["Platform and reliability"]
---

## Introduction

Most of the services that we generally encounter in our daily job will simply take some input from users and populate database , and read from database and show it on UI. Each database has a schema, now it is rarely happens that we know each nitty gritty of our domain and come up with a perfect schema at day one. Most of the time we go thorugh an iterative process of modifying our schema as and when requirments change. Now all these migrations can reside separately to the application and can be handled by database teams but what if we can keep this migration script along side of our application in a versioned way and apply them on application startup? That’s what flyway does for us.

Flyway (<https://flywaydb.org/documentation/>) is an open source database migration tool that is used to migrate your database schema. It can be used standalone but it has nice integration with spring boot. Liquibase in another option for database migration most of the time we will see one of these two being used in production services.

Flyway supports a vast list of databases check in documentation for more details.

## Flyway Convention

Flyway favours simplicity and convention over configuration.

- Each flyway migration script file has the format of `V<version>__<description>.sql`
- If we want to undo a migration, we can put that script in a file names `U<version>__<description>.sql`
- Flyway also stores the checksum of file. So once a file has been applied the content cannot be changed.
- It tracks the migration history in a table named `flyway_schema_history` .

## Flyway with Spring Boot

To show case flyway with spring boot I am going to use our last `inventory-service`. So our intention is to start creating a schema and that should be applied by flyway.

To integrate flyway with spring boot we start with adding below dependency to our service.

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

We start by adding `spring-boot-starter-data-jpa` this gives us jpa and hibernate capabilities to our application.

As we are using here postgresql we need to add `postgresql` driver dependency.

And last but not the least we need to add `flyway-core` dependency.

Flyway uses spring datasource configuration to find out database uri and credentials. Let’s add the details of our database to our application.

For this blog I have created a db named inventory and a R/W user named `inventory_rw` . So our application config looks like as following.

``` yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/inventory
    username: inventory_rw
    password: '*****'
```

So till now flyway knows which database to connect with which credentials. Now we need to provide the migration script. We name our first migration script as `V1__init.sql` and it looks like as following-

``` sql
CREATE SCHEMA IF NOT EXISTS inv;

SET search_path TO inv,public;

CREATE TABLE IF NOT EXISTS products(
 id UUID PRIMARY KEY,
 product_name  VARCHAR(255) NOT NULL,
 stock NUMERIC NOT NULL default 0,
 manufacturer VARCHAR(255) NOT NULL
);
```

Now if we run the application, the application starts up. We see some logs like below -

     Successfully applied 1 migration to schema "public", now at version v1 (execution time 00:00.046s)

if we connect to our database we see the following -

![application-table](/application-table.png)

So there is our table . We also see the owner is our user .

if we check in public schema we can say the flyway_schema_history table is also created.

![flyway-schema-history](/flyway-schema-history.png)

And here is how it looks like -

![flyway-schema-content](/flyway-schema-content.png)

Now let’s try to change our migration script. We get below exception

    Caused by: org.flywaydb.core.api.exception.FlywayValidateException: Validate failed: Migrations have failed validation
    Migration checksum mismatch for migration version 1
    -> Applied to database : 2071614183
    -> Resolved locally    : 387884339. Either revert the changes to the migration, or run repair to update the schema history.

So once your schema is applied by default we cannot change the script. However if we want we can disable this check by `spring.flyway.validate-on-migrate=false`.

Let’s add a column to our table. To do this we need to add a new migration script with version number greater than previous version.

Let’s create a new file name `V2_created_on_column.sql` and add the following script.

``` sql
ALTER TABLE products ADD COLUMN created_on TIMESTAMP NOT NULL ;
```

On logs we see below lines -

     Current version of schema "public": 1
     Migrating schema "public" to version "2 - created on"
     Successfully applied 1 migration to schema "public", now at version v2 (execution time 00:00.051s)

if we check our table we can see the new column is added. ![flyway-created-on](/flyway-created-on.png)

## Introducing flyway in already existing database

If we already have a database that is not created using flyway and the public schema (or the schema in which we want our flyway-schema-history-table to be created) already has other tables we will see below exception.

    Found non-empty schema(s) "public" but no schema history table. Use baseline() or set baselineOnMigrate to true to initialize the schema history table.

Flyway basically refuses to migrate on a non empty database for which it does not have a schema history table. To overcome this issue, we need to provide a baseline rto flyway. a baseline is a way to tell flway that don’t care what happended till this version and apply any changes above this version. With spring config we achieve this by adding below property to our application.

``` yml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: '0'
```

Now here’s a small catch if you don’t provide which version to baseline the default value provided is 1. So here I am providing a custom value 0 so that my scripts like V1 onwards are applied. The schema history table will look somehting like below -

![flyway-baseline](/flyway-baseline.png)

We now have a new entry called `<<Flyway Baseline>>` in our schema history table.

## Flyway custom user

Till now in our configuration, flyway is using the user provided in spring datasource. However it may be possible that your application uses a read-only user . In this scenario to provide a different user to flyway we need to specify user for flyway.

The below config uses a separate R/W user to do the migration -

``` yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/inventory
    username: inventory_ro
    password: '******'
  flyway:
    baseline-on-migrate: true
    baseline-version: '0'
    user: inventory_rw
    password: '******'
```

Flyway integration of spring boot has a lot of properties to configure, for example we can define custom schema where flyway_schema_history table will be created, cherry pick migration script, default schema name etc. A list of these properties can be found <https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html>.

So that’s it from this post. We now know how to iterate over our database and apply migrations using flyway. In next post wit this knowledge I will show how to integrate a database with spring boot application.
