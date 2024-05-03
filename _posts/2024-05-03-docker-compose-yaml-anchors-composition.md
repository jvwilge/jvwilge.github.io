---
layout: default
title: How to use YAML anchors as a template for Docker Compose services - also fixes `depends_on cannot be extended`
exerpt: 
date: 2024-05-03 08:02:14
tags : docker
categories : en
---
How to use YAML anchors as a template for Docker Compose services - also fixes `depends_on cannot be extended`
===

After getting the `services with 'depends_on' cannot be extended` error I started looking for a solution that also better reflects that we're actually cloning a service and changing some parameters instead of a 'is a' relation.

<!--more-->

I will demonstrate the solution using an OpenSearch 2 service since it has some snags you might also encounter in the real world, but you can apply this to most services. Our starting point is the [Sample Docker Compose file by OpenSearch](https://opensearch.org/docs/latest/install-and-configure/install-opensearch/docker/#sample-docker-compose-file-for-development).

We will use that example as a base and add a second service. The most naïve way would be to copy/paste, but that'll make quite a short article.

## Using extends and depends_on simultaneously

To add a second service the easiest step is to ues the [extends attribute](https://docs.docker.com/compose/multiple-compose-files/extends/) and change some parameters in the extended service. Technically you're not extending here, but in many cases you can get away with it. In our case we also used the [depends_on attribute](https://docs.docker.com/compose/compose-file/05-services/#depends_on) and after a recent (early 2024) Docker Compose upgrade it gave the `depends_on cannot be extended` error.

So why did you get `services with 'depends_on' cannot be extended`? The  [Docker Compose documentation](https://docs.docker.com/compose/compose-file/05-services/#restrictions:~:text=%2C-,depends_on,-.) explains the reasoning behind this not working. If you want more details search for the error and a lot of issues will pop up (with thorough explanations).

In [GitHub issue 1988](https://github.com/docker/compose/issues/1988) I found quite some pieces to create to a proper solution.

The proper solution would something in the direction of an abstract/base service or a template. In the actual `services` section we create 'instances' of the base service and overwrite/add some parameters.

## The difference between `extends` and an extension

One of the reasons it took me a while to find a fitting solution is that I thought `extends` and extension were the same thing.

[The `extends` attribute](https://docs.docker.com/compose/compose-file/05-services/#extends) is used to share configuration (even across files).
[An extension](https://docs.docker.com/compose/compose-file/11-extension/) is used to modularize configuration.

Well that still sounds the same, doesn't it? In the next paragraph things will become more clear. In my head I pretend that an extension is a template you can define as a top level element (which is normally not allowed by the YAML schema of Docker Compose) for later re-use .

## Defining the service template as an extension

Defining a top level element is allowed though if you prefix it with `x-`. This element is ignored by Docker Compose, which is exactly what we want since we want to fill in the details later. As you might've guessed by now the `x-` stands for [extension](https://docs.docker.com/compose/compose-file/11-extension/).

To use this template somwhere in the (same) YAML we can use what I call the programmatic copy/paste of YAML: [YAML anchors](https://support.atlassian.com/bitbucket-cloud/docs/yaml-anchors/). As always, be careful with the spaces here!

As a simple first step let's only pull out the image name to an extension:

```yaml
x-opensearch-service: &opensearch-service
  image: opensearchproject/opensearch:2.12.0

services:
  opensearch-node1:
    <<: *opensearch-service
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
#... rest of file cut away for brevity ...#
``` 

Instead of running `docker compose up` and wait for errors I'd recommend using `docker compose config`, this will show the [canonical compose.yaml](https://docs.docker.com/reference/cli/docker/compose/config/). The fact that something works isn't a guarantee that it is correct or has unwanted side effects.

As you can see the image is now inside `services.opensearch-node1`.

All right, the base is set up.

## How to circumvent sequences for environment variables

The first snag you'll probably hit is that you want to override environment variables (or at least add new ones besides the common ones). Unfortunately this is a YAML sequence, [where merging is not allowed](https://docs.docker.com/compose/compose-file/11-extension/#:~:text=can%27t%20be%20used%20with%20sequences.). 

Luckily there is an escape hatch in this specific case, [the `env_file` attribute](https://docs.docker.com/compose/compose-file/05-services/#env_file). Here you can specify a file with environment variables. The values from (different) `env_file`s are combined with `environment` to `environment`.  It feels a bit like cheating, but on the other hand our `compose.yaml` becomes a bit more readable.

Create a file called `compose-opensearch.env` and add the following values:
```bash
discovery.type=single-node
bootstrap.memory_lock=true
OPENSEARCH_JAVA_OPTS="-Xms512m -Xmx512m"
DISABLE_INSTALL_DEMO_CONFIG=true
DISABLE_SECURITY_PLUGIN=true
```

Replace the earlier extension with:
```yaml
x-opensearch-service: &opensearch-service
  image: opensearchproject/opensearch:2.12.0
  env_file: compose-opensearch.env
```

And you can remove the redundant environment values from the `opensearch-node1` service.

Now check `docker compose config` and verify that the enviroment section of `opensearch-node1` looks correct.

# Summary

To summarize:

First create an extension that acts as a base template. Then create actual services where you override/add the attributes to the template using YAML anchors (except sequences!). `docker compose config` saves a lot of debugging time.

# Links

Here are some links that didn't make it to the article but might still be helpful. 

- [stackoverflow - What is the << (double left arrow) syntax in YAML called, and where's it specified?](https://stackoverflow.com/questions/41063361/what-is-the-double-left-arrow-syntax-in-yaml-called-and-wheres-it-specifi)
- [Github Issue 3320 - services with 'depends_on' cannot be extended](https://github.com/docker/compose/issues/3220)
- [Medium - Don’t Repeat Yourself with Anchors, Aliases and Extensions in Docker Compose Files](https://medium.com/@kinghuang/docker-compose-anchors-aliases-extensions-a1e4105d70bd)

> First published on May 3, 2024 at [jvwilge.github.io](http://jvwilge.github.io)