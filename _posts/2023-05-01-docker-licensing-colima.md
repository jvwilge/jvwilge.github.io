---
layout: default
title: How to get around Docker licensing on Mac with Colima
exerpt: 
date: 2023-05-01 05:44:19
tags : docker
categories : en
---
How to get around Docker licensing on Mac with Colima
===

In this article I'll show you how to get around the Docker desktop licensing issues by using Colima on a Mac. Note that this is completely by the rules and you might even consider to still purchase a license because it's not a very smooth ride.

<!--more-->

# Our use case

In our project we're planning to use Docker and Docker Compose, so I will focus on how to get these two working. We use Docker compose via a Maven plugin : [fabric8io/docker-maven-plugin](https://dmp.fabric8.io/) or with [Testcontainers](https://www.testcontainers.org/) solution. The article will keep things generic so you can also apply it to many other use cases. 

# So when do you need a license?

So word around the campfire was that you need a license for Docker. To be more precise: you need a license for Docker Desktop. For me it was a bit unclear what consists of Docker Desktop and when you exactly need a license. A good read is 
[Looking for a Docker Alternative? Consider This.](https://www.docker.com/blog/looking-for-a-docker-alternative-consider-this/)

The main point you're probably interested in is this :
> That means all the binaries (Docker Engine, Docker Daemon, Docker CLI, Docker Compose, BuildKit, libraries, etc) and anything open source continues to be free of charge.

So that means that if you're able to get Docker and Docker Compose working standalone you're staying within the boundaries of what is allowed.

Fortunately this is possible, with some small caveats.

# Colima to the rescue

The tool to replace Docker Desktop is called [Colima](https://github.com/abiosoft/colima). I'm aware this is a gross oversimplification, but it's quite a high layer cake and will make this article too complicated. If you want to know more I suggest starting with [their FAQ](https://github.com/abiosoft/colima/blob/main/docs/FAQ.md)).

# Install Docker, Docker Compose and Colima

Technically you can get the job done without Colima, but by only installing Docker and Docker Compose there is quite some manual labour involved. By using Colima as the glue, much of this work can be skipped.

To install everything via brew:

```bash
brew install docker docker-compose colima
```

Now start Colima

```bash
colima start
```

If you forget this step Testcontainers will complain with
>```Could not find a valid Docker environment. Please see logs and check configuration```

or the Maven plugin will complain with 

>```Cannot create docker access object```


# Prevent `DOCKER_HOST` error

The next step is add `DOCKER_HOST` as an environment variable (`~/.bashrc` or `~/.zshrc`):
```bash
export DOCKER_HOST="unix://${HOME}/.colima/docker.sock"
```

If you don't add this last step you'll get an error that looks like :
>```No <dockerHost> given, no DOCKER_HOST environment variable, no read/writable '/var/run/docker.sock' or '//./pipe/docker_engine' and no external provider like Docker machine configured```

When using the fabric8 plugin you're also able to set the `<dockerHost>` as suggested, but this environment variable is used by other tools too, so setting the environment variable has my preference.

## Prevent 'Could not start container' when using Testcontainers

If you're using Testcontainers also add :
```bash
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE="/var/run/docker.sock"
```

This will prevent errors like: 
> ```INFO - Container testcontainers/ryuk:0.4.0 is starting: 1e617402f1fdbeecfb1bb9b8c09410ff3a9c623f61a57d0aa7153d46bd12b016```
> ```ERROR - Could not start container```


# Fix credential helper error

You might run into an error with the credential helper:
> ```Error getting the credentials for https://index.docker.io/v1/ from the configured credential helper [Failed to start 'docker-credential-desktop get' : Cannot run program "docker-credential-desktop": error=2, No such file or directory]```

I'm not sure whether you'll always get this error or only if you previously had Docker Desktop installed. But I found a fix on [this Stackoverflow page](https://stackoverflow.com/questions/61221890/docker-for-mac-cannot-run-program-docker-credential-desktop)

I had to change `credsStore` to `credStore` in `~/.docker/config.json` (note the removal of the lower case `s` before the capital `S`).

## Fix ownership errors (permission denied)

After this I ran into an error with the `postgresql:15.2` image:

> ```chown: changing ownership of '/var/lib/postgresql/data': Permission denied```

I solved it by defining a volume in the root-section of the `compose.yaml` (or `docker-compose.yaml`) instead of under the service.

I'm still a bit in the dark here on the exact mechanics here and hope to improve this explanation, but it might help you further in a relatively clean way.

# Fix chown errors

Another thing I ran into was :
>```Unable to start container id [08763523c32c] : {"message":"error while creating mount source path '/Users/harrie/sandbox-maven/target/docker/db/data': chown /Users/harrie/sandbox-maven/target/docker/db/data: permission denied"} (Internal Server Error: 500) -> [Help 1]```

Changing the volume mentioned before also helps here, but it can also be solved by creating the path in advance.

# docker compose vs. docker-compose

A final issue that needs further investigation from my side is the `docker compose` vs `docker-compose`. The one with the space is the preferred way nowadays, but it doesn't seem to work with Colima.

[This page](https://stackoverflow.com/questions/66514436/difference-between-docker-compose-and-docker-compose) on StackOverflow gives some insights in what is going on.


# Conclusion

You can make most things work with Colima, but you really should ask yourself whether it's worth all the hassle for a few company dollars a month. Getting funding is probably the biggest hurdle, not the amount.

Things like volumes work just a tad smoother with Docker Desktop. For now it is just a tool to make testing easier, so no need to dive into the deep details (because the volume issue is most likely also due to a lack of knowledge on my side).


# Links
- [Demystifying Docker Volumes for Mac and PC Users](https://gist.github.com/onlyphantom/0bffc5dcc25a756e247cb526c01072c0)
- [Using volumes in Docker Compose](https://devopscell.com/docker/docker-compose/volumes/2018/01/16/volumes-in-docker-compose.html)


> First published on May 1, 2022 at [jvwilge.github.io](http://jvwilge.github.io)
