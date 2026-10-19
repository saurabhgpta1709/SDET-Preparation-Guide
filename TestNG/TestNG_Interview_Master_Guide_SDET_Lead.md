# TestNG Interview Master Guide — Basics to SDET Lead

This guide takes you from beginner-level TestNG concepts to SDET Lead interview scenarios, with Java examples, framework design, parallel execution, debugging, and CI/CD.

## Learning roadmap

1. **Fundamentals:** TestNG, annotations, lifecycle, assertions, `testng.xml`, groups, priorities, dependencies.
2. **Framework engineering:** Data providers, listeners, retries, factories, reusable framework structure.
3. **Advanced execution:** Parallelism, thread safety, test isolation, reporting, Maven.
4. **SDET Lead scenarios:** Architecture, flaky tests, CI/CD strategy, scalability, code review, release quality gates.

---

# Part 1: TestNG fundamentals

## 1. What is TestNG?

**Interview answer:** TestNG is a Java testing framework inspired by JUnit. It provides annotations, test configuration, grouping, parameterization, dependencies, parallel execution, listeners, and suite orchestration.

TestNG stands for *Test Next Generation*.

### Why use it?

For a suite containing hundreds or thousands of API, UI, and mobile tests, TestNG helps you:

- Execute smoke tests before deployments and regression tests nightly.
- Run independent tests in parallel.
- Supply multiple datasets to one test method.
- Capture failures and generate reports.
- Integrate test execution with Maven and CI/CD.

```java
import org.testng.Assert;
import org.testng.annotations.Test;

public class LoginTest {
    @Test
    public void validLoginShouldSucceed() {
        String actualStatus = "SUCCESS";
        Assert.assertEquals(actualStatus, "SUCCESS");
    }
}
```

`@Test` identifies the test method. The assertion verifies the result. TestNG executes the test and records its outcome.

### TestNG vs. JUnit 5

| Feature | TestNG | JUnit 5 |
|---|---|---|
| Test grouping | Groups | Tags |
| Data-driven tests | `@DataProvider` | `@ParameterizedTest` |
| Method dependencies | `dependsOnMethods` | No direct equivalent in the core model |
| XML suite configuration | `testng.xml` | Different configuration mechanisms |
| Parallel execution | Built-in suite/test/data-provider configuration | Supported through platform and execution configuration |
| Extensions | Listeners | Extensions |

Neither is universally better. Choose based on framework requirements, existing tooling, and team familiarity.

## 2. How do you add TestNG to Maven?

Add the dependency to `pom.xml` (the version is an example; use a version approved for your project):

```xml
<dependency>
    <groupId>org.testng</groupId>
    <artifactId>testng</artifactId>
    <version>7.11.0</version>
    <scope>test</scope>
</dependency>
```

Example structure:

```text
src/test/java/
  tests/
  listeners/
  utils/
pom.xml
testng.xml
```

Run tests:

```bash
mvn test
```

Run a specific suite:

```bash
mvn test -Dsurefire.suiteXmlFiles=testng.xml
```

Configure Maven Surefire to use the intended suite file and pin the plugin version in a production project.

## 3. What are the important TestNG annotations?

| Annotation | Purpose |
|---|---|
| `@Test` | Marks a test method |
| `@BeforeSuite` / `@AfterSuite` | Suite-level setup and teardown |
| `@BeforeTest` / `@AfterTest` | Before/after methods in an XML `<test>` block |
| `@BeforeClass` / `@AfterClass` | Before first test method / after test methods in a class instance |
| `@BeforeMethod` / `@AfterMethod` | Before/after each applicable test method |
| `@DataProvider` | Supplies test data |
| `@Factory` | Creates test class instances |
| `@Parameters` | Reads XML-defined parameters |
| `@Listeners` | Registers listener implementations |

Example:

```java
public class PaymentTest {
    @BeforeClass
    public void setupClass() {
        System.out.println("Initialize reusable resources");
    }

    @BeforeMethod
    public void setupMethod() {
        System.out.println("Prepare test-specific data");
    }

    @Test
    public void paymentSuccess() {
        System.out.println("Execute payment test");
    }

    @AfterMethod(alwaysRun = true)
    public void cleanupMethod() {
        System.out.println("Clean test data");
    }

    @AfterClass(alwaysRun = true)
    public void cleanupClass() {
        System.out.println("Release class resources");
    }
}
```

Use `@BeforeMethod` and `@AfterMethod` for per-test setup and cleanup. Avoid sharing mutable state across parallel tests.

## 4. Explain the TestNG lifecycle

For a simple suite with one class and one test method, the conceptual order is:

1. `@BeforeSuite`
2. `@BeforeTest`
3. `@BeforeClass`
4. `@BeforeMethod`
5. `@Test`
6. `@AfterMethod`
7. `@AfterClass`
8. `@AfterTest`
9. `@AfterSuite`

Actual ordering depends on suite structure, configuration dependencies, groups, inheritance, and execution mode.

