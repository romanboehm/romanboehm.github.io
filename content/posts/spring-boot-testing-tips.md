+++
title = "Keep Your Properties Close, but Your @TestConfiguration Closer - Selected Spring Boot Testing Tips"
date = "2021-10-02"
tags = [
    "spring-boot",
    "spring",
    "testing"
]
draft = false
toc = true
+++

##  Introduction

Spring Boot is great and the best thing about it is how easy it makes it to test your application. So here are some selected tips for writing tests with Spring Boot. The most recent stable Spring Boot version at the time of publishing was 2.5.5.

## Keep Configuration Close to the Tests

The configuration and setup of the tests should be as close to them as possible. This reduces the possibility of one test interfering with another and makes it much easier for the reader of the test to grok its purpose.
Here are two means to achieve that goal:

### `@TestConfiguration` As Inner Class

I often use this one to re-configure beans for a single testsuite only. Like so:

```java
@DataJpaTest
class StatementCountTest {

    @TestConfiguration
    static class ProxyDataSourceConfig {

        @Bean
        DataSource dataSource(DataSourceProperties baseDataSourceProperties) {
            return new StatementListeningDataSource(
                baseDataSourceProperties.initializeDataSourceBuilder().build()
            );
        }
    }

    @Autowired
    private EventRepository eventRepository;

    @Test
    void executesSingleStatementToFindAllEvents() {
        StatementCountValidator validator = new StatementCountValidator();
        eventRepository.findAll(); // Uses the `DataSource` implementation provided above internally.
        assertThat(validator.getStatementCount()).isEqualTo(1);
    }
}
```

### Properties Declared At Class Level

Tests should be as close to your production environment as possible. Declaring your properties with `@SpringBootTest(properties = { "x=y", "n=1" })` (same goes for [test slices](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.spring-boot-applications.autoconfigured-tests)) at the class level helps with that: It allows to 
1) provide only the needed properties, in case you do not keep properties files, or to
2) selectively override individual properties, like URLs, for tests.

## Don't Dirty Your Context

Sounds like innuendo, but it's supposed to signal something entirely different: Do not use `@DirtiesContext` to reset your [ApplicationContext](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html) because your test suite will become horribly slow. 
Rather reset your state (e.g. database) manually through `@AfterEach`/`@AfterClass` or employ other means which allow Spring to [cache your context](https://rieckpil.de/improve-build-times-with-context-caching-from-spring-test/).

Therefore I'd also suggest to ...

## Use @SpringBootTest religiously

Even though it may seem counter-intuitive at first, try using `@SpringBootTest` "by default". My rule of thumb is: for any unit test where you need to nest your mocks and cannot use a test slice. This provides a lot of advantages in my opinion:

1) It makes tests cleaner. An `@Autowired` is most often much shorter than `new ABean(new BBean(MyConfigurationFromSomeWhere()))`.
2) It's not only cleaner, but the beans in your tests will be set up the same as in your production environment. There's fewer surprises later on in case the way you manually set up your beans, or your mocks does not reflect your production system anymore.
3) It may even be faster or as fast as tests with a lot of mocks and spies due to the above mentioned context caching.

## Use Testcontainers

If you've never heard of it: [Testcontainers](https://www.testcontainers.org/) provides a truly extraordinary library to run your tests against "real" infrastructure. Your database in your test will be on-par feature-wise with your production database because _it's the same database_ (only running within a Docker container), your Kafka cluster won't be some half-assed in-house in-memory implementation, it'll be the real deal.
I use their Postgres container for my [showcase application](https://github.com/rmnbhm/wichtelnng) and it's just a single line to have an actual Postgres database available for my `@SpringBootTest`s:
```properties
# In src/test/resources/config/application.properties
spring.datasource.url=jdbc:tc:postgresql:latest:///postgres?TC_DAEMON=true
```
No configuration needed in the tests themselves! Also note the `?TC_DAEMON=true`. This will keep the container running even without open connections to the database, meaning it'll be available super quickly for the next test.

## Conclusion

Spring Boot is opinionated and so were these tips. I found by honoring them my tests became easier to read, easier to maintain, and more meaningful, sometimes even faster. When in doubt just measure... or find what works for you, a fully-featured application context is waiting at your finger tips.
