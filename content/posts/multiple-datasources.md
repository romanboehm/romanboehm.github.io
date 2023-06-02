---
title: "How to: Configure Multiple DataSources in Spring Boot"
date: 2022-08-16
tags:
  - spring-boot
  - spring
  - hikari
draft: false
---


## Introduction

A few days ago a colleague of mine and I wanted to know "[w]hat’s the Spring Boot way of defining three different DataSources through externalized config (application.yaml), with each sharing the same Hikari settings?" That is: We basically had a working solution, we just weren't sure it was how you'd do it making use of all the goodness Spring Boot offers. Therefore I consequently tried [offloading the task of figuring it out to Twitter](https://twitter.com/0xromanboehm/status/1557374298536525825?s=20&t=09VLy1imZC-GJ5BtLaH7Cw). -.-

And ... it worked! Thanks to [Michael Simons' hint](https://twitter.com/rotnroll666/status/1557377897693945856?s=20&t=09VLy1imZC-GJ5BtLaH7Cw), and the always excellent [docs.spring.io resource](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.data-access.configure-two-datasources) we knew we were right on track. We just had to [iron out some wrinkles](https://twitter.com/dreadwarrior/status/1557407059922112514?s=20&t=09VLy1imZC-GJ5BtLaH7Cw).

## Solutions

I created a demo to figure out and validate the solutions. You can find it on [GitHub](https://github.com/romanboehm/multiple-datasources).

### Validation

Last things first, this is the test that any solution would have to pass:

```java
@SpringBootTest
@Testcontainers
class MultipleDatasourcesApplicationTests {

	@Container
	static PostgreSQLContainer<?> databaseOne = new PostgreSQLContainer<>("postgres:latest");

	@Container
	static PostgreSQLContainer<?> databaseTwo = new PostgreSQLContainer<>("postgres:latest");

	@DynamicPropertySource
	static void loadDatabaseProperties(DynamicPropertyRegistry registry) {
		registry.add("datasources.datasource-one.url", databaseOne::getJdbcUrl);
		registry.add("datasources.datasource-one.jdbcUrl", databaseOne::getJdbcUrl); // For `HikariConfig`.
		registry.add("datasources.datasource-one.password", databaseOne::getPassword);
		registry.add("datasources.datasource-one.username", databaseOne::getUsername);
		registry.add("datasources.datasource-two.url", databaseTwo::getJdbcUrl);
		registry.add("datasources.datasource-two.jdbcUrl", databaseTwo::getJdbcUrl); // For `HikariConfig`.
		registry.add("datasources.datasource-two.password", databaseTwo::getPassword);
		registry.add("datasources.datasource-two.username", databaseTwo::getUsername);
	}

	@Autowired
	@Qualifier("dataSourceOne")
	private DataSource one;

	@Autowired
	@Qualifier("dataSourceTwo")
	private DataSource two;

	@Test
	void canUseBothDataSources() throws SQLException {
		try (
				var connOne = one.getConnection();
				var connTwo = two.getConnection();
		) {
			boolean oneSucceeded = connOne.createStatement().execute("SELECT 1;");
			boolean twoSucceeded = connTwo.createStatement().execute("SELECT 1;");

			assertThat(oneSucceeded).isTrue();
			assertThat(twoSucceeded).isTrue();
		}
	}

	@Test
	void dataSourceOneConfiguredCorrectly() {
		assertThat(one).isInstanceOfSatisfying(HikariDataSource.class, hikariDataSource -> {
			assertThat(hikariDataSource.getPoolName()).isEqualTo("one");
			assertThat(hikariDataSource.getConnectionTimeout()).isEqualTo(30001);
			assertThat(hikariDataSource.getIdleTimeout()).isEqualTo(600001);
			assertThat(hikariDataSource.getMaxLifetime()).isEqualTo(1800001);
		});
	}

	@Test
	void dataSourceTwoConfiguredCorrectly() {
		assertThat(two).isInstanceOfSatisfying(HikariDataSource.class, hikariDataSource -> {
			assertThat(hikariDataSource.getPoolName()).isEqualTo("two");
			assertThat(hikariDataSource.getConnectionTimeout()).isEqualTo(30001);
			assertThat(hikariDataSource.getIdleTimeout()).isEqualTo(600001);
			assertThat(hikariDataSource.getMaxLifetime()).isEqualTo(1800001);
		});
	}

}
```

You can see it checks for general ability to use the `DataSource` instances and their individual configuration. Here, both the values from the shared "base" config as well as individual settings, represented by the `poolName` property, are being asserted on. 

### Solution A: Making use of `HikariConfig`

We can frame this as the "programmatic" solution because it relies on certain configuration classes exposed through Hikari's API.

#### Externalized Configuration

```yaml
# file: application-prog.yaml
datasources:
  hikari-base:
    connectionTimeout: 30001
    idleTimeout: 600001
    maxLifetime: 1800001
  datasource-one:
    username: "username-set-dynamically-in-test"
    password: "password-set-dynamically-in-test"
    jdbcUrl: "url-set-dynamically-in-test"
    poolName: "one"
  datasource-two:
    username: "username-set-dynamically-in-test"
    password: "password-set-dynamically-in-test"
    jdbcUrl: "url-set-dynamically-in-test"
    poolName: "two"
```

#### `@Configuration` class

```java
@Profile("prog")
@Configuration
@EnableConfigurationProperties
public class ProgrammaticDataSourceConfiguration {

    @Bean("hikariConfigBase") // (1)
    @ConfigurationProperties("datasources.hikari-base")
    HikariConfig hikariBaseConfig() {
        return new HikariConfig();
    }

    @Bean("hikariConfigOne") // (2)
    @ConfigurationProperties("datasources.datasource-one")
    HikariConfig hikariConfigOne(@Qualifier("hikariConfigBase") HikariConfig hikariConfigBase) {
        HikariConfig compositeConfig = new HikariConfig();
        hikariConfigBase.copyStateTo(compositeConfig);
        return compositeConfig;
    }

    @Bean("dataSourceOne") // (3)
    public HikariDataSource dataSourceOne(@Qualifier("hikariConfigOne") HikariConfig hikariConfigOne) {
        return new HikariDataSource(hikariConfigOne);
    }


    @Bean("hikariConfigTwo") // (2)
    @ConfigurationProperties("datasources.datasource-two")
    HikariConfig hikariConfigTwo(@Qualifier("hikariConfigBase") HikariConfig hikariConfigBase) {
        HikariConfig compositeConfig = new HikariConfig();
        hikariConfigBase.copyStateTo(compositeConfig);
        return compositeConfig;
    }

    @Bean("dataSourceTwo") // (3)
    public HikariDataSource dataSourceTwo(@Qualifier("hikariConfigTwo") HikariConfig hikariConfigTwo) {
        return new HikariDataSource(hikariConfigTwo);
    }

}
```

Let's walk through it (look for comments containing steps' ordinals):
1. Expose "base" config as a bean. Hikari offers `HikariConfig` as a config class to use with `@ConfigurationProperties`.
2. "Enhance" config from step 1 with data source-specific settings. This is achieved by creating a new `HikariConfig` instance for each data source that contains the old values. Luckily, we can use `HikariConfig`'s own API method `c.z.h.HikariConfig#copyStateTo` for that. `@ConfigurationProperties` then takes care of binding the `datasources-datasource-...` properties to the new instance.
3. Use configs from step 2 to create the final `HikariDataSource`s.

Note:
- `HikariConfig` binds to `jdbcUrl`, not `url`.
- We cannot skip step 2 and use `@ConfigurationProperties("datasources-datasource-...")` in the `HikariDataSource` factory methods directly: When instantiating the data source with `new HikariDataSource(hikariConfig)` we'd need the url. The url, however, will only be set _after_ the instance exists.

### Solution B I: Using YAML anchors

This is one of the two solutions based on [YAML anchors](https://support.atlassian.com/bitbucket-cloud/docs/yaml-anchors/), so it'll work only if the externalized configuration is a .yaml properties file.

#### Externalized Configuration

```yaml
# file: application-yaml-simple.yaml
datasources:
  hikari-base: &hikari-base
    connectionTimeout: 30001
    idleTimeout: 600001
    maxLifetime: 1800001
  datasource-one:
    username: "username-set-dynamically-in-test"
    password: "password-set-dynamically-in-test"
    jdbcUrl: "url-set-dynamically-in-test"
    <<: *hikari-base
    poolName: "one"
  datasource-two:
    username: "username-set-dynamically-in-test"
    password: "password-set-dynamically-in-test"
    jdbcUrl: "url-set-dynamically-in-test"
    <<: *hikari-base
    poolName: "two"
```

#### `@Configuration` class

```java
@Profile("yaml-simple")
@Configuration
@EnableConfigurationProperties
public class SimpleYamlAnchorDataSourceConfiguration {

    @Bean("dataSourceOneProperties") // (1)
    @ConfigurationProperties("datasources.datasource-one")
    HikariConfig dataSourceOneProperties() {
        return new HikariConfig();
    }

    @Bean("dataSourceOne") // (2)
    public HikariDataSource dataSourceOne(
            @Qualifier("dataSourceOneProperties") HikariConfig dataSourceOneProperties
    ) {
        return new HikariDataSource(dataSourceOneProperties);
    }

    @Bean("dataSourceTwoProperties") // (1)
    @ConfigurationProperties("datasources.datasource-two")
    HikariConfig dataSourceTwoProperties() {
        return new HikariConfig();
    }

    @Bean("dataSourceTwo") // (2)
    public HikariDataSource dataSourceTwo(
            @Qualifier("dataSourceTwoProperties") HikariConfig dataSourceTwoProperties
    ) {
        return new HikariDataSource(dataSourceTwoProperties);
    }

}
```

Like with the aforementioned solution, let's walk through this one, too:

1. Bind the `datasources.datasource-...`-prefixed blocks via `@ConfigurationProperties` to `HikariConfig` instances, the latter being more or less a bean intended for mapping data source-related configuration onto it with some additional helper methods for constructing a `DataSource`.
2. Use the the `HikariConfig` of every declared data source section to create the actual `HikariDataSource` from it.

### Solution B II: Using YAML anchors

This is the other solution based on YAML anchors. It comes with a different properties structure, but essentially works the same.

#### Externalized Configuration

```yaml
# file: application-yaml-structured.yaml
datasources:
  hikari-base: &hikari-base
    connectionTimeout: 30001
    idleTimeout: 600001
    maxLifetime: 1800001
  datasource-one:
    username: "username-set-dynamically-in-test"
    password: "password-set-dynamically-in-test"
    url: "url-set-dynamically-in-test"
    hikari:
      <<: *hikari-base
      poolName: "one"
  datasource-two:
    username: "username-set-dynamically-in-test"
    password: "password-set-dynamically-in-test"
    url: "url-set-dynamically-in-test"
    hikari:
      <<: *hikari-base
      poolName: "two"
```

#### `@Configuration` class

```java
@Profile("yaml-structured")
@Configuration
@EnableConfigurationProperties
public class StructuredYamlAnchorDataSourceConfiguration {

    @Bean("dataSourceOneProperties") // (1)
    @ConfigurationProperties("datasources.datasource-one")
    DataSourceProperties dataSourceOneProperties() {
        return new DataSourceProperties();
    }

    @Bean("dataSourceOne") // (2)
    @ConfigurationProperties("datasources.datasource-one.hikari")
    public HikariDataSource dataSourceOne(
            @Qualifier("dataSourceOneProperties") DataSourceProperties dataSourceOneProperties
    ) {
        return dataSourceOneProperties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    @Bean("dataSourceTwoProperties") // (1)
    @ConfigurationProperties("datasources.datasource-two")
    DataSourceProperties dataSourceTwoProperties() {
        return new DataSourceProperties();
    }

    @Bean("dataSourceTwo") // (2)
    @ConfigurationProperties("datasources.datasource-two.hikari")
    public HikariDataSource dataSourceTwo(
            @Qualifier("dataSourceTwoProperties") DataSourceProperties dataSourceTwoProperties
    ) {
        return dataSourceTwoProperties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

}
```

The steps are essentially the same as with B I, but there's a bit more properties binding involved:

1. Bind the `DataSource`-based properties of the `datasources.datasource-...`-prefixed blocks via `@ConfigurationProperties` to `DataSource` instances.
2. From the `DataSourceProperties` instances create `HikariDataSource`s by using Spring Boot's "translation"/building mechanism, which e.g. transfers `DataSource#url` to `HikariConfig#jdbcUrl`. The `datasources.datasource-....hikari` properties then get bound by way of `@ConfigurationProperties` to the new `HikariDataSource`s.

Note:
- The binding in step 2 hinges on the fact that we don't need the properties declared under `datasources.datasource-....hikari` to create the `HikariDataSource` instances, but can set them following their instantiation.

## Evaluation

We _did_ land on B II, using YAML anchors with a "structured" externalized configuration for the following reasons:

- The externalized configuration of every data source reflects the way you would declare a single Hikari data source, i.e. `spring.datasource` with the `url`, `username`, and `password` properties; `spring.datasource.hikari` with Hikari-related properties.
- The `@Configuration` providing the `DataSource` beans is relatively straightforward (yet not as simple as in B I), and doesn't have to make use of `HikariConfig`'s programmatic API.

Depending on how many Hikari-related properties you have, one might also prefer the "simple" YAML-anchor-based solution over the "structured" one. The demo _does_ use very few Hikari-related properties and seems to suggest B I; our real world use case favors B II.

Moreover, as I hope to have demonstrated here, YAML anchors are a general solution to "shared" properties without having to resort to `${...}` property placeholder syntax. 

But, and those are big "but"s (and I cannot lie):
1. Don't overdo the YAML abuse. It might turn simple externalized property declarations into complex code. You wouldn't want to have that.
2. If you can't declare your properties in YAML, solutions B I and II are out of the question anyways.