**Remember:**
- **Suite:** overall suite setup/teardown.
- **Test:** XML `<test>` configuration.
- **Class:** class-level setup/teardown.
- **Method:** setup/teardown around each test method.

`@BeforeTest` does **not** mean before every `@Test` method. It refers to an XML `<test>` block.

## 5. `@BeforeMethod` vs. `@BeforeClass`

- `@BeforeMethod` normally runs before each applicable test method.
- `@BeforeClass` normally runs once before the first test method in that class instance.

For 10 applicable methods, `@BeforeMethod` typically runs 10 times, while `@BeforeClass` runs once for that instance.

Use class-level setup for genuinely reusable resources and method-level setup for test-specific data and state. Shared mutable class fields are risky when methods run concurrently.

## 6. What is `testng.xml`?

It configures test classes, groups, parameters, listeners, and execution settings.

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="RegressionSuite" parallel="false">
    <test name="PaymentTests">
        <classes>
            <class name="tests.PaymentTest"/>
            <class name="tests.RefundTest"/>
        </classes>
    </test>
</suite>
```

Run with:

```bash
mvn test -Dsurefire.suiteXmlFiles=testng.xml
```

Separate suites may be maintained for smoke, regression, API, and mobile testing.

## 7. `priority` vs. `dependsOnMethods`

**Priority** influences the order in which eligible tests are scheduled. Lower numbers generally run before higher numbers when comparing otherwise eligible tests in the same context.

```java
@Test(priority = 1)
public void createUser() {}

@Test(priority = 2)
public void validateProfile() {}
```

Priority does not require the first test to succeed before the second runs.

A dependency expresses a prerequisite:

```java
@Test
public void createUser() {}

@Test(dependsOnMethods = "createUser")
public void validateProfile() {}
```

By default, if the dependency fails, the dependent test is skipped.

**Lead-level recommendation:** Avoid long dependency chains. Prefer independent tests with their own setup and cleanup, especially for parallel execution.

## 8. Hard vs. soft assertions

A hard assertion stops the current test method when it fails:

```java
Assert.assertEquals(actualStatus, "SUCCESS");
Assert.assertEquals(actualCode, 200);
```

If the first assertion fails, the second line is not reached.

A soft assertion collects failures until `assertAll()`:

```java
import org.testng.asserts.SoftAssert;

@Test
public void validatePaymentResponse() {
    SoftAssert soft = new SoftAssert();
    soft.assertEquals(actualStatus, "SUCCESS");
    soft.assertEquals(actualCode, 200);
    soft.assertEquals(actualCurrency, "INR");
    soft.assertAll();
}
```

Create a fresh `SoftAssert` for each test execution. Omitting `assertAll()` can mean the collected failures do not fail the test. Do not continue unsafe actions after a critical prerequisite fails.

## 9. How do you group tests?

Groups let you select tests by purpose or feature:

```java
@Test(groups = {"smoke", "api"})
public void verifyHealthEndpoint() {}

@Test(groups = {"regression", "payment"})
public void verifyRefundFlow() {}
```

Example XML group filter:

```xml
<suite name="SmokeSuite">
    <test name="SmokeTests">
        <groups>
            <run>
                <include name="smoke"/>
            </run>
        </groups>
        <packages>
            <package name="tests.*"/>
        </packages>
    </test>
</suite>
```

Common groups include `smoke`, `regression`, `sanity`, `api`, `ui`, `mobile`, `critical`, and `slow`. Groups should map to execution intent and pipeline decisions.

## 10. `enabled`, `invocationCount`, and `timeOut`

```java
@Test(enabled = false)
public void disabledTest() {}

@Test(invocationCount = 5)
public void repeatOperation() {}

@Test(timeOut = 3000)
public void shouldCompleteWithinLimit() {}
```

- `enabled = false`: excludes the method.
- `invocationCount = 5`: requests five invocations.
- `timeOut = 3000`: specifies a 3,000 ms timeout.

Repeated tests can create duplicate data or payment side effects if not designed to be repeatable. A TestNG timeout is not a replacement for well-designed API/UI timeouts or bounded polling.

---

# Part 2: Data-driven testing and framework design

## 11. What is `@DataProvider`?

It lets one test method run with multiple input datasets.

```java
@DataProvider(name = "loginData")
public Object[][] getLoginData() {
    return new Object[][] {
        {"validUser", "validPassword", true},
        {"invalidUser", "wrongPassword", false},
        {"validUser", "", false}
    };
}

