This page contains links to articles I've written.
If you use (parts of) an article please add a reference, thank you!

----

## Recent posts

<ul class="posts">
  {% for post in site.posts limit:3 %}
    <li class="post">
      <a href="{{ post.url }}">{{ post.title }}</a>
      <time class="publish-date" datetime="{{ post.date | date: '%F' }}">
        {{ post.date | date: "%B %-d, %Y" }}
      </time>
    </li>
  {% endfor %}
</ul>

### 2022

- [Using EasyRandom and Lombok's .toBuilder to improve the sustainability of your Unit tests](https://jvwilge.github.io/en/2022/08/01/easy-random-to-builder.html)
- [Splitting up your fast and slow tests in Gradle to receive faster feedback](/2022/04/21/gradle-split-tests.html)

### 2021

- [Unit testing your MapStruct mapper for omitted parameters - EasyRandom to the rescue](/en/2021/08/31/mapstruct-easyrandom.html)

### 2020

- [Running your java integration tests with Gradle and docker-compose with only one command (optionally with Bitbucket Pipelines)
](https://vanwilgenburg.wordpress.com/2020/09/02/docker-compose-gradle-bitbucket/)
- [Using a ConnectableFlux to do background batching on elasticsearch](https://vanwilgenburg.wordpress.com/2020/01/09/connectableflux-with-elasticsearch/)

### 2019

- [Running your elasticsearch integration tests with JUnit 5, Karate and TestContainers (Docker)](https://vanwilgenburg.wordpress.com/2019/07/08/elasticsearch-junit5-karate-testcontainers/)
- [How to remotely reload classes with Spring Developer Tools without opening extra ports](https://vanwilgenburg.wordpress.com/2019/06/03/spring-dev-tools/)
- [Writing integration tests for CORS headers (with Karate)](https://vanwilgenburg.wordpress.com/2019/05/03/writing-integration-tests-for-cors-headers-with-karate/)
- [Using ZAP-proxy and nginx to debug and tamper with HTTP traffic – Emulate timeouts and other unexpected behaviour](https://vanwilgenburg.wordpress.com/2019/01/22/embedded-elasticsearch-junit5-spring-boot/)

### 2018

- [Using ZAP-proxy and nginx to debug and tamper with HTTP traffic – Emulate timeouts and other unexpected behaviour](https://vanwilgenburg.wordpress.com/2018/10/02/zap-proxy-and-nginx/)
- [Lessons learned after serving thousands of concurrent users in a devops team for a year](https://vanwilgenburg.wordpress.com/2018/08/22/lessons-learned-after-serving-thousands-of-concurrent-users-in-a-devops-team-for-a-year/)
- [Introduction to Java heap tuning – Some easy steps to improve response times
  ](https://vanwilgenburg.wordpress.com/2018/03/05/introduction-to-java-heap-tuning-some-easy-steps-to-improve-response-times/)

### [Older](https://vanwilgenburg.wordpress.com/)
<!--
[2017]() | [2016]() | [2015]() | [2014]() | [2013]()
---
-->

------
[Github profile](http://github.com/jvwilge) | [Stackoverflow profile](https://stackoverflow.com/users/833009/jvwilge) | [articles written before 2021](https://vanwilgenburg.wordpress.com/)
