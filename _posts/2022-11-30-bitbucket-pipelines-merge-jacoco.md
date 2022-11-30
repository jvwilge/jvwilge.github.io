---
layout: default
title: Merging Jacoco coverage files with Gradle to run parallel builds on Bitbucket Pipelines
exerpt: 
date: 2022-11-30 09:39:18
tags : java gradle jacoco sonar bitbucket-pipelines
categories : en
---
Merging Jacoco coverage files with Gradle to run parallel builds on Bitbucket Pipelines
===

Our complete build pipeline takes about 20 minutes to run. In this article I'll explain how to split up the pipeline in order to improve the build time. Then I will show you how to merge the coverage reports (since every build step now has its own file).

<!--more-->

A build of 20 minutes is of course annoying. Below that threshold you usually get away with it, but more and more often we touched 20 minutes so time to tackle that problem.

Our main issue is that we have an integration test that takes up most of the time. It takes a while because it uses Docker containers with Kafka and Mysql, two things that can not easily be parallelized on a test level. Because most of the time is spent polling for new messages I thought about duplicating topic names and the database scheme and running two instances of the spring boot app. But this will require some effort and we have to make sure it still runs on less beefier laptops by non-developers.

An easier solution would be to make the split at a higher level, at Bitbucket Pipelines.

If you're only interested in how to do the merging in Gradle you can skip to [Combining the coverage files](#combining-the-coverage-files).

# Splitting the pipeline

In Bitbucket-pipelines there is an option to [run steps in parallel](https://support.atlassian.com/bitbucket-cloud/docs/set-up-or-run-parallel-steps/).
We split the pipeline into 3 steps and a Sonar step after that (`bitbucket-pipelines.yml`):

```yaml
pipelines:
  default:
    - parallel:
        steps:
          - step: *check
          - step: *check2
          - step: *check3
    - step:
        <<: *sonar
```

`check`, `check2` and `check3` will run in parallel. The results of the three steps have to be combined (e.g. coverage reports) in the Sonar step. I'll show you how to do the combining later).

Now that there are 3 steps you of course need to make a split somewhere. It really depends on you build where to make it, I will show you how we did it, that might help you get an idea on how to do it on your build.

The condensed version of `check2` looks like this (`bitbucket-pipelines.yml`):

```yaml
- &check2
  name: Gradle integrationTest testSet B
  script:
    - export KARATE_TESTSET=b
    - ./gradlew :integrationTest --tests "com.awesomecompany.KarateIntegrationTest.testAll"
    - cp build/jacoco/integrationTest.exec build/jacoco/integrationTestB.exec
  artifacts:
    - build/jacoco/integrationTestB.exec
 ```

It doesn't really matter that we use Karate here, this can be done with any system. The Karate test checks for an environment variable `KARATE_TESTSET` and then runs a subset of the integration test. `check3` looks almost the same and used `A` as test set. How you implement this is completely up to you. We made the split by directory, but you can do anything as long as it is deterministic.

The 'main' run uses set `C`. Note that you should be careful with how you split up the test set. `C` is `<all> minus B minus A`. This way you're sure that no test is missed (that also speeds up your build, but in a nasty way, by skipping important tests). 

In the main run you run your regular build, minus set `A` and `B` (we call that `C`)  (`bitbucket-pipelines.yml`):

```yaml
- &check
  name: Gradle check, integrationTest set C
  script:
    - export KARATE_TESTSET=c
    - ./gradlew check
  artifacts:
    # all files so we don't need another fresh checkout
    - "**"
```

Splitting the pipeline in three doubled the speed. There's only one but...

We also use Sonar and Sonar needs a coverage file. Since we had three runs we have at least 3 coverage files (the unit tests also have a coverage file).

# Artifacts

As you might have spotted we have `artifacts` at the end of each step. [This carries over some files](https://support.atlassian.com/bitbucket-cloud/docs/use-artifacts-in-steps/) (or all files in case of test set `C`) to the next step (also see [this site](https://community.atlassian.com/t5/Bitbucket-questions/Is-there-a-way-of-sharing-data-between-steps-in-bitbucket/qaq-p/777977))

We carry over all the files of set `C` so we don't have to checkout and compile the code again.

# Combining the coverage files

To combine the coverage files for Sonar you'll probably run into `JacocoMerge`, **don'
t** use this since the docs state it: _"is deprecated and will be removed in the next major version of Gradle."_

The right step to take is creating a task and output the file as xml (since the combined `.exec` only works for Java and not the other JVM languages (and is also [deprecated](https://docs.sonarqube.org/8.6/analysis/coverage/)))
(`build.gradle`):

```groovy
task combineJacocoReports(type: JacocoReport) {
    // sources: 
    // - https://igorski.co/join-jacoco-reports/
    // - https://stackoverflow.com/questions/65982262/aggregate-several-jacoco-exec-files-into-a-single-coverage-report-with-gradle
    executionData fileTree(project.rootDir.absolutePath).include("**/build/jacoco/*.exec")
    sourceSets sourceSets.main
    reports.xml.required = true
}

sonarqube {
    properties {
        property "sonar.coverage.jacoco.xmlReportPaths", "$buildDir/reports/jacoco/combineJacocoReports/combineJacocoReports.xml"
    }
}
```

Running combineJacocoReports will merge all the `.exec` files to a single xml file which you can pick up in the Sonar step (`bitbucket-pipelines.yml`):

```yaml
- &sonar
  name: Sonar
  clone:
    enabled: false
  script:
    #exclude all tasks except sonar, others are already run
     - ./gradlew -x check combineJacocoReports sonar -x jacocoTestReport -x compileTestJava -x compileJava -i
```

Since we carried over the compiled classes there is no need to clone and build the project again, so you can [disable cloning](https://bitbucket.org/blog/disabling-clones-in-pipelines-steps).
It feels a bit error prone to exclude all the tasks except two, so let me know if there's a better way!

# Conclusion

Taking these steps really sped up our build into 3 steps. Don't go overboard by splitting your build into 50 parallel threads. There's a startup time and you'll probably get bothered by [the finance department at some point](https://support.atlassian.com/subscriptions-and-billing/docs/manage-your-bill-for-enterprise-plans/). I think a sensible maximum build time is between 10 and 15 minutes.

If you have any comments, improvements or spotted a mistake don't hesitate to reach out on [twitter.com/jvwilge](https://twitter.com/jvwilge), thank you for reading!

> First published on November 30, 2022 at [jvwilge.github.io](http://jvwilge.github.io)