@Test(dataProvider = "loginData")
public void verifyLogin(
        String username, String password, boolean expectedResult) {
    boolean actualResult = performLogin(username, password);
    Assert.assertEquals(actualResult, expectedResult);
}
```

TestNG invokes the provider, passes each dataset to the test, and records each invocation.

Use it for API payload variations, roles, boundary values, and payment amount/currency validation. For large datasets, use JSON, CSV, databases, or test-data services rather than hardcoding thousands of rows.

## 12. `@DataProvider` vs. `@Parameters`

| `@DataProvider` | `@Parameters` |
|---|---|
| Multiple datasets for a test | Values from TestNG XML |
| Good for positive/negative cases | Good for environment/configuration |
| Can return structured datasets | Commonly configures a test or configuration method |

XML:

```xml
<suite name="Suite">
    <test name="QA">
        <parameter name="baseUrl" value="https://qa.example.com"/>
        <classes>
            <class name="tests.UserApiTest"/>
        </classes>
    </test>
</suite>
```

Java:

```java
@Parameters("baseUrl")
@Test
public void verifyEnvironment(String baseUrl) {
    Assert.assertTrue(baseUrl.startsWith("https://"));
}
```

Use a secret store or protected CI credentials for secrets, not committed XML.

## 13. Can a data provider run tests in parallel?

Yes:

```java
@DataProvider(name = "users", parallel = true)
public Object[][] getUsers() {
    return new Object[][] {
        {"user1"}, {"user2"}, {"user3"}
    };
}

@Test(dataProvider = "users")
public void verifyUser(String user) {
    // Execute isolated test
}
```

Consider shared mutable state, duplicate data, rate limits, and thread safety. Parallel execution is safe only when the tests and supporting infrastructure are designed for concurrency.

## 14. What is `@Factory`?

`@Factory` creates multiple test class instances, often from different configurations.

```java
public class BrowserTest {
    private final String browser;

    public BrowserTest(String browser) {
        this.browser = browser;
    }

    @Test
    public void verifyBrowser() {
        System.out.println("Testing: " + browser);
    }

    @Factory
    public static Object[] createTests() {
        return new Object[] {
            new BrowserTest("chrome"),
            new BrowserTest("firefox"),
            new BrowserTest("edge")
        };
    }
}
```

Use a factory for multiple configured class instances. Use a data provider when one test method should execute with different datasets.

## 15. What is a test dependency?

```java
@Test
public void createAccount() {}

@Test(dependsOnMethods = "createAccount")
public void verifyAccount() {}
```

Dependencies can also be expressed with `dependsOnGroups`. Use them when a real workflow relationship exists. Avoid making unrelated tests depend on previous tests; each test should ideally create or obtain its own required data.

## 16. What is `expectedExceptions`?

```java
@Test(expectedExceptions = IllegalArgumentException.class)
public void negativeAmountShouldBeRejected() {
    validateAmount(-100);
}
```

The test passes if the expected exception is thrown. If it is not thrown, the test fails. For API tests, normally validate the HTTP status, response body, and error contract instead.

## 17. `assertEquals`, `assertTrue`, and `assertThrows`

```java
Assert.assertEquals(statusCode, 200);
Assert.assertTrue(responseBody.contains("SUCCESS"));

Assert.assertThrows(
    IllegalArgumentException.class,
    () -> validateAmount(-100)
);
```

- `assertEquals`: compares values.
- `assertTrue`: checks a condition.
- `assertThrows`: verifies an expected exception.

For a payment API, HTTP 200 alone is not enough; also validate transaction status, identifier, amount, and downstream effects.

## 18. `dependsOnMethods` vs. `alwaysRun`

- `dependsOnMethods` declares an execution dependency.
- `alwaysRun = true` changes behavior for supported annotations and circumstances.

```java
@AfterMethod(alwaysRun = true)
public void cleanup() {
    // Release resources
}
```

Cleanup must be idempotent, tolerate partially initialized resources, and log its own failures. `alwaysRun` does not guarantee cleanup will succeed.

## 19. What is `@Optional`?

It supplies a default value when a TestNG XML parameter is absent:

```java
@Parameters("environment")
@Test
public void verifyEnvironment(
        @Optional("qa") String environment) {
    System.out.println(environment);
}
```

Use defaults carefully. Silently defaulting an important target environment can cause tests to run against the wrong system.

## 20. How do you structure a reusable framework?

Separate test intent from infrastructure and execution concerns.

```text
src/test/java/
  tests/
    LoginTest.java
    PaymentTest.java
  services/
    LoginService.java
    PaymentService.java
  pages/
    LoginPage.java
  listeners/
    TestListener.java
  data/
    TestDataProvider.java
  config/
    ConfigManager.java
  utils/
    TestDataFactory.java
    RetryAnalyzer.java
