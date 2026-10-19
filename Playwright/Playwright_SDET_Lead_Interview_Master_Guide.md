# Playwright Interview Master Guide — SDET Lead
**From beginner to advanced | TypeScript | Framework design | CI/CD | Real-world scenarios**

This guide is designed for an experienced QA/SDET who is new to Playwright. The goal is not to memorize APIs: it is to understand the browser model, write reliable tests, design a maintainable framework, and explain trade-offs as an SDET Lead.

> Examples use TypeScript and `@playwright/test`. Replace sample URLs, selectors, credentials, and endpoints with those from your application.

---

## How to study this guide

1. Learn Questions 1–12 and run every code sample.
2. Build a small UI framework using the architecture in Questions 13–22.
3. Practise API, network, authentication, and parallel execution from Questions 23–32.
4. Answer the SDET Lead scenarios in Questions 33–45 out loud.
5. Build the capstone project at the end.

## 1. Fundamentals

### Q1. What is Playwright, and why is it used?

**Answer:** Playwright is an end-to-end browser automation framework. It can drive Chromium, Firefox, and WebKit, interact with web pages, inspect browser behavior, make API requests, intercept network traffic, and produce debugging artifacts.

It is more than a browser-driving library: Playwright Test adds a test runner, assertions, fixtures, parallel execution, retries, projects, reporters, and trace support.

**How it works:** Your test calls Playwright APIs. Playwright communicates with the browser and performs actions such as navigation, clicking, typing, and reading page state. The test runner schedules tests, creates fixtures, collects results, and writes reports.

**Example**
```ts
import { test, expect } from '@playwright/test';

test('user can open the documentation', async ({ page }) => {
  await page.goto('https://playwright.dev/');
  await expect(page).toHaveTitle(/Playwright/);
  await page.getByRole('link', { name: 'Get started' }).click();
  await expect(page.getByRole('heading', { name: 'Installation' })).toBeVisible();
});
```

**Lead-level point:** Choose Playwright when reliable modern web automation, parallel execution, strong diagnostics, browser coverage, and UI/API workflows matter. It does not replace every testing tool: native mobile app automation may still require Appium, and load testing requires a performance tool.

### Q2. Playwright vs Selenium: what are the differences?

| Area | Playwright | Selenium |
|---|---|---|
| Browser communication | Uses browser-specific automation protocols and Playwright's control layer | WebDriver standard and driver/browser implementations |
| Waiting | Auto-waits for actionability; web-first assertions retry | Explicit waits are commonly designed by the team |
| Isolation | Fresh BrowserContext per test by default in Playwright Test | Depends on how driver/session setup is designed |
| Test runner | Playwright Test is integrated | Often paired with TestNG, JUnit, NUnit, etc. |
| Network mocking | Built-in route interception and response mocking | Usually needs additional setup/tools |
| Diagnostics | Trace Viewer, screenshots, videos, reports | Often assembled from framework/listeners/plugins |
| Browser support | Chromium, Firefox, WebKit | Broad browser and language ecosystem |
| Languages | Playwright supports TypeScript/JavaScript, Python, Java, and .NET; Playwright Test is the JS/TS runner | Broad language bindings including Java, Python, C#, JS, Ruby |
| Mobile | Browser emulation; not a replacement for native-app automation | Appium is commonly used for native/mobile apps |

**Interview answer:** “I would not claim one tool is universally better. Playwright offers integrated auto-waiting, isolated contexts, network tooling, and strong traces. Selenium has a mature WebDriver ecosystem, broad language support, and may fit an existing Java/TestNG grid. I would compare reliability, application needs, team skills, grid infrastructure, and migration cost before deciding.”

### Q3. Explain Browser, BrowserContext, and Page.

- **Browser:** A running browser instance, such as Chromium.
- **BrowserContext:** An isolated browser session with its own cookies, local storage, permissions, and session state.
- **Page:** A tab or page inside a context.

Typical hierarchy:

```text
Browser
 ├── Context A (user/session A)
 │    ├── Page 1
 │    └── Page 2
 └── Context B (user/session B)
      └── Page 1
```

Playwright Test normally provides a fresh context and page for each test, reducing state leakage.

```ts
test('two isolated users', async ({ browser }) => {
  const contextA = await browser.newContext();
  const contextB = await browser.newContext();

  const userPage = await contextA.newPage();
  const adminPage = await contextB.newPage();

  await userPage.goto('https://example.com');
  await adminPage.goto('https://example.com');

  await contextA.close();
  await contextB.close();
});
```

**When it matters:** Testing two roles, separate sessions, cookies, permissions, or concurrent users. Do not confuse two pages in the same context with two isolated users: pages in one context share session state.

### Q4. What is a locator? Which locator strategies do you prefer?

A locator is a reusable description of how to find an element. Playwright resolves it when an operation runs and automatically waits for relevant actionability conditions.

