+++
title = "Tell Me Your Environment and I Tell You What Your Context Is"
date = "2022-08-06"
tags = [
    "spring",
    "testing",
    "dependencyinjection"
]
draft = false
+++

# Introduction
At my previous work we had this problem where we couldn't run through our entire workflow in our integration system because we were missing the necessary input data to some central service's integration instance. We therefore took it upon us to provide said data by asynchronously _mirroring_ data from prod to int.

This is definitely not a "Five tips on how to deal with numbers in JavaScript -- NaN will shock you!" or "The best Spring annotations of 2021" kind of post. There aren't going to be any checklists, best practices, or tips on anything. I just wanted to write down an elegant solution to a mildly interesting problem.

# The Problem
We had a service dedicated to source product data from an end-of-life system, let's call it _Legacy Service_ for simplicity's sake. We triggered the service, let's call it _Legacy Accumulator_, or _Accumulator_ if product data was missing certain pieces of information. You can regard _Accumulator_ as some sort of bridge: It brought the data up to the standard we required later on in our context.

Seems unfortunate enough already, but here's the catch:

* _Accumulator_ had a production instances querying the production _Legacy Service_.
* _Legacy Service_ did not provide an integration instance of itself. (As is tradition ...)
* Our integration services handled the same product data as our production services.
* Our integration system should _not_, however, at any stage query _Legacy Service_'s production instance. We had no business interfering with production systems for our testing needs. We also actually din't _want_ to query _Legacy Service_ twice because that cost quite a bit of time.

# The Solution
Our solution was providing data to integration storage by means of **asynchronously mirroring the `save` (to storage) calls on production to integration**. Hacky, but it had a few upsides:

1) It barely taxed our production instance of _Accumulator_, and there was no extra load on _Legacy Service_. Spring picked an idle thread from the pool and gave it a bit of I/O work to handle. 
2) Using Spring, and building upon a well-designed codebase by my former team, the extra functionality was quite easy to add. Moreover, it would then be quick to get rid of or deactivate. Everything hinged upon two corresponding extra properties being set at application startup, which we could provide through environment variables passed along to the Docker container.

## Our Starting Point
The bean dealing with storage access used to be provided by reading the two necessary properties and calling our storage client library's factory method with them:
```
@Configuration
class StorageConfig {

    @Bean
    StorageAccess storageAccess(
        @Value("${storage.env}) env,
        @Value("${storage.pw}) pw,
    ) {
        return Storage.createStorageAccess(env, pw);
    }
}
```

`StorageAccess` was an interface with methods like `save(ProductData data)` and it was `@Autowired` in a `@Service` bean of Accumulator, serving the application when persisting product data.

## Conditionality
In order to keep interference to a minimum we provided the mirroring functionality through an additional conditional bean:

```
@Configuration
@ConditionalOnProperty({"${mirroring.storage.env}", "${mirroring.storage.pw}"})
class MirroringStorageConfig {
    
    @Bean("mirroringStorageAccess")
    StorageAccess mirroringStorageAccess(
        @Value("${mirroring.storage.env}) env,
        @Value("${mirroring.storage.pw}) pw,
    ) {
        return Storage.createStorageAccess(env, pw);
    }
}
```

We could've left out the qualifier for the bean, but it made it easier to read and less prone to breakage upon refactoring method names. The above mentioned OG `StorageAccess` also received a qualifier: "storageAccess".

In order to make mirroring transparent at the call-site we chose to provide our own implementation of `StorageAccess` which delegated its work to the regular storage access implementation for all but one of its operations: Only when calling `save(...)` it would delegate to the regular storage access instance first _and then, asynchronously_, to the mirroring storage access:

```
@Component("saveMirroringStorageAccess")
@ConditionalOnConfiguration(MirroringStorageConfig.class)
@Primary
class SaveMirroringStorageAccess implements StorageAccess {

    @Autowired
    @Qualifier("storageAccess")
    private final StorageAccess primary;

    @Autowired
    @Qualifier("mirroringStorageAccess")
    private final StorageAccess mirror;

    @Override
    public Product load(String id) {
        return primary.load(id);
    }

    // More methods delegated to exclusively primary.

    @Override
    public void save(ProductData data) {
        primary.save(data);
        mirror.save(data);
    }
}
```

What's neat about that? 
1. The whole config existed only when mirroring was activated through the two corresponding properties due to `@ConditionalOnProperty(...)` and  `@ConditionalOnConfiguration(...)`. 
2. Devs did not have to change call-site code or to even care which implementation to inject. As soon as the properties were set, our mirroring implementation would be the primary one due to what is essentially a conditional `@Primary` bean.