```

Typical layers:

1. **Test layer:** TestNG tests, assertions, groups, providers.
2. **Service/page layer:** API clients, business actions, page objects.
3. **Infrastructure:** configuration, auth, drivers, DB/Kafka clients.
4. **Execution/reporting:** listeners, logs, reports, Maven, Jenkins.

TestNG should orchestrate tests; it should not contain all HTTP, database, business, and reporting logic.

---

# Part 3: Listeners, retries, and advanced features

## 21. What are TestNG listeners?

Listeners react to execution events without duplicating reporting logic in each test.

| Listener | Main use |
|---|---|
| `ITestListener` | Test lifecycle and outcomes |
| `ISuiteListener` | Suite start and finish |
| `IInvokedMethodListener` | Before/after invoked methods |
| `IExecutionListener` | Overall TestNG execution lifecycle |
| `IAnnotationTransformer` | Modify supported annotations programmatically |
| `IReporter` | Generate reports after execution |
| `IMethodInterceptor` | Modify selected method lists |

Example:

```java
import org.testng.ITestListener;
import org.testng.ITestResult;
import org.testng.ITestContext;

public class TestListener implements ITestListener {
    @Override
    public void onTestStart(ITestResult result) {
        System.out.println("START: " + result.getName());
    }

    @Override
    public void onTestSuccess(ITestResult result) {
        System.out.println("PASS: " + result.getName());
    }

    @Override
    public void onTestFailure(ITestResult result) {
        System.out.println("FAIL: " + result.getName());
        Throwable error = result.getThrowable();
        if (error != null) {
            error.printStackTrace();
        }
    }

    @Override
    public void onTestSkipped(ITestResult result) {
        System.out.println("SKIP: " + result.getName());
    }

    @Override
    public void onStart(ITestContext context) {
        System.out.println("TEST START: " + context.getName());
    }

    @Override
    public void onFinish(ITestContext context) {
        System.out.println("TEST FINISH: " + context.getName());
    }
}
```

Register in XML:

```xml
<suite name="RegressionSuite">
    <listeners>
        <listener class-name="listeners.TestListener"/>
    </listeners>
    <test name="API">
        <classes>
            <class name="tests.PaymentTest"/>
        </classes>
    </test>
</suite>
```

A production listener might record duration, exception details, redacted API request/response information, UI screenshots, environment, and build metadata. Delegate heavy work to dedicated reporting and artifact services.

## 22. `ITestListener` vs. `IInvokedMethodListener`

`ITestListener` focuses on test lifecycle and outcomes. `IInvokedMethodListener` provides hooks around invoked methods, including test and configuration methods.

Use the former for pass/fail reporting and the latter for method-level tracing or timing. Neither replaces clear setup and teardown design.

## 23. How do you implement retries?

```java
import org.testng.IRetryAnalyzer;
import org.testng.ITestResult;

public class RetryAnalyzer implements IRetryAnalyzer {
    private int retryCount = 0;
    private static final int MAX_RETRIES = 2;

    @Override
    public boolean retry(ITestResult result) {
        if (retryCount < MAX_RETRIES) {
            retryCount++;
            return true;
        }
        return false;
    }
}
```

Attach it:

```java
@Test(retryAnalyzer = RetryAnalyzer.class)
public void verifyPaymentStatus() {
    // Execute test
}
```

This requests up to two retries after the initial failure.

A strong framework records the initial failure, each retry and its reason, the eventual outcome, and the retry rate. Retries are a diagnostic mechanism, not a cure for flaky tests. Do not repeat non-idempotent operations blindly.

## 24. Can you rerun only failed tests?

Yes. TestNG supports retry analyzers and failed-test XML output, including a commonly generated `testng-failed.xml` file under its output directory.

These approaches differ:
- `IRetryAnalyzer` requests another attempt during the current execution.
- Failed-test XML can be used for a later rerun.
- CI can launch a controlled rerun using failure artifacts.

Preserve the original report before rerunning so an eventual pass does not erase the first-run failure.

## 25. `IRetryAnalyzer` vs. `IAnnotationTransformer`

`IRetryAnalyzer` decides whether a failed test should be retried. `IAnnotationTransformer` modifies supported TestNG annotations before execution and can assign retry analyzers centrally.

```java
public class RetryTransformer implements IAnnotationTransformer {
    @Override
    public void transform(
            ITestAnnotation annotation,
            Class testClass,
            Constructor testConstructor,
            Method testMethod) {
        annotation.setRetryAnalyzer(RetryAnalyzer.class);
    }
}
```

Register the transformer using a supported mechanism for your TestNG version. Transformer registration has special lifecycle constraints; `@Listeners` is not a universal registration mechanism for every transformer use case.

Avoid blanket retries that hide defects.

## 26. What is `IMethodInterceptor`?

It intercepts selected methods and returns a modified list. Possible uses include custom filtering or controlled selection logic. It is not a replacement for groups or XML suite configuration. Keep selection deterministic and visible so failures can be reproduced.

## 27. What is `IReporter`?

`IReporter` is an extension point for generating a report after TestNG execution. Reports can summarize totals, outcomes, duration, failure details, and environment/build metadata.

An existing reporting library is often preferable to building a full reporting engine from scratch. Reporting must not change test outcomes merely to make a report green.

## 28. What else can `IAnnotationTransformer` do?

It can modify supported annotation attributes before execution, depending on the annotation and TestNG version. This can include selected settings such as enablement, groups, and retry configuration.

Use dynamic behavior sparingly, log changes, and prefer static configuration when it is easier to review and reproduce.

## 29. How do you capture screenshots after Selenium failures?

Use `ITestListener.onTestFailure()` and a screenshot utility. Resolve the driver belonging to the correct test execution or thread.

```java
@Override
public void onTestFailure(ITestResult result) {
    Object instance = result.getInstance();

    if (instance instanceof BaseUiTest) {
        BaseUiTest test = (BaseUiTest) instance;
        test.captureScreenshot(result.getName());
    }
}
```

This assumes `BaseUiTest` exposes a safe capture method. Capture should fail gracefully if the browser has closed or crashed. For API tests, capture redacted request/response diagnostics instead.

## 30. How do you create a custom marker annotation?

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface CriticalTest {}
```

