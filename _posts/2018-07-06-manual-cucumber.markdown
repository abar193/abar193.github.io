---
layout: post
title:  "Running manual Cucumber tests in Java"
date:   2018-07-06
categories: cucumber
tags: [java, cucumber, manual]
---
# Running manual Cucumber tests in Java

In theory, it would be nice to have everything automated. In reality, it might be the case that for some reasons you have to run and manually verify test results.

There can be a number of ways for running manual Cucumber tests, for Ruby there's even whole [CukeHub](https://cukehub.com/) to run and organize manual tests, but my research yielded no results on how to do manual testing in Java.

So, let's implement it our self.

## Used example

I picked some simple [demo for ATM](https://github.com/pavradev/cucumber-demo). It had some outdated dependencies, but could serve as a good starting point.

## Defining feature

Let's take one of scenarios in feature file and add the desired functionality

###### src/test/resources/atm/withdraw.feature 
```
    @manual
    Scenario: Manual test 1
        Given I have 50 SEK on my account
        When I withdraw -1 SEK
        Then I get 0 SEK from the ATM
        And error message about incorrect amount is displayed
        And evaluate manually
```

Step we want is the last one. Cucumber should run its tests, evaluate some of the results and then await for users' final decision. Since we want to exclude manual tests from automated runs, we should also annotate those scenarios somehow. I went with '@manual' tag.

Now to define this step


###### src/test/java/atm/WithdrawSteps.java
```java
    @Then("^evaluate manually$")
    public void evaluate_manually() throws Throwable {
        System.out.println("Mark feature as [p]assed or [f]ailed?");
        Scanner sc = new Scanner(System.in);
        String s = sc.nextLine();
        while (!s.equals("f") && !s.equals("p")) {
            System.out.println("Unknown input: " + s);
            s = sc.nextLine();
        }
        assertThat("User marked test as failed", s, is("p"));
    }
```

Keep asking user to provide his assessment until he finally gives up and gives us what we want — either "p" or "f". Then throw an exception if input isn't "p". With such simple code nothing can go wrong, right?

Wrong.

## Problem with Maven

If you run your Cucumber tests with 'mvn clean test' (the way every Cucumber tutorial suggests) then you get but one problem.
Maven blocks System input, preventing our Scanner from giving us any input -- no matter how hard user smashes his "enter" key, our app just won't receive a single line of input and will stay soft locked.

To work around this problem and get an easy-to-use solution we need 2 components. One is 'exec-maven-plugin', which allows us to execute custom java code bypassing mvn test restrictions, and second is [cucumber.api.cli.Main usage info](https://github.com/cucumber/cucumber-jvm/blob/master/core/src/main/resources/cucumber/api/cli/USAGE.txt). All we need to do is add exec-maven-plugin, specify 'mainClass' as 'cucumber.api.cli.Main', define 'classpathScope' as 'test' and provide in arguments path to our feature and glue code files, and Cucumber plugins.

## Result

###### pom.xml
```xml
    <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>1.6.0</version>
        <executions>
            <execution>
                <goals>
                    <goal>java</goal>
                </goals>
                <!-- Uncomment the following line if you want to run manual tests with mvn test command -->
                <!--<phase>test</phase>-->
            </execution>
        </executions>
        <configuration>
            <mainClass>cucumber.api.cli.Main</mainClass>
            <classpathScope>test</classpathScope>
            <arguments>
                <!-- Specify the location of .feature files -->
                <argument>classpath:atm</argument>
                <!-- Specify the location of gluecode classes - in our case that's WithdrawSteps -->
                <argument>--glue</argument>
                <argument>classpath:atm</argument>
                <!-- Define plugins - first one ensures that console output is colorful,
                    second one gives location for report file -->
                <argument>--plugin</argument>
                <argument>pretty</argument>
                <argument>-p</argument>
                <argument>json:target/cucumber/manual.json</argument>
                <!-- Cucumber Tag expression.  -->
                <argument>--tags</argument>
                <argument>@manual</argument>
            </arguments>
        </configuration>
    </plugin> 
```

I find it best to annotate Runner class to prevent manual tests from being executed with automated:

###### src/test/java/atm/RunCukesTest.java
```java
@RunWith(Cucumber.class)
@CucumberOptions(plugin = {"pretty", "json:target/cucumber/cucumber.json"},
        tags = {"not @manual"})
public class RunCukesTest {
}
```

## Running

Now we can run automated tests with `mvn test` and manual with `mvn test-compile exec:java`. 
With current configuration it will provide 2 json report files which can be merged in several ways, depending on how you store your results. 

If you prefer to have results in html format, you can merge reports by copying one report.js file starting from second line and excluding last into penultimate line of another report.js.

## Example 

Complete example for this post can be found [here](https://github.com/abar193/cucumber-demo).