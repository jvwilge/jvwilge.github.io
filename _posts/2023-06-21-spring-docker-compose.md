---
layout: default
title: Automatically spin up a Docker Compose environment for your local and integration test environment with Spring Boot 3.1.0+
exerpt: 
date: 2023-06-21 06:38:02
tags : java spring docker
categories : en
---
Automatically spin up a Docker Compose environment for your local and integration test environment with Spring Boot 3.1.0+
===

With the release of Spring Boot 3.1.0 it is possible to automatically spin up a Docker Compose environment to run the application against locally or use it as an integration test environment. All the datasource properties are set automatically by Spring, even if you're using a random port.

<!--more-->

Using Docker Compose for your application would be a very short article, so here I'll show you how to use the same environment for your integration tests using multiple database schema's. 
All the examples are in Maven, but are also applicable to Gradle, there are no plugins involved.
I also assume `docker` and `docker compose` are available on your command line.
I'll use a Postgresql database in this example, but the possibilities are endless (see [Service Connections](https://docs.spring.io/spring-boot/docs/3.1.0/reference/html/features.html#features.docker-compose.service-connections)).

## Setup your project and sample code

I created a project from scratch to demonstrate what's happening. It's [a default Maven Spring Boot 3.1.0 project with Spring Web, Spring Data  Jdbc and Docker Compose Support](https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.1.0&packaging=jar&jvmVersion=17&groupId=io.github.jvwilge&artifactId=compose-demo&name=compose-demo&description=Demo%20project%20for%20Spring%20Boot&packageName=io.github.jvwilge.compose-demo&dependencies=web,data-jdbc,docker-compose)