```java
@CriticalTest
@Test
public void verifyPaymentSettlement() {
    // Test logic
}
```

A custom Java annotation is metadata. TestNG does not automatically assign behavior to arbitrary custom annotations; your interceptor, listener, or other framework component must interpret it.

---

# Part 4: Parallel execution and thread safety

## 31. How do you execute tests in parallel?

```xml
<suite name="ParallelSuite"
       parallel="methods"
       thread-count="5">
    <test name="APIRegression">
        <classes>
            <class name="tests.PaymentTest"/>
            <class name="tests.AccountTest"/>
        </classes>
    </test>
</suite>
```

Common modes:

| Mode | What is parallelized |
|---|---|
| `methods` | Test methods |
| `classes` | Test classes |
| `tests` | XML `<test>` blocks |
| `instances` | Test class instances |

Choose based on independence and resource isolation. Parallel execution is a design choice, not just a speed setting.

## 32. `parallel="methods"` vs. `parallel="classes"`

Method-level parallelism can run methods from the same class concurrently. Shared fields become especially risky.

```java
public class PaymentTest {
    private String paymentId;

    @Test
    public void createPayment() {
        paymentId = "PAY-100";
    }

    @Test
    public void verifyPayment() {
        System.out.println(paymentId);
    }
}
```

Under parallel method execution, `verifyPayment()` may run before `createPayment()`. Class-level parallelism typically runs different classes concurrently while keeping methods within each class sequential under the usual configuration.

Better design: make tests independent and obtain their own data.

## 33. How would you run 2,000 tests in parallel without data contamination?

**Interview answer:** I would not simply increase thread count. I would first make tests independent, isolate mutable state, and control shared resources, then tune concurrency based on application and infrastructure capacity.

1. Generate unique data per test invocation.
2. Avoid per-test state in static fields or shared class instances.
3. Use thread-safe clients and immutable configuration.
4. Isolate database records and cleanup.
5. Start with measured concurrency and increase gradually.
6. Monitor response times, errors, resource usage, and rate limits.
7. Log test IDs, data identifiers, environment, and correlation IDs.

Example:

```java
String testRunId = UUID.randomUUID().toString();
String userId = "user-" + testRunId;
```

Uniqueness helps, but does not replace proper creation, ownership, cleanup, or handling of shared business constraints.

## 34. What is `ThreadLocal`?

`ThreadLocal` stores a separate value for each Java thread. It is commonly used for per-thread WebDriver instances.

```java
public class DriverManager {
    private static final ThreadLocal<WebDriver> DRIVER =
        new ThreadLocal<>();

    public static void setDriver(WebDriver driver) {
        DRIVER.set(driver);
    }

    public static WebDriver getDriver() {
        return DRIVER.get();
    }

    public static void unloadDriver() {
        WebDriver driver = DRIVER.get();
        try {
            if (driver != null) {
                driver.quit();
            }
        } finally {
            DRIVER.remove();
        }
    }
}
```

The driver must be created and assigned on the intended thread. Call `remove()` because worker threads can be reused.

`ThreadLocal` does not make other shared objects thread-safe.

## 35. How do you make Rest Assured tests thread-safe?

- Keep environment configuration immutable.
- Avoid modifying a shared mutable request specification.
- Isolate authentication and user-specific state.
- Generate unique test data.
- Avoid static response objects or transaction IDs.
- Make test-data and reporting utilities concurrency-safe.

```java
@Test
public void verifyAccountBalance() {
    RequestSpecification request =
        new RequestSpecBuilder()
            .setBaseUri(baseUrl)
            .addHeader("Authorization", token)
            .build();

    Response response = given()
        .spec(request)
        .get("/accounts/balance");

    Assert.assertEquals(response.statusCode(), 200);
}
```

This is illustrative. Whether a shared specification is safe depends on how it is constructed and whether it is modified after publication.

## 36. How do you avoid race conditions?

A race condition occurs when results depend on the timing of concurrent operations.

Prefer:
- Separate test accounts or records.
- Unique data and explicit ownership.
- Appropriate database transaction isolation.
- Condition-based polling rather than arbitrary sleeps.
- Locks or queues only where exclusive access is genuinely required.

