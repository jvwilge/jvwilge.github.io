---
layout: default
title: Sharing build logic across multi repository Gradle projects - A parent pom on steroids
exerpt: 
date: 2022-09-22 06:52:40
tags : java gradle
categories : en
---
Sharing build logic across multi repository Gradle projects - A parent pom on steroids
===

Coming from a Maven background I sometimes still struggle doing things the Gradle-way. Usually you search on Google how you would do things in Maven and add Gradle to the search terms. Not with reusing build logic. 

<!--more-->

This article is about sharing it across multiple projects across multiple source repositories (most articles about this subject are aimed at sub-projects), publishing it to a Maven repository and handling the small issues you might run into. You might even consider this as an alternative for an archetype (with automatic updates when you update the convention plugin version).

# Introduction

I'm kind of OCD when it comes to keeping build files across projects consistent. Recently it kicked in when I was doing my regular inspection round. It appeared a few projects slipped through the cracks and had a different setup and lineage.

For most people this isn't a problem, but since I update build files by doing [plain text diffs](https://www.jetbrains.com/go/guide/tips/compare-with-clipboard/#:~:text=Select%20any%20pair%20of%20files,file%20to%20compare%20it%20to.) it really took much more time to keep things similar. Since this will probably happen again and we have about 8 micro services under our control right now, I thought it would be nice if we could re-use some logic across those services (preferrably without copy/pasting of course). Another thing I kept in mind is to make it useful for the rest of the organisation too.

# Convention plugin

Coming from Maven I started to search for things like parent pom but didn't find a satisfying solution.
Asking around in other teams my search only became more difficult because of some compelling ideas people came up with.

