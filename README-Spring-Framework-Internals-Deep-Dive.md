# Spring Framework Internals --- Configuration Parsing, Post-Processors & Imports

> A structured deep-dive reconstructed from handwritten notes, with
> terminology normalized and a few important technical nuances
> clarified.

## Table of Contents

1.  [Big Picture](#1-big-picture)
2.  [Core Post-Processor Hierarchy](#2-core-post-processor-hierarchy)
3.  [Why ConfigurationClassPostProcessor Runs So
    Early](#3-why-configurationclasspostprocessor-runs-so-early)
4.  [What ConfigurationClassPostProcessor
    Does](#4-what-configurationclasspostprocessor-does)
5.  [ImportBeanDefinitionRegistrar
    Lifecycle](#5-importbeandefinitionregistrar-lifecycle)
6.  [BeanFactoryPostProcessor Execution
    Order](#6-beanfactorypostprocessor-execution-order)
7.  [How to Run a Custom BFPP Very
    Early](#7-how-to-run-a-custom-bfpp-very-early)
8.  [ApplicationContextInitializer](#8-applicationcontextinitializer)
9.  [PropertySource vs Placeholder
    Resolution](#9-propertysource-vs-placeholder-resolution)
10. [Why @Value Is Not Used Directly Inside
    Condition](#10-why-value-is-not-used-directly-inside-condition)
11. [What @Import Can Import](#11-what-import-can-import)
12. [ImportSelector Processing](#12-importselector-processing)
13. [DeferredImportSelector](#13-deferredimportselector)
14. [What an ImportSelector May
    Return](#14-what-an-importselector-may-return)
15. [End-to-End Context Refresh](#15-end-to-end-context-refresh)
16. [Interview-Level Takeaways](#16-interview-level-takeaways)

------------------------------------------------------------------------

## 1. Big Picture

A Spring `ApplicationContext` must discover and register the complete
set of `BeanDefinition`s **before normal singleton creation begins**.

One of the most important infrastructure components involved is:

`ConfigurationClassPostProcessor`

It is responsible for processing configuration classes and discovering
additional bean definitions contributed through mechanisms such as:

-   `@Configuration`
-   `@ComponentScan`
-   `@Import`
-   `@ImportResource`
-   `@Bean`
-   `ImportSelector`
-   `DeferredImportSelector`
-   `ImportBeanDefinitionRegistrar`

A useful mental model is:

``` mermaid
flowchart TD
    A["ApplicationContext.refresh()"] --> B["Prepare BeanFactory"]
    B --> C["invokeBeanFactoryPostProcessors()"]
    C --> D["ConfigurationClassPostProcessor"]
    D --> E["Parse configuration classes"]
    E --> F["@ComponentScan"]
    E --> G["@Import"]
    E --> H["@ImportResource"]
    E --> I["@Bean methods"]
    G --> J["ImportSelector"]
    G --> K["DeferredImportSelector"]
    G --> L["ImportBeanDefinitionRegistrar"]
    J --> M["More configuration/classes"]
    K --> M
    L --> N["Additional BeanDefinitions"]
    M --> N
    N --> O["Other BFPP/BDRPP processing"]
    O --> P["registerBeanPostProcessors()"]
    P --> Q["Instantiate non-lazy singletons"]
```

The central idea is:

> **Spring first builds and transforms the bean-definition model; only
> later does it create ordinary application objects.**

------------------------------------------------------------------------

## 2. Core Post-Processor Hierarchy

The handwritten notes capture this relationship:

``` text
BeanFactoryPostProcessor
        ^
        |
BeanDefinitionRegistryPostProcessor
        ^
        |
ConfigurationClassPostProcessor
```

More precisely:

``` mermaid
classDiagram
    class BeanFactoryPostProcessor {
        <<interface>>
        +postProcessBeanFactory(beanFactory)
    }

    class BeanDefinitionRegistryPostProcessor {
        <<interface>>
        +postProcessBeanDefinitionRegistry(registry)
    }

    class ConfigurationClassPostProcessor {
        +postProcessBeanDefinitionRegistry(registry)
        +postProcessBeanFactory(beanFactory)
    }

    BeanFactoryPostProcessor <|-- BeanDefinitionRegistryPostProcessor
    BeanDefinitionRegistryPostProcessor <|.. ConfigurationClassPostProcessor
```

### BeanFactoryPostProcessor

A `BeanFactoryPostProcessor` operates on the `BeanFactory` **after bean
definitions exist but before ordinary beans are instantiated**.

Typical use:

``` java
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(
        ConfigurableListableBeanFactory beanFactory);
}
```

### BeanDefinitionRegistryPostProcessor

`BeanDefinitionRegistryPostProcessor` extends `BeanFactoryPostProcessor`
and gets an **earlier hook**:

``` java
void postProcessBeanDefinitionRegistry(
    BeanDefinitionRegistry registry);
```

This matters because a BDRPP can **register additional bean
definitions**, and Spring can subsequently discover more post-processors
from those definitions.

### ConfigurationClassPostProcessor

`ConfigurationClassPostProcessor` is a Spring infrastructure BDRPP.

Its most important early work occurs through:

``` java
postProcessBeanDefinitionRegistry(...)
```

That is where configuration-class processing begins.

------------------------------------------------------------------------

## 3. Why ConfigurationClassPostProcessor Runs So Early

Configuration processing can itself discover new bean definitions.

For example:

``` java
@Configuration
@Import(MyRegistrar.class)
class AppConfig {
}
```

`MyRegistrar` might register:

``` text
paymentService
auditService
securityInfrastructure
...
```

Those definitions need to exist **before later BeanFactoryPostProcessors
inspect or modify the registry**.

Conceptually:

``` mermaid
sequenceDiagram
    participant AC as ApplicationContext
    participant CCPP as ConfigurationClassPostProcessor
    participant Parser as ConfigurationClassParser
    participant Registrar as ImportBeanDefinitionRegistrar
    participant Registry as BeanDefinitionRegistry
    participant BFPP as Later BFPPs

    AC->>CCPP: postProcessBeanDefinitionRegistry()
    CCPP->>Parser: parse configuration classes
    Parser->>Parser: discover imports/scans/@Bean methods
    Parser->>Registrar: discover/store registrar
    CCPP->>Registrar: registerBeanDefinitions(...)
    Registrar->>Registry: add BeanDefinitions
    AC->>BFPP: execute later processors
    BFPP->>Registry: inspect complete definitions
```

Therefore, the important principle is:

> Configuration-derived definitions should become available early enough
> for subsequent registry/factory post-processing.

------------------------------------------------------------------------

## 4. What ConfigurationClassPostProcessor Does

At a high level, its registry-processing phase:

1.  Locates configuration-class candidates.
2.  Parses them through `ConfigurationClassParser`.
3.  Processes component scanning.
4.  Processes imports.
5.  Processes imported resources.
6.  Discovers `@Bean` methods.
7.  Handles `ImportSelector` and `DeferredImportSelector`.
8.  Discovers `ImportBeanDefinitionRegistrar`s.
9.  Uses `ConfigurationClassBeanDefinitionReader` to load the resulting
    bean definitions.

Conceptual flow:

``` text
ConfigurationClassPostProcessor
        |
        v
ConfigurationClassParser
        |
        +--> @ComponentScan
        |
        +--> @Import
        |       |
        |       +--> ordinary configuration/component class
        |       +--> ImportSelector
        |       +--> DeferredImportSelector
        |       +--> ImportBeanDefinitionRegistrar
        |
        +--> @ImportResource
        |
        +--> @Bean methods
        |
        v
ConfigurationClassBeanDefinitionReader
        |
        v
BeanDefinitionRegistry
```

### Important distinction

The **parser** discovers and models configuration metadata.

The **reader** turns that parsed configuration model into actual bean
definitions and invokes registrar callbacks at the appropriate stage.

That distinction is useful when reading Spring source code.

------------------------------------------------------------------------

## 5. ImportBeanDefinitionRegistrar Lifecycle

Suppose:

``` java
@Configuration
@Import(MyRegistrar.class)
class AppConfig {
}
```

and:

``` java
public class MyRegistrar
        implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {

        RootBeanDefinition bd =
            new RootBeanDefinition(MyService.class);

        registry.registerBeanDefinition("myService", bd);
    }
}
```

### Phase 1 --- configuration parsing

While parsing `AppConfig`, Spring encounters:

``` java
@Import(MyRegistrar.class)
```

The registrar is identified and associated with the current
configuration model.

A useful simplified mental model is:

``` text
ConfigurationClassParser
        |
        +--> discovers MyRegistrar
        |
        +--> associates registrar with ConfigurationClass metadata
```

### Phase 2 --- load bean definitions

After parsing, `ConfigurationClassBeanDefinitionReader` processes the
parsed `ConfigurationClass` objects.

Conceptually:

``` text
ConfigurationClassBeanDefinitionReader
        |
        +--> loadBeanDefinitions(...)
                |
                +--> process @Bean methods
                +--> process imported resources
                +--> invoke ImportBeanDefinitionRegistrar
```

The registrar callback then executes:

``` java
registrar.registerBeanDefinitions(...);
```

and can dynamically add bean definitions to the registry.

### Phase 3 --- later post-processors see those definitions

After registrar-contributed definitions are registered, normal
registry/factory post-processing continues.

``` mermaid
flowchart TD
    A["Parse @Configuration"] --> B["Encounter @Import(MyRegistrar.class)"]
    B --> C["Discover/store registrar with configuration metadata"]
    C --> D["Finish configuration parsing"]
    D --> E["ConfigurationClassBeanDefinitionReader"]
    E --> F["Invoke registrar.registerBeanDefinitions(...)"]
    F --> G["New BeanDefinitions enter registry"]
    G --> H["Later BFPP/BDRPP processing"]
```

### Key clarification

It is tempting to say that Spring simply discovers the registrar and
immediately invokes it while parsing `@Import`.

A better model is:

> The registrar is **discovered during configuration parsing**, while
> its `registerBeanDefinitions(...)` callback is performed as Spring
> loads bean definitions from the parsed configuration model.

This separation keeps parsing/discovery distinct from registration.

------------------------------------------------------------------------

## 6. BeanFactoryPostProcessor Execution Order

Spring distinguishes post-processors using:

1.  `PriorityOrdered`
2.  `Ordered`
3.  no ordering interface

Conceptually:

``` text
PriorityOrdered
      |
      v
Ordered
      |
      v
Unordered
```

Within each category, ordering rules apply.

### ConfigurationClassPostProcessor

`ConfigurationClassPostProcessor` implements `PriorityOrdered` and uses
very high precedence.

This is intentional: configuration parsing can discover many additional
bean definitions and even additional post-processors.

### PropertySourcesPlaceholderConfigurer

`PropertySourcesPlaceholderConfigurer` is a BFPP used to resolve
`${...}` placeholders in bean-definition property values and annotation
metadata handled through the bean factory.

Example:

``` java
@Value("${database.url}")
private String url;
```

Its role is different from configuration-class parsing.

### Simplified execution picture

``` mermaid
flowchart TD
    A["invokeBeanFactoryPostProcessors()"] --> B["Programmatically added processors"]
    B --> C["BeanDefinitionRegistryPostProcessors"]
    C --> D["PriorityOrdered"]
    D --> E["Ordered"]
    E --> F["Remaining BDRPPs"]
    F --> G["BeanFactoryPostProcessors"]
    G --> H["PriorityOrdered"]
    H --> I["Ordered"]
    I --> J["Remaining BFPPs"]
```

### Nuance: programmatically registered processors

Processors added directly with:

``` java
context.addBeanFactoryPostProcessor(...)
```

are treated separately from processors discovered as bean definitions.

Their programmatic registration order matters; ordering interfaces are
not the mechanism Spring uses to sort that initial manually supplied
list.

------------------------------------------------------------------------

## 7. How to Run a Custom BFPP Very Early

Suppose you want custom processing to occur before auto-detected
infrastructure processors.

Simply declaring:

``` java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class MyProcessor implements BeanFactoryPostProcessor {
}
```

does **not** guarantee that it executes before Spring's internal
configuration processing.

Why?

Because Spring must first discover that component as a bean definition,
and configuration processing itself may be responsible for discovering
it.

### Technique 1 --- programmatically register before refresh

``` java
AnnotationConfigApplicationContext context =
        new AnnotationConfigApplicationContext();

context.addBeanFactoryPostProcessor(
        new MyEarlyBeanFactoryPostProcessor());

context.register(AppConfig.class);
context.refresh();
```

Conceptually:

``` text
create context
    |
    +--> addBeanFactoryPostProcessor(custom)
    |
    +--> register configuration
    |
    +--> refresh()
            |
            +--> manually supplied BFPP
            +--> auto-detected infrastructure processors
            +--> other BFPPs
```

This is fundamentally different from declaring the processor as an
ordinary Spring bean.

------------------------------------------------------------------------

## 8. ApplicationContextInitializer

An even earlier extension point is `ApplicationContextInitializer`.

Example:

``` java
public class MyContextInitializer
        implements ApplicationContextInitializer<
            ConfigurableApplicationContext> {

    @Override
    public void initialize(
            ConfigurableApplicationContext context) {

        context.addBeanFactoryPostProcessor(
            new MyEarlyBeanFactoryPostProcessor());
    }
}
```

For Spring Boot, it can be supplied programmatically:

``` java
new SpringApplicationBuilder(MyApplication.class)
        .initializers(new MyContextInitializer())
        .run(args);
```

Historically/framework-specifically, initializers may also be discovered
through framework bootstrap metadata mechanisms.

### Timing

``` mermaid
flowchart LR
    A["Create ApplicationContext"] --> B["ApplicationContextInitializer"]
    B --> C["Register early BFPP / customize context"]
    C --> D["refresh()"]
    D --> E["Configuration parsing"]
    E --> F["BFPP chain"]
    F --> G["BeanPostProcessors"]
    G --> H["Singleton creation"]
```

### Important conclusion

You cannot make an ordinary auto-discovered BFPP reliably precede
configuration-class processing merely by assigning a smaller `@Order`
value.

If you truly require a pre-refresh hook, use an appropriate
**programmatic/bootstrap extension point**.

------------------------------------------------------------------------

## 9. PropertySource vs Placeholder Resolution

### `@PropertySource`

Example:

``` java
@Configuration
@PropertySource("classpath:custom.properties")
class AppConfig {
}
```

Its purpose is to add a property source to Spring's:

``` text
Environment
```

Then values can be accessed with:

``` java
environment.getProperty("app.name");
```

### Does `@PropertySource` itself resolve `${...}`?

No.

It primarily contributes properties to the `Environment`.

Placeholder resolution in bean definitions historically requires an
appropriate placeholder configurer such as:

``` java
PropertySourcesPlaceholderConfigurer
```

### Spring Boot

Spring Boot already loads standard external configuration such as:

``` text
application.properties
application.yml
```

into the `Environment`.

Therefore, `@PropertySource` is usually unnecessary for those standard
files.

Use it mainly when introducing an additional custom property resource.

### Mental model

``` mermaid
flowchart TD
    A["application.properties / YAML / @PropertySource"] --> B["Environment / PropertySources"]
    B --> C["Environment.getProperty(...)"]
    B --> D["Placeholder resolution infrastructure"]
    D --> E["${...} in supported bean metadata / @Value"]
```

------------------------------------------------------------------------

## 10. Why @Value Is Not Used Directly Inside Condition

A Spring `Condition` participates during configuration/bean-definition
processing.

Example:

``` java
public class MyCondition implements Condition {

    @Override
    public boolean matches(
            ConditionContext context,
            AnnotatedTypeMetadata metadata) {

        String mode =
            context.getEnvironment()
                   .getProperty("app.mode");

        return "prod".equals(mode);
    }
}
```

### Why not this?

``` java
@Value("${app.mode}")
private String mode;
```

Because condition evaluation happens **very early**, before normal bean
instantiation and dependency injection.

At that point:

``` text
@Configuration parsing
        |
        +--> evaluate @Conditional
        |
        +--> decide whether definitions should exist
        |
        v
normal bean creation happens much later
```

Therefore, a condition should use the facilities supplied by
`ConditionContext`, especially:

``` java
context.getEnvironment()
```

### Correct mental model

``` mermaid
flowchart LR
    A["Environment already available"] --> B["Condition.matches(...)"]
    B --> C{"Condition true?"}
    C -->|Yes| D["Register/process configuration"]
    C -->|No| E["Skip configuration"]
    D --> F["Later: instantiate beans"]
    F --> G["@Value / dependency injection"]
```

------------------------------------------------------------------------

## 11. What @Import Can Import

`@Import` can bring several kinds of classes into configuration
processing.

``` java
@Import({
    MyConfiguration.class,
    MySelector.class,
    MyRegistrar.class
})
```

Important categories:

-   ordinary configuration/component classes
-   `ImportSelector`
-   `DeferredImportSelector`
-   `ImportBeanDefinitionRegistrar`

Conceptually:

``` mermaid
flowchart TD
    A["@Import"] --> B{"Imported type"}
    B --> C["@Configuration / ordinary class"]
    B --> D["ImportSelector"]
    B --> E["DeferredImportSelector"]
    B --> F["ImportBeanDefinitionRegistrar"]
    C --> G["Process as imported configuration/class"]
    D --> H["selectImports()"]
    E --> I["deferred selectImports()"]
    F --> J["registerBeanDefinitions()"]
```

This mechanism is the foundation behind many Spring `@Enable...`
annotations.

For example, an enabling annotation can effectively be:

``` java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(SomeInfrastructureRegistrar.class)
public @interface EnableSomething {
}
```

------------------------------------------------------------------------

## 12. ImportSelector Processing

An `ImportSelector` returns class names:

``` java
public class MySelector implements ImportSelector {

    @Override
    public String[] selectImports(
            AnnotationMetadata importingClassMetadata) {

        return new String[] {
            FeatureConfiguration.class.getName()
        };
    }
}
```

Spring then processes the returned classes through its configuration
parsing machinery.

### Important consequence

The returned classes may themselves lead to more imports.

``` text
ImportSelector
    |
    +--> returns ConfigA
             |
             +--> @Import(SelectorB)
                         |
                         +--> returns ConfigC
```

So configuration processing is naturally recursive.

### General order

A useful high-level model is:

``` text
1. Parse current configuration
2. Process immediate ImportSelector results
3. Recursively process classes returned by selectors
4. Process deferred selectors at their deferred stage
5. Load definitions and invoke registrars
```

Do not rely on the literal order of classes inside an `@Import({...})`
array as a universal execution-order contract between selectors and
registrars. Spring's parser categorizes imported types and processes
them according to framework semantics.

------------------------------------------------------------------------

## 13. DeferredImportSelector

`DeferredImportSelector` extends the import-selection model by deferring
selection until the parser has processed the current round of
configuration classes.

It is useful when an import decision should see a more complete
configuration picture.

Conceptually:

``` mermaid
sequenceDiagram
    participant P as ConfigurationClassParser
    participant C as Regular Configurations
    participant I as ImportSelector
    participant D as DeferredImportSelector
    participant R as Definition Reader / Registrars

    P->>C: Parse configuration classes
    P->>I: Process immediate selectors
    I-->>P: Return imports
    P->>P: Parse returned imports recursively
    P->>D: Process deferred selector group
    D-->>P: Return deferred imports
    P->>P: Parse deferred imports
    P->>R: Load BeanDefinitions / invoke registrars
```

Spring Boot's auto-configuration infrastructure has historically made
important use of deferred import selection.

### Why defer?

Because framework-level configuration often needs to make decisions
after user configuration has had an opportunity to contribute metadata.

Mental model:

``` text
User configuration first
        |
        v
Deferred framework imports
        |
        v
Final configuration model
```

------------------------------------------------------------------------

## 14. What an ImportSelector May Return

`ImportSelector.selectImports(...)` technically returns:

``` java
String[]
```

Those strings are fully qualified class names.

The returned classes are then interpreted by Spring according to what
they are.

They may represent:

### Configuration classes

``` java
@Configuration
class FeatureConfig {
}
```

This is the most intuitive use.

### Another ImportSelector

Spring can recursively process the returned selector.

``` text
Selector A
   |
   +--> Selector B
           |
           +--> Configuration C
```

### ImportBeanDefinitionRegistrar

A returned/imported registrar can participate in the registrar mechanism
and later contribute bean definitions.

### Ordinary classes

A class imported through Spring configuration processing can become a
bean definition even if it does not carry `@Component`.

That is why:

``` java
@Import(MyService.class)
```

can register `MyService` even without:

``` java
@Component
```

### Summary

``` mermaid
flowchart LR
    A["ImportSelector.selectImports()"] --> B["String[] class names"]
    B --> C["Configuration class"]
    B --> D["Another selector"]
    B --> E["Registrar"]
    B --> F["Ordinary class"]
    C --> G["Configuration processing"]
    D --> G
    E --> H["Registrar callback later"]
    F --> I["Imported bean definition"]
```

------------------------------------------------------------------------

## 15. End-to-End Context Refresh

The following diagram ties the entire topic together.

``` mermaid
flowchart TD
    A["ApplicationContext created"] --> B["ApplicationContextInitializer(s)"]
    B --> C["refresh()"]

    C --> D["Prepare/obtain BeanFactory"]
    D --> E["invokeBeanFactoryPostProcessors()"]

    E --> F["Programmatically supplied BFPPs"]
    E --> G["BeanDefinitionRegistryPostProcessors"]

    G --> H["ConfigurationClassPostProcessor"]
    H --> I["ConfigurationClassParser"]

    I --> J["@ComponentScan"]
    I --> K["@Import"]
    I --> L["@ImportResource"]
    I --> M["@Bean"]

    K --> N["Immediate ImportSelector"]
    K --> O["DeferredImportSelector"]
    K --> P["ImportBeanDefinitionRegistrar"]

    N --> Q["More imported classes/configuration"]
    O --> Q
    Q --> I

    I --> R["ConfigurationClassBeanDefinitionReader"]
    R --> S["Register @Bean/configuration definitions"]
    R --> T["Invoke registrar callbacks"]
    T --> U["Registrar-contributed BeanDefinitions"]

    S --> V["Registry now richer"]
    U --> V

    V --> W["Remaining BDRPP/BFPP processing"]
    W --> X["registerBeanPostProcessors()"]
    X --> Y["BeanPostProcessors ready"]
    Y --> Z["finishBeanFactoryInitialization()"]
    Z --> AA["Instantiate non-lazy singletons"]
    AA --> AB["BPP before/after initialization; proxies may be created"]
```

### Definition phase vs object phase

This is the most useful architectural distinction:

  -----------------------------------------------------------------------------------
  Phase                            Works primarily with       Examples
  -------------------------------- -------------------------- -----------------------
  Configuration parsing            metadata / bean            `@Configuration`,
                                   definitions                `@Import`, selectors

  Registry post-processing         `BeanDefinitionRegistry`   BDRPP, registrars

  Factory post-processing          bean-definition metadata / BFPP, placeholder
                                   factory                    configurer

  Bean post-processor registration BPP definitions/instances  AOP infrastructure,
                                                              autowiring processors

  Bean instantiation               actual objects             constructors,
                                                              dependency injection

  Initialization/post-processing   actual bean instances      BPP callbacks, proxy
                                                              creation
  -----------------------------------------------------------------------------------

------------------------------------------------------------------------

## 16. Interview-Level Takeaways

### 1. BFPP and BPP solve different problems

``` text
BeanFactoryPostProcessor
    -> operates before normal bean creation
    -> modifies bean definitions / factory metadata

BeanPostProcessor
    -> operates on actual bean instances
    -> participates during bean creation
```

### 2. BDRPP is earlier and more powerful for registration

``` text
BeanFactoryPostProcessor
        ^
        |
BeanDefinitionRegistryPostProcessor
```

A BDRPP can contribute new definitions before ordinary BFPP processing
completes.

### 3. ConfigurationClassPostProcessor is central to annotation configuration

Without its configuration-class processing, mechanisms such as:

``` text
@Configuration
@ComponentScan
@Import
@Bean
```

would not be interpreted in the normal annotation-configured way.

### 4. ImportBeanDefinitionRegistrar is not a normal BFPP

A registrar is a callback used **by configuration-class processing** to
contribute bean definitions.

``` text
Configuration parsing
       |
       v
discover registrar
       |
       v
definition reader
       |
       v
registrar.registerBeanDefinitions(...)
```

### 5. ImportSelector chooses imports; Registrar directly registers definitions

``` text
ImportSelector
    -> returns class names
    -> Spring processes those classes

ImportBeanDefinitionRegistrar
    -> receives BeanDefinitionRegistry
    -> directly registers definitions
```

### 6. DeferredImportSelector postpones import selection

It is designed for imports that should be considered after the current
configuration parsing has progressed further.

### 7. @Conditional happens before ordinary bean injection

Therefore:

``` java
context.getEnvironment().getProperty(...)
```

is appropriate inside a `Condition`, rather than depending on normal
`@Value` injection into the condition object.

### 8. @PropertySource and placeholder resolution are separate concepts

``` text
@PropertySource
    -> adds a source to Environment

PropertySourcesPlaceholderConfigurer
    -> participates in resolving ${...} placeholders
```

### 9. Programmatic BFPP registration is special

``` java
context.addBeanFactoryPostProcessor(...)
```

registers a processor directly with the context rather than waiting for
it to be discovered as a bean.

This can provide an earlier hook than ordinary auto-detection.

### 10. The deepest mental model

Think of Spring startup as two broad worlds:

``` mermaid
flowchart LR
    A["WORLD 1<br/>Build the object blueprint"] --> B["BeanDefinitions"]
    B --> C["Configuration parsing"]
    C --> D["BDRPP"]
    D --> E["BFPP"]
    E --> F["WORLD 2<br/>Create and enhance objects"]
    F --> G["Instantiate beans"]
    G --> H["Dependency injection"]
    H --> I["BeanPostProcessors"]
    I --> J["AOP proxies / initialized beans"]
```

**Spring does not begin by creating your services. It first constructs,
expands, filters, and transforms a registry describing what those
services should be.**

That single idea explains why configuration parsing, `ImportSelector`,
registrars, conditions, BDRPPs, and BFPPs all occur before ordinary
singleton creation.

------------------------------------------------------------------------

## Quick Revision Map

``` text
ApplicationContextInitializer
        |
        v
refresh()
        |
        v
invokeBeanFactoryPostProcessors()
        |
        +--> programmatically supplied processors
        |
        +--> BeanDefinitionRegistryPostProcessors
                |
                +--> ConfigurationClassPostProcessor
                        |
                        +--> ConfigurationClassParser
                        |       |
                        |       +--> @ComponentScan
                        |       +--> @Import
                        |       |      +--> ImportSelector
                        |       |      +--> DeferredImportSelector
                        |       |      +--> Registrar discovery
                        |       +--> @ImportResource
                        |       +--> @Bean metadata
                        |
                        +--> ConfigurationClassBeanDefinitionReader
                                |
                                +--> load BeanDefinitions
                                +--> invoke Registrar callbacks
        |
        +--> remaining BFPP processing
        |
        v
registerBeanPostProcessors()
        |
        v
instantiate singleton beans
        |
        v
BeanPostProcessor callbacks / proxies
```

------------------------------------------------------------------------

## Source

Prepared from the supplied handwritten notes, in image order.
Terminology, code formatting, and lifecycle descriptions were normalized
for readability, and ambiguous lifecycle statements were refined to
better match Spring's conceptual processing model.