Preferred order in most projects:
1. `getByRole()` with accessible role and name.
2. `getByLabel()` for labelled form controls.
3. `getByPlaceholder()` when placeholder text is a stable user-facing contract.
4. `getByText()` for meaningful visible text.
5. `getByTestId()` when the team has agreed on stable test IDs.
6. CSS/XPath only when better semantic options are unavailable.

```ts
await page.getByRole('button', { name: 'Submit payment' }).click();
await page.getByLabel('Email address').fill('qa@example.com');
await page.getByTestId('payment-status').waitFor();
```

**Why:** User-facing locators are easier to understand and less coupled to DOM implementation. A selector such as `.btn-primary:nth-child(3)` can break after harmless styling or layout changes.

**Lead-level practice:** Agree with developers on accessible labels and test-ID conventions. Do not use test IDs as an excuse to ignore accessibility.

### Q5. What is auto-waiting? Does it eliminate all waits?

No. Before actions such as `click()`, Playwright checks whether the target is uniquely resolved and actionable—for example, visible, stable, enabled, and able to receive events as appropriate. Web-first assertions retry until the expected condition becomes true or the timeout expires.

```ts
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.getByText('Changes saved')).toBeVisible();
```

This is usually better than:
```ts
await page.waitForTimeout(3000); // Avoid fixed sleeps as a synchronization strategy
```

Auto-waiting does **not** mean Playwright understands every business process. If a backend job or asynchronous event has not completed, assert a meaningful UI/API condition or wait for a specific event/response.

**Interview distinction:** Actionability waiting happens around actions; web-first assertion retrying happens while checking expected state. Neither guarantees the backend business operation succeeded unless you assert its result.

### Q6. What is the difference between `locator`, `page.$`, and `locator.all()`?

Use `locator()` and semantic locator methods for modern tests. Locators are lazy and retryable; many older element-handle patterns are more brittle.

`locator.all()` returns the current matching locators immediately and does not itself wait for a dynamic list to finish loading. First wait for a stable condition, then iterate or assert the list.

```ts
const rows = page.getByRole('row');
await expect(rows).toHaveCount(4);

for (const row of await rows.all()) {
  console.log(await row.innerText());
}
```

For assertions about lists, prefer retryable assertions such as `toHaveCount()` or `toHaveText()` where possible.

### Q7. How do you handle dropdowns?

For a native HTML `<select>`:
```ts
await page.getByLabel('Country').selectOption({ label: 'India' });
await expect(page.getByLabel('Country')).toHaveValue('IN');
```

For a custom dropdown built with buttons/listboxes, interact as a user would:
```ts
await page.getByRole('combobox', { name: 'Country' }).click();
await page.getByRole('option', { name: 'India' }).click();
```

**Important:** First inspect the actual DOM/accessibility tree. `selectOption()` does not work on arbitrary custom dropdown widgets.

### Q8. How do you handle JavaScript alerts, confirms, and prompts?

Register the dialog handler before triggering the action:
```ts
page.once('dialog', async dialog => {
  expect(dialog.type()).toBe('confirm');
  await dialog.accept();
});
await page.getByRole('button', { name: 'Delete account' }).click();
```

To cancel, call `dialog.dismiss()`. To provide prompt text, call `dialog.accept('text')`.

Avoid installing a handler that blindly accepts every dialog across the suite; it can hide unexpected alerts and cause false positives.

### Q9. How do you work with iframes?

Use `frameLocator()` to locate content inside a frame:
```ts
const paymentFrame = page.frameLocator('iframe[title="Payment details"]');
await paymentFrame.getByLabel('Card number').fill('4111111111111111');
```

Prefer stable frame selectors such as title or test ID. If the iframe is cross-origin, Playwright can still interact with its rendered frame content, but your test cannot freely access its internals through arbitrary page JavaScript.

### Q10. How do you handle new tabs or popups?

Listen for the popup event before clicking:
```ts
const popupPromise = page.waitForEvent('popup');
await page.getByRole('link', { name: 'Open statement' }).click();
const popup = await popupPromise;
await expect(popup).toHaveURL(/statement/);
```

Registering the wait first prevents a race where the popup opens before the listener is installed. For a popup opened from a particular page, `page.waitForEvent('popup')` is usually the clearest option.

### Q11. How do you upload and download files?

**Upload**
```ts
await page.getByLabel('Upload document').setInputFiles('test-data/sample.pdf');
```

**Download**
```ts
const downloadPromise = page.waitForEvent('download');
await page.getByRole('button', { name: 'Download statement' }).click();
const download = await downloadPromise;
await download.saveAs(`test-results/${download.suggestedFilename()}`);
```

In CI, use a known output directory and validate file existence/content when that is part of the requirement. Do not assume a download succeeded merely because the button was clicked.

### Q12. What are assertions in Playwright?

