+++
title = 'Javalin & Flyway'
date = 2023-04-11T21:48:22+01:00
+++

## Motivation
Some time ago, while working on a college assignment using Javalin, I encountered some labs that required running SQL statements manually on the database before deploying the application to [Heroku](https://www.heroku.com/). This was a tedious task that had to be repeated every time there were changes or additions to the tables. Although I knew about Flyway and its ability to automate database migrations, I couldn't find any tutorial on the [Javalin site](https://javalin.io/tutorials/) that explained how to use it. Therefore, I decided to write a short blog post that could be helpful to anyone facing a similar situation.

## Introduction
[Javalin](https://javalin.io/) is a lightweight web framework for Java and Kotlin that aims to simplify web development by providing a concise and intuitive API. It is built on top of the popular Jetty web server and includes features such as routing, request and response handling, websockets, and JSON serialization.

[Flyway](https://flywaydb.org/) is a database migration tool that enables developers to version and automate their database changes using simple SQL scripts. It supports a wide range of databases, including PostgreSQL, MySQL, Oracle, and SQL Server, and integrates seamlessly with popular build tools such as Maven and Gradle.

In this article, we will explore how to use Flyway with Javalin to automate database migrations and streamline the development process. Specifically, we will focus on how to set up and configure Flyway within a Javalin application, and how to write and apply database migration scripts using Flyway's API. Whether you are a seasoned Java developer or just starting out with Javalin, this guide will provide you with the knowledge and tools you need to manage your database changes efficiently and effectively.

## Steps
1. Add Flyway to your build tool by adding the following Maven dependency to your `pom.xml` file:
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>9.16.3</version>
</dependency>
```
If you are using Gradle, add the following to your `build.gradle` file:
```
implementation 'org.flywaydb:flyway-core:9.16.3'
```

2. Create a `flyway/migrations` directory in your project's `src/main/resources` folder. This directory will contain all your database migration scripts.

3. Create an example migration script, such as `V1_0__create_user_schema.sql`, in the `flyway/migrations` directory. The script should contain the SQL statements to create the required database tables, columns or data. For example:
```sql
CREATE TABLE users (
    id serial PRIMARY KEY,
    name VARCHAR (100) UNIQUE NOT NULL,
    email VARCHAR (255) UNIQUE NOT NULL
);
```

4. Run Flyway to migrate the database schema before connecting to the database. Here is an example Kotlin code that does this:
```kotlin
// Set up the database connection details
val pg = EmbeddedPostgres.start()

val dbUrl = pg.getJdbcUrl("postgres", "postgres")
val dbUser = "postgres"
val dbPassword = "postgres"

// Run Flyway to migrate the database schema
val flyway = Flyway.configure()
    .dataSource(dbUrl, dbUser, dbPassword)
    .locations("filesystem:src/main/resources/flyway/migrations")
    .load()
flyway.migrate()

// Connect to the database
Database.connect(dbUrl, "org.postgresql.Driver", dbUser, dbPassword)
```
This function configures Flyway to use the given database URL, username, and password, and to look for migration scripts in the `flyway/migrations` directory. It then runs the migration using the `flyway.migrate()` method, which applies any pending migration scripts to the database. Finally, it connects to the database using the `Database.connect()`.

5. When Flyway is properly configured and integrated into the Javalin application, the following logs will be generated at application startup. These logs confirm that Flyway has validated the migration files, created the schema history table, and migrated the database schema to the target version without any issues. This serves as an indication that the database migration process has been executed successfully, and the Javalin application is ready to use.
```log
23:08:02.089 [main] INFO  o.f.core.internal.command.DbValidate - Successfully validated 1 migration (execution time 00:00.008s)
23:08:02.096 [main] INFO  o.f.c.i.s.JdbcTableSchemaHistory - Creating Schema History table "public"."flyway_schema_history" ...
23:08:02.112 [main] INFO  o.f.core.internal.command.DbMigrate - Current version of schema "public": << Empty Schema >>
23:08:02.115 [main] INFO  o.f.core.internal.command.DbMigrate - Migrating schema "public" to version "1.0 - create user schema"
23:08:02.125 [main] INFO  o.f.core.internal.command.DbMigrate - Successfully applied 1 migration to schema "public", now at version v1.0 (execution time 00:00.016s)
```

## Source Code
A sample project can be found at my GitHub [here](https://github.com/KevFan/javalin-flyway) for reference. The project uses Maven as the build tool and an embedded [Postgres](https://www.postgresql.org/) database as the database connection.


## Conclusion
In conclusion, using Flyway with Javalin can greatly simplify the database migration process and save developers time and effort. With Flyway, you can version and automate your database changes using simple SQL scripts and integrate seamlessly with popular build tools such as Maven and Gradle. In this article, we have explored the steps required to use Flyway with Javalin, including how to add Flyway to your build tool, create migration scripts, and configure and run Flyway within a Javalin application. I hope that this guide has been helpful in getting you started with using Flyway and Javalin together and that it will help you manage your database changes efficiently and effectively in your future projects.