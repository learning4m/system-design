# Spring Container Internals and AOP --- Study Notes

> A consolidated reference from our discussion: Spring configuration
> processing, import mechanisms, post-processors, proxy creation,
> Advisors, Interceptors, JDK proxies, and CGLIB.

## Table of Contents

1.  [Big Picture: Spring Startup](#1-big-picture-spring-startup)
2.  [What `@Enable...` Annotations Do](#2-what-enable-annotations-do)
3.  [`@EnableScheduling` Example](#3-enablescheduling-example)
4.  [Configuration Phase:
    `ConfigurationClassPostProcessor`](#4-configuration-phase-configurationclasspostprocessor)
5.  [`@Conditional` and `@Profile`](#5-conditional-and-profile)
6.  [`ImportSelector`](#6-importselector)
7.  [`DeferredImportSelector`](#7-deferredimportselector)
8.  [`ImportBeanDefinitionRegistrar`](#8-importbeandefinitionregistrar)
9.  [Selector vs Registrar](#9-selector-vs-registrar)
10. [Can Selectors and Registrars
    Chain?](#10-can-selectors-and-registrars-chain)
11. [BDRPP vs BFPP vs BPP](#11-bdrpp-vs-bfpp-vs-bpp)
12. [`PostProcessorRegistrationDelegate`](#12-postprocessorregistrationdelegate)
13. [External Framework BDRPP
    Registration](#13-external-framework-bdrpp-registration)
14. [`@MapperScan` / MyBatis Example](#14-mapperscan--mybatis-example)
15. [Why `static @Bean` for
    BFPP/BDRPP](#15-why-static-bean-for-bfppbdrpp)
16. [Where AOP Proxy Creation Fits](#16-where-aop-proxy-creation-fits)
17. [`@EnableAspectJAutoProxy`](#17-enableaspectjautoproxy)
18. [`AbstractAutoProxyCreator`](#18-abstractautoproxycreator)
19. [Advisor vs Advice vs
    Interceptor](#19-advisor-vs-advice-vs-interceptor)
20. [Multiple Advices and One Proxy](#20-multiple-advices-and-one-proxy)
21. [JDK Proxy Pseudocode](#21-jdk-proxy-pseudocode)
22. [CGLIB Proxy Pseudocode](#22-cglib-proxy-pseudocode)
23. [`ReflectiveMethodInvocation`](#23-reflectivemethodinvocation)
24. [JDK Proxy vs CGLIB Caveats](#24-jdk-proxy-vs-cglib-caveats)
25. [`@Transactional` + `@Async` +
    `@Scheduled`](#25-transactional--async--scheduled)
26. [End-to-End Mental Model](#26-end-to-end-mental-model)
27. [Quick Revision Sheet](#27-quick-revision-sheet)

------------------------------------------------------------------------

## 1. Big Picture: Spring Startup

A useful simplified lifecycle is:

``` text
AbstractApplicationContext.refresh()
        |
        v
invokeBeanFactoryPostProcessors()
        |
        +--> BeanDefinitionRegistryPostProcessors
        |       |
        |       +--> ConfigurationClassPostProcessor
        |       +--> Remaining BDRPPs
        |
        +--> BeanFactoryPostProcessors
        |
        v
registerBeanPostProcessors()
        |
        v
Instantiate singleton beans
        |
        +--> dependency injection
        +--> initialization
        +--> BeanPostProcessors
        +--> AOP proxy creation
```

The most important distinction is:

``` text
Early phases                  Later phases
------------                  ------------
BeanDefinition work           Bean instance work

CCPP                          Constructors
BDRPP                         @Autowired
BFPP                          @PostConstruct
                              BPP
                              AOP proxies
```

------------------------------------------------------------------------

## 2. What `@Enable...` Annotations Do

Spring does not have one universal engine that searches for annotation
names beginning with `@Enable`.

An `@Enable...` annotation is normally a composed annotation using
`@Import`.

Examples conceptually:

``` java
@Import(SchedulingConfiguration.class)
public @interface EnableScheduling {
}
```

or:

``` java
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
}
```

Therefore:

``` text
@EnableSomething
      |
      v
@Import(...)
      |
      +--> Configuration class
      +--> ImportSelector
      +--> DeferredImportSelector
      +--> ImportBeanDefinitionRegistrar
```

The imported infrastructure enables the feature.

------------------------------------------------------------------------

## 3. `@EnableScheduling` Example

Conceptually:

``` text
@EnableScheduling
      |
      v
@Import(SchedulingConfiguration.class)
      |
      v
ScheduledAnnotationBeanPostProcessor BeanDefinition
      |
      v
BPP is instantiated and registered
      |
      v
Application beans are created
      |
      v
BPP detects @Scheduled methods
      |
      v
Methods are registered with a scheduler
```

Important: `@EnableScheduling` itself does not scan every bean and run
scheduled methods.

It causes the scheduling infrastructure to be registered.

------------------------------------------------------------------------

## 4. Configuration Phase: `ConfigurationClassPostProcessor`

`ConfigurationClassPostProcessor` (CCPP) is a very important:

``` java
BeanDefinitionRegistryPostProcessor
```

It processes annotation-based configuration.

Conceptually:

``` text
ConfigurationClassPostProcessor
        |
        v
ConfigurationClassParser
        |
        +--> @Configuration
        +--> @ComponentScan
        +--> @Import
        +--> @ImportResource
        +--> @Bean
        +--> @Conditional
```

Its purpose is primarily to turn configuration metadata into
`BeanDefinition`s.

### Important distinction

`ConfigurationClassParser` is an internal parser.

`ConfigurationClassPostProcessor` is the actual BDRPP participating in
the container lifecycle.

### What does Phase 1 produce?

Mostly metadata:

``` text
BeanDefinitionRegistry
    |
    +-- orderService BeanDefinition
    +-- customerService BeanDefinition
    +-- transaction infrastructure definitions
    +-- auto-proxy creator definition
    +-- framework post-processor definitions
```

At this point most application beans have **not been instantiated**.

------------------------------------------------------------------------

## 5. `@Conditional` and `@Profile`

`@Profile` is built using `@Conditional`.

Conceptually:

``` java
@Conditional(ProfileCondition.class)
public @interface Profile {
}
```

During configuration parsing Spring uses a `ConditionEvaluator`.

``` text
Configuration candidate
        |
        v
ConditionEvaluator.shouldSkip(...)
        |
        +-- condition false --> skip
        |
        +-- condition true  --> continue processing
```

For a skipped configuration class, its configuration contribution is not
processed.

For example:

``` java
@Configuration
@Conditional(MyCondition.class)
@Import(MyRegistrar.class)
class Config {
}
```

If the condition fails:

``` text
Config skipped
   |
   +--> its @Import is not processed
   +--> MyRegistrar is not discovered through that import
```

Conditions can also be associated with configuration methods such as
`@Bean` methods.

------------------------------------------------------------------------

## 6. `ImportSelector`

Interface conceptually:

``` java
public interface ImportSelector {

    String[] selectImports(
        AnnotationMetadata importingClassMetadata);
}
```

It returns **fully-qualified class names**.

Example:

``` java
class MySelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        return new String[] {
            ConfigA.class.getName(),
            ConfigB.class.getName()
        };
    }
}
```

The returned imports may lead Spring to process:

-   configuration classes,
-   ordinary imported classes,
-   another `ImportSelector`,
-   a `DeferredImportSelector`,
-   an `ImportBeanDefinitionRegistrar`.

Think:

> `ImportSelector` selects **classes to import**.

------------------------------------------------------------------------

## 7. `DeferredImportSelector`

`DeferredImportSelector` is an `ImportSelector` whose selection is
deferred until regular configuration parsing has progressed.

Conceptually:

``` text
Parse regular configuration
        |
        +--> normal ImportSelector runs
        |
        +--> DeferredImportSelector collected
        |
        v
regular parsing reaches deferred-import stage
        |
        v
process deferred selectors
        |
        v
parse their selected imports
```

### Spring Boot

A key example is:

``` text
AutoConfigurationImportSelector
        |
        v
DeferredImportSelector
```

This supports Boot auto-configuration being considered after user
configuration has been discovered sufficiently for conditional
auto-configuration decisions.

In modern Spring Boot, auto-configuration candidates are listed in:

``` text
META-INF/spring/
org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Older Boot versions used `spring.factories` for
`EnableAutoConfiguration` candidates.

Not every `spring.factories` entry was a deferred import.
`spring.factories` historically supported many unrelated extension
points.

------------------------------------------------------------------------

## 8. `ImportBeanDefinitionRegistrar`

Conceptually:

``` java
public interface ImportBeanDefinitionRegistrar {

    void registerBeanDefinitions(
        AnnotationMetadata importingClassMetadata,
        BeanDefinitionRegistry registry);
}
```

Unlike an `ImportSelector`, it works directly with the registry.

Example:

``` java
registry.registerBeanDefinition(
    "orderService",
    new RootBeanDefinition(OrderService.class));
```

A registrar can programmatically register definitions for:

-   normal beans,
-   infrastructure beans,
-   `BeanPostProcessor`s,
-   `BeanFactoryPostProcessor`s,
-   `BeanDefinitionRegistryPostProcessor`s,
-   `FactoryBean`s,
-   configuration classes.

Think:

> Registrar = "I will construct and register the `BeanDefinition`
> myself."

------------------------------------------------------------------------

## 9. Selector vs Registrar

  -------------------------------------------------------------------------------
  Question                ImportSelector          ImportBeanDefinitionRegistrar
  ----------------------- ----------------------- -------------------------------
  Main output             Class names             BeanDefinitions

  Works directly with     No                      Yes
  registry                                        

  Dynamic choice          Yes                     Yes

  Controls bean           Limited                 Yes
  name/metadata directly                          

  Common use              Select                  Programmatic infrastructure
                          configuration/imports   registration
  -------------------------------------------------------------------------------

Mental model:

``` text
ImportSelector
    |
    v
"Spring, import these classes."

Registrar
    |
    v
"Registry, add these exact BeanDefinitions."
```

------------------------------------------------------------------------

## 10. Can Selectors and Registrars Chain?

### ImportSelector -\> ImportSelector

Yes.

``` text
FirstSelector
      |
      v
returns SecondSelector
      |
      v
Spring recognizes ImportSelector
      |
      v
executes SecondSelector
```

### ImportSelector -\> Registrar

Yes.

``` text
Selector
   |
   v
returns MyRegistrar class
   |
   v
configuration parser recognizes registrar
   |
   v
registrar participates in import processing
```

### Registrar -\> another Registrar as a plain BeanDefinition

This is different.

If Registrar A merely does:

``` java
registry.registerBeanDefinition(
    "registrarB",
    new RootBeanDefinition(RegistrarB.class));
```

Spring does **not** execute Registrar B merely because a bean implements
`ImportBeanDefinitionRegistrar`.

`ImportBeanDefinitionRegistrar` is an `@Import` parsing callback, not a
post-processor type that Spring globally discovers from bean
definitions.

Contrast that with BDRPP:

``` text
Registrar registers MyBDRPP BeanDefinition
        |
        v
Spring searches for BDRPP types
        |
        v
MyBDRPP executes
```

------------------------------------------------------------------------

## 11. BDRPP vs BFPP vs BPP

### BeanDefinitionRegistryPostProcessor

Runs against the registry.

``` java
postProcessBeanDefinitionRegistry(
    BeanDefinitionRegistry registry)
```

Can add and modify bean definitions.

Also extends:

``` java
BeanFactoryPostProcessor
```

### BeanFactoryPostProcessor

Works on the configured bean factory / bean definitions before normal
bean creation.

``` java
postProcessBeanFactory(
    ConfigurableListableBeanFactory beanFactory)
```

### BeanPostProcessor

Works with actual bean instances.

``` java
postProcessBeforeInitialization(...)
postProcessAfterInitialization(...)
```

Comparison:

  Extension   Works mainly on             Timing
  ----------- --------------------------- ----------------------
  BDRPP       BeanDefinitionRegistry      Earliest
  BFPP        BeanFactory / definitions   Before normal beans
  BPP         Bean objects                During bean creation

------------------------------------------------------------------------

## 12. `PostProcessorRegistrationDelegate`

`AbstractApplicationContext.refresh()` eventually invokes post-processor
orchestration.

Conceptually:

``` text
invokeBeanFactoryPostProcessors()
        |
        v
PostProcessorRegistrationDelegate
```

The important behavior is:

``` text
1. Invoke BDRPP.postProcessBeanDefinitionRegistry()
        |
        +--> ConfigurationClassPostProcessor
        +--> other PriorityOrdered BDRPPs
        +--> Ordered BDRPPs
        +--> remaining BDRPPs
        +--> repeat discovery for newly registered BDRPPs
        |
2. Invoke postProcessBeanFactory() on BDRPPs
        |
3. Invoke remaining BFPPs
```

The repeated BDRPP discovery is important.

A BDRPP can register another BDRPP definition, and Spring can discover
that new BDRPP before leaving the registry-post-processing stage.

------------------------------------------------------------------------

## 13. External Framework BDRPP Registration

External frameworks can contribute BDRPPs in multiple ways.

### 1. Registrar

``` text
@EnableFramework
      |
      v
@Import(FrameworkRegistrar)
      |
      v
register MyBDRPP BeanDefinition
```

### 2. `@Bean`

``` java
@Bean
static MyBDRPP myBDRPP() {
    return new MyBDRPP();
}
```

### 3. Component scanning

``` java
@Component
class MyBDRPP
        implements BeanDefinitionRegistryPostProcessor {
}
```

This requires its package to be scanned.

### 4. Boot auto-configuration

``` text
AutoConfigurationImportSelector
        |
        v
FrameworkAutoConfiguration
        |
        v
@Bean static MyBDRPP
```

### 5. Programmatic registration

A context/framework can add a post-processor instance before refresh.

------------------------------------------------------------------------

## 14. `@MapperScan` / MyBatis Example

This is a good example of two distinct stages.

Conceptually:

``` text
@MapperScan
    |
    v
@Import(MapperScannerRegistrar)
    |
    v
MapperScannerRegistrar
    |
    v
registers MapperScannerConfigurer BeanDefinition
```

`MapperScannerConfigurer` is a BDRPP.

Later:

``` text
PostProcessorRegistrationDelegate
       |
       v
finds MapperScannerConfigurer
       |
       v
instantiates it
       |
       v
postProcessBeanDefinitionRegistry()
       |
       v
scan mapper interfaces
       |
       v
register mapper-related BeanDefinitions
```

The key distinction:

``` text
@MapperScan processing
        !=
mapper interface scanning
```

The first installs the scanner infrastructure.

The second is performed by that infrastructure.

------------------------------------------------------------------------

## 15. Why `static @Bean` for BFPP/BDRPP

If a post-processor is declared through an `@Bean` method:

``` java
@Configuration
class Config {

    @Bean
    MyBFPP bfpp() {
        return new MyBFPP();
    }
}
```

Spring may need the post-processor extremely early.

Calling a non-static factory method requires creating the configuration
instance early.

Prefer:

``` java
@Configuration
class Config {

    @Bean
    static MyBFPP bfpp() {
        return new MyBFPP();
    }
}
```

Same principle applies to a BDRPP declared through `@Bean`, because
BDRPP extends BFPP.

This recommendation concerns the **factory method**, not the
post-processor class itself.

------------------------------------------------------------------------

## 16. Where AOP Proxy Creation Fits

AOP proxies are generally **not created during CCPP, BDRPP, or BFPP
processing**.

Those earlier phases install the infrastructure.

Proxy wrapping occurs during bean creation through post-processors.

``` text
Configuration processing
        |
        v
register auto-proxy infrastructure
        |
        v
register BeanPostProcessors
        |
        v
create application bean
        |
        v
BeanPostProcessors run
        |
        v
auto-proxy creator determines matching Advisors
        |
        v
proxy created if necessary
```

------------------------------------------------------------------------

## 17. `@EnableAspectJAutoProxy`

Conceptually:

``` text
@EnableAspectJAutoProxy
        |
        v
@Import(AspectJAutoProxyRegistrar)
        |
        v
AspectJAutoProxyRegistrar
        |
        v
registers AutoProxyCreator BeanDefinition
```

A common auto-proxy creator is:

``` text
AnnotationAwareAspectJAutoProxyCreator
```

Hierarchy conceptually:

``` text
BeanPostProcessor
      ^
      |
AbstractAutoProxyCreator
      ^
      |
AbstractAdvisorAutoProxyCreator
      ^
      |
AspectJAwareAdvisorAutoProxyCreator
      ^
      |
AnnotationAwareAspectJAutoProxyCreator
```

The registrar does not proxy `OrderService` directly.

It installs the BPP that can later proxy `OrderService`.

------------------------------------------------------------------------

## 18. `AbstractAutoProxyCreator`

During bean lifecycle Spring eventually reaches
initialization/post-processing.

Conceptually:

``` java
Object result =
    postProcessAfterInitialization(bean, beanName);
```

`AbstractAutoProxyCreator` can perform logic similar to:

``` java
if (shouldSkip(beanClass, beanName)) {
    return bean;
}

Object[] specificInterceptors =
    getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);

if (specificInterceptors == DO_NOT_PROXY) {
    return bean;
}

return createProxy(...);
```

High-level:

``` text
Bean instance
    |
    v
Should it be proxied?
    |
    +-- no --> original bean
    |
    +-- yes
          |
          v
     determine Advisors
          |
          v
       ProxyFactory
          |
          v
     JDK or CGLIB proxy
```

Infrastructure classes such as `Advice`, `Pointcut`, `Advisor`, and AOP
infrastructure itself are normally excluded from ordinary auto-proxying.

------------------------------------------------------------------------

## 19. Advisor vs Advice vs Interceptor

This distinction is fundamental.

### Advisor

An Advisor combines:

``` text
Advisor
   |
   +--> Pointcut: WHERE should advice apply?
   |
   +--> Advice: WHAT behavior should apply?
```

### Advice

Represents the cross-cutting behavior.

Some advice types are already `MethodInterceptor`s; others can be
adapted into interceptors.

### MethodInterceptor

Executes at runtime around a method invocation:

``` java
Object invoke(MethodInvocation invocation) {
    // before
    Object result = invocation.proceed();
    // after
    return result;
}
```

A good mental model:

``` text
Advisor:
"Does this concern apply here, and what advice belongs to it?"

Interceptor:
"Execute the runtime behavior for this invocation."
```

### Real examples

  ----------------------------------------------------------------------------------------------------------------
  Feature                 Advisor                                          Runtime interceptor/advice
  ----------------------- ------------------------------------------------ ---------------------------------------
  Transaction             `BeanFactoryTransactionAttributeSourceAdvisor`   `TransactionInterceptor`

  Async                   `AsyncAnnotationAdvisor`                         `AnnotationAsyncExecutionInterceptor`

  Cache                   `BeanFactoryCacheOperationSourceAdvisor`         `CacheInterceptor`
  ----------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 20. Multiple Advices and One Proxy

Suppose a method is affected by several concerns:

``` java
@Transactional
@Async
@Cacheable("orders")
public Order load(...) {
    ...
}
```

Conceptually, Spring may create an interceptor chain:

``` text
Proxy
  |
  v
Interceptor A
  |
  v
Interceptor B
  |
  v
Interceptor C
  |
  v
Target
```

The actual order depends on Advisor ordering.

Important nuance:

> Do not assume that every annotation-based Spring feature necessarily
> originates from one identical auto-proxy creator or that nested
> proxies can never occur.

Spring attempts to integrate Advisors into proxy infrastructure, but
different BPP-based infrastructure and ordering/configuration can
matter. The useful mental model is still an ordered interceptor chain
around the target.

------------------------------------------------------------------------

## 21. JDK Proxy Pseudocode

Suppose:

``` java
interface OrderService {
    String placeOrder();
}

class OrderServiceImpl implements OrderService {
    public String placeOrder() {
        return "OK";
    }
}
```

Conceptually a JDK proxy behaves like:

``` java
class GeneratedJdkProxy implements OrderService {

    InvocationHandler handler;

    @Override
    public String placeOrder() {
        return (String) handler.invoke(
            this,
            OrderService.class.getMethod("placeOrder"),
            new Object[0]
        );
    }
}
```

Spring's handler then delegates through AOP infrastructure.

Conceptually:

``` java
Object invoke(Object proxy, Method method, Object[] args) {

    List<Object> chain =
        getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    MethodInvocation invocation =
        new ReflectiveMethodInvocation(
            proxy,
            target,
            method,
            args,
            targetClass,
            chain
        );

    return invocation.proceed();
}
```

The proxy implements interfaces rather than subclassing the concrete
target class.

------------------------------------------------------------------------

## 22. CGLIB Proxy Pseudocode

For:

``` java
class OrderService {

    public String placeOrder() {
        return "OK";
    }
}
```

Conceptually Spring generates a subclass:

``` java
class OrderService$$SpringCGLIB
        extends OrderService {

    MethodInterceptor callback;

    @Override
    public String placeOrder() {

        return (String) callback.intercept(
            this,
            PLACE_ORDER_METHOD,
            EMPTY_ARGS,
            PLACE_ORDER_METHOD_PROXY
        );
    }
}
```

The callback enters Spring's AOP invocation machinery and executes the
interceptor chain before eventually invoking the target.

The important conceptual difference is:

``` text
JDK:
Proxy implements target interfaces

CGLIB:
Proxy subclasses target class
```

------------------------------------------------------------------------

## 23. `ReflectiveMethodInvocation`

This is the core chain-walking idea.

Simplified pseudocode:

``` java
class ReflectiveMethodInvocation
        implements MethodInvocation {

    List<?> interceptors;
    int currentInterceptorIndex = -1;

    public Object proceed() {

        if (currentInterceptorIndex ==
                interceptors.size() - 1) {

            return invokeTargetMethod();
        }

        Object interceptor =
            interceptors.get(++currentInterceptorIndex);

        return ((MethodInterceptor) interceptor)
            .invoke(this);
    }
}
```

Example chain:

``` text
proceed()
   |
   v
TransactionInterceptor.invoke()
   |
   +--> begin transaction
   |
   +--> invocation.proceed()
              |
              v
        LoggingInterceptor.invoke()
              |
              +--> log before
              |
              +--> invocation.proceed()
                         |
                         v
                       Target
              |
              +--> log after
   |
   +--> commit
```

This explains why interceptors can wrap each other.

------------------------------------------------------------------------

## 24. JDK Proxy vs CGLIB Caveats

  -----------------------------------------------------------------------
  Concern                 JDK Dynamic Proxy       CGLIB
  ----------------------- ----------------------- -----------------------
  Basic mechanism         Implements interfaces   Subclasses class

  Interface required      Yes for exposed proxy   No
                          contract                

  Proxy assignable to     Usually no              Yes
  implementation class                            

  `final` class           Interface proxy may     Cannot subclass final
                          still proxy interface   class
                          contract of a final     
                          implementation          

  `final` method          Interface-level         Cannot intercept by
                          interception can differ overriding
                          by setup; subclass      
                          proxy cannot override   
                          it                      

  `private` method        Not exposed through     Cannot
                          proxy contract          override/intercept
                                                  normally

  Constructor             No                      No normal Spring AOP
  interception                                    interception

  Self invocation         Bypasses proxy          Bypasses proxy

  Concrete-type injection Problematic with JDK    Usually works
                          proxy                   
  -----------------------------------------------------------------------

### Self invocation

``` java
class OrderService {

    public void methodA() {
        methodB();
    }

    @Transactional
    public void methodB() {
    }
}
```

External call:

``` text
Client -> Proxy -> methodA()
```

Inside target:

``` text
this.methodB()
```

does not re-enter the external proxy.

Therefore proxy-based advice on `methodB()` is bypassed for that
self-invocation.

### Final/private methods

Subclass-based proxying relies on overriding methods, so `final` and
`private` methods cannot be overridden.

### Proxy choice

A common Spring AOP rule is:

``` text
proxyTargetClass = true
        |
        v
class-based proxy

otherwise:
proxying strategy depends on interfaces and proxy configuration
```

Spring Boot commonly configures class-based proxying by default, so do
not assume every modern Boot application with interfaces necessarily
uses JDK proxies.

------------------------------------------------------------------------

## 25. `@Transactional` + `@Async` + `@Scheduled`

These annotations represent different infrastructure concerns.

### `@Scheduled`

`ScheduledAnnotationBeanPostProcessor` discovers scheduled methods and
registers scheduled tasks.

The scheduled invocation should ultimately invoke the bean reference
Spring has resolved for scheduling, which may be a proxy.

### `@Transactional`

Transaction infrastructure provides an Advisor/Interceptor combination.

Conceptually:

``` text
Transaction Advisor
      |
      +--> transaction pointcut / attributes
      |
      +--> TransactionInterceptor
```

### `@Async`

Async infrastructure contributes async advice/interception.

A key runtime issue is **ordering**.

Consider conceptually:

``` text
Async interceptor
      |
      v
submit task to executor
      |
      v
new worker thread
      |
      v
remaining invocation chain
```

If transaction interception occurs **inside** the async execution, the
transaction is opened on the worker thread, which is normally what you
want.

A transaction context is generally thread-bound; you should not expect a
transaction/connection context opened on one thread to simply move to
another thread.

Therefore, when mixing `@Async` and `@Transactional`, understand the
actual interceptor/proxy ordering rather than reasoning only from
annotation order in source code.

------------------------------------------------------------------------

## 26. End-to-End Mental Model

Here is the complete connection from startup to runtime:

``` text
Application starts
      |
      v
refresh()
      |
      v
PostProcessorRegistrationDelegate
      |
      v
ConfigurationClassPostProcessor
      |
      +--> parse @Configuration
      +--> evaluate @Conditional
      +--> @ComponentScan
      +--> @Import
      |      +--> ImportSelector
      |      +--> DeferredImportSelector
      |      +--> ImportBeanDefinitionRegistrar
      |
      +--> discover/register @Bean definitions
      |
      v
BeanDefinitionRegistry populated
      |
      v
remaining BDRPPs
      |
      v
BFPPs
      |
      v
register BeanPostProcessors
      |
      +--> dependency-related BPPs
      +--> scheduling BPP
      +--> auto-proxy creator(s)
      |
      v
create application bean
      |
      +--> instantiate target
      +--> populate dependencies
      +--> initialize
      |
      v
AutoProxyCreator examines bean
      |
      v
find applicable Advisors
      |
      +-- none --> return target
      |
      +-- found
             |
             v
          ProxyFactory
             |
             +--> JDK proxy
             |       or
             +--> class-based proxy
             |
             v
        exposed bean reference
             |
             v
Application invokes method
             |
             v
Proxy
             |
             v
Interceptor chain
             |
             v
Target method
```

------------------------------------------------------------------------

## 27. Quick Revision Sheet

### `@Enable...`

Usually:

``` text
@EnableX
   |
   v
@Import(...)
   |
   v
register/import infrastructure
```

### Import mechanisms

``` text
ImportSelector
    = choose classes

DeferredImportSelector
    = choose classes later in configuration parsing

ImportBeanDefinitionRegistrar
    = directly register BeanDefinitions
```

### Post-processors

``` text
BDRPP
    = modify/add BeanDefinitions through registry

BFPP
    = modify BeanFactory / BeanDefinitions

BPP
    = process actual bean objects
```

### AOP

``` text
Advisor
    = Pointcut + Advice

Pointcut
    = where?

Advice
    = what cross-cutting behavior?

MethodInterceptor
    = runtime invocation form used in chain
```

### Proxy

``` text
JDK Proxy
    = implements interfaces

CGLIB/class proxy
    = subclasses target class
```

### Runtime

``` text
Client
  |
  v
Proxy
  |
  v
Interceptor 1
  |
  v
Interceptor 2
  |
  v
Target
```

### Most important classes to remember

``` text
AbstractApplicationContext
PostProcessorRegistrationDelegate

ConfigurationClassPostProcessor
ConfigurationClassParser

ImportSelector
DeferredImportSelector
ImportBeanDefinitionRegistrar

BeanDefinitionRegistryPostProcessor
BeanFactoryPostProcessor
BeanPostProcessor

AspectJAutoProxyRegistrar
AnnotationAwareAspectJAutoProxyCreator
AbstractAutoProxyCreator
AbstractAdvisorAutoProxyCreator

Advisor
Pointcut
MethodInterceptor

ProxyFactory
ReflectiveMethodInvocation
```

------------------------------------------------------------------------

## Final mental shortcut

Spring startup can be remembered in four words:

``` text
DISCOVER -> DEFINE -> ENHANCE -> CREATE
```

``` text
DISCOVER
@Configuration / scanning / imports

DEFINE
BeanDefinitions enter the registry

ENHANCE
BDRPP + BFPP prepare definitions
BPP infrastructure is registered

CREATE
Beans are instantiated and BPPs can wrap them in proxies
```

And Spring AOP runtime can be remembered as:

``` text
MATCH -> PROXY -> CHAIN -> TARGET
```

``` text
MATCH
Advisors determine applicability

PROXY
Spring creates JDK/class proxy

CHAIN
MethodInterceptors execute in order

TARGET
Actual business method runs
```
