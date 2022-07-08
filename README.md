# Camel Migration
Exploring how to migrate a small Camel 2.x application to Camel 3.x

Camel is a very flexible framework that is designed around [Enterprise Integration Patterns (EIP)](https://www.enterpriseintegrationpatterns.com/).
The transition from the 2.x to 3.x represents a major and non-backward compatible change. The 3.0.0 release came out in November 2019.
This repo will explore how to use the various Camel migrations guides to migrate from Camel 2.x to Camel 3.x using a small application.

The application will be built using the Spring Java-Config capability to configure the `CamelContext`. 
This expects a configuration class to extend the `CamelConfiguration` which will allow configuration of Camel context.

The starting point:
- a small Camel 2.x application
- using Spring 5.x
- using JDK 8

The goal:
- a small Camel 3.x application
- using Spring 5.x
- using JDK 17

This application will consume EDI X12 messages from a directory and write them to a directory as JSON
(using [the Gozer parser](https://github.com/walmartlabs/gozer) and object model).

## Camel LTS releases
With 3.x Camel [moved to an LTS model like the JDK](https://camel.apache.org/blog/2020/03/LTS-Release-Schedule/).
The LTS versions will be supported for 1 year, be more stable and will not get new features.
The non-LTS versions will not have patch releases and will feature more innovations.
The LTS versions:
- 3.4.0 (EOL in June 2021)
- 3.7.0 (EOL in Dec 2021)
- 3.11.0 (EOL in June 2022)
- 3.14.0 (EOL in Dec 2022)

Some major changes start with the non-LTS release of Camel 3.15.0
- [removes support for JDK 8](https://camel.apache.org/releases/release-3.15.0/)
- [removes support for spring-java-config](https://issues.apache.org/jira/browse/CAMEL-17354)

In addition, Camel 3.17 (non-LTS) will support JDK 17 as well as JDK 11

# Camel as a standalone application
This application is built using Camel `Main` which allows the [application to be run as a standalone Java application](http://people.apache.org/~dkulp/camel/running-camel-standalone.html)
(instead of packaging it up as a WAR). The application will run until it is terminated (Ctrl-C).

**Notes**
- Camel 2.25.x would not work with the Camel Spring Main standalone approach so decided to use the 2.24.x versions
- Camel Spring Main in 2.x requires a Spring XML file to kick things off

## Building and starting the application
Build the application

	mvn clean install

Run the application

	mvn camel:run

# The Migration Plan
- (Starting point) A Camel 2.24.x application
- (Phase 1) Migrate the Camel application from 2.24.x to Camel 3.14.x.
  - this moves the application to the current LTS
  - it holds off on dealing with the JDK and Java-Config changes
- (Phase 2) Migrate the application from 3.14.0 to Camel 3.17.0
  - move to JDK 17
  - evaluate options with Java-Config removal

The branch `camel2x` will contain the Camel 2.x version of the application
To access that version of the code locally

    git checkout tags/camel2x -b camel2xBranch

The branch `camel3xPhase1` will contain the Camel 3.x version of the application after phase 1 is completed
To access that version of the code locally

    git checkout tags/camel3xPhase1 -b camel3xPhase1Branch

The `main` branch will contain the Camel 3.x version of the application after phase 2 is completed


# Major Highlights for Camel Migration (Phase 1)
The [Apache Camel migration guide](https://camel.apache.org/manual/camel-3-migration-guide.html) is being used to start things off.

### Change the Camel version in the pom
Change the Camel version from

    <camel.version>2.24.3</camel.version>

To

    <camel.version>3.14.3</camel.version>

The majority of the errors will be in compiling the unit tests

![Issues](Camel2to3issues.png)

Here are some run-time errors

    Caused by: org.apache.camel.spring.javaconfig.CamelSpringJavaconfigInitializationException: org.apache.camel.FailedToCreateRouteException: Failed to create route gozerSplitMessage at: >>> Log[Finished splitting X12 File into ${property.total.tx.sets} documents] <<< in route: Route(gozerSplitMessage)[From[direct:gozerSplitMessage] -> [... because of Unknown function: property.total.tx.sets at location 33
    Finished splitting X12 File into ${property.total.tx.sets} documents

### Looking at some of the changes
- the simple language `property` function was deprecated in Camel 2.x and has been removed. Use `exchangeProperty` instead.

         .log(LoggingLevel.INFO, "Finished splitting X12 File into ${exchangeProperty.total.tx.sets} documents")

- the `getMessage` method on the `Exchange` is the preferred approach to accessing the headers, body and other data over `getIn` and `getOut`
  -- Note: the `getOut` method was deprecated. The `getIn` was not. Since these methods return a `Message` this sample application was updated to use `getMessage` instead of `getIn`

### Getting the tests running again
- the `DefaultExchange` was imported from `import org.apache.camel.impl`. That needs to be changed to `org.apache.camel.support`

- the `JndiRegistry` and `createRegistry` are no longer needed.

          //
          // old way
          //
          @Override
          protected JndiRegistry createRegistry() throws Exception {
              JndiRegistry registry = super.createRegistry();
              registry.bind("x12SplitterProcessor", x12SplitterProcessor);
              return registry;
          }

         //
         // new way
         //
         context.getRegistry().bind("x12SplitterProcessor", x12SplitterProcessor);

- the `adviceWith` method was replaced with `AdviceWith`.

          //
          // old way
          //
-         context.getRouteDefinition(CamelConstants.WRITE_FILE_ROUTE_ID).adviceWith(context, new RouteBuilder() {
          @Override
          public void configure() {
                  interceptSendToEndpoint(CamelConstants.OUTBOX_ENDPOINT)
                      .skipSendToOriginalEndpoint()
                      .to(MOCK_OUTBOX_ENDPOINT);
              }
          });

          //
          // new way
          //
          AdviceWith.adviceWith(context, CamelConstants.WRITE_FILE_ROUTE_ID, a -> {
              a.interceptSendToEndpoint(CamelConstants.OUTBOX_ENDPOINT)
                    .skipSendToOriginalEndpoint()
                    .to(MOCK_OUTBOX_ENDPOINT);
          });

- There were issues with using `AdviceWidth` when trying to intercept and mock an endpoint from `onException`
  -- solved it by creating a separate route to write the parser error file which could be intercepted 