Do not solve every race condition with `synchronized`; it can serialize the suite and hide an isolation problem.

## 37. How does parallelism work with `@DataProvider`?

```java
@DataProvider(name = "accounts", parallel = true)
public Object[][] accounts() {
    return new Object[][] {
        {"account-A"}, {"account-B"}, {"account-C"}
    };
}
```

A suite may configure the provider pool:

```xml
<suite name="Suite" data-provider-thread-count="8">
    <!-- Tests -->
</suite>
```

Do not assume all thread settings across methods, classes, suites, and providers form one universal concurrency limit. Verify behavior for your TestNG version and suite layout.

## 38. TestNG parallelism vs. Maven parallelism

TestNG can parallelize methods, classes, instances, and XML tests. Maven Surefire can also fork JVMs and execute test classes or suites concurrently depending on configuration.

Concurrency can multiply: multiple JVM forks, each with multiple TestNG workers, may create far more load than expected.

Start with one primary concurrency strategy, calculate total potential concurrency, set resource limits, and scale only after measuring stability.

## 39. How do you manage WebDriver lifecycle in parallel?

1. Create a driver during setup.
2. Assign it to the correct thread or isolated test context.
3. Execute the test.
4. Capture diagnostic evidence before shutdown.
5. Quit the driver during teardown.
6. Remove the thread-local reference.

Handle driver-creation failures and partially initialized state. Never use one global static WebDriver for concurrent browser tests.

## 40. How do you investigate intermittent parallel failures?

Compare:
- Test/invocation IDs, thread names, timestamps.
- Correlation IDs and API request/response details.
- Test-data collisions and ownership.
- Browser and driver logs.
- Database and asynchronous event state.
- Service latency, rate limits, and infrastructure health.

Reproduce at lower concurrency, then increase gradually. If a failure disappears without parallelism, it suggests a concurrency or timing issue but does not prove the cause.

---

# Part 5: Execution control, reporting, and CI/CD

## 41. Configuration failure vs. test failure

A test failure occurs in test logic or assertions. A configuration failure occurs in setup or teardown methods such as `@BeforeMethod` or `@AfterMethod`.

Configuration failure examples include browser startup problems, invalid configuration, and unavailable dependencies. Distinguish these from business assertion failures when reporting and triaging defects.

## 42. Failed test vs. skipped test

- **Failed:** The test executed and did not meet expectations, or encountered an unhandled failure.
- **Skipped:** The test did not complete as a normal executed test, perhaps because a dependency failed or it was deliberately skipped.

Critical skipped tests may need to block a release even if the failure count is zero. Release gates should consider failures, skips, flaky retries, and coverage.

## 43. How do you run TestNG in Jenkins?

Typical pipeline:
1. Check out source.
2. Set up JDK and Maven.
3. Load configuration and secrets securely.
4. Run the selected suite.
5. Publish reports.
6. Preserve logs and artifacts.
7. Apply release-quality gates.

```bash
mvn clean test -Dsurefire.suiteXmlFiles=suites/smoke.xml
```

Illustrative Jenkins pipeline:

```groovy
stage('API Smoke Tests') {
    steps {
        sh 'mvn clean test -Dsurefire.suiteXmlFiles=suites/smoke.xml'
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'
            archiveArtifacts(
                artifacts: 'target/surefire-reports/**',
                allowEmptyArchive: true
            )
        }
    }
}
```

Confirm the actual report path and XML format for your project. Preserve reports even when tests fail.

## 44. How do you separate smoke and regression suites?

Example:
- Pull request: smoke and focused API tests.
- Deployment validation: smoke and critical workflows.
- Nightly: broader regression.
- Scheduled extended runs: cross-browser, mobile, and performance tests.

Keep smoke tests fast and valuable. Do not reduce them so much that critical failures go unnoticed.

## 45. What is `preserve-order`?

The XML `preserve-order` setting influences the order in which TestNG processes classes or methods represented in XML, subject to execution mode and scheduling rules.

It does not make a parallel suite sequential and should not be used to create implicit dependencies. Model real dependencies explicitly.

## 46. How do `include` and `exclude` groups work?

```xml
<groups>
    <run>
        <include name="smoke"/>
        <exclude name="slow"/>
    </run>
</groups>
```

Keep group names meaningful and stable. A critical release-validation group should have documented purpose, clear ownership, and predictable membership.

## 47. How do you manage QA, staging, and production configuration?

Keep environment configuration out of test logic.

```text
config/
  qa.properties
  staging.properties
```

Example:

```bash
mvn test -Denv=qa
```

Use secure credentials for secrets. Do not commit passwords or tokens. Restrict production validation to explicitly authorized, non-destructive operations.

## 48. `@BeforeGroups` vs. `@BeforeClass`

`@BeforeGroups` executes before the first test method in specified groups that is about to run. `@BeforeClass` executes before the first test method in a class instance.

