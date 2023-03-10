_This page provides guidance on upgrading to Spring Framework 6.0._

## Upgrading to Version 6.0

### Core Container

The JSR-330 based `@Inject` annotation is to be found in `jakarta.inject` now. The corresponding JSR-250 based
annotations `@PostConstruct` and `@PreDestroy` are to be found in `jakarta.annotation`. For the time being,
Spring keeps detecting their `javax` equivalents as well, covering common use in pre-compiled binaries.

The core container performs basic bean property determination without `java.beans.Introspector` by default.
For full backwards compatibility with 5.3.x in case of sophisticated JavaBeans usage, specify the following
content in a `META-INF/spring.factories` file which enables 5.3-style full `java.beans.Introspector` usage:
`org.springframework.beans.BeanInfoFactory=org.springframework.beans.ExtendedBeanInfoFactory`

When staying on 5.3.x for the time being, you may enforce forward compatibility with 6.0-style property
determination (and better introspection performance!) through a custom `META-INF/spring.factories` file:
`org.springframework.beans.BeanInfoFactory=org.springframework.beans.SimpleBeanInfoFactory`

`LocalVariableTableParameterNameDiscoverer` is deprecated now and logs a warning for each successful
resolution attempt (it only kicks in when `StandardReflectionParameterNameDiscoverer` has not found names).
Compile your Java sources with the common Java 8+ `-parameters` flag for parameter name retention (instead
of relying on the `-debug` compiler flag) in order to avoid that warning, or report it to the maintainers
of the affected code. With the Kotlin compiler, we recommend the `-java-parameters` flag for completeness.

`LocalValidatorFactoryBean` relies on standard parameter name resolution in Bean Validation 3.0 now,
just configuring additional Kotlin reflection if Kotlin is present. If you refer to parameter names in
your Bean Validation setup, make sure to compile your Java sources with the Java 8+ `-parameters` flag.

`ListenableFuture` has been deprecated in favor of `CompletableFuture`. 
See [27780](https://github.com/spring-projects/spring-framework/issues/27780).

`SimpleEvaluationContext` disables array allocations now, aligned with regular constructor resolution.

### Data Access and Transactions

Due to the Jakarta EE migration, make sure to upgrade to Hibernate ORM 5.6.x with the `hibernate-core-jakarta`
artifact, alongside switching your `javax.persistence` imports to `jakarta.persistence` (Jakarta EE 9).
Alternatively, consider migrating to Hibernate ORM 6.1 right away (exclusively based on `jakarta.persistence`,
compatible with EE 9 as well as EE 10) which is the Hibernate version that Spring Boot 3.0 comes with.

The corresponding Hibernate Validator generation is 7.0.x, based on `jakarta.validation` (Jakarta EE 9).
You may also choose to upgrade to Hibernate Validator 8.0 right away (aligned with Jakarta EE 10).

For EclipseLink as the persistence provider of choice, the reference version is 3.0.x (Jakarta EE 9),
with EclipseLink 4.0 as the most recent supported version (Jakarta EE 10).

Spring's default JDBC exception translator is the JDBC 4 based `SQLExceptionSubclassTranslator` now,
detecting JDBC driver subclasses as well as common SQL state indications (without database product name
resolution at runtime). As of 6.0.3, this includes a common SQL state check for `DuplicateKeyException`,
addressing a long-standing difference between SQL state mappings and legacy default error code mappings.

`CannotSerializeTransactionException` and `DeadlockLoserDataAccessException` are deprecated as of 6.0.3
due to their inconsistent JDBC semantics, in favor of the `PessimisticLockingFailureException` base class
and consistent semantics of its common `CannotAcquireLockException` subclass (aligned with JPA/Hibernate)
in all default exception translation scenarios.

For full backwards compatibility with database-specific error codes, consider re-enabling the legacy
`SQLErrorCodeSQLExceptionTranslator`. This translator kicks in for user-provided `sql-error-codes.xml`
files. It can simply pick up Spring's legacy default error code mappings as well when triggered by an
empty user-provided file in the root of the classpath.

### Web Applications

Due to the Jakarta EE migration, make sure to upgrade to Tomcat 10, Jetty 11, or Undertow 2.2.19 with the
`undertow-servlet-jakarta` artifact, alongside switching your `javax.servlet` imports to `jakarta.servlet`
(Jakarta EE 9). For the latest server generations, consider Tomcat 10.1 and Undertow 2.3 (Jakarta EE 10).

Several outdated Servlet-based integrations have been dropped: e.g. Apache Commons FileUpload (`org.springframework.web.multipart.commons.CommonsMultipartResolver`), and Apache Tiles as well as
FreeMarker JSP support in the corresponding `org.springframework.web.servlet.view` subpackages. We recommend
`org.springframework.web.multipart.support.StandardServletMultipartResolver` for multipart file uploads
and regular FreeMarker template views if needed, and a general focus on REST-oriented web architectures.

Spring MVC and Spring WebFlux no longer detect controllers based solely on a type-level `@RequestMapping`
annotation. That means interface-based AOP proxying for web controllers may no longer work. Please,
enable class-based proxying for such controllers; otherwise the interface must also be annotated with `@Controller`.
See [22154](https://github.com/spring-projects/spring-framework/issues/22154).

`HttpMethod` is now a class and no longer an enum. Though the public API has been maintained, some 
migration might be necessary (i.e. change from `EnumSet<HttpMethod>` to `Set<HttpMethod>`, use `if 
else` instead of `switch`). For the rationale behind this decision, see 
[27697](https://github.com/spring-projects/spring-framework/issues/27697).

The Kotlin extension function to `WebTestClient.ResponseSpec::expectBody` now returns the Java `BodySpec`
type and no longer uses the workaround type `KotlinBodySpec`. Spring 6.0 uses Kotlin 1.6, which fixed the
bug that needed this workaround ([KT-5464](https://youtrack.jetbrains.com/issue/KT-5464)).
This means that `consumeWith` is no longer available.

`RestTemplate`, or rather the `HttpComponentsClientHttpRequestFactory`, now requires Apache HttpClient 5.

The Spring-provided Servlet mocks (`MockHttpServletRequest`, `MockHttpSession`) require Servlet 6.0 now,
due to a breaking change between the Servlet 5.0 and 6.0 API jars. They can be used for testing Servlet
5.0 based code but need to run against the Servlet 6.0 API (or newer) on the test classpath. Note that
your production code may still compile against Servlet 5.0 and get integration-tested with Servlet 5.0
based containers; just mock-based tests need to run against the Servlet 6.0 API jar.

`SourceHttpMessageConverter` is not configured by default anymore in Spring MVC and `RestTemplate`.
As a consequence, Spring web applications using `javax.xml.transform.Source` now need to configure
`SourceHttpMessageConverter` explicitly. Note that the order of converter registration is important,
and `SourceHttpMessageConverter` should typically be registered before "catch-all" converters like
`MappingJackson2HttpMessageConverter` for example.