When I almost gave up I accidentally found a sample project : [Sharing build logic in a multi-repo setup Sample](https://docs.gradle.org/current/samples/sample_publishing_convention_plugins.html). You can download a [zip](https://docs.gradle.org/current/samples/zips/sample_publishing_convention_plugins-groovy-dsl.zip) with code examples.

The main thing to note here is that I was used to inheritance in Maven and in Gradle a better way is to use composition. The solution is called 'convention plugin'. Most examples you'll find are about sharing it across sub-projects though, but I want to share it across multiple projects (in different code repositories) and eventually publish it to a (Maven) repository.

# More details

You can skip this section if you're not interested in my quest for a solution.

One of the articles that made me stop thing the Maven way was [Composition over inheritance: Gradle vs Maven](https://melix.github.io/blog/2021/12/composition-in-gradle.html). It also gave a bit of direction but not a real solution.

Another nice read was [How to write Gradle plugins - answers to common questions and alternative implementation solutions](https://androidexample365.com/how-to-write-gradle-plugins-answers-to-common-questions-and-alternative-implementation-solutions/), I can't really quote anything related to this article, but it contains useful information and maybe on a subconscious level it provided input for this article.

# Determining the granularity

The next step was splitting up our `build.gradle` in usable sub-sections. We want to be flexible but also don't have to many pieces you can use. For our projects I came up with 8 'global' plugins and two common ones for our 8 micro services.

The main component I called `java-root`. This one will be used in every java-project.
I will share the reasoning of some of the parts to invite you to think about how you can split  up your own projects:
- `itest` for using the [integration test Gradle plugin](https://plugins.gradle.org/plugin/com.softeq.gradle.itest) because not everybody made a hard split between `test` and `itest`
- `docker-compose` uses the [Docker compose plugin](https://github.com/avast/gradle-docker-compose-plugin) (uses `itest`) Not all projects use `itest` and some use [Testcontainers](https://www.testcontainers.org/)
- `jib` to build Docker images, because it is not used for library projects
- `logback` to make sure the right logging jars end up in production, because other projects might use another library (like [log4j2](https://logging.apache.org/log4j/2.x/))
- `spring-boot` for spring-boot projects (uses `itest`) because it is not used for libraries and we might use another application framework in the future

# Setting up the convention plugin

I don't want to repeat the excellent sample project so I'll do a quick summary and you can check the sample code yourself. If you put this article next to it should be easy to grasp.

You have to apply the `groovy-gradle-plugin` plugin (plugin-plugin is **not** a typo) and create a file in `src/groovy/`. Note that the filename will determine the path the plugin will end up in in your (Maven) repository.

If the file is called `com.myorg.java-conventions.gradle` you will find it in `com/myorg/java-conventions/com.myorg.java-conventions.gradle.plugin` on your (Maven) repository.

In the `.gradle` file you'll write all the build logic you want to re-use just like you do in a normal `build.gradle`.

In order to publish the plugin the `maven-publish` plugin is used. To publish to your local Maven repository usually `~/.m2/repository` you can run `./gradlew publishToMavenLocal`.


# Using plugins in the convention plugin

Most projects use (regular) plugins and defining a plugin is also build logic that can be shared. It is possible,  but got me confused a bit when passing a version because of a comment in the sample:

```java
// NOTE: external plugin version is specified in implementation dependency artifact of the project's build file
```

Luckily a colleague showed me how to interpret the line. The line means that if you use a plugin you have to define the version in the dependencies block of your `conventions-plugin-groovy/build.gradle` file (so the build file that builds the convention plugin, **not** the plugin `.gradle` file!).

# Overriding dependency versions (without version catalog)

Sometimes you want to override a version defined by the plugin. We do this by defining the overrides in `gradle.properties`. In your convention plugin you need to take some special action because overriding doesn't always work as expected.

In the convention plugin we can't use `gradle.properties` to define defaults because this will is not published to the (Maven) repository, if it was Gradle wouldn't be able to use it anyway.

If you define the property in the `ext` block of the convention plugin but don't define it in the project using the convention plugin it will fail because the property is not found. Defining the property in your project is a solution, but ideally you only want to override the property in special cases and use the default version otherwise.

To solve this issue you can use the following construction in the convention plugin :

```groovy
ext["junitVersion"] = findProperty("junitVersion") ?: "5.9.0"
```

This uses version `5.9.0` of JUnit which you can down/upgrade to another version by defining `junitVersion` in `gradle.properties` of your project.

This construction still feels a bit strange, but it seems like an accepted way to do it (example: [1](https://discuss.gradle.org/t/ext-properties-and-p-overrides/21269),
[2](https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/htmlsingle/#running-your-application.passing-system-properties),
[3](https://blog.mrhaki.com/2016/05/gradle-goodness-get-property-or-default.html))

An alternative might be to use a [version catalog](https://sourcediving.com/manage-your-gradle-dependencies-with-version-catalog-not-only-in-android-114117647cdb), which has recently became a stable feature of Gradle. This however deserves another article and I still have to smooth outh some wrinkles there.

# Using it in your own project

To use a convention plugin in your own project you first add a `pluginManagement` section to `settings.gradle`. Note that has to be the first element in this file. You have to do this unless you publish to the [Gradle Plugin Portal](https://plugins.gradle.org/).

For local testing it would look like this:
```groovy
pluginManagement {
    repositories {
        mavenLocal()
        gradlePluginPortal()
    }
}
```

But you probably should replace `mavenLocal()` with your company repository.

The next step is to define the plugin version in `gradle.properties` :

```properties
myOrgConventionPluginVersion=1.2.3
```

And finally add the plugin to `build.gradle`:

```groovy
plugins {
    id 'com.myorg.java-conventions' version "$myOrgConventionPluginVersion"
}
```

Of course you can also define the version directly in `build.gradle`, but I thinks this is a cleaner way to do it.

# Conclusion

I learned a lot while writing this article. Besides using Google it can be very useful to just do some random exploration in the Gradle docs. I hope this article was helpful.

If you have any comments, improvements or spotted a mistake don't hesitate to reach out on [twitter.com/jvwilge](https://twitter.com/jvwilge), thank you for reading!

> First published on September 22, 2022 at [jvwilge.github.io](http://jvwilge.github.io)