Use group setup only when the resource truly belongs to the group. Shared resources can become complicated under parallel execution.

## 49. How do you run one test method?

```bash
mvn test -Dtest=PaymentTest#verifyPayment
```

This depends on the Surefire version and project configuration supporting that selector for the chosen test provider. You can also use a focused XML suite or IDE execution.

## 50. TestNG report vs. CI report

TestNG generates execution results; CI tools or reporting plugins ingest those results and present them. For example, TestNG output may feed Maven Surefire XML, which Jenkins ingests as JUnit-compatible results.

Ensure reruns and retries do not distort pass rates or hide first-run failures.

---

# Part 6: Advanced scenarios and debugging

## 51. Configuration failure vs. test failure

```java
@BeforeMethod
public void setup() {
    driver = createDriver();
    Assert.assertNotNull(driver);
}

@Test
public void verifyHomePage() {
    // Test logic
}
```

If setup fails, the test may not execute normally. Distinguish setup failures from business assertion failures because they often have different root causes.

## 52. What is `configfailurepolicy`?

This suite-level setting controls handling of certain subsequent configuration methods after a configuration failure. Common values are `skip` and `continue`.

```xml
<suite name="Suite" configfailurepolicy="skip">
    <!-- Tests -->
</suite>
```

Do not interpret `continue` as permission to execute tests whose prerequisites are invalid. A test without a valid driver or authenticated client cannot safely proceed.

## 53. How do you test Kafka workflows?

Example payment workflow:
1. Submit payment request.
2. Capture transaction/correlation ID.
3. Validate immediate API response.
4. Wait for the matching Kafka event.
5. Validate event payload and key.
6. Confirm expected database state.
7. Clean up data where appropriate.

Use bounded condition-based polling, correlation IDs, and clear timeout diagnostics. Avoid fixed long sleeps and ensure your consumer can reliably identify the intended event.

## 54. How do you handle flaky tests?

- Categorize failures from evidence.
- Reproduce locally and in CI.
- Inspect timing, data collisions, and shared state.
- Replace arbitrary sleeps with explicit waits or bounded polling.
- Fix isolation and cleanup.
- Track flakiness as a quality metric.
- Quarantine only with an owner and deadline.

A flaky critical test is still a gap in release confidence.

## 55. How would you design a retry policy?

| Test type | Suggested policy |
|---|---|
| Read-only API check | Limited retry may suit known transient failures |
| UI rendering test | Fix synchronization before relying on retries |
| Payment creation | Never retry blindly; use idempotency and reconciliation |
| Database consistency test | Investigate eventual consistency and isolation |
| Contract/schema test | Deterministic assertion failures generally should not be retried |

Retries must be observable, configurable, and safe for the operation being tested.

## 56. TestNG vs. Cucumber

TestNG is a Java test execution framework. Cucumber supports behavior-driven development with executable specifications often written in Gherkin.

```gherkin
Scenario: Successful payment
  Given a valid customer account
  When the customer makes a payment
  Then the payment should be successful
```

Cucumber step definitions implement the behavior. TestNG can be used through an appropriate Cucumber integration. Choose Cucumber when business-readable specifications justify the additional layer.

## 57. Integrate Rest Assured with TestNG

```java
import org.testng.Assert;
import org.testng.annotations.Test;
import static io.restassured.RestAssured.given;

public class HealthApiTest {
    @Test(groups = {"smoke", "api"})
    public void healthEndpointReturnsSuccess() {
        int statusCode =
            given()
                .baseUri("https://api.example.com")
            .when()
                .get("/health")
            .then()
                .extract()
                .statusCode();

        Assert.assertEquals(statusCode, 200);
    }
}
```

The URL is illustrative. Use controlled configuration and reusable clients/services in a real framework.

## 58. Integrate Appium with TestNG

A typical design includes:
- Driver factory for Android/iOS.
- Device/capability configuration.
- Page or screen objects.
- Per-test setup/teardown.
- Device allocation and concurrency control.
- Failure screenshots and device logs.
- CI integration with a device farm or local devices.

Parallel workers must not unintentionally control the same device session.

## 59. Tests pass locally but fail in Jenkins

| Area | Investigate |
|---|---|
| Java/Maven | JDK, Maven, dependencies |
| Configuration | URLs, profiles, missing parameters |
| Secrets | Credential availability and permissions |
| Timing | Slow machines, missing waits, race conditions |
| Parallelism | Shared state and data collisions |
| Infrastructure | Network, DNS, service and driver availability |
| Reports | Stack traces, timestamps, screenshots, request IDs |

Reproduce with the same suite, configuration, and concurrency, then narrow down the differences. Increasing wait time without investigating is not the default fix.

## 60. Migrate an unstable framework to enterprise-ready quality

