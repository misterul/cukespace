Cucumber-JVM/Arquillian Integration
===================================

This project allows you to deploy and run Cucumber features into the
application server of your choice using the Arquillian test framework.

## Supported application servers:

The following application servers are supported with Cucumber-JVM/Arquillian.
The artifact ```cucumber-arquillian-core``` is required for all servers, and
additional dependencies have been listed for each.

| Server      | Additional Dependencies   |
|-------------|---------------------------|
| JBoss AS 7  | cucumber-arquillian-jbas7 |
| Glassfish 3 |                           |

# Quickstart

This quickstart assumes you're already very familiar with [Arquillian](http://www.arquillian.org/)
and [Cucumber-JVM](http://www.github.com/cucumber/cucumber-jvm).

## Installation

Before you start writing features, you'll want to pull down the source for
Cucumber-JVM/Arquillian and install it to your local repository using the
following command:

```mvn install```

If you're feeling confident, you can even do it without testing:

```mvn install -DskipTests```

## Project Setup

You'll want at least the following dependency in your pom.xml:

```xml
<dependency>
    <groupId>info.cukes.runtime.arquillian</groupId>
    <artifactId>cucumber-arquillian-core</artifactId>
    <version>{VERSION}</version>
    <scope>test</scope>
</dependency>
```

You'll also want to add dependencies for the application server you wish to
test against. Here's an example dependency for JBoss AS 7:

```xml
<dependency>
    <groupId>info.cukes.runtime.arquillian</groupId>
    <artifactId>cucumber-arquillian-jbas7</artifactId>
    <version>{VERSION}</version>
    <scope>test</scope>
</dependency>
```

## Creating Features for Server-side Execution

All you have to do is extend ```Cucumber```, create the test deployment, and
tailor the Cucumber runtime options:

```java
package my.features;

import cucumber.runtime.arquillian.junit.Cucumber;
import my.features.domain.Belly;
import my.features.glue.BellySteps;

public class CukesInBellyFeature extends Cucumber {
    
    @Deployment
    public static Archive<?> createDeployment() {
        return ShrinkWrap.create(WebArchive.class)
            .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml")
            .addAsResource("my/features/cukes.feature")
            .addClass(Belly.class)
            .addClass(BellySteps.class)
            .addClass(CukesInBellyFeature.class);
    }
    
    @Override
    protected void initializeRuntimeOptions() {
        RuntimeOptions runtimeOptions = this.getRuntimeOptions();
        runtimeOptions.featurePaths.add("classpath:my/features");
        runtimeOptions.glue.add("classpath:my/features/glue");
    }
}
```

Arquillian will then package up all the necessary dependencies along with your
test deployment and execute the feature in the application server. Your step
definitions will also be serviced by Arquillian's awesome test enrichers, so
your steps will have access to any resource supported by the Arquillian
container you choose to use:

```java
public class CukesInBellySteps {
    
    @EJB
    private CukeService service;
    
    @Resource
    private Connection connection;
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Inject
    private CukeLocator cukeLocator;
    
    @When("^I persist my cuke$")
    public void persistCuke() {
        this.entityManager.persist(this.cukeLocator.findCuke());
    }
}
``` 

### Creating Features for Functional UI Testing

[This guide](http://arquillian.org/guides/functional_testing_using_drone/) will
help you get started with using the Arquillian Drone extension for functional
testing.

To create features for functional UI testing, you first want to add all
necessary Drone dependencies to your project's POM, then mark your deployment
as untestable and inject a webdriver:

```java
public class CukesInBellyClientFeature extends Cucumber {
    
    @Drone
    DefaultSelenium browser;
    
    @Deployment(testable = false)
    public static Archive<?> createDeployment() {
        return ShrinkWrap.create(WebArchive.class)
            .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml")
            .addAsWebInfResource(new StringAsset("<faces-config version=\"2.0\"/>"), "faces-config.xml")
            .addAsWebResource(new File("src/main/webapp/belly.xhtml"), "belly.xhtml")
            .addClass(Belly.class)
            .addClass(BellyController.class);
    }
    
    // ...
}
```

You can then access your Drone from any step definition.

```java
public class IrresistibleButtonSteps {
    
    @Drone
    private DefaultSelenium browser;
    
    @When("^I click on an irresistible button$")
    public void click() {
        this.browser.click("id=irresistible-button");
    }
}
```

Be sure to remember to inject the webdriver into your test fixture, or you
won't be able to inject it into any of your step definitions. You'll know when
you've forgotten because you'll get the following error:

```java.lang.IllegalArgumentException: Drone Test context should not be null```

## Running the Examples

To run the default configuration:

```mvn verify``

The following command line properties allow you to specify the target server
and browser:

| Property | Values | Example |
|----------|--------|---------|
| jbas7 | managed, remote | ```mvn verify -Djbas7=managed``` |
| glassfish3 | managed | ```mvn verify -Dglassfish3=managed``` |
| browser | [see here](http://stackoverflow.com/questions/2569977/list-of-selenium-rc-browser-launchers) | ```mvn verify -Dbrowser=*googlechrome``` |

