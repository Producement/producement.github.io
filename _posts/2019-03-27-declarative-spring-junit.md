---
layout: post
title: 'Declarative configuration and mocking in Spring Boot tests'
author: maido
categories: [tech, spring-boot, test, kotlin]
image: /assets/images/annotations.png
featured: true
hidden: false
---

There are multiple ways how to set up mocks and configuration in [Spring Boot](https://spring.io/guides/gs/spring-boot/) tests. Mostly the projects we have been working on have some base classes that you can extend depending on the types of your tests. For example you will have `IntegrationTest`, `RepositoryTest` or `UnitTest` base classes that you extend while writing a new test class, which contain all the mocks and configuration that is needed. but we are big fans of composition and declarative configuration, which is extensively used in Spring Boot itself, so I'm going to show you some examples that we use and have found to work quite nicely for us. We are also big fans of Kotlin so I'm going to use it in the examples, but it works almost the same way in Java.

## Configuration annotations

Instead of having base classes for test, each test type has its own annotation. For unit tests we have

```
@Target(CLASS)
@Retention(RUNTIME)
@TestInstance(Lifecycle.PER_CLASS)
annotation class UnitTest
```

This is a really simple one and probably only useful in Kotlin where we do not have static methods. So if we want to use `@BeforeAll` and `@AfterAll` annotations for setup and cleanup on instance methods we need `@TestInstance(Lifecycle.PER_CLASS)` to create only one instance of the test per class.

Let's take an example that is a bit more useful: test for Spring repositories. It is using test slicing to only start up the beans that we need to test a repository class, without running service or presentation layer beans.

```
@Target(CLASS)
@Retention(RUNTIME)
@TestInstance(Lifecycle.PER_CLASS)
@DataJpaTest
annotation class RepositoryTest
```

Quite simple as well, but adding this annotation to your class enables you to test a specific repository

```
@RepositoryTest
class UserRepositoryTest @Autowired constructor(
  private val userRepository: UserRepository
) {
    ...
```

Not a lot of useful configuration yet, but these annotations give us a base that we can improve upon.

## Setting up mock data

We want to test our repository, but we don't really want to use another repository to set up the data for it. Spring Boot provides a `@Sql` annotation that you can use to run raw SQL.

```
@Sql (statements = ["insert into users(id,name) values(1, 'someuser')"])
@Sql (statements = ["delete from users"], executionPhase = AFTER_TEST_METHOD)
@Test
fun `testing something that needs a user` () {
    ...
}
```

This is easy to set up and you can use a meta annotation to clean up your code.

```
@Sql (statements = ["insert into users(id,name) values(1, 'someuser')"])
annotation WithTestUser
```
```
@Sql (statements = ["delete from users"], executionPhase = AFTER_TEST_METHOD)
annotation Cleanup
```

Now you can use those annotations on your tests instead of using `@Sql` directly.

But this is also not perfect for multiple reasons. For example the data is static since strings in annotations need to be constant and there is not a lot of room for customization.

To tackle these issues we have come up with something like this.

```
// Kotlin doesn't support @Repeatable so we need a grouping annotation
annotation class WithAll(val value: Array<With>)

annotation class With(
  val value: KClass<out TestFixture>
)

interface TestFixture {
  fun apply(jdbcTemplate: JdbcTemplate)
}
```

Now you can create your own annotations with fixtures for mock data.

```
object UserFixture : TestFixture {

  override fun apply(jdbcTemplate: JdbcTemplate) {
    SimpleJdbcInsert(jdbcTemplate)
      .withTableName("users")
      .execute(
        mapOf(
          "id" to aUser.id,
          "name" to aUser.name,
        )
      )
  }

  val aUserId = RandomUtils.nextInt().toLong()
  val aUser = User(aUserId, "someuser")
}

@With(UserFixture::class)
annotation class WithTestUser
```

To make it work for our Repository annotation we need to create a `TestExecutionListener`.

```
class FixtureAnnotationTestExecutionListener : AbstractTestExecutionListener() {

  override fun beforeTestExecution(testContext: TestContext) {
    val template = testContext.applicationContext.getBean(JdbcTemplate::class.java)
    val annotations = AnnotatedElementUtils.getMergedRepeatableAnnotations(
      testContext.testMethod, With::class.java, WithAll::class.java
    )
    val fixtures = annotations.map { it.value.objectInstance ?: it.value.createInstance() }
    fixtures.forEach { it.apply(template) }
  }
}
```

And add it to the `@RepositoryTest` annotation.

```
@Target(CLASS)
@Retention(RUNTIME)
@TestInstance(Lifecycle.PER_CLASS)
@DataJpaTest
@TestExecutionListeners(listeners = [FixtureAnnotationTestExecutionListener::class], mergeMode = MERGE_WITH_DEFAULTS)
@Cleanup
annotation class RepositoryTest
```

As you can see I also added the `@Cleanup` annotation here that we previously defined to automatically clean up all the mock data we used.

## Summary

Metaannotations (annotations on annotations) are a powerful thing that can be used in the application code and during testing to simplify configuration. Use them, but as always try not to go overboard :)