Use `expect` from `@playwright/test`. Web-first assertions such as `toBeVisible()`, `toHaveText()`, `toHaveURL()`, and `toHaveCount()` retry until success or timeout.

```ts
await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
await expect(page).toHaveURL(/dashboard/);
await expect(page.getByTestId('balance')).toHaveText('₹10,000');
```

Avoid checking a transient condition once with `isVisible()` if the page is still loading:
```ts
// May return false immediately if not visible yet:
const visible = await locator.isVisible();

// Better for eventual state:
await expect(locator).toBeVisible();
```

Use soft assertions sparingly. They allow a test to continue after a failure and can be useful for collecting multiple independent validation failures, but critical preconditions should generally fail immediately.

---

## 2. Playwright Test runner and framework design

### Q13. What is Playwright Test?

Playwright Test is the official JavaScript/TypeScript test runner. It provides:
- Test declaration and grouping (`test`, `test.describe`).
- Fixtures such as `page`, `context`, `browser`, and `request`.
- Assertions (`expect`).
- Hooks.
- Parallel workers and retries.
- Projects for browser/device/environment matrices.
- Reporters, attachments, and tracing.
- Tags, filtering, and test annotations.

Install and initialize:
```bash
npm init playwright@latest
npx playwright test
npx playwright show-report
```

### Q14. Explain fixtures. How are they different from hooks?

A fixture supplies a dependency to a test and controls its setup/teardown lifecycle. The built-in `page` fixture provides a page; custom fixtures can provide a logged-in page object, API client, database helper, or test-data service.

```ts
import { test as base, expect, type Page } from '@playwright/test';

type Fixtures = { dashboardPage: Page };

export const test = base.extend<Fixtures>({
  dashboardPage: async ({ page }, use) => {
    await page.goto('/dashboard');
    await use(page);
    // Fixture teardown belongs after use().
  },
});

export { expect };
```

A hook runs code at a defined test/suite lifecycle point, for example `beforeEach`. A fixture is dependency-oriented and can have a scope and teardown. Use fixtures for reusable resources and setup tied to a dependency; use hooks for simple lifecycle actions that are genuinely suite-wide.

**Lead-level caution:** Avoid turning every helper into a fixture. Over-engineered fixture graphs can make test setup hard to follow.

### Q15. Explain `beforeEach`, `afterEach`, `beforeAll`, and `afterAll`.

- `beforeEach`: runs before each test in its scope.
- `afterEach`: runs after each test, including when a test fails.
- `beforeAll`: runs once for the relevant worker/suite execution, not necessarily once globally across all machines.
- `afterAll`: runs after that suite's tests in the worker lifecycle.

Use `beforeEach` for per-test navigation or setup. Use `beforeAll` for expensive worker-level setup only when safe to share. Do not create one mutable browser page in `beforeAll` and share it among parallel tests; it undermines isolation.

### Q16. What should `playwright.config.ts` contain?

Typical responsibilities:
- `testDir`, timeouts, retries, and worker count.
- `baseURL`.
- `use` defaults: browser, headless mode, viewport, tracing, screenshots, video.
- Projects for browsers or environments.
- Reporter configuration.
- Test output directory and web server setup.

Example:
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  reporter: [['list'], ['html', { open: 'never' }]],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://127.0.0.1:3000',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
});
```

The exact tracing and retry policies should reflect CI cost and debugging needs. Keep secrets out of the config file and source control.

### Q17. Explain Page Object Model (POM).

POM wraps a page or component's locators and user actions in a class. Tests express business intent; page objects encapsulate UI mechanics.

```ts
// pages/LoginPage.ts
import { type Page, type Locator } from '@playwright/test';

export class LoginPage {
  readonly email: Locator;
  readonly password: Locator;
  readonly signIn: Locator;

  constructor(private readonly page: Page) {
    this.email = page.getByLabel('Email');
    this.password = page.getByLabel('Password');
    this.signIn = page.getByRole('button', { name: 'Sign in' });
  }

