---
layout: default
title: Splitting up your fast and slow tests in Gradle to receive faster feedback
content: This article will show you how to run your slow unit/integration after the fast ones. This way you'll get negative feedback quicker so you can spend your time more efficient. 
date: 2022-04-21 07:51:55
tags : mapstruct easy-random java
categories : en
---
Splitting up your fast and slow tests in Gradle to receive faster feedback
===

This article will show you how to run your slow unit/integration after the fast ones. This way you'll get negative feedback quicker so you can spend your time more efficient.

Our project is growing and building it takes about 10 minutes because of our extensive integration test which is waiting a lot on Kafka messages. There also is an ArchUnit test which technically is a unit test, but takes tens of seconds to run.  If there is an error I usually have to wait for those tests to finish since they're executed early in the build flow. So running the fast tests separate from the long ones (with the fast ones first of course) will improve the response time of a build failure if there is an error.

Our environment
---

We build our project with Gradle and use JUnit 5 for our unit and integration tests. The integration tests talk to a Docker compose environment spun up with the [Avast Docker Compose Plugin](https://github.com/avast/gradle-docker-compose-plugin). Unit and integration tests are split with the [itest :: gradle plugin](https://softeq.github.io/itest-gradle-plugin/), so splitting both in fast and slow will effectively become a build in four steps. A few things other projects might not have are:
- [The Jacoco plugin](https://docs.gradle.org/current/userguide/jacoco_plugin.html), there is a small thing you have to change when splitting the tests.
- [ArchUnit](https://www.archunit.org/) , ArchUnit does not (yet) have support for the `@Order` annotation.

What about `--fail-fast`?
---

It is tempting to think that the `--fail-fast` is the solution for this problem. It behaves a bit different since you have no control about the order. So let's say the slowest test is executed first you still have to wait a long time for negative feedback. Of course you can combine the solution of this article with `--fail-fast` to squeeze the last seconds out of your failing build.

Note that you only can add `--fail-fast` [in combination with a Test task](https://github.com/gradle/gradle/issues/4562). So
`./gradlew build --fail-fast` won't work, but `./gradlew build test --fail-fast` or `./gradlew build integrationTest --fail-fast` will.  

Preparation
---

In order to split the tests I decided to mark slow tests with `@Tag("slow")`, a solution found in [this article](https://igorski.co/running-junit-5-tests-with-gradle/).

I picked `slow` as a name because that's the path of least resistance since there are not a lot of slow tests.

Note that if you're [coming from JUnit 4](https://junit.org/junit5/docs/current/user-guide/#migrating-from-junit4-tips):
> `@Category` no longer exists; use `@Tag` instead

An example test class would look like:
```java
@Tag("slow")
class MySlowTest {
    // do slow tests here
}
```


Adapting the unit test task
---

Some searching on stackoverflow showed me [this answer](https://stackoverflow.com/a/67582830/833009) and inspired me to this solution:

```groovy
task testFast(type: Test) {
    useJUnitPlatform {
        excludeTags 'slow'
    }

    group 'verification'
    description 'run unit tests without @Tag("slow") < 2 seconds'
}

test {
    useJUnitPlatform {
        includeTags 'slow'
    }

    description 'run unit tests tagged with @Tag("slow") > 2 seconds'
}

test.dependsOn tasks.testFast
```

So important to note here is that I let `test` depend on `testFast`. This means that if you run the `test` tasks it will also run `testFast`. My philosophy here is that if someone doesn't know about `testFast` then running `test` should run all tests just like before so no tests are missed when testing locally.

If you don't want this behaviour you can use `test.shouldRunAfter tasks.testFast` instead so they always run separate. Also add `check.dependsOn(testFast)` otherwise it won't run as part of the `check` task.

Note that the `group` is just decorative here, it is good to have it show up in the right section when running `./gradlew tasks`.

The only differences when you run all the tests is that a new report will show up in the `build/reports/tests` directory and a Jacoco coverage file (`.exec`) in `build/jacoco`.

To combine the coverage reports you can define the following task in your `build.gradle` :


```groovy
jacocoTestReport {
    executionData(test, testFast)
    reports.xml.required = true
}

```


Adapting the integration test
---

To run the integration tests we use [itest :: gradle plugin](https://softeq.github.io/itest-gradle-plugin/) to run JUnit 5 tests. 

We're using the [Avast Docker Compose Plugin](https://github.com/avast/gradle-docker-compose-plugin)  to spin up our Docker containers.

Again we mark the slow tests with `@Tag("slow")` and redefine the task (note that this is slightly different than the `test` task) :

```groovy
task integrationTestFast(type: Test) {
    useJUnitPlatform {
        excludeTags 'slow'
    }

    group 'verification'
    description 'integration tests tagged without @Tag("slow") < 2 minutes'
}

integrationTest {
    testClassesDirs = sourceSets.itest.output.classesDirs
    classpath += sourceSets.itest.runtimeClasspath

    useJUnitPlatform {
        includeTags 'slow'
    }

    group 'verification'
    description 'integration tests tagged with @Tag("slow") > 2 minutes'
}

integrationTest.dependsOn tasks.integrationTestFast
```

The original `integrationTest` is modified


Make sure the `dockerCompose` task is moved under the `integrationTestFast` since the `isRequiredBy` can't point downwards and can only have one argument. 

Add (or modify `isRequiredBy`) :
```groovy
isRequiredBy(tasks.integrationTestFast)
```

Otherwise you'll get the following error:

> `Could not get unknown property 'integrationTestFast' for task set of type org.gradle.api.internal.tasks.DefaultTaskContainer`

Since `integrationTest` depends on `integrationTestFast` the `composeUp` task will run before any integration test and `composeDown` runs after all the integration tests (or even if you run only `integrationTestFast`). This was also a reason to use `dependsOn` instead of `shouldRunAfter`.
When you start experimenting yourself the `./gradlew <task> --dry-run` command is useful to show all the tasks involved and make sure all the tasks are executed so you won't end up in an inconsistent state.

Again we get an extra test report and Jacoco file so we have to include it in the report :

```groovy
jacocoTestReport {
    executionData(testFast, test, integrationTestFast, integrationTest)
    reports.xml.required = true
}
```

ArchUnit
---

For ArchUnit it is important to add `@ArchTag("slow")` (and also `@Tag` if you mix regular tests in the ArchUnit test class), otherwise the tag won't be respected.


Conclusion
----

After some discussions with colleagues we went for the `@Order` annotation and the command line option `--fail-fast` which you probably didn't expect after reading the article.

The `@Order` annotation on class level sounded interesting but it didn't work with ArchUnit and since I was almost finished with this article I decided to publish it anyway since it still contains valuable information and our final solutions uses a lot of the techniques mentioned here.
Source: [Stackoverflow: JUnit test class order](https://stackoverflow.com/questions/57624495/junit-test-class-order)

I [filed an issue](https://github.com/TNG/ArchUnit/issues/852) with ArchUnit and got a quick reply that it is hard to support `@Order` because they (understandably) don't want a dependency on JUnit Jupiter API so splitting ArchUnit in a separate task might be the best way to go.

If you have any comments, improvements or spotted a mistake please contact me on [twitter.com/jvwilge](https://twitter.com/jvwilge), thank you for reading!


> First published on April 21, 2022 at [jvwilge.github.io](http://jvwilge.github.io)