To be on the safe side we added a test, making sure the wiring actually worked:

```
// Set basic common properties to get application started.
@TestPropertySource(properties = { accumulator.foo=1, accumulator.bar=2 })
class StorageAccessTest {

    @Nested
    @SpringBootTest(properties = { storage.env=test, storage.pw=mypass })
    class RegularStorageAccess {

        @Autowired
        private StorageAccess storageAccess;

        @Test
        void providesRegularStorageAccess() {
            assertThat(storageAccess).isInstanceOf(StorageAccessImpl.class);
        }
    }

    @Nested
    @SpringBootTest(properties = { 
        storage.env=test, 
        storage.pw=mypass,
        mirroring.storage.env=mirror,
        mirroring.storage.pw=mymirrorpass
    })
    class MirrorStorageAccess {

        @Autowired
        private StorageAccess storageAccess;

        @Test
        void providesSaveMirroringStorageAccessIfPropertiesAreProvided() {
            assertThat(storageAccess).isInstanceOf(SaveMirroringStorageAccess.class);
        }
    }
}
```

Yes, it's started full blown contexts and tested implementation details, and yes, it bordered on testing framework functionality! But as I said: We did want to be sure our whole context wass still valid -- better safe than sorry.

## Asynchrony
What we've seen above wasn't the whole truth as asynchrony was yet missing from the new implementation. Since Spring has an `@Async` annotation to make classes or methods asynchronous that should be rather succint, yes? Well, _quite_ succint, since proxying forbade us from just annotating the `save` method within `SaveMirroringStorageAccess` -- the async method had to sit in a different class from the call-site.
We achieved that with a wrapping class:

```
@Component
@ConditionalOnConfiguration(MirroringStorageConfig.class)
class AsyncMirroringWrapper {
    
    @Autowired
    @Qualifier("mirroringStorageAccess")
    private final StorageAccess mirror;

    @Async
    void saveAsync(ProductData data) {
        mirror.save(data);
    }
}
```

And injected said wrapper to our _delegating_ `StorageAccess` version instead:

```
// ...
class SaveMirroringStorageAccess implements StorageAccess {

    @Autowired
    @Qualifier("storageAccess")
    private final StorageAccess primary;

    // The following field used to be the mirroring `StorageAccess` instance.
    @Autowired
    private final AsyncMirroringWrapper wrapper; 

    // ...

    @Override
    public void save(ProductData data) {
        primary.save(data);
        wrapper.saveAsync(data);
    }
}
```
Asynchrony might introduce a thousand problems to your code, but in our case the main issue with regards to testing was to make sure the code _did actually run async_ and not block.

```
@SpringBootTest(properties = { 
    storage.env=test, 
    storage.pw=mypass,
    mirroring.storage.env=mirror,
    mirroring.storage.pw=mymirrorpass,
    // other props
})
@AutoConfigureMockMvc
class SaveMirroringStorageAccessTest {

    @Autowired
    prvivate MockMvc mvc;

    @MockBean(name = "storageAccess")
    private StorageAccess primary;

    @MockBean(name = "mirroringStorageAccess")
    private StorageAccess mirror;

    @SpyBean(name= "saveMirroringStorageAccess")
    private StorageAccess combined;
    
    @Test
    void doesNotBlockWhenMirroring() {
        AtomicBoolean hasMirrored = new AtomicBoolean(false);
        CountDownLatch latch = new CountDownLatch(1);

        doAnswer(inv -> {
            latch.await();
            hasMirrored.compareAndSet(false, true);
            return null;
        }).when(mirror).save(any());
        
        mvc.perform(post("/", ...))
            .andExpect(status().is2xxSuccessful());

        assertThat(hasMirrored).isFalse();
        verify(primary, times(1)).save(any());

        latch.countDown();

        await().atMost(1, SECONDS).untilTrue(hasMirrored);
    }
}
```

Again, internal knowledge of how classes were set up, mocking and stubbing within a `@SpringBootTest`, and other violations of what is good and right. I know... But that was the price for our feature having minimal impact on existing code. And we chose to pay it. And for testing non-trivial, non-business-logic, async code it reads pretty nicely in my opinion.

# Conclusion
So much for our unorthodox solution to collecting data for integration testing. Would I recommend it to somebody else? Not if you can avoid it. Was it worth it? I hope so. Can we get rid of it again? Yes, it can be very easily deactivated or even removed without causing a bunch of follow-up patches to code that has been added in the meantime. Is the code easy to follow? I rather think so, thanks to Spring (and JUnit 5, Mockito, and Awaitility).