  async login(email: string, password: string) {
    await this.email.fill(email);
    await this.password.fill(password);
    await this.signIn.click();
  }
}
```

```ts
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('valid user can sign in', async ({ page }) => {
  const login = new LoginPage(page);
  await page.goto('/login');
  await login.login('qa@example.com', process.env.TEST_PASSWORD!);
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
});
```

**Good POM rules:**
- Keep selectors in one place.
- Expose business actions, not dozens of low-level wrappers.
- Keep assertions primarily in tests, except where a page object's explicit contract makes an assertion appropriate.
- Avoid a giant `BasePage` full of unrelated methods.
- Model reusable widgets as component objects where helpful.

### Q18. What is the difference between POM, fixtures, and utility classes?

- **POM/component object:** understands a page/component and its UI behavior.
- **Fixture:** manages lifecycle and injects a dependency into tests.
- **Utility/service:** performs a cross-cutting operation such as generating data, formatting values, making a domain API call, or querying a database.

Example: `LoginPage` is a POM; a `loggedInPage` fixture creates it in an authenticated context; `TestDataService` creates a user through an API. Keep responsibilities separate.

### Q19. How should an enterprise Playwright framework be structured?

One possible structure:
```text
project/
├── tests/
│   ├── ui/
│   ├── api/
│   └── integration/
├── pages/
├── components/
├── fixtures/
├── clients/             # API clients / domain services
├── data/                # non-secret static test data
├── factories/           # test-data builders
├── utils/
├── config/
├── schemas/
├── playwright.config.ts
├── package.json
└── tsconfig.json
```

Use this as a starting point, not a mandatory folder template. Organize by product/domain when the suite becomes large. Keep tests readable and domain-focused. Avoid a framework so abstract that a new engineer must navigate ten layers to understand one click.

### Q20. Which design patterns are useful in Playwright?

- **Page Object / Component Object:** encapsulate UI details.
- **Factory:** create test data or role-specific objects.
- **Builder:** construct complex request payloads with readable defaults.
- **Strategy:** choose environment-specific or authentication-specific behavior.
- **Adapter:** standardize different external services behind one interface.
- **Facade:** offer a simple API over a complicated subsystem.
- **Dependency injection through fixtures:** supply services and test dependencies.

Use patterns to solve real duplication or complexity. Do not add Singleton to every helper or build an elaborate inheritance hierarchy without a need. Playwright's fixture model already provides lifecycle-aware dependency injection.

### Q21. How do you manage environments and secrets?

Use environment variables or an approved secret manager for credentials and tokens. Validate required settings early, fail with a clear error, and never print secret values in logs.

```ts
const baseURL = process.env.BASE_URL;
if (!baseURL) throw new Error('BASE_URL must be set');

const username = process.env.TEST_USERNAME;
const password = process.env.TEST_PASSWORD;
if (!username || !password) throw new Error('Test credentials are missing');
```

Separate non-secret configuration (URLs, timeouts) from secrets. Use dedicated test accounts, rotate credentials, restrict permissions, and ensure traces/screenshots do not expose sensitive customer data.

### Q22. How do you reuse authentication?

Playwright can save and load browser storage state. A common pattern is to authenticate in a setup project and reuse a state file. Keep that file out of version control because it can contain cookies or tokens that impersonate a user.

```ts
// After successful UI login:
await page.context().storageState({ path: 'playwright/.auth/user.json' });
```

Then configure:
```ts
use: {
  storageState: 'playwright/.auth/user.json',
}
```

**Important trade-off:** Shared state is fine for read-only tests that do not interfere with one another. If tests mutate server-side state, use separate accounts or isolated data per worker/test. Authentication can also be bootstrapped via API if supported. Remember that state expires, and sessionStorage may need special handling.

---

## 3. Advanced Playwright capabilities

### Q23. How do you perform API testing in Playwright?

Use the built-in `request` fixture or create an `APIRequestContext`. Validate status, headers, response body, and business rules. UI and API tests can share the same runner and reports.

```ts
import { test, expect } from '@playwright/test';

test('create customer API', async ({ request }) => {
  const response = await request.post('/api/customers', {
    data: { name: 'Test Customer', email: 'customer@example.com' },
  });

  expect(response.status()).toBe(201);
  const body = await response.json();
  expect(body.name).toBe('Test Customer');
  expect(body.id).toBeTruthy();
});
```

Configure `baseURL` or create a context with the appropriate base URL and authentication headers. Do not blindly reuse UI cookies for every API; the service's authentication model determines the right approach.

### Q24. How do you combine API setup with UI validation?

Create preconditions through APIs when the purpose of the test is UI behavior, then use the UI to validate the user journey. This reduces setup time and makes the test more focused.

Example approach:
1. Create a customer via API.
2. Open the customer page in the browser.
3. Assert the user-visible fields and actions.
4. Delete the customer via API in teardown, or use unique disposable data.

Do not use API setup if the test's purpose is to verify the UI's own creation flow. That flow needs to be exercised through the UI.

### Q25. What is network interception? Explain `page.route()`.

`page.route()` lets a test intercept matching network requests and continue, modify, abort, or fulfill them. It is useful for deterministic edge cases, controlled error responses, and frontend tests that should not depend on a third party.

```ts
test('shows a friendly error when catalogue API fails', async ({ page }) => {
  await page.route('**/api/catalogue', async route => {
    await route.fulfill({
      status: 503,
      contentType: 'application/json',
      body: JSON.stringify({ message: 'Service unavailable' }),
    });
  });

  await page.goto('/catalogue');
  await expect(page.getByText(/try again later/i)).toBeVisible();
});
```

**Caution:** Mocking can prove the frontend handles a simulated response, but it does not prove the real backend works. Maintain separate integration tests against real services.

### Q26. How do you wait for a network response?

Set up the response wait before the action that triggers it:
```ts
const responsePromise = page.waitForResponse(response =>
  response.url().includes('/api/payments') &&
  response.request().method() === 'POST'
);

