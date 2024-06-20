---
title: "Programmatically find unused properties in a Spring Boot app"
date: 2024-06-20
tags:
  - spring-boot
  - spring
tootId: "112650692876985992"
draft: false
---

## Problem

[Spring Boot's externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html) is a powerful feature. But with great power comes, ahem, at least sometimes, a bunch of unused properties polluting your environment. That `what=where-does-this-get-used` in `application-doineedthem.properties`? You might be able to remove it.

## Solution

I created a solution that you can incorporate into your test setup, so

1. you get a failing build if you have unused properties (in a linter kind of way), and
2. you do not pollute your regular classpath with code related to meta topics such as unused properties.

First, create an `EnvironmentPostProcessor`, which "[a]llows for customization of the application's Environment prior to the application context being refreshed." This should be the correct time to _snapshot_ your properties and inject property-usage tracking capabilities.

```java
public class UnusedPropertiesEnvironmentPostProcessor implements EnvironmentPostProcessor {

    private static Set<String> getAllPropertyNames(ConfigurableEnvironment environment) {
        var result = new HashSet<String>();
        for (var ps : environment.getPropertySources()) {
            // One can pick one's own heuristic on what to track. 
            // `instance of OriginTrackedMapPropertySource` should cover externalized config files
            // such as application.properties.
            if (ps instanceof OriginTrackedMapPropertySource otps) {
                result.addAll(Arrays.asList(otps.getPropertyNames()));
            }
        }
        return result;
    }

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        var propertyNames = getAllPropertyNames(environment);
        environment.getPropertySources().addFirst(new PropertySource<>("Usage tracking property source adapter") {
            @Override
            public Object getProperty(String name) {
                if ("unused.properties".equals(name)) {
                    return propertyNames;
                }
                propertyNames.remove(name);
                return null;
            }
        });
    }
}
```

Essentially, we're adding a dummy `PropertySource` instance that gets consulted _before_ all the others. For every `getProperty(String name)` invocation, it will simply mark that property as used, i.e. remove it from the set of all previously detected properties, and, by returning `null`, will let the next `PropertySource` instance in line handle the call.

Whenever one retrieves the value of the `unused.properties` property, one gets the set of the properties that haven't been used anywhere until this point in time.

Next, create a `spring.factories` file under `src/test/resources/META-INF` with the following content to bootstrap the above `EnvironmentPostProcessor` instance:

```text
org.springframework.boot.env.EnvironmentPostProcessor=\
  replace.with.your.package.UnusedPropertiesEnvironmentPostProcessor
```

Finally, we can use our detection mechanism in a `@SpringBootTest` like so:

```java
@SpringBootTest
class UnusedPropertiesTest {

  @Value("${unused.properties}")
  Set<String> unusedProperties;

  @Test
  void containsNoUnusedProperties() {
    assertThat(unusedProperties).isEmpty();
  }
}
```

If you have e.g. a property `foo=bar` in your `application.properties` that you do not `@Value("${foo}")` anywhere, this test is going to fail.
