# Spring MVC to Spring Boot — A Deep Reference from `web.xml` to Modern Boot

This README reconstructs the full mental model from classic Spring XML applications to modern Spring Boot, including:

- `applicationContext.xml`
- `spring-mvc.xml`
- `web.xml`
- Java-based Spring configuration
- Spring Boot MVC auto-configuration
- `WebMvcConfigurer`
- `HandlerMapping`
- `HttpMessageConverter`
- Servlet filters
- Spring Security filters
- `DelegatingFilterProxy`
- `FilterChainProxy`
- `SecurityFilterChain`
- custom Servlets
- `DispatcherServlet`
- complete request execution order

The goal is not only to know **what replaced what**, but to understand **what still exists underneath**.

---

# Table of Contents

1. [Core Mental Model](#1-core-mental-model)
2. [Phase 1 — Classic XML-Based Spring](#2-phase-1--classic-xml-based-spring)
3. [Phase 2 — Annotation-Based Beans While XML Remained](#3-phase-2--annotation-based-beans-while-xml-remained)
4. [Phase 3 — Java Configuration Replaces Spring XML](#4-phase-3--java-configuration-replaces-spring-xml)
5. [Phase 4 — `web.xml` Becomes Optional](#5-phase-4--webxml-becomes-optional)
6. [Phase 5 — Spring Boot](#6-phase-5--spring-boot)
7. [Direct Mapping of the Three Old Files](#7-direct-mapping-of-the-three-old-files)
8. [Spring MVC vs Spring Boot](#8-spring-mvc-vs-spring-boot)
9. [`@EnableWebMvc` vs Spring Boot MVC Auto-Configuration](#9-enablewebmvc-vs-spring-boot-mvc-auto-configuration)
10. [`WebMvcConfigurer` in Spring Boot](#10-webmvcconfigurer-in-spring-boot)
11. [What Can Be Customized Through `WebMvcConfigurer`](#11-what-can-be-customized-through-webmvcconfigurer)
12. [How `RequestMappingHandlerMapping` Gets Registered](#12-how-requestmappinghandlermapping-gets-registered)
13. [How Controller Methods Become URL Mappings](#13-how-controller-methods-become-url-mappings)
14. [How `HttpMessageConverter`s Get Registered](#14-how-httpmessageconverters-get-registered)
15. [Jackson and `ObjectMapper`](#15-jackson-and-objectmapper)
16. [Adding Servlet Filters Without `web.xml`](#16-adding-servlet-filters-without-webxml)
17. [Servlet Filter Chain](#17-servlet-filter-chain)
18. [Spring Security Filter Architecture](#18-spring-security-filter-architecture)
19. [`DelegatingFilterProxy`](#19-delegatingfilterproxy)
20. [`FilterChainProxy`](#20-filterchainproxy)
21. [`SecurityFilterChain`](#21-securityfilterchain)
22. [How Servlet and Security Filter Chains Are Effectively Chained](#22-how-servlet-and-security-filter-chains-are-effectively-chained)
23. [What Exists Before and After `DelegatingFilterProxy`](#23-what-exists-before-and-after-delegatingfilterproxy)
24. [Adding Your Own Filter at Different Locations](#24-adding-your-own-filter-at-different-locations)
25. [Adding a Custom Servlet Without `web.xml`](#25-adding-a-custom-servlet-without-webxml)
26. [Can One Servlet Execute After Another?](#26-can-one-servlet-execute-after-another)
27. [Filter vs Interceptor vs Servlet vs AOP](#27-filter-vs-interceptor-vs-servlet-vs-aop)
28. [Complete Request Lifecycle](#28-complete-request-lifecycle)
29. [Historical Mapping Cheat Sheet](#29-historical-mapping-cheat-sheet)
30. [Final Architecture to Remember](#30-final-architecture-to-remember)

---

# 1. Core Mental Model

The most important fact:

> Spring Boot did not replace Spring MVC, the Servlet API, `DispatcherServlet`, `Filter`, `HandlerMapping`, `HandlerAdapter`, or the Spring IoC container.

Spring Boot mainly automated their registration and configuration.

The evolution is:

```text
Classic Spring
==============
web.xml
applicationContext.xml
spring-mvc.xml

        ↓

Java Config Spring
==================
WebApplicationInitializer
@Configuration
@EnableWebMvc
WebMvcConfigurer

        ↓

Spring Boot
===========
@SpringBootApplication
SpringApplication.run()
AutoConfiguration
Embedded Servlet Container
```

But internally:

```text
Configuration metadata
        ↓
BeanDefinition
        ↓
BeanDefinitionRegistry
        ↓
BeanFactory
        ↓
BeanFactoryPostProcessor
        ↓
BeanPostProcessor
        ↓
Bean creation
        ↓
Dependency Injection
        ↓
Proxy creation
        ↓
Runtime application
```

So XML, Java config, `@Enable...`, and Boot auto-configuration are mostly **different ways of supplying configuration metadata to the same underlying Spring container**.

---

# 2. Phase 1 — Classic XML-Based Spring

A classic web application looked like:

```text
                         Tomcat
                           |
                        web.xml
                     /             \
                    /               \
                   v                 v
        ContextLoaderListener    DispatcherServlet
                   |                 |
                   v                 v
      applicationContext.xml     spring-mvc.xml
                   |                 |
          Services/DAO/etc.       Controllers/MVC
```

Typical `web.xml`:

```xml
<web-app>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>

    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

Typical `applicationContext.xml`:

```xml
<bean id="employeeService"
      class="com.example.EmployeeServiceImpl">
    <property name="employeeDao" ref="employeeDao"/>
</bean>

<bean id="employeeDao"
      class="com.example.EmployeeDaoImpl"/>
```

Typical `spring-mvc.xml`:

```xml
<context:component-scan base-package="com.example.web"/>
<mvc:annotation-driven/>
```

---

# 3. Phase 2 — Annotation-Based Beans While XML Remained

Spring then reduced explicit `<bean>` declarations.

Instead of:

```xml
<bean id="employeeService"
      class="com.example.EmployeeServiceImpl"/>
```

you could write:

```java
@Service
public class EmployeeServiceImpl {
}
```

and enable:

```xml
<context:component-scan base-package="com.example"/>
```

Similarly:

```java
@Controller
@RequestMapping("/employees")
public class EmployeeController {
}
```

The role of XML changed from:

```text
"Declare every bean"
```

toward:

```text
"Enable infrastructure and scanning rules"
```

---

# 4. Phase 3 — Java Configuration Replaces Spring XML

Spring introduced:

```java
@Configuration
@Bean
@ComponentScan
@Import
@PropertySource
```

Example:

```java
@Configuration
@ComponentScan({
        "com.example.service",
        "com.example.repository"
})
@EnableTransactionManagement
public class RootConfig {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

The important internal point:

```text
XML <bean>
      \
       \
        ---> BeanDefinition
       /
@Bean /
```

Both are configuration metadata sources.

---

# 5. Phase 4 — `web.xml` Becomes Optional

With Servlet 3, programmatic initialization became possible.

Spring introduced:

```java
WebApplicationInitializer
```

A convenient Spring MVC bootstrap class is:

```java
AbstractAnnotationConfigDispatcherServletInitializer
```

Example:

```java
public class WebInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[]{RootConfig.class};
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{WebConfig.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```

Now:

```text
web.xml                         Java Config
-------                         -----------

ContextLoaderListener     →     initializer creates root context

DispatcherServlet         →     initializer registers DispatcherServlet

URL mapping "/"           →     getServletMappings()

applicationContext.xml    →     RootConfig.class

spring-mvc.xml            →     WebConfig.class
```

---

# 6. Phase 5 — Spring Boot

Spring Boot simplified bootstrap further.

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

The lifecycle becomes:

```text
main()
  |
  v
SpringApplication.run()
  |
  v
ApplicationContext
  |
  v
Auto-Configuration
  |
  v
Embedded Tomcat
  |
  v
DispatcherServlet registration
```

Traditional deployment:

```text
Tomcat starts
   ↓
Tomcat reads web.xml
   ↓
Spring starts
```

Boot:

```text
Java main()
   ↓
Spring starts
   ↓
Spring starts embedded Tomcat
```

This is an inversion of ownership.

---

# 7. Direct Mapping of the Three Old Files

## `applicationContext.xml`

Old responsibility:

```text
Business and infrastructure beans

Services
Repositories
DataSource
TransactionManager
JPA
etc.
```

Modern plain Spring:

```java
@Configuration
@ComponentScan
@EnableTransactionManagement
public class RootConfig {
}
```

Modern Boot:

```text
@SpringBootApplication
+
AutoConfiguration
+
your @Configuration classes
```

---

## `spring-mvc.xml`

Old responsibility:

```text
MVC infrastructure
Controllers
HandlerMappings
HandlerAdapters
ViewResolvers
MessageConverters
```

Old:

```xml
<mvc:annotation-driven/>
<context:component-scan base-package="com.example.web"/>
```

Modern plain Spring:

```java
@Configuration
@EnableWebMvc
@ComponentScan("com.example.web")
public class WebConfig implements WebMvcConfigurer {
}
```

Modern Boot:

```text
WebMvcAutoConfiguration
+
your optional WebMvcConfigurer
```

---

## `web.xml`

Old responsibility:

```text
Servlet container bootstrap
DispatcherServlet
ContextLoaderListener
Servlet mappings
Filters
Listeners
```

Modern plain Spring:

```text
WebApplicationInitializer
```

Modern Boot:

```text
SpringApplication.run()
Embedded Servlet container
Registration beans
AutoConfiguration
```

---

# 8. Spring MVC vs Spring Boot

Spring MVC is the **web framework**.

Spring Boot is the **bootstrap and automatic configuration layer**.

```text
Spring MVC
==========
DispatcherServlet
HandlerMapping
HandlerAdapter
@Controller
@RestController
HttpMessageConverter
ViewResolver
ExceptionResolver


Spring Boot
===========
Application bootstrap
Embedded Tomcat
MVC auto-configuration
DispatcherServlet registration
Jackson configuration
Dependency management
```

So:

> Spring Boot configures Spring MVC; it does not replace Spring MVC.

---

# 9. `@EnableWebMvc` vs Spring Boot MVC Auto-Configuration

Plain Spring:

```java
@Configuration
@EnableWebMvc
public class WebConfig {
}
```

`@EnableWebMvc` conceptually leads to:

```text
@EnableWebMvc
     ↓
@Import
     ↓
DelegatingWebMvcConfiguration
     ↓
WebMvcConfigurationSupport
     ↓
MVC infrastructure beans
```

In Boot:

```text
spring-webmvc present
       +
Servlet web application
       ↓
WebMvcAutoConfiguration
       ↓
Spring MVC infrastructure
```

Normally in Boot, do **not** use `@EnableWebMvc`.

Why?

Because Boot's MVC auto-configuration uses a condition conceptually like:

```java
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
```

`@EnableWebMvc` imports a configuration derived from:

```java
WebMvcConfigurationSupport
```

Therefore adding `@EnableWebMvc` says, effectively:

```text
"I want explicit Spring MVC control."
```

and Boot backs away from much of its normal MVC configuration.

---

# 10. `WebMvcConfigurer` in Spring Boot

A common Boot customization is:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // customizations
    }
}
```

Important:

> Providing your own `WebMvcConfigurer` does NOT replace Boot's MVC configuration.

Spring can have multiple configurers.

Conceptually:

```text
Boot MVC configuration
        +
Your WebMvcConfigurer
        +
Other WebMvcConfigurer beans
        ↓
Delegating configuration
        ↓
Final MVC infrastructure
```

The condition is **not**:

```java
@ConditionalOnMissingBean(WebMvcConfigurer.class)
```

Therefore your configurer does not cause Boot to remove its default MVC setup.

---

# 11. What Can Be Customized Through `WebMvcConfigurer`

Major extension points include:

| Method | Purpose |
|---|---|
| `configurePathMatch()` | URL/path matching |
| `configureContentNegotiation()` | media-type negotiation |
| `configureAsyncSupport()` | MVC async processing |
| `configureDefaultServletHandling()` | default Servlet handling |
| `addFormatters()` | converters and formatters |
| `addInterceptors()` | MVC interceptors |
| `addResourceHandlers()` | static resources |
| `addCorsMappings()` | MVC CORS rules |
| `addViewControllers()` | direct URL-to-view mapping |
| `configureViewResolvers()` | view resolution |
| `addArgumentResolvers()` | custom controller arguments |
| `addReturnValueHandlers()` | custom controller return values |
| `configureMessageConverters()` | direct converter configuration |
| `extendMessageConverters()` | extend existing converters |
| `configureHandlerExceptionResolvers()` | configure exception resolvers |
| `extendHandlerExceptionResolvers()` | extend existing resolvers |
| `getValidator()` | MVC validator |
| `getMessageCodesResolver()` | validation message-code strategy |

## Example — interceptor

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new AuditInterceptor())
            .addPathPatterns("/api/**");
}
```

## Example — CORS

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
            .allowedOrigins("https://example.com")
            .allowedMethods("GET", "POST");
}
```

## Example — custom converter

```java
@Override
public void addFormatters(FormatterRegistry registry) {
    registry.addConverter(new StringToEmployeeIdConverter());
}
```

Remember:

```text
Converter / Formatter
---------------------
String/value ↔ Java type

@PathVariable
@RequestParam
form values


HttpMessageConverter
--------------------
HTTP body ↔ Java object

@RequestBody
@ResponseBody
```

---

# 12. How `RequestMappingHandlerMapping` Gets Registered

Plain Spring MVC:

```text
@EnableWebMvc
     ↓
DelegatingWebMvcConfiguration
     ↓
WebMvcConfigurationSupport
     ↓
@Bean requestMappingHandlerMapping(...)
     ↓
RequestMappingHandlerMapping bean
```

Boot:

```text
@SpringBootApplication
      ↓
@EnableAutoConfiguration
      ↓
WebMvcAutoConfiguration
      ↓
MVC configuration infrastructure
      ↓
RequestMappingHandlerMapping bean
```

The important point:

> Boot does not invent a different handler mapping. It configures Spring MVC's normal `RequestMappingHandlerMapping`.

---

# 13. How Controller Methods Become URL Mappings

Suppose:

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    @GetMapping("/{id}")
    public Employee getEmployee(@PathVariable Long id) {
        return null;
    }
}
```

During initialization:

```text
RequestMappingHandlerMapping
        ↓
afterPropertiesSet()
        ↓
initHandlerMethods()
        ↓
scan handler beans
        ↓
find @Controller / @RestController beans
        ↓
inspect methods
        ↓
@RequestMapping / @GetMapping / @PostMapping
        ↓
create RequestMappingInfo
        ↓
register RequestMappingInfo → HandlerMethod
```

Conceptually:

```text
GET /employees/{id}
        ↓
EmployeeController#getEmployee(Long)
```

Runtime:

```text
GET /employees/10
        ↓
DispatcherServlet
        ↓
RequestMappingHandlerMapping
        ↓
HandlerMethod
        ↓
RequestMappingHandlerAdapter
        ↓
Controller method
```

Two phases must be distinguished:

```text
Phase 1:
Register RequestMappingHandlerMapping bean

Phase 2:
That bean discovers and registers controller methods
```

---

# 14. How `HttpMessageConverter`s Get Registered

`RequestMappingHandlerMapping` answers:

```text
"Which controller method?"
```

`RequestMappingHandlerAdapter` answers:

```text
"How do I invoke it?"
```

`HttpMessageConverter` answers:

```text
"How do I convert HTTP body ↔ Java object?"
```

Plain MVC:

```text
@EnableWebMvc
       ↓
WebMvcConfigurationSupport
       ↓
RequestMappingHandlerAdapter
       ↓
List<HttpMessageConverter<?>>
```

Boot adds auto-configured converters depending on the classpath.

Typical converter types include:

```text
ByteArrayHttpMessageConverter
StringHttpMessageConverter
ResourceHttpMessageConverter
FormHttpMessageConverter
MappingJackson2HttpMessageConverter
...
```

---

# 15. Jackson and `ObjectMapper`

For:

```java
@PostMapping("/employees")
public Employee create(@RequestBody Employee employee) {
    return employee;
}
```

Request body:

```json
{
  "name": "John"
}
```

Flow:

```text
HTTP JSON
   ↓
DispatcherServlet
   ↓
RequestMappingHandlerAdapter
   ↓
RequestResponseBodyMethodProcessor
   ↓
HttpMessageConverter selection
   ↓
MappingJackson2HttpMessageConverter
   ↓
ObjectMapper
   ↓
Employee
```

Response:

```text
Employee
   ↓
RequestResponseBodyMethodProcessor
   ↓
MappingJackson2HttpMessageConverter
   ↓
ObjectMapper
   ↓
JSON response
```

So the hierarchy is:

```text
RequestMappingHandlerMapping
        ↓
Which method?

RequestMappingHandlerAdapter
        ↓
How to invoke?

Argument/return-value processors
        ↓
How to handle parameters/return values?

HttpMessageConverter
        ↓
How to translate HTTP body?

MappingJackson2HttpMessageConverter
        ↓
JSON converter

ObjectMapper
        ↓
Actual Jackson serialization/deserialization
```

---

# 16. Adding Servlet Filters Without `web.xml`

Old:

```xml
<filter>
    <filter-name>auditFilter</filter-name>
    <filter-class>com.example.AuditFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>auditFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

Modern Boot option 1:

```java
@Component
public class AuditFilter implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        chain.doFilter(request, response);
    }
}
```

Modern Boot option 2 — explicit control:

```java
@Bean
public FilterRegistrationBean<AuditFilter> auditFilter() {

    FilterRegistrationBean<AuditFilter> registration =
            new FilterRegistrationBean<>();

    registration.setFilter(new AuditFilter());
    registration.addUrlPatterns("/api/*");
    registration.setName("auditFilter");
    registration.setOrder(10);

    return registration;
}
```

Direct mapping:

```text
web.xml                       Boot
-------                       ----

<filter>                →     FilterRegistrationBean#setFilter()

<filter-name>           →     setName()

<url-pattern>           →     addUrlPatterns()

ordering                →     setOrder()

init-param              →     addInitParameter()
```

---

# 17. Servlet Filter Chain

A Servlet container builds something conceptually like:

```text
ApplicationFilterChain

Filter A
   ↓
Filter B
   ↓
Filter C
   ↓
Target Servlet
```

Every filter implements:

```java
jakarta.servlet.Filter
```

and receives:

```java
FilterChain chain
```

Typical filter:

```java
public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain) {

    before();

    chain.doFilter(request, response);

    after();
}
```

This means filters execute as nested calls:

```text
Filter A before
    Filter B before
        Servlet
    Filter B after
Filter A after
```

---

# 18. Spring Security Filter Architecture

Typical high-level security order:

```text
SecurityContext handling
        ↓
Headers / CORS / CSRF / Logout
        ↓
Authentication filters
        ↓
Anonymous authentication
        ↓
ExceptionTranslationFilter
        ↓
AuthorizationFilter
```

Possible authentication filters include:

```text
UsernamePasswordAuthenticationFilter
BasicAuthenticationFilter
BearerTokenAuthenticationFilter
OAuth2LoginAuthenticationFilter
...
```

For JWT Resource Server:

```text
BearerTokenAuthenticationFilter
        ↓
AuthenticationManager
        ↓
JwtAuthenticationProvider
        ↓
JwtDecoder
        ↓
JwtAuthenticationToken
        ↓
SecurityContext
        ↓
AuthorizationFilter
```

---

# 19. `DelegatingFilterProxy`

`DelegatingFilterProxy` is a Servlet `Filter`.

Its job is to bridge:

```text
Servlet container
      ↓
Spring bean
```

Conceptually:

```java
public void doFilter(req, res, servletChain) {

    Filter delegate =
            applicationContext.getBean(
                    "springSecurityFilterChain");

    delegate.doFilter(req, res, servletChain);
}
```

The target bean is usually:

```text
springSecurityFilterChain
        ↓
FilterChainProxy
```

So:

```text
Tomcat
   ↓
DelegatingFilterProxy
   ↓
FilterChainProxy
```

---

# 20. `FilterChainProxy`

`FilterChainProxy` is Spring Security's central filter.

Important facts:

- it is itself a Servlet `Filter`;
- Tomcat normally does not know every individual security filter;
- it contains/selects Spring Security `SecurityFilterChain`s;
- it executes security filters using its own internal chain.

Conceptually:

```text
DelegatingFilterProxy
        ↓
FilterChainProxy
        ↓
choose matching SecurityFilterChain
```

---

# 21. `SecurityFilterChain`

A `SecurityFilterChain` conceptually contains:

```text
RequestMatcher
+
List<Filter>
```

Example:

```java
@Bean
@Order(1)
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {

    http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated());

    return http.build();
}
```

Multiple chains are possible:

```text
FilterChainProxy
    |
    +-- SecurityFilterChain #1
    |      matcher: /api/**
    |
    +-- SecurityFilterChain #2
    |      matcher: /admin/**
    |
    +-- SecurityFilterChain #3
           matcher: /**
```

First matching chain wins.

---

# 22. How Servlet and Security Filter Chains Are Effectively Chained

This was the key conceptual point.

At the Java type level, all of these are filters:

```text
LoggingFilter
DelegatingFilterProxy
FilterChainProxy
SecurityContextHolderFilter
BearerTokenAuthenticationFilter
AuthorizationFilter
```

But they are **not all registered in the same chain**.

Tomcat's outer chain:

```text
LoggingFilter
      ↓
DelegatingFilterProxy
      ↓
CompressionFilter
      ↓
DispatcherServlet
```

When `DelegatingFilterProxy` runs:

```text
DelegatingFilterProxy
      ↓
FilterChainProxy
      ↓
Virtual/Internal Security Chain
```

Conceptually Spring Security builds:

```java
VirtualFilterChain(
    originalServletChain,
    securityFilters
)
```

Pseudo-code:

```java
class VirtualFilterChain implements FilterChain {

    List<Filter> securityFilters;
    FilterChain originalChain;
    int position = 0;

    public void doFilter(
            ServletRequest request,
            ServletResponse response) {

        if (position < securityFilters.size()) {

            Filter next =
                    securityFilters.get(position++);

            next.doFilter(
                    request,
                    response,
                    this);

        } else {

            originalChain.doFilter(
                    request,
                    response);
        }
    }
}
```

Therefore:

```text
Tomcat chain
    ↓
DelegatingFilterProxy
    ↓
FilterChainProxy
    ↓
Security Filter 1
    ↓
Security Filter 2
    ↓
Security Filter N
    ↓
original Tomcat chain resumes
    ↓
next Servlet Filter
    ↓
DispatcherServlet
```

This is **nested filter chains**.

---

# 23. What Exists Before and After `DelegatingFilterProxy`

Outer Servlet chain may look like:

```text
CorrelationIdFilter
       ↓
LoggingFilter
       ↓
DelegatingFilterProxy
       ↓
CompressionFilter
       ↓
DispatcherServlet
```

Inside `DelegatingFilterProxy`:

```text
DelegatingFilterProxy
       ↓
FilterChainProxy
       ↓
SecurityFilterChain
       ↓
Security filters
```

Important:

```text
DelegatingFilterProxy
        ↓
FilterChainProxy
```

is a **direct delegation call**.

There is no Servlet-container insertion slot between them.

So this cannot be achieved by normal Servlet ordering:

```text
DelegatingFilterProxy
       ↓
My normal Servlet Filter       ❌
       ↓
FilterChainProxy
```

---

# 24. Adding Your Own Filter at Different Locations

## A. Before Spring Security entirely

Register a normal Servlet filter ordered before `DelegatingFilterProxy`.

```text
MyFilter
   ↓
DelegatingFilterProxy
   ↓
Spring Security
```

Useful for:

```text
Correlation IDs
low-level request logging
container-level behavior
```

---

## B. Inside the Spring Security chain

Use:

```java
http.addFilterBefore(...)
http.addFilterAfter(...)
http.addFilterAt(...)
```

Example:

```java
http.addFilterAfter(
        myFilter,
        BearerTokenAuthenticationFilter.class);
```

Flow:

```text
DelegatingFilterProxy
      ↓
FilterChainProxy
      ↓
BearerTokenAuthenticationFilter
      ↓
MyFilter
      ↓
AuthorizationFilter
```

Useful when your filter requires:

```java
SecurityContextHolder.getContext().getAuthentication()
```

---

## C. Immediately after entry into Security, before built-in security filters

Use a security filter placed before an early known security filter:

```java
http.addFilterBefore(
        myFilter,
        SecurityContextHolderFilter.class);
```

Conceptually:

```text
DelegatingFilterProxy
      ↓
FilterChainProxy
      ↓
MyFilter
      ↓
SecurityContextHolderFilter
      ↓
...
```

---

## D. After all Spring Security processing

Register an ordinary Servlet filter ordered after the Spring Security Servlet filter.

Conceptually:

```text
DelegatingFilterProxy
      ↓
entire SecurityFilterChain
      ↓
MyServletFilter
      ↓
DispatcherServlet
```

Do not confuse this with `addFilterAfter()`, which means **inside** the security chain.

---

# 25. Adding a Custom Servlet Without `web.xml`

Servlets are not added to a chain.

They are registered with URL mappings.

Example:

```java
public class MyServlet extends HttpServlet {

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        response.getWriter()
                .write("Hello from MyServlet");
    }
}
```

Register:

```java
@Bean
public ServletRegistrationBean<MyServlet> myServlet() {

    ServletRegistrationBean<MyServlet> registration =
            new ServletRegistrationBean<>(
                    new MyServlet(),
                    "/custom/*");

    registration.setName("myServlet");
    registration.setLoadOnStartup(1);

    return registration;
}
```

Old:

```xml
<servlet>
...
</servlet>

<servlet-mapping>
...
</servlet-mapping>
```

Modern Boot:

```text
ServletRegistrationBean
```

Also possible:

```java
@WebServlet("/custom/*")
public class MyServlet extends HttpServlet {
}
```

with:

```java
@ServletComponentScan
```

---

# 26. Can One Servlet Execute After Another?

Normally, no.

Filters form a chain.

Servlets are selected by routing.

Suppose:

```text
DispatcherServlet → "/"
MyServlet         → "/custom/*"
```

For:

```text
GET /custom/test
```

Tomcat selects the best Servlet mapping:

```text
Tomcat
   |
   +-- "/"         → DispatcherServlet
   |
   +-- "/custom/*" → MyServlet   ← selected
```

It does **not** execute:

```text
DispatcherServlet
      ↓
MyServlet
```

Servlets are not chained like filters.

Advanced Servlet APIs such as:

```java
RequestDispatcher.forward(...)
RequestDispatcher.include(...)
```

can explicitly dispatch from one resource to another, but this is not ordinary Servlet chaining.

---

# 27. Filter vs Interceptor vs Servlet vs AOP

Use the correct extension point.

| Requirement | Mechanism |
|---|---|
| Run before/after entire Servlet processing | Servlet `Filter` |
| Run inside Spring Security chain | Security filter via `HttpSecurity` |
| Run before/after controller | `HandlerInterceptor` |
| Create separate endpoint outside MVC | custom `Servlet` |
| Cross-cutting Java method behavior | Spring AOP |
| Customize controller argument resolution | `HandlerMethodArgumentResolver` |
| Customize body conversion | `HttpMessageConverter` |
| Customize MVC infrastructure | `WebMvcConfigurer` |

---

# 28. Complete Request Lifecycle

A modern Boot + MVC + Security request can be visualized as:

```text
                              HTTP Request
                                   |
                                   v
                         Embedded Tomcat
                                   |
                                   v
                        Servlet Filter Chain
                                   |
                    +--------------+--------------+
                    |                             |
                    v                             |
             Custom Servlet Filter               |
                    |                             |
                    v                             |
          DelegatingFilterProxy                  |
                    |                             |
                    v                             |
              FilterChainProxy                   |
                    |                             |
                    v                             |
        Select SecurityFilterChain              |
                    |                             |
                    v                             |
          SecurityContextHolderFilter           |
                    |                             |
                    v                             |
                  CORS                           |
                    |
                    v
                  CSRF
                    |
                    v
        Authentication Filter(s)
        e.g. BearerTokenAuthenticationFilter
                    |
                    v
        AnonymousAuthenticationFilter
                    |
                    v
        ExceptionTranslationFilter
                    |
                    v
            AuthorizationFilter
                    |
                    | security chain ends
                    v
        Original Servlet chain resumes
                    |
                    v
         Any Servlet Filter ordered later
                    |
                    v
             DispatcherServlet
                    |
                    v
             HandlerMapping
                    |
                    v
       RequestMappingHandlerMapping
                    |
                    v
               HandlerMethod
                    |
                    v
       HandlerInterceptor.preHandle()
                    |
                    v
       RequestMappingHandlerAdapter
                    |
          +---------+---------+
          |                   |
          v                   v
Argument Resolvers     Message Converters
          |                   |
          +---------+---------+
                    |
                    v
               Controller
                    |
                    v
       ReturnValueHandler
                    |
                    v
       HttpMessageConverter
                    |
                    v
            JSON / response
                    |
                    v
       Interceptor.postHandle()
                    |
                    v
    Interceptor.afterCompletion()
                    |
                    v
      DispatcherServlet returns
                    |
                    v
   Servlet Filter "after" portions
                    |
                    v
               HTTP Response
```

For a JSON request specifically:

```text
HTTP JSON
   ↓
Servlet filters
   ↓
Spring Security
   ↓
DispatcherServlet
   ↓
RequestMappingHandlerMapping
   ↓
HandlerMethod
   ↓
RequestMappingHandlerAdapter
   ↓
RequestResponseBodyMethodProcessor
   ↓
MappingJackson2HttpMessageConverter
   ↓
ObjectMapper
   ↓
Java object
   ↓
Controller
```

---

# 29. Historical Mapping Cheat Sheet

| Old Spring | Java Config Spring | Spring Boot |
|---|---|---|
| `applicationContext.xml` | `@Configuration` root config | `@SpringBootApplication` + auto-config + explicit config |
| `spring-mvc.xml` | `@EnableWebMvc` + `WebMvcConfigurer` | `WebMvcAutoConfiguration` + optional `WebMvcConfigurer` |
| `web.xml` | `WebApplicationInitializer` | `SpringApplication.run()` + embedded server |
| `<bean>` | `@Bean` | often auto-configured |
| `<context:component-scan>` | `@ComponentScan` | included in `@SpringBootApplication` |
| `<mvc:annotation-driven>` | `@EnableWebMvc` | MVC auto-configuration |
| `<filter>` | programmatic Servlet registration | `FilterRegistrationBean` / Filter bean |
| `<servlet>` | programmatic Servlet registration | `ServletRegistrationBean` |
| Spring Security filter in `web.xml` | `DelegatingFilterProxy` setup | Boot registers proxy automatically |
| custom MVC config XML | `WebMvcConfigurer` | `WebMvcConfigurer` while keeping Boot defaults |

---

# 30. Final Architecture to Remember

The whole architecture can be compressed into this:

```text
                        SPRING BOOT APPLICATION
                                 |
                                 v
                       SpringApplication.run()
                                 |
                                 v
                        ApplicationContext
                                 |
                                 v
                         Embedded Tomcat
                                 |
               +-----------------+-----------------+
               |                                   |
               v                                   v
       Servlet registrations                Filter registrations
               |                                   |
        DispatcherServlet                 DelegatingFilterProxy
        Custom Servlets                   Custom Servlet Filters
                                                   |
                                                   v
                                           FilterChainProxy
                                                   |
                                                   v
                                          SecurityFilterChain
                                                   |
                                                   v
                                           Security Filters
                                                   |
                                                   v
                                           DispatcherServlet
                                                   |
                                                   v
                                               Spring MVC
                                                   |
                    +------------------------------+-----------------------------+
                    |                              |                             |
                    v                              v                             v
          RequestMappingHandlerMapping   RequestMappingHandlerAdapter    ExceptionResolvers
                    |                              |
                    v                              v
             HandlerMethod                ArgumentResolvers
                                                   |
                                                   v
                                        HttpMessageConverters
                                                   |
                                                   v
                                              Controller
```

The most important statements to retain are:

1. **`web.xml` disappeared as a configuration file, not as an architectural concept.**
2. **The Servlet container still owns Filters, Servlets, mappings, and listeners.**
3. **Spring Boot performs those registrations programmatically.**
4. **Spring MVC is still built around `DispatcherServlet`.**
5. **`RequestMappingHandlerMapping` finds the controller method.**
6. **`RequestMappingHandlerAdapter` invokes it.**
7. **`HttpMessageConverter` handles request/response bodies.**
8. **Jackson's `ObjectMapper` sits underneath `MappingJackson2HttpMessageConverter`.**
9. **`WebMvcConfigurer` customizes Spring MVC without replacing Boot MVC defaults.**
10. **`@EnableWebMvc` in Boot means taking greater control and can cause Boot MVC auto-configuration to back off.**
11. **Servlet Filters form an outer container-managed filter chain.**
12. **`DelegatingFilterProxy` bridges that Servlet chain into Spring Security.**
13. **`FilterChainProxy` executes an inner Spring Security filter chain.**
14. **Security filters are Servlet `Filter`s, but they are managed inside Spring Security rather than individually by Tomcat.**
15. **Servlets are routed by URL mapping; they do not execute as a chain.**
16. **Use a Filter for before/after Servlet processing, an Interceptor for controller-level hooks, a Security filter for security-chain logic, and a Servlet for a separate endpoint.**

---

## One-Line Historical Summary

```text
web.xml + applicationContext.xml + spring-mvc.xml
                    ↓
WebApplicationInitializer + @Configuration + @EnableWebMvc
                    ↓
@SpringBootApplication + AutoConfiguration + Embedded Tomcat
```

The underlying Servlet and Spring MVC architecture is still there — **modern Spring Boot mainly hides the repetitive bootstrap configuration while preserving the same core runtime model.**