await page.getByRole('button', { name: 'Pay now' }).click();
const response = await responsePromise;

expect(response.status()).toBe(201);
```

Do not match only a broad URL if multiple requests can match it. Include method, endpoint, and, when needed, request payload or another unique condition. Also assert the user-visible outcome if the requirement is end-to-end.

### Q27. How do you test WebSockets or asynchronous event-driven UI?

Think in terms of observable outcomes and correlation:
1. Trigger a business action.
2. Capture the correlation/reference ID if available.
3. Wait for the UI to display the expected final state, using a retryable assertion.
4. If needed, inspect a WebSocket frame or query a trusted backend/event store in an integration test.
5. Apply a bounded timeout and report useful diagnostics.

Do not use arbitrary sleeps such as “wait 10 seconds for Kafka.” Prefer event-driven synchronization, API polling with a deadline, or a UI assertion that retries. For payment-like flows, assert intermediate and final statuses and ensure duplicate events do not create duplicate business effects.

### Q28. What are retries, and how should you use them?

Retries rerun failed tests, which can help collect evidence for intermittent failures. They must not be used to conceal flaky tests.

```ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
});
```

Classify failures:
- Product defect.
- Test defect or brittle selector.
- Environment/infrastructure failure.
- Test-data collision.
- Timing/race condition.

Track first-attempt pass rate and flaky-test rate separately from final pass rate. A test that passes only on retry is still a reliability signal. Fix root causes and use retries as a diagnostic/containment mechanism, not as a quality metric.

### Q29. How does parallel execution work?

Playwright Test uses worker processes to run tests in parallel. Configure worker count to fit available CPU, memory, environment capacity, and backend rate limits.

```bash
npx playwright test --workers=4
```

Tests should be independent: unique records, isolated sessions, no order dependency, and reliable cleanup. Workers cannot share ordinary in-memory state with one another.

**Lead-level answer:** “I start with isolated tests and unique test data, then benchmark throughput and backend stability while increasing workers. More workers do not always mean faster execution if the environment is saturated.”

### Q30. How do you avoid test-data collisions?

- Generate unique identifiers per test/run.
- Use API factories to create and delete data.
- Allocate one account per worker when state is mutable.
- Avoid shared records that tests update concurrently.
- Make cleanup idempotent and safe to retry.
- Tag test data with run ID/test ID when possible.
- Do not rely on test order.

For example, an email could include a generated suffix. For financial systems, use controlled test accounts and avoid unsafe operations against production data. If the backend has eventual consistency, wait for a specific state rather than assuming writes are immediately visible.

### Q31. Explain projects, devices, and cross-browser testing.

Projects let the same test suite run with different settings, such as Chromium, Firefox, WebKit, mobile viewport emulation, or different environments.

```ts
projects: [
  { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
  { name: 'webkit', use: { ...devices['Desktop Safari'] } },
]
```

Prioritize a browser matrix based on customer usage and risk. A common strategy is broad Chromium coverage on every pull request and a wider cross-browser matrix nightly or before release. Emulating a mobile browser is not the same as testing a native iOS/Android app.

### Q32. What are traces, screenshots, videos, and the HTML report?

- **Trace:** Timeline of actions, DOM snapshots, network information, and related evidence useful for debugging.
- **Screenshot:** A visual snapshot at a point in time.
- **Video:** A recording of a page/context, if configured.
- **HTML report:** A summary of test outcomes, errors, attachments, and run information.

Example config:
```ts
use: {
  trace: 'retain-on-failure',
  screenshot: 'only-on-failure',
  video: 'retain-on-failure',
}
```

Keep artifact retention and access controlled because traces, screenshots, and videos can contain personal or financial data. Prefer trace-based debugging over adding more sleeps.

---

## 4. SDET Lead and architecture questions

### Q33. Design an enterprise Playwright framework.

**Interview-ready structure:**

```text
Test Specs
   |
   +-- UI page/component objects
   +-- API/domain clients
   +-- Test-data factories
   +-- Fixtures (lifecycle + dependency injection)
   |
Playwright Test Runner
   +-- config / projects / retries / workers
   +-- reporters / traces / attachments
   |
CI pipeline
   +-- lint + type-check
   +-- install pinned browsers
   +-- run smoke/regression by tags
   +-- publish reports and artifacts
   |
Systems under test
   +-- Web UI / APIs / DB / Kafka / dependent services
```

**Design principles:**
- Keep tests independent and business-focused.
- Use TypeScript types and linting to catch errors early.
- Put selectors in page/component objects.
- Use API clients and factories for setup, not giant helper methods.
- Keep configuration environment-driven.
- Use least-privilege credentials.
- Make failures diagnosable through traces, logs, correlation IDs, and clear reporting.
- Separate deterministic mocks from true integration coverage.
- Measure suite duration, first-attempt pass rate, flake rate, and defect escape trends.

Do not describe an architecture only by listing patterns. Explain why each layer exists and how it improves maintenance or reliability.

### Q34. How would you design CI/CD for Playwright?

A practical pipeline:
1. Install a pinned Node.js version and dependencies using the lockfile.
2. Run lint and TypeScript checks.
3. Install only the required Playwright browsers/dependencies.
4. Run fast smoke tests on pull requests.
5. Run broader regression and cross-browser suites at an appropriate stage.
6. Split long suites into shards when justified.
7. Publish reports, traces, screenshots, and logs even when tests fail.
8. Fail the pipeline based on defined release criteria.
9. Track flaky tests separately and route failures to owners.

Example commands:
```bash
npm ci
npx playwright install --with-deps chromium
npx tsc --noEmit
npx playwright test --grep @smoke
npx playwright show-report
```

In real CI, ensure report publication runs even when the test step fails. Pin dependencies and avoid relying on whatever browser version happens to be installed on an agent.

### Q35. How would you run 5,000 tests in a reasonable time?

First profile the suite: test durations, setup cost, browser launch cost, bottlenecks, and flaky tests. Then:
- Remove redundant tests while preserving risk coverage.
- Use API setup instead of repeated UI setup when appropriate.
- Reuse authentication safely.
- Run independent tests in parallel.
- Shard across CI agents.
- Tune workers based on resource limits.
- Separate smoke, pull-request regression, nightly, and release suites.
- Avoid slow global fixtures and unnecessary browser matrices.
- Track p50/p95 test duration and infrastructure saturation.

Do not promise that a particular number of workers will achieve a target runtime without benchmarking. Parallelism can increase database load, rate-limit pressure, and collisions.

### Q36. How would you troubleshoot flaky tests?

Use a repeatable investigation:
1. Check trace, screenshot, video, browser console, network logs, and server logs.
2. Determine whether failure is selector/actionability, timing, data, product, or infrastructure.
3. Reproduce locally and in CI with the same environment.
4. Remove fixed sleeps and replace them with observable conditions.
5. Make data unique and tests isolated.
6. Check animation, overlays, network variability, and async backend processing.
7. Add a regression test or monitoring for the root cause.
8. Track the test's flake rate after the fix.

**Bad fix:** Increase every timeout or add retries to every test.  
**Good fix:** Identify the missing synchronization condition or shared state and correct it.

### Q37. How do you test database and Kafka workflows?

Playwright is not a database or Kafka testing platform by itself, but TypeScript tests can call approved clients or helper services. Keep system-level validation separate from UI checks where possible.

For a workflow:
1. Create a unique entity via API or an approved setup mechanism.
2. Trigger the action through the UI or API, depending on the test objective.
3. Capture a business/correlation ID.
4. Verify the UI reaches the expected state.
5. For integration tests, query a read-only DB view or event test harness and verify the expected event/business record.
6. Use bounded polling for eventual consistency.
7. Clean up safely.

Avoid coupling every UI test directly to database internals. Validate DB/Kafka behavior in targeted integration tests, and retain end-to-end tests for the highest-value user journeys. Never consume production Kafka messages or write production data as part of ordinary automation.

### Q38. How would you test a payment journey?

Design test layers:
- **API/service tests:** validations, authorization, idempotency, business rules, response contracts.
- **UI tests:** amount entry, confirmation, status rendering, error handling, accessibility.
- **Integration tests:** payment request → event publication → ledger/status update → UI state.
- **Negative tests:** insufficient balance, timeout, declined payment, duplicate submission, expired session, unavailable dependency.
- **Resilience tests:** retry behavior, delayed events, duplicate events, and recovery.
- **Security tests:** authorization boundaries, sensitive-data exposure, and safe logging.

Use deterministic test accounts and a payment sandbox. Assert that a duplicate request does not create a second debit. For asynchronous processing, correlate the request and event and assert the final business state, not just an HTTP 200.

### Q39. How do you decide what to automate?

Prioritize cases by business risk, frequency, stability, repeatability, execution cost, and value of fast feedback. Automate stable, repeatable, high-value regression paths first. Keep exploratory testing for areas where human observation and discovery are more valuable.

A useful matrix:
- **High risk + frequent:** automate early.
- **High risk + complex/unstable:** stabilize the product/test seam first; combine automation and exploratory testing.
- **Low risk + rarely repeated:** may not justify automation.
- **Visual/accessibility requirements:** use targeted visual/accessibility checks, not only DOM assertions.

Measure value using feedback time, maintenance effort, escaped defects, and risk coverage—not just test count or automation percentage.

### Q40. Which metrics should an SDET Lead track?

- First-attempt pass rate.
- Flaky-test rate and repeat offenders.
- Mean/p95 suite duration.
- Time to diagnose and recover from failures.
- Critical business journey coverage.
- Defect leakage/escape rate and severity.
- CI pipeline stability.
- Test maintenance effort.
- Release-blocking failure accuracy (true vs false positives).
- Test data/environment availability.

Automation percentage alone is weak: a large suite of low-value tests can produce a high percentage without reducing release risk.

### Q41. How do you migrate a Selenium framework to Playwright?

1. Inventory existing tests and classify by business risk and value.
2. Establish baseline: runtime, flake rate, maintenance cost, and coverage.
3. Choose a pilot journey with stable expected behavior.
4. Define TypeScript conventions, fixtures, POM/component object standards, reporting, and CI.
5. Rebuild selected tests with Playwright-native locators and assertions rather than translating every line literally.
6. Run both suites during transition where needed and compare outcomes.
7. Remove duplicate coverage only after proving equivalent behavior.
8. Train the team and document patterns.
9. Migrate incrementally, based on evidence.

Do not promise that a rewrite will automatically reduce cost. Migration has a temporary maintenance burden and should be justified by measurable benefits.

### Q42. How would you test accessibility?

Use semantic locators and accessible names in ordinary tests, then add automated accessibility scanning and manual keyboard/screen-reader checks for critical journeys. Automated scans catch only a subset of accessibility issues.

Example:
```ts
await page.keyboard.press('Tab');
await expect(page.getByRole('link', { name: 'Skip to content' })).toBeFocused();
```

Check keyboard navigation, focus visibility, labels, headings, contrast, error announcements, and dialogs. Treat accessibility as a quality requirement, not just a plugin run.

### Q43. How do you secure a Playwright framework?

- Use secret managers or CI secret variables.
- Use least-privilege test accounts.
- Never commit storage-state files or tokens.
- Scrub sensitive data from logs and reports.
- Restrict artifact access and retention.
- Use test/sandbox environments for destructive workflows.
- Avoid production personal data.
- Pin and update dependencies.
- Review third-party packages and CI permissions.
- Prevent secrets from being printed on assertion failures.

A trace can include page content and network details; treat it as potentially sensitive evidence.

### Q44. How do you manage a large test suite by tags and risk?

Use a consistent tagging scheme such as `@smoke`, `@regression`, `@critical`, and `@slow`. Use tags to select suites, not as a replacement for clear ownership and coverage strategy.

```ts
test('customer can view balance @smoke', async ({ page }) => {
  // ...
});
```

A team may prefer test annotations or title tags based on its conventions. Define what each tag means and which pipeline stage runs it. Keep pull-request suites fast and valuable; schedule broader coverage where time and environment capacity allow.

### Q45. How would you lead adoption of Playwright in a QA team?

A strong answer covers both technology and people:
1. Assess team skills, existing framework pain points, and application architecture.
2. Build a small proof of concept covering a real critical journey.
3. Agree on coding standards, locator rules, test isolation, data strategy, and code review.
4. Integrate reporting and CI before scaling.
5. Train through pairing and code reviews.
6. Define ownership for flaky tests and framework components.
7. Publish metrics and review them regularly.
8. Expand incrementally based on outcomes, not tool enthusiasm.

**Sample interview response:**  
“I would start with a risk-based pilot rather than rewriting the entire suite. I’d define conventions for locators, fixtures, page objects, test data, and diagnostics; integrate the pilot into CI; and measure runtime, first-attempt reliability, and maintenance effort. Once the team has a repeatable pattern, I’d migrate the highest-value journeys in stages and use code reviews and ownership to keep the suite sustainable.”

---

## 5. Extra scenario questions interviewers may ask

### Q46. A click intermittently times out. What do you do?

Inspect the trace and locator count first. Verify that the element is unique, enabled, visible, stable, and not covered by an overlay. Check whether the page is in the expected state and whether a network response or asynchronous render is still pending. Replace brittle selectors and arbitrary sleeps with a user-facing locator plus an assertion for the relevant state. Do not use `force: true` until you understand why normal actionability fails; forcing a click can hide a real user-facing defect.

### Q47. The test passes locally but fails in CI. Why?

Possible causes include browser/version mismatch, missing dependencies, different viewport or timezone, slower resources, environment configuration, parallel data collisions, network restrictions, and timing assumptions. Compare versions and config, inspect CI traces/artifacts and server logs, rerun the isolated test, and reproduce with the same container image. Fix the root cause rather than merely increasing global timeouts.

### Q48. When would you use `waitForTimeout()`?

Rarely. It can be useful temporarily during debugging or for a genuinely time-based requirement, but it is usually a poor synchronization mechanism because it either wastes time or remains too short under load. Prefer `expect(locator).toBeVisible()`, `waitForResponse()`, `waitForURL()`, or a bounded poll for a business condition.

### Q49. How do you handle an element that appears more than once?

Scope the locator to a meaningful parent, filter by text or another locator, and assert uniqueness when appropriate.

```ts
const row = page.getByRole('row').filter({ hasText: 'Customer 123' });
await row.getByRole('button', { name: 'Edit' }).click();
```

Avoid blindly using `.first()` or `.nth(1)` unless ordering is itself part of the requirement. Multiple matches can indicate an ambiguous test or a product defect.

### Q50. What is the difference between mocking and contract/integration testing?

- **Mocking:** Controls a dependency response so a test can reliably validate the caller's behavior.
- **Contract testing:** Checks that services agree on request/response expectations.
- **Integration testing:** Validates real interactions between components.
- **End-to-end testing:** Validates a business journey across the system from a user or external perspective.

A balanced suite uses mocks for hard-to-trigger failures and deterministic frontend cases, contract/integration tests for service boundaries, and a smaller set of end-to-end tests for critical user journeys.

---

## 6. Commands to know

```bash
# Run all tests
npx playwright test

# Run a specific file
npx playwright test tests/ui/login.spec.ts

# Run tests matching a title
npx playwright test -g "valid user"

# Run one browser project
npx playwright test --project=chromium

# Run with a controlled worker count
npx playwright test --workers=4

# Debug interactively
npx playwright test --debug

# Open HTML report
npx playwright show-report

# Run a test in UI mode
npx playwright test --ui

# Install browser binaries
npx playwright install

# Install Chromium and its OS dependencies on supported Linux CI agents
npx playwright install --with-deps chromium
```

Use `npx playwright test --help` for the exact options supported by the version installed in your repository.

## 7. A four-week hands-on plan

### Week 1 — Fundamentals
- Install Node.js, TypeScript, and Playwright.
- Write tests using locators, assertions, dropdowns, dialogs, iframes, uploads, and popups.
- Learn browser/context/page and actionability.
- Deliverable: 15 reliable UI tests.

### Week 2 — Framework
- Create page/component objects and custom fixtures.
- Add environment configuration, authentication, data factories, and tags.
- Add linting and TypeScript checks.
- Deliverable: a clean repository with reusable patterns.

### Week 3 — Advanced and CI
- Add API tests, route mocking, response waits, retries, browser projects, traces, and HTML reporting.
- Run tests in CI with artifacts published on failure.
- Test parallel execution and data isolation.
- Deliverable: a working CI pipeline.

### Week 4 — Lead-level design
- Add one critical integration journey and a test-data cleanup strategy.
- Benchmark execution time and reliability.
- Practise flaky-test triage, Selenium migration, 5,000-test scaling, payment workflows, and team adoption answers.
- Deliverable: a 10-minute architecture walkthrough plus a 30-minute mock interview.

## 8. Capstone project: build this to prove mastery

Create a small customer/payments application test framework with:
- UI: login, customer search, payment form, payment status.
- API: create customer, initiate payment, fetch payment status.
- Test data: unique IDs and cleanup.
- Authentication: reusable state for safe read-only tests; isolated accounts for state-mutating tests.
- Network: mock a dependency timeout and verify the error state.
- Integration: correlate payment ID with eventual status.
- CI: smoke on pull request, full regression on schedule, reports/traces on failure.
- Quality: lint, type-check, no committed secrets, stable locator conventions.
- Documentation: setup instructions, architecture diagram, failure-triage steps, known limitations.

## 9. How to answer in an SDET Lead interview

Use this structure for most questions:
1. **Definition:** Explain the concept in one or two sentences.
2. **Mechanism:** Explain how it works.
3. **Example:** Give a concise real implementation example.
4. **Trade-off:** Mention when you would and would not use it.
5. **Leadership angle:** Explain how you would standardize it, measure it, or keep it reliable at scale.

For scenario questions, explain the investigation steps and decision criteria instead of jumping straight to code.

## Official references for deeper study

- Playwright documentation: https://playwright.dev/docs/intro
- Best practices: https://playwright.dev/docs/best-practices
- Locators: https://playwright.dev/docs/locators
- Auto-waiting: https://playwright.dev/docs/actionability
- Fixtures: https://playwright.dev/docs/test-fixtures
- Page Object Models: https://playwright.dev/docs/pom
- Authentication: https://playwright.dev/docs/auth
- Parallelism: https://playwright.dev/docs/test-parallel
- API testing: https://playwright.dev/docs/api-testing
- Network: https://playwright.dev/docs/network
- Trace Viewer: https://playwright.dev/docs/trace-viewer
- CI: https://playwright.dev/docs/ci

**Final advice:** For a lead interview, being able to explain why a framework is reliable is more valuable than reciting every Playwright method. Build it, run it in CI, deliberately introduce failures, and practise diagnosing them from evidence.
