# REST Assured Service Object Model (SOM) Framework

<div align="center">

**Enterprise-ready API automation framework built with Java, REST Assured, TestNG and the Service Object Model (SOM) pattern.**

![Java](https://img.shields.io/badge/Java-17+-orange?logo=openjdk)
![REST Assured](https://img.shields.io/badge/REST%20Assured-5.x-green)
![TestNG](https://img.shields.io/badge/TestNG-7.x-red)
![Maven](https://img.shields.io/badge/Maven-3.x-blue?logo=apachemaven)
![Framework](https://img.shields.io/badge/Architecture-Service%20Object%20Model-purple)

</div>

> **Purpose:** Build clean, reusable and scalable API automation by separating business test scenarios from API implementation and framework infrastructure.

---

## 📚 Table of Contents

- [Framework Overview](#-framework-overview)
- [Why Service Object Model (SOM)?](#-why-service-object-model-som)
- [Key Principle](#-key-principle-of-som)
- [Benefits of SOM](#-benefits-of-som)
- [Project Structure](#-complete-project-structure)
- [Directory and Class Guide](#-directory-explanation)
- [Request Flow](#-end-to-end-request-flow)
- [Complete User Flow](#-example-complete-user-flow)
- [SOM vs Direct REST Assured](#-som-vs-direct-rest-assured-tests)
- [Design Principles](#-design-philosophy)
- [Enterprise Scalability](#-why-som-is-suitable-for-enterprise-api-automation)
- [Future Enhancements](#-future-enhancements)
- [Interview Explanation](#-one-line-interview-explanation)
- [Architecture Diagram](#-framework-in-one-picture)
- [Conclusion](#-conclusion)

---

## 1. Framework Overview

`rest-assured-som-framework` is an enterprise-style API automation
framework built using **Java, Rest Assured, TestNG, Maven, and the
Service Object Model (SOM)** design pattern.

The primary objective of this framework is to separate:

-   API implementation
-   Test scenarios
-   Request/response models
-   Configuration
-   Authentication
-   Endpoints
-   Assertions
-   Test data
-   Reusable utilities

This separation makes the framework easier to **maintain, scale, debug,
reuse, and extend** as the number of APIs and test cases increases.

The framework follows a layered architecture:

``` text
Test Layer
    |
    v
Service Layer
    |
    +---- Request Models
    +---- Response Models
    +---- Endpoints
    |
    v
Base Service
    |
    +---- Configuration
    +---- Authentication
    |
    v
Rest Assured
    |
    v
Application APIs
```

The architecture and directory organization used here are based on the
supplied framework design. fileciteturn0file0

------------------------------------------------------------------------

# 2. Why Service Object Model (SOM)?

## What problem are we solving?

A basic Rest Assured test can look like this:

``` java
given()
    .baseUri("https://api.example.com")
    .header("Authorization", "Bearer " + token)
    .contentType(ContentType.JSON)
    .body(requestBody)
.when()
    .post("/users")
.then()
    .statusCode(201);
```

This works for a small project, but imagine having:

-   500+ APIs
-   2,000+ test cases
-   multiple authentication mechanisms
-   multiple environments
-   common headers
-   dynamic tokens
-   request/response models
-   retries
-   logging
-   reporting
-   parallel execution

If every test contains Rest Assured implementation, the test suite
becomes difficult to maintain.

For example, if `/users` changes to `/api/v2/users`, we may have to
update the endpoint in many test classes.

SOM solves this problem by moving API operations into **service
classes**.

Instead of:

``` java
@Test
public void createUserTest() {

    given()
        .baseUri(baseUrl)
        .header("Authorization", token)
        .body(request)
    .when()
        .post("/users")
    .then()
        .statusCode(201);
}
```

we write:

``` java
@Test
public void createUserTest() {

    UserRequest request =
            new UserRequest("Saurabh", "SDET");

    Response response =
            userService.createUser(request);

    response.then()
            .statusCode(201);
}
```

The test focuses on the **business scenario**, while `UserService` owns
the **API implementation**.

------------------------------------------------------------------------

# 3. Key Principle of SOM

The most important rule is:

> **Service classes perform API operations. Test classes validate
> business scenarios.**

For example:

``` text
UserTest
   |
   | createUser()
   v
UserService
   |
   | POST /users
   v
API
```

This creates a clean separation between **what we are testing** and
**how the API is called**.

------------------------------------------------------------------------

# 4. Benefits of SOM

## 4.1 Reusability

The same service method can be used by multiple tests.

``` java
userService.createUser(request);
```

can be used by:

-   positive tests
-   negative tests
-   regression tests
-   integration tests
-   end-to-end tests

------------------------------------------------------------------------

## 4.2 Maintainability

Suppose the endpoint changes:

``` text
/users
```

to:

``` text
/api/v2/users
```

The change can be centralized in:

``` java
Endpoints.USERS
```

instead of changing dozens of test cases.

------------------------------------------------------------------------

## 4.3 Readable Tests

A test becomes business-readable:

``` java
UserResponse user =
        userService.createUser(request);

assertEquals(user.getName(), "Saurabh");
```

rather than exposing low-level HTTP implementation everywhere.

------------------------------------------------------------------------

## 4.4 Separation of Concerns

Each layer has a specific responsibility.

  Layer             Responsibility
  ----------------- -----------------------------
  Tests             Business scenarios
  Services          API operations
  Request Models    Request payload structure
  Response Models   Response structure
  Base Service      Common Rest Assured setup
  Config            Environment/configuration
  Auth              Token management
  Constants         Endpoints
  Utils             Common helper functionality
  Assertions        Reusable validations

------------------------------------------------------------------------

# 5. Complete Project Structure

``` text
rest-assured-som-framework
│
├── pom.xml
│
├── src
│   ├── main
│   │   └── java
│   │       └── com.company.api
│   │
│   │           ├── base
│   │           │   └── BaseService.java
│   │           │
│   │           ├── services
│   │           │   ├── UserService.java
│   │           │   ├── AccountService.java
│   │           │   └── PaymentService.java
│   │           │
│   │           ├── models
│   │           │   ├── request
│   │           │   │   ├── LoginRequest.java
│   │           │   │   ├── UserRequest.java
│   │           │   │   └── PaymentRequest.java
│   │           │   │
│   │           │   └── response
│   │           │       ├── LoginResponse.java
│   │           │       ├── UserResponse.java
│   │           │       └── PaymentResponse.java
│   │           │
│   │           ├── constants
│   │           │   └── Endpoints.java
│   │           │
│   │           ├── config
│   │           │   └── ConfigManager.java
│   │           │
│   │           ├── auth
│   │           │   └── TokenManager.java
│   │           │
│   │           ├── utils
│   │           │   ├── JsonUtils.java
│   │           │   └── TestDataUtils.java
│   │           │
│   │           └── assertions
│   │               └── ResponseAssertions.java
│   │
│   └── test
│       └── java
│           └── com.company.api.tests
│               ├── UserTest.java
│               ├── PaymentTest.java
│               └── AccountTest.java
│
└── src
    └── test
        └── resources
            ├── config.properties
            └── testdata
```

------------------------------------------------------------------------

# 6. Directory Explanation

## `pom.xml`

Maven's Project Object Model file.

It controls:

-   project dependencies
-   Java version
-   Rest Assured dependency
-   TestNG
-   Jackson
-   Lombok
-   Maven plugins
-   test execution

Typical dependencies include:

``` xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.5.6</version>
</dependency>

<dependency>
    <groupId>org.testng</groupId>
    <artifactId>testng</artifactId>
    <version>7.11.0</version>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.19.2</version>
</dependency>
```

------------------------------------------------------------------------

# 7. `src/main/java`

This directory contains the **framework implementation**.

The code here should generally be reusable across different test
scenarios.

Think of it as:

``` text
src/main/java
       |
       +--- Framework code
```

It should not contain individual business test cases.

------------------------------------------------------------------------

# 8. `base` Package

``` text
base
└── BaseService.java
```

## `BaseService.java`

`BaseService` is the parent class for all service classes.

Its responsibility is to provide common API configuration such as:

-   Base URI
-   Content type
-   Common headers
-   Authentication
-   Request specification
-   Common Rest Assured configuration

Example:

``` java
public class BaseService {

    protected RequestSpecification requestSpec;

    public BaseService() {

        requestSpec = new RequestSpecBuilder()
                .setBaseUri(ConfigManager.get("base.url"))
                .setContentType(ContentType.JSON)
                .build();
    }

    protected RequestSpecification request() {
        return given().spec(requestSpec);
    }
}
```

Now services can extend it:

``` java
public class UserService extends BaseService {
}
```

This prevents duplicate Rest Assured setup.

### Why do we need BaseService?

Without it:

``` text
UserService
    -> creates request specification

PaymentService
    -> creates request specification

AccountService
    -> creates request specification
```

With it:

``` text
                 BaseService
                /     |      \
               /      |       \
       UserService PaymentService AccountService
```

Common functionality is implemented once and reused everywhere.

------------------------------------------------------------------------

# 9. `services` Package

``` text
services
├── UserService.java
├── AccountService.java
└── PaymentService.java
```

This is the **core SOM layer**.

Each service represents a logical API/domain.

------------------------------------------------------------------------

## `UserService.java`

Responsible for user-related API operations.

Typical operations:

``` text
POST /users
GET /users/{id}
PUT /users/{id}
DELETE /users/{id}
```

Example:

``` java
public class UserService extends BaseService {

    public Response createUser(UserRequest requestBody) {

        return request()
                .body(requestBody)
        .when()
                .post(Endpoints.USERS);
    }

    public Response getUser(int userId) {

        return request()
                .pathParam("id", userId)
        .when()
                .get(Endpoints.USER_BY_ID);
    }
}
```

The test does not need to know the HTTP implementation.

It simply calls:

``` java
userService.createUser(request);
```

------------------------------------------------------------------------

## `AccountService.java`

Contains account-related operations.

For example:

``` text
createAccount()
getAccount()
updateAccount()
closeAccount()
getAccountBalance()
```

Example:

``` java
public class AccountService extends BaseService {

    public Response getAccount(String accountId) {

        return request()
                .pathParam("id", accountId)
        .when()
                .get(Endpoints.ACCOUNT_BY_ID);
    }
}
```

In a banking system, this service could represent operations such as:

``` text
Account creation
Account details
Balance
Account status
Beneficiary
Transaction history
```

------------------------------------------------------------------------

## `PaymentService.java`

Contains payment-related APIs.

Typical operations:

``` text
createPayment()
getPaymentStatus()
cancelPayment()
refundPayment()
```

Example:

``` java
public class PaymentService extends BaseService {

    public Response createPayment(
            PaymentRequest requestBody) {

        return request()
                .body(requestBody)
        .when()
                .post(Endpoints.PAYMENTS);
    }

    public Response getPaymentStatus(
            String paymentId) {

        return request()
                .pathParam("id", paymentId)
        .when()
                .get(Endpoints.PAYMENT_BY_ID);
    }
}
```

This structure is particularly useful for financial/payment APIs because
payment operations can be reused across many business scenarios.

------------------------------------------------------------------------

# 10. `models` Package

``` text
models
├── request
└── response
```

Models represent the data exchanged between the test framework and APIs.

There are two major categories:

``` text
Request Model
     |
     v
Application API
     |
     v
Response Model
```

------------------------------------------------------------------------

# 11. `models/request`

Contains Java POJOs representing request payloads.

``` text
request
├── LoginRequest.java
├── UserRequest.java
└── PaymentRequest.java
```

## `LoginRequest.java`

Represents authentication input.

Example JSON:

``` json
{
  "username": "testuser",
  "password": "password"
}
```

Java:

``` java
public class LoginRequest {

    private String username;
    private String password;

    public LoginRequest(String username, String password) {
        this.username = username;
        this.password = password;
    }

    public String getUsername() {
        return username;
    }

    public String getPassword() {
        return password;
    }
}
```

------------------------------------------------------------------------

## `UserRequest.java`

Represents user creation/update payload.

``` java
public class UserRequest {

    private String name;
    private String job;

    public UserRequest(String name, String job) {
        this.name = name;
        this.job = job;
    }

    public String getName() {
        return name;
    }

    public String getJob() {
        return job;
    }
}
```

The advantage is that the test does not need to manually build JSON
strings.

------------------------------------------------------------------------

## `PaymentRequest.java`

Represents payment payload.

For example:

``` json
{
  "sourceAccount": "ACC1001",
  "destinationAccount": "ACC2001",
  "amount": 5000,
  "currency": "INR"
}
```

Java:

``` java
public class PaymentRequest {

    private String sourceAccount;
    private String destinationAccount;
    private double amount;
    private String currency;
}
```

This gives us type-safe request construction.

------------------------------------------------------------------------

# 12. `models/response`

Contains response POJOs.

``` text
response
├── LoginResponse.java
├── UserResponse.java
└── PaymentResponse.java
```

## `LoginResponse.java`

Represents the authentication API response.

Example:

``` json
{
  "token": "abc123",
  "expiresIn": 3600
}
```

Java:

``` java
public class LoginResponse {

    private String token;
    private int expiresIn;

    public String getToken() {
        return token;
    }

    public int getExpiresIn() {
        return expiresIn;
    }
}
```

------------------------------------------------------------------------

## `UserResponse.java`

Represents user API response.

Example:

``` json
{
  "id": 101,
  "name": "Saurabh",
  "job": "SDET"
}
```

------------------------------------------------------------------------

## `PaymentResponse.java`

Represents payment response.

For a banking/payment application, this could contain:

``` text
paymentId
transactionId
status
amount
currency
timestamp
failureReason
```

This allows the framework to deserialize JSON into a Java object.

Example:

``` java
PaymentResponse response =
        paymentService.createPayment(request);
```

------------------------------------------------------------------------

# 13. `constants` Package

``` text
constants
└── Endpoints.java
```

## `Endpoints.java`

Centralizes API endpoint paths.

Example:

``` java
public final class Endpoints {

    private Endpoints() {
    }

    public static final String LOGIN =
            "/auth/login";

    public static final String USERS =
            "/users";

    public static final String USER_BY_ID =
            "/users/{id}";

    public static final String PAYMENTS =
            "/payments";

    public static final String PAYMENT_BY_ID =
            "/payments/{id}";
}
```

### Why centralize endpoints?

Bad:

``` java
post("/users");
get("/users/{id}");
put("/users/{id}");
```

Better:

``` java
post(Endpoints.USERS);
get(Endpoints.USER_BY_ID);
put(Endpoints.USER_BY_ID);
```

Benefits:

-   easier maintenance
-   avoids duplicate strings
-   reduces spelling mistakes
-   supports API version changes
-   provides a single source of truth

------------------------------------------------------------------------

# 14. `config` Package

``` text
config
└── ConfigManager.java
```

## `ConfigManager.java`

Responsible for reading configuration values.

For example:

``` text
base.url
username
password
environment
timeout
```

Example:

``` java
public class ConfigManager {

    private static final Properties properties =
            new Properties();

    static {
        try (InputStream input =
                ConfigManager.class
                    .getClassLoader()
                    .getResourceAsStream("config.properties")) {

            properties.load(input);

        } catch (IOException e) {
            throw new RuntimeException(
                    "Unable to load config", e);
        }
    }

    public static String get(String key) {
        return properties.getProperty(key);
    }
}
```

Usage:

``` java
String baseUrl =
        ConfigManager.get("base.url");
```

This avoids hardcoding environment-specific values.

------------------------------------------------------------------------

# 15. `auth` Package

``` text
auth
└── TokenManager.java
```

## `TokenManager.java`

Manages API authentication.

Responsibilities can include:

-   generating token
-   caching token
-   refreshing expired token
-   providing token to services
-   handling OAuth/JWT authentication

Example:

``` java
public class TokenManager {

    private static String token;

    public static String getToken() {

        if (token == null) {
            token = generateToken();
        }

        return token;
    }
}
```

Then BaseService can use:

``` java
.addHeader(
    "Authorization",
    "Bearer " + TokenManager.getToken()
)
```

This prevents every test from implementing authentication separately.

------------------------------------------------------------------------

# 16. `utils` Package

``` text
utils
├── JsonUtils.java
└── TestDataUtils.java
```

Utilities contain reusable helper functions that do not belong to a
specific service.

------------------------------------------------------------------------

## `JsonUtils.java`

Responsible for JSON-related operations.

Potential responsibilities:

``` text
Object -> JSON
JSON -> Object
Read JSON file
Extract JSON values
Serialize request
Deserialize response
```

Example:

``` java
public static String toJson(Object object) {
    return objectMapper.writeValueAsString(object);
}
```

------------------------------------------------------------------------

## `TestDataUtils.java`

Responsible for generating or loading test data.

Examples:

``` text
random email
random mobile number
random account number
random payment reference
test data from JSON
test data from CSV
```

Example:

``` java
public static String randomEmail() {

    return "user" +
            System.currentTimeMillis() +
            "@example.com";
}
```

This helps tests avoid hardcoded, duplicate test data.

------------------------------------------------------------------------

# 17. `assertions` Package

``` text
assertions
└── ResponseAssertions.java
```

## `ResponseAssertions.java`

Contains reusable API validations.

For example:

``` java
public class ResponseAssertions {

    public static void assertStatusCode(
            Response response,
            int expectedStatusCode) {

        response.then()
                .statusCode(expectedStatusCode);
    }
}
```

More advanced assertions can validate:

``` text
status code
response time
headers
JSON schema
response body
business fields
error codes
```

Example:

``` java
ResponseAssertions.assertStatusCode(
        response, 201);
```

The goal is to avoid repeating complicated validation logic across
tests.

------------------------------------------------------------------------

# 18. `src/test/java`

This directory contains the actual automated test cases.

``` text
src/test/java
└── com.company.api.tests
```

Think of it as:

``` text
src/main/java
    = Framework

src/test/java
    = Test Scenarios
```

------------------------------------------------------------------------

# 19. `UserTest.java`

Contains user-related test scenarios.

Example:

``` java
public class UserTest {

    private UserService userService;

    @BeforeClass
    public void setup() {
        userService = new UserService();
    }

    @Test
    public void createUserTest() {

        UserRequest request =
                new UserRequest(
                        "Saurabh",
                        "SDET");

        Response response =
                userService.createUser(request);

        response.then()
                .statusCode(201)
                .body("name",
                        equalTo("Saurabh"));
    }
}
```

Notice that the test does not contain:

``` text
base URL
token generation
request specification
endpoint string
Rest Assured configuration
```

That is the power of SOM.

------------------------------------------------------------------------

# 20. `PaymentTest.java`

Contains payment business scenarios.

Examples:

``` text
Create payment successfully
Invalid amount
Insufficient balance
Invalid beneficiary
Duplicate payment
Payment status verification
Payment cancellation
Idempotency validation
```

Example:

``` java
@Test
public void createPaymentTest() {

    PaymentRequest request =
            TestDataUtils.createPaymentRequest();

    Response response =
            paymentService.createPayment(request);

    ResponseAssertions.assertStatusCode(
            response, 201);
}
```

------------------------------------------------------------------------

# 21. `AccountTest.java`

Contains account-related scenarios.

Examples:

``` text
Create account
Get account
Verify balance
Update account
Invalid account
Account status
Account closure
```

The test should focus on the expected business behavior rather than HTTP
implementation details.

------------------------------------------------------------------------

# 22. `src/test/resources`

Contains external test resources.

``` text
src/test/resources
├── config.properties
└── testdata
```

This keeps configuration and test data outside Java code.

------------------------------------------------------------------------

# 23. `config.properties`

Example:

``` properties
base.url=https://api.example.com
username=testuser
password=password
```

For multiple environments, you can later introduce:

``` text
config-dev.properties
config-qa.properties
config-stage.properties
config-prod.properties
```

or environment variables.

For CI/CD, secrets should not be committed to GitHub. Use Jenkins/GitHub
Actions secret management or environment variables.

------------------------------------------------------------------------

# 24. `testdata`

This directory stores external test data.

Possible structure:

``` text
testdata
├── users.json
├── payments.json
├── accounts.json
└── login.json
```

Example:

``` json
{
  "name": "Saurabh",
  "job": "SDET"
}
```

This allows test data to be changed without modifying Java test logic.

------------------------------------------------------------------------

# 25. End-to-End Request Flow

A typical request travels through the framework like this:

``` text
UserTest
    |
    | createUser(request)
    v
UserService
    |
    | extends
    v
BaseService
    |
    +---- ConfigManager
    |        |
    |        +---- base.url
    |
    +---- TokenManager
    |        |
    |        +---- JWT/OAuth token
    |
    +---- RequestSpecification
    |
    v
Rest Assured
    |
    | POST /users
    v
Application API
    |
    v
Response
    |
    v
UserResponse / ResponseAssertions
    |
    v
Test Result
```

------------------------------------------------------------------------

# 26. Example: Complete User Flow

## Step 1 --- Create request

``` java
UserRequest request =
        new UserRequest(
                "Saurabh",
                "SDET");
```

## Step 2 --- Call service

``` java
Response response =
        userService.createUser(request);
```

## Step 3 --- Service calls API

``` java
return request()
        .body(requestBody)
.when()
        .post(Endpoints.USERS);
```

## Step 4 --- BaseService supplies configuration

``` text
Base URL
Content-Type
Authorization
Common headers
```

## Step 5 --- Rest Assured executes request

``` text
POST https://api.example.com/users
```

## Step 6 --- Test validates response

``` java
response.then()
        .statusCode(201);
```

------------------------------------------------------------------------

# 27. Separation of Responsibilities

A strong SOM framework follows this rule:

``` text
                    TEST
                     |
              "What should happen?"
                     |
                     v
                  SERVICE
                     |
              "How do I call API?"
                     |
                     v
              BASE SERVICE
                     |
           "How is API configured?"
                     |
                     v
              REST ASSURED
```

### 🧪 Test Layer

Responsible for:

``` text
Scenario
Expected behavior
Business validation
Test data combination
```

### ⚙️ Service Layer

Responsible for:

``` text
HTTP method
Endpoint
Path parameters
Query parameters
Request body
API operation
```

### 🧱 Base Layer

Responsible for:

``` text
Base URI
Headers
Authentication
Request specification
Common Rest Assured configuration
```

### 📦 Model Layer

Responsible for:

``` text
Request objects
Response objects
Serialization
Deserialization
```

------------------------------------------------------------------------

# 28. SOM vs Direct Rest Assured Tests

## Without SOM

``` java
@Test
public void paymentTest() {

    given()
        .baseUri(baseUrl)
        .header("Authorization", token)
        .contentType(ContentType.JSON)
        .body(paymentJson)
    .when()
        .post("/payments")
    .then()
        .statusCode(201);
}
```

Problems:

-   duplicated configuration
-   duplicated endpoints
-   duplicated authentication
-   tests become large
-   difficult maintenance
-   low reusability

## With SOM

``` java
@Test
public void paymentTest() {

    PaymentRequest request =
            new PaymentRequest(...);

    Response response =
            paymentService.createPayment(request);

    response.then()
            .statusCode(201);
}
```

Much cleaner.

------------------------------------------------------------------------

# 29. SOM Design Principle

A useful way to remember SOM is:

> **One service represents a logical API capability/domain, and each
> method represents an API operation.**

For example:

``` text
UserService
    |
    +-- createUser()
    +-- getUser()
    +-- updateUser()
    +-- deleteUser()

PaymentService
    |
    +-- createPayment()
    +-- getPaymentStatus()
    +-- cancelPayment()
    +-- refundPayment()

AccountService
    |
    +-- createAccount()
    +-- getAccount()
    +-- getBalance()
    +-- closeAccount()
```

------------------------------------------------------------------------

# 30. Why SOM Is Suitable for Enterprise API Automation

For an enterprise API automation project, the number of APIs and tests
grows continuously.

SOM gives us a foundation where new APIs can be added without disturbing
existing tests.

For example, adding:

``` text
BeneficiaryService
TransactionService
KycService
NotificationService
CardService
```

does not require redesigning the entire framework.

We simply create additional service objects extending `BaseService`.

``` java
public class BeneficiaryService
        extends BaseService {
}
```

This makes the architecture scalable.

------------------------------------------------------------------------

# 31. Future Enhancements

The current structure can be extended with enterprise features such as:

``` text
Schema Validation
Contract Testing
Pact
Retry Mechanism
Request/Response Logging
Allure Reporting
Extent Reporting
Parallel Execution
Thread-safe Token Management
Multiple Environments
OAuth 2.0
JWT
Data Factory
Database Validation
Kafka Validation
Docker
Jenkins/GitHub Actions
Performance Testing
JMeter
Security Testing
API Mocking
AI-assisted Test Generation
AI-based Failure Analysis
```

These can be added without changing the fundamental SOM architecture.

------------------------------------------------------------------------

# 32. Design Philosophy

The framework follows these principles:

1.  **Single Responsibility** --- each class has a focused
    responsibility.
2.  **DRY** --- common API configuration is not duplicated.
3.  **Separation of Concerns** --- tests, services, models,
    configuration, authentication and utilities are separated.
4.  **Reusability** --- service methods and utilities can be reused
    across tests.
5.  **Maintainability** --- endpoint/configuration changes are
    centralized.
6.  **Scalability** --- new API domains can be added as new service
    classes.
7.  **Readability** --- tests describe business behavior rather than
    low-level HTTP implementation.
8.  **Testability** --- individual layers can be independently
    maintained and enhanced.

------------------------------------------------------------------------

# 33. Quick Reference

  Directory/Class                        Purpose
  -------------------------------------- ----------------------------------
  `pom.xml`                              Dependency and build management
  `base/BaseService.java`                Common API configuration
  `services/UserService.java`            User API operations
  `services/AccountService.java`         Account API operations
  `services/PaymentService.java`         Payment API operations
  `models/request`                       Request payload POJOs
  `models/response`                      Response POJOs
  `constants/Endpoints.java`             Centralized endpoint paths
  `config/ConfigManager.java`            Configuration management
  `auth/TokenManager.java`               Authentication/token management
  `utils/JsonUtils.java`                 JSON operations
  `utils/TestDataUtils.java`             Test data generation/management
  `assertions/ResponseAssertions.java`   Reusable response validations
  `UserTest.java`                        User scenarios
  `PaymentTest.java`                     Payment scenarios
  `AccountTest.java`                     Account scenarios
  `config.properties`                    Environment/configuration values
  `testdata`                             External test data

------------------------------------------------------------------------

# 34. One-Line Interview Explanation

If an interviewer asks **"Why did you use Service Object Model in your
Rest Assured framework?"**, a strong answer is:

> "I used the Service Object Model to separate API implementation from
> test scenarios. Each service class encapsulates operations for a
> specific business domain, while common configuration and
> authentication are centralized in BaseService. Request and response
> POJOs provide type-safe payload handling, and tests focus only on
> business validation. This improves reusability, maintainability,
> readability and scalability as the API suite grows."

------------------------------------------------------------------------

# 35. Framework in One Picture

``` text
                       TEST LAYER
          ┌──────────────┼──────────────┐
          │              │              │
      UserTest      PaymentTest    AccountTest
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                    SOM SERVICES
          ┌──────────────┼──────────────┐
          │              │              │
     UserService   PaymentService  AccountService
          │              │              │
          └──────────────┼──────────────┘
                         │
               ┌─────────┼─────────┐
               │         │         │
          Endpoints   Request    Response
                     Models      Models
               │
               ▼
             BaseService
               │
        ┌──────┴────────┐
        │               │
 ConfigManager     TokenManager
        │               │
        └──────┬────────┘
               │
               ▼
           Rest Assured
               │
               ▼
             API
```

------------------------------------------------------------------------

## Conclusion

The main goal of this framework is not simply to send HTTP requests
using Rest Assured. The goal is to build a **maintainable API automation
platform** where test scenarios remain clean while the complexity of API
communication is encapsulated inside reusable framework components.

The **Service Object Model is the central architectural pattern**
because it creates a clear boundary between:

``` text
Business Scenario
       ↓
Service/API Operation
       ↓
Common API Infrastructure
       ↓
Application
```

This approach provides a strong foundation for scaling the framework
from a small API test suite to an enterprise-level automation solution.
