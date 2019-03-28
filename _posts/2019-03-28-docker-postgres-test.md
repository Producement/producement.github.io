---
layout: post
title: 'Running tests against a real Postgres instance'
author: maido
categories: [tech, spring-boot, test, postgres, kotlin]
image: /assets/images/testcontainers.png
featured: false
hidden: false
---

From our personal experience we have learned that it is a really good idea to use an actual DB that you use in production to run your tests against so that there won't be any nasty surprises after you go to production. So no [H2](https://h2database.com/html/main.html) for us. It is possible to use an [embedded DB](https://github.com/yandex-qatools/postgresql-embedded), but we had some issues with those as well. What has worked for us is just running the DB in [Docker](https://www.docker.com/) and using it to run tests. It's quite easy to set it up using the [testcontainers](https://www.testcontainers.org/) project.

```
class DbContainerInitializer : ApplicationContextInitializer<ConfigurableApplicationContext> {

  override fun initialize(applicationContext: ConfigurableApplicationContext) {
    // Only start Postgres when not running in CI
    if (!applicationContext.environment.acceptsProfiles(Profiles.of("ci"))) {
      postgres.start()
      TestPropertyValues.of(
        "spring.datasource.url=${postgres.jdbcUrl}",
        "spring.datasource.username=${postgres.username}",
        "spring.datasource.password=${postgres.password}"
      ).applyTo(applicationContext.environment)
    }
  }

  companion object {
    // Lazy because we only want it to be initialized when accessed
    private val postgres: KPostgreSQLContainer by lazy {
      KPostgreSQLContainer("postgres:10-alpine")
        .withDatabaseName("databasename")
        .withUsername("username")
        .withPassword("password")
    }
  }
}

// Hack needed because testcontainers use of generics confuses Kotlin
class KPostgreSQLContainer(imageName: String) : PostgreSQLContainer<KPostgreSQLContainer>(imageName)
```

This class will start a Postgres database instance container and injects the properties needed to connect to this database to the Spring environment. If profile called `ci` is active, this container will not be initialized because the CI platform we use already provides a Postgres DB to the container it runs tests in so no need to run Docker in Docker.

You can use this class with `@ContextConfiguration` annotation on your test annotation class. 

```
@Target(CLASS)
@Retention(RUNTIME)
@TestInstance(Lifecycle.PER_CLASS)
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@ContextConfiguration(initializers = [DbContainerInitializer::class])
@TestExecutionListeners(listeners = [FixtureAnnotationTestExecutionListener::class], mergeMode = MERGE_WITH_DEFAULTS)
@Cleanup
annotation class RepositoryTest
```

Easy!

What it does is that it starts up a Postgres instance inside Docker when the `DbContainerInitializer.initialize` method is called. Since the `postgres` variable is in companion object, it will only be initialized once when the testing starts and the same instance will be used throughout the tests. You can change it to an instance variable if you want, but this will slow down the tests. We just use a `@Cleanup` annotation that we created in our [previous blog post](https://blog.producement.com/tech/spring-boot/test/kotlin/2019/03/27/declarative-spring-junit.html) to make sure that the DB is pristine after every test run.