1. **Baseline:** Measure pass rate, flaky rate, duration, failure categories, and maintenance effort.
2. **Standardize:** Establish conventions for annotations, assertions, naming, and configuration.
3. **Separate concerns:** Move API/UI/database/messaging operations into reusable layers.
4. **Isolate tests:** Remove shared mutable state and use unique data.
5. **Improve observability:** Add consistent logs, correlation IDs, diagnostics, and reports.
6. **Optimize execution:** Introduce controlled parallelism and reliable suite selection.
7. **Enforce quality gates:** Track critical failures, skips, retries, and trends.
8. **Improve continuously:** Review failures and framework metrics with the team.

Faster execution is valuable only when reliability and diagnostic quality remain strong.

---

# Part 7: Five coding exercises

## Exercise 1: Run a test with three datasets

```java
@DataProvider(name = "accountTypes")
public Object[][] accountTypes() {
    return new Object[][] {
        {"SAVINGS", true},
        {"CURRENT", true},
        {"INVALID", false}
    };
}

@Test(dataProvider = "accountTypes")
public void validateAccountType(String type, boolean expectedValid) {
    boolean actualValid = isValidAccountType(type);
    Assert.assertEquals(actualValid, expectedValid);
}

private boolean isValidAccountType(String type) {
    return "SAVINGS".equals(type) || "CURRENT".equals(type);
}
```

Explain how each dataset becomes an invocation and how the expected result maps to business behavior.

## Exercise 2: Ensure teardown runs

```java
@BeforeMethod
public void setup() {
    initializeTestResources();
}

@Test
public void verifyTransaction() {
    Assert.assertTrue(isTransactionSuccessful());
}

@AfterMethod(alwaysRun = true)
public void teardown() {
    releaseTestResources();
}
```

Explain how `alwaysRun` helps teardown remain eligible after failure and why cleanup must tolerate partially initialized resources.

## Exercise 3: Add a listener

Implement `ITestListener`, record the test name and duration, capture the exception on failure, and delegate screenshots/logging to utilities.

## Exercise 4: Avoid duplicate test data

Generate unique data per invocation, track ownership, and clean up safely. Explain what happens if creation succeeds but the test crashes before cleanup.

## Exercise 5: Execute only smoke tests

Mark the appropriate tests with the `smoke` group, select it in XML, and configure Jenkins to run that suite. Explain which failures should block deployment.

---

# Part 8: Common interview traps

| Question | Correct understanding |
|---|---|
| Does `@BeforeTest` run before every `@Test`? | No; it relates to XML `<test>`. |
| Does priority create a dependency? | No; it influences ordering. |
| Does parallelism make shared fields safe? | No; concurrency needs deliberate design. |
| Can soft assertions fail to report errors? | Yes, if `assertAll()` is omitted. |
| Does retrying fix flaky tests? | No; it can hide symptoms. |
| Is `ThreadLocal` a complete thread-safety solution? | No; it isolates only its own values. |
| Should every test depend on one setup test? | Usually not; this creates coupling. |
| Does a green report guarantee release readiness? | No; critical skips and missing coverage matter. |
| Should every API failure be retried? | No; consider failure type and idempotency. |
| Should a listener contain all framework logic? | No; delegate to dedicated services. |

---

# Part 9: Seven-day mastery plan

## Day 1 — Fundamentals
- Create a Maven TestNG project.
- Practice lifecycle annotations.
- Write hard and soft assertions.

## Day 2 — Execution control
- Create `testng.xml`.
- Practice groups and priorities.
- Use dependencies and XML parameters.

## Day 3 — Data-driven testing
- Write a data provider.
- Read test data from JSON or CSV.
- Practice `@Factory` and exception tests.

## Day 4 — Listeners and retries
- Implement `ITestListener`.
- Capture failure diagnostics.
- Implement and measure retry behavior.

## Day 5 — Parallel execution
- Configure parallel methods.
- Use `ThreadLocal` for WebDriver.
- Test unique data and cleanup.

## Day 6 — Enterprise integration
- Integrate Rest Assured.
- Generate Maven reports.
- Execute a suite in Jenkins.

## Day 7 — SDET Lead practice
- Design a TestNG framework.
- Debug a simulated flaky test.
- Answer architecture questions aloud.

---

# Part 10: Ten questions an SDET Lead must answer confidently

1. How would you design a TestNG framework supporting UI, API, and mobile automation?
2. How would you execute 2,000 tests in parallel without data contamination?
3. How do you distinguish a real defect from a flaky test?
4. How do you design retries without hiding defects?
5. How would you integrate TestNG with Maven and Jenkins?
6. How do you capture diagnostic information after failure?
7. How do you reduce regression time without reducing meaningful coverage?
8. How do you handle shared state, thread safety, and isolation?
9. How would you report flaky-test rate, duration, and critical failures?
10. How do you decide which tests block a release?

## Final recommendation

Start with annotations, lifecycle, assertions, groups, and `testng.xml`. Then focus heavily on listeners, retries, parallel execution, and thread safety. Those topics help you move from knowing TestNG to demonstrating SDET Lead-level engineering judgment.