All the code in this article is available on [Github](https://github.com/jvwilge/spring-boot-docker-compose-demo).

To add Docker Compose support to an existing project add the following dependency (you can skip this step if you're starting from the generated project):
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-docker-compose</artifactId>
  <scope>runtime</scope>
  <optional>true</optional>
</dependency>
```

Now we can add the Docker Compose configuration. Fill `compose.yaml` with :
```yaml
version: "3.8"

services:
  postgresql:
    image: 'postgres:15.3-bullseye'
    container_name: 'demo-postgres'
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=demodb
    ports:
      - "5432"
    user: root
    volumes:
      - 'pg_data:/var/lib/postgresql/data'
      - './src/test/resources/db:/docker-entrypoint-initdb.d'

volumes:
  pg_data:
```

This uses a default Postgresql docker image, bound to a random port. Usually a random port can get tricky, but Spring detects the port and sets the properties automatically. It's even possible to use more exotic images, as long as the image names conforms to a certain string ("postgres" in this case).
`pg_data` is the data directory and I'll explain the `docker-entrypoint-initdb.d` later.

The final thing to do is adding a Postgresql driver to your `pom.xml`:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.6.0</version>
</dependency>
```

This is basically it. With this file your Spring application spins up the Docker Compose environment and sets all the datasource properties automatically!

# Running and verifying

Now start your application (i.e. `ComposeDemoApplication`) and you'll notice the `compose.yaml` is detected and started:

```
o.s.b.d.c.l.DockerComposeLifecycleManager : Using Docker Compose file '/Users/jeroen/compose-demo/compose.yaml'
```

A call to `docker ps` will result in something like this:

```
CONTAINER ID   IMAGE                    COMMAND                  CREATED             STATUS             PORTS                     NAMES
43577912f3e5   postgres:15.3-bullseye   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:64481->5432/tcp   demo-postgres
```

This Postgresql runs on port 64481 and you can connect to it by using the username/password `postgres`/`password`.

If you stop the application the Docker Compose environment will also shut down.

You can also start the Docker Compose environment via `docker compose up` and most of the things will work the same. Spring will detect your running Docker Compose environment, but it won't shut down the Docker Compose enivironment afterwards (which is good if you want to keep it running, more on this later).

## Initialize test data

To have some test data available you can add sql files to `src/test/resources/db`. These files will be detected at startup of the Postgresql container (this is a feature of the Postgresql image, most other database images have a similar freature). Another thing you can add here is your database snapshot (e.g. if you're using [Flyway](https://flywaydb.org/) or [Liquibase](https://www.liquibase.org/)) to speed up startup.

## Run your (integration) tests with Docker compose

If you also want to use you (integration) test environment with Docker Compose you need to change one property in `src/test/resources/application-test.properties`:

```properties
spring.docker.compose.skip.in-tests=false
```

## Keep the Docker Compose environment running

To keep your environment running set the property `spring.docker.compose.lifecycle-management` to `start-only`. The default property `start-and-stop` stops the environment after finishing.
Keeping the environment running makes subsequent startups faster since it's already running.

If you also use it for your test environment it makes turnarond time much faster and allows you to inspect the database after your tests are finished.

However, this is something you don't want on your CI environment so I advice you to include this as en environment variable:

```shell
export SPRING_DOCKER_COMPOSE_LIFECYCLE-MANAGEMENT=start-only
```

As an alternative you can manually start Docker Compose via `docker compose up`, this also keeps the environment running.

## How to use different schema's for your local and test environment.

Of course it would be nice to have separate databases for each environment. You can (kind of) achieve this by using multiple schema's. Do this by adding a property file to your property file (e.g. application.properties):

```properties
spring.datasource.hikari.schema=local
```

Thank you Boudewijn for asking the right questions during my research, otherwise I would've missed this property.

Don't forget to create separate schema's in your sql init files or you might end up with obscure error messages like:
```
org.postgresql.util.PSQLException: ERROR: relation "car" does not exist
```


## Using multiple Docker Compose environments

As an alternative you can use multiple `compose.yaml` files. There are some arguments for this (e.g. cleaning up the test environment, but keeping your local data). 
I like to use a file called `compose-base.yaml` and [extend from there](https://docs.docker.com/compose/extends/). Make sure to set the `name` of your Docker Compose environments (to different values), otherwise the directory name is used and Docker Compose gets confused.

Now set a property in the property file for your enviroment (e.g. `application-test.properties`):

```properties
spring.docker.compose.file=compose-test.yaml
```

A next step would be to have a volume for your test environment in your `target` directory so it gets cleaned up with a `mvn clean`. Especially with Flyway or Liquibase changesets you might want to clean up your volumes a bit more often.


## Cleaning up volumes the right way

Volumes and networks are [never cleaned](https://docs.docker.com/engine/reference/commandline/compose_down/#:~:text=Networks%20and%20volumes%20defined%20as%20external%20are%20never%20removed.) by default. To clean a volume use:

```shell
docker compose down --volumes
```

The volume used in this example is called `compose-demo_pg_data`. You can look this up by calling `docker volume ls`.

If you're dealing with a lot of breaking database changes it's good to know how to clean up volumes.
If you see a message:

>```PostgreSQL Database directory appears to contain a database; Skipping initialization```

and don't expect this it might also be caused by an incorrect cleanup.


# Conclusion

This new Spring feature is pretty powerful. I didn't dive into the details of what happens if you have multiple Postgresql images, don't use Hikari for connection pooling or other edge cases, but for the straightforward cases (and let's be honest, most of them should be) it saves us a lot of time.

# Links

- [Spring Docker Compose](https://docs.spring.io/spring-boot/docs/3.1.0/reference/html/features.html#features.docker-compose)
- [Spring Docker Compose properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#appendix.application-properties.docker-compose)
- [Docker Compose v3 Reference](https://docs.docker.com/compose/compose-file/compose-file-v3/)
- [StackOverflow: "docker-compose and create db in Postgres on init"](https://stackoverflow.com/questions/59715622/docker-compose-and-create-db-in-postgres-on-init)
- [Do you still need testcontainers with Spring Boot 3.1?](https://softwaremill.com/do-you-still-need-testcontainers-with-spring-boot-3-1/#new-spring-boot-docker-compose-module-in-action)


> First published on June 21, 2022 at [jvwilge.github.io](http://jvwilge.github.io)
