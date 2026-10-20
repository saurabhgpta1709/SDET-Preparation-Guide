# Playwright Mastery Guide
## Beginner to Advanced | SDET Lead, Senior QA Automation Engineer & Software Engineer Interviews

**Goal:** Build practical Playwright skills with TypeScript, design a maintainable automation framework, and prepare for senior/lead interviews.

**Recommended study time:** 60–90 minutes daily for 4–6 weeks. For every concept, write code, deliberately cause a failure, inspect the trace, fix the root cause, and explain the trade-off aloud.

## Table of Contents
1. Learning roadmap
2. Playwright overview
3. Setup and first test
4. TypeScript essentials
5. Tests and assertions
6. Locators
7. Browser, context, page, fixtures
8. Synchronization and waiting
9. Forms, dialogs, frames, files, popups
10. API testing and network mocking
11. Authentication and test data
12. Page Object Model and design patterns
13. Framework architecture
14. Parallelism, retries, and flakiness
15. Configuration, projects, and environments
16. CI/CD and reporting
17. Debugging
18. Accessibility, visual, and cross-browser testing
19. Performance and security
20. SDET Lead design
21. Interview questions and answers
22. Scenario-based questions
23. Coding exercises
24. Best practices and cheat sheet
25. Readiness checklist

---

# 1. Learning Roadmap

| Week | Topics | Practical deliverable |
|---|---|---|
| 1 | TypeScript, setup, locators, assertions | 10 stable UI tests |
| 2 | Fixtures, waits, forms, dialogs, files | A small end-to-end suite |
| 3 | POM, API tests, authentication, test data | Reusable framework |
| 4 | Parallelism, configuration, CI, reports | CI pipeline with artifacts |
| 5 | Network mocking, accessibility, visual tests, advanced TypeScript | Advanced scenarios |
| 6 | Architecture, reliability metrics, interview practice | Explain and defend your design |

**Skill levels**
- **Beginner:** Can write and run tests, use locators, and assert results.
- **Intermediate:** Can use fixtures, API requests, authentication, and debug failures.
- **Advanced:** Can build reusable configuration, isolate data, and integrate CI.
- **Lead:** Can define strategy, risk-based coverage, quality gates, metrics, and team standards.

# 2. Playwright Overview

Playwright automates web applications using Chromium, Firefox, and WebKit. `@playwright/test` adds a test runner, fixtures, assertions, parallel execution, retries, projects, reporters, and debugging tools.

### Strengths
- Auto-waiting and retrying assertions.
- Browser contexts for session isolation.
- Cross-browser projects.
- Network interception and API testing.
- Trace Viewer, screenshots, videos, and HTML reports.
- TypeScript-first developer experience.

### Important limitations
- Playwright does not replace unit tests or dedicated load testing.
- A passing UI test does not prove all backend invariants.
- Automation does not automatically create a good test strategy.
- A framework cannot fix unstable data or a broken test environment.

### Test portfolio
Use the cheapest reliable layer that validates the requirement:
- **Unit/component:** business logic and components.
- **API/service:** contracts, authorization, validation, business rules.
- **UI end-to-end:** a focused set of critical user journeys.
- **Non-functional:** performance, accessibility, security, resilience, compatibility.

Avoid testing every rule through the browser. UI tests are relatively expensive and have more environmental failure modes.

# 3. Setup and First Test

Install a supported Node.js LTS release:

```bash
mkdir playwright-mastery
cd playwright-mastery
npm init -y
npm init playwright@latest
npx playwright install
```

For Linux CI, browser dependencies may be needed:

```bash
npx playwright install --with-deps
```

Run tests:

```bash
npx playwright test
npx playwright test --headed
npx playwright test --ui
npx playwright test --debug
```

Example test:

```typescript
import { test, expect } from '@playwright/test';

test('home page displays the expected title', async ({ page }) => {
  await page.goto('https://playwright.dev/');
  await expect(page).toHaveTitle(/Playwright/);
  await expect(
    page.getByRole('link', { name: 'Get started' })
  ).toBeVisible();
});
```

- `test` declares a test.
- `expect` provides assertions.
- `page` is a fixture provided by Playwright Test.
- `getByRole` uses semantic role and accessible name.
- `toHaveTitle` and `toBeVisible` retry until they pass or time out.

Useful commands:

```bash
npx playwright test tests/login.spec.ts
npx playwright test -g "invalid credentials"
npx playwright show-report
```

# 4. TypeScript Essentials

## 4.1 Types, objects, and unions

```typescript
const baseUrl: string = 'https://qa.example.test';
const isCI: boolean = process.env.CI === 'true';

type User = {
  id: string;
  email: string;
  active: boolean;
};

type Environment = 'local' | 'qa' | 'staging';
```

Prefer `const` unless reassignment is required. Let TypeScript infer obvious types; add annotations where they clarify contracts.

## 4.2 Functions and optional fields

```typescript
type Profile = {
  name: string;
  phone?: string;
};

function formatName(profile: Profile): string {
  return profile.name.trim();
}
```

Use strict type checking. Avoid frequent non-null assertions (`!`), which silence compiler warnings without proving a value exists.

## 4.3 Interfaces, types, and classes

```typescript
interface Credentials {
  username: string;
  password: string;
}

type LoginResult =
  | { success: true; userId: string }
  | { success: false; reason: string };

class LoginData {
  constructor(
    public readonly username: string,
    public readonly password: string,
  ) {}
}
```

An interface is useful for object contracts and can be extended. A type alias can represent unions, intersections, tuples, primitives, and object shapes. Choose the construct that expresses the contract clearly.

## 4.4 Generics and runtime validation

```typescript
async function parseJson<T>(response: { json(): Promise<unknown> }): Promise<T> {
  return (await response.json()) as T;
}
```

The cast does **not** validate the response at runtime. For untrusted API data, use a runtime schema validator or explicit checks.

## 4.5 Async/await

```typescript
test('search shows results', async ({ page }) => {
  await page.goto('/search');
  await page.getByLabel('Search').fill('invoice');
  await page.getByRole('button', { name: 'Search' }).click();
  await expect(page.getByRole('heading', { name: /results/i })).toBeVisible();
});
```

Common mistakes:
- Forgetting `await`.
- Starting a promise but neither awaiting nor returning it.
- Using `forEach(async ...)` and assuming it waits.
- Swallowing errors in an empty `catch`.

For sequential asynchronous work, use `for...of`:

```typescript
for (const term of ['invoice', 'payment', 'refund']) {
  await page.getByLabel('Search').fill(term);
  await page.getByRole('button', { name: 'Search' }).click();
}
```

Learn next: `unknown` vs `any`, `Partial<T>`, `Pick<T>`, `Omit<T>`, `Record<K,V>`, discriminated unions, type guards, modules, `readonly`, and `tsconfig` strictness.

# 5. Tests and Assertions

```typescript
test.describe('Catalogue', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/products');
  });

  test('shows the product heading', async ({ page }) => {
    await expect(page.getByRole('heading', { name: 'Products' })).toBeVisible();
  });
});
```

Use hooks for genuinely shared setup. Do not hide complicated business workflows in many hooks; explicit setup is often easier to debug.

### Web-first assertions

```typescript
await expect(page.getByText('Payment successful')).toBeVisible();
await expect(page.getByRole('button', { name: 'Submit' })).toBeEnabled();
await expect(page).toHaveURL(/\/orders\/\w+/);
await expect(page.getByTestId('status')).toHaveText('Completed');
```

Avoid a one-time snapshot for an asynchronous condition:

```typescript
// Fragile: count() returns once; it does not retry like an assertion.
expect(await page.getByRole('listitem').count()).toBe(3);

// Better:
await expect(page.getByRole('listitem')).toHaveCount(3);
```

Soft assertions (`expect.soft`) allow additional independent checks to run after a failure. Do not use them for prerequisites where later actions would become meaningless.

Use behavior-focused titles such as `blocks checkout when billing address is missing`, not `test 17`.

# 6. Locators and Reliable Selection

Preferred locator order, in general:
1. `getByRole()` with accessible name
2. `getByLabel()` for form controls
3. `getByPlaceholder()` when appropriate
4. `getByText()` for meaningful visible text
5. `getByTestId()` for an explicit test contract
6. CSS/XPath when semantic options do not fit

```typescript
page.getByRole('button', { name: 'Save changes' });
page.getByLabel('Email address');
page.getByPlaceholder('Search');
page.getByTestId('invoice-status');
```

Scope repeated controls to the relevant row/card:

```typescript
const row = page.getByRole('row').filter({ hasText: 'INV-1042' });
await row.getByRole('button', { name: 'Open' }).click();
```

Avoid brittle selectors such as `div:nth-child(4) > div:nth-child(2) > button`. Avoid using `nth()` merely to silence a strict-mode violation; first determine why the locator matches multiple elements. If position is truly part of the requirement, make it explicit and verify context.

**Locator vs ElementHandle:** Locators resolve when actions/assertions run and can re-resolve after DOM changes. Element handles point to a particular DOM node and can become stale. Prefer locators for normal test automation.

# 7. Browser, Context, Page, and Fixtures

```text
Browser process
└── BrowserContext (isolated session: cookies/storage/permissions)
    ├── Page (tab)
    └── Page (another tab)
```

- **Browser:** Running browser instance.
- **BrowserContext:** Isolated browser session.
- **Page:** A tab or popup.
- **Locator:** A resilient query for an element.

Contexts let tests simulate different users without sharing cookies or local storage.

```typescript
test('two users have separate sessions', async ({ browser }) => {
  const adminContext = await browser.newContext();
  const guestContext = await browser.newContext();

  try {
    const adminPage = await adminContext.newPage();
    const guestPage = await guestContext.newPage();
    await adminPage.goto('/login');
    await guestPage.goto('/login');
    // Authenticate each role and assert role-specific behavior.
  } finally {
    await adminContext.close();
    await guestContext.close();
  }
});
```

Fixtures provide dependencies and manage setup/teardown. Built-ins include `page`, `context`, `browser`, and `request`.

```typescript
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

type AppFixtures = { loginPage: LoginPage };

export const test = base.extend<AppFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
});

export { expect } from '@playwright/test';
```

Use fixtures for reusable dependencies with clear lifecycle rules. Avoid turning them into hidden workflows that make test behavior hard to follow.

# 8. Synchronization and Waiting

Playwright auto-waits for actionability conditions relevant to an action, such as uniqueness, visibility, stability, enabled state, and ability to receive events. Web-first assertions retry.

Avoid fixed sleeps:

```typescript
await page.waitForTimeout(5000); // Usually fragile and wasteful
```

Prefer a business condition:

```typescript
await page.getByRole('button', { name: 'Submit' }).click();
await expect(page.getByText('Request submitted')).toBeVisible();
```

Wait for a specific response when the response itself matters:

```typescript
const responsePromise = page.waitForResponse(response =>
  response.url().includes('/api/orders') &&
  response.request().method() === 'POST' &&
  response.status() === 201
);

await page.getByRole('button', { name: 'Place order' }).click();
const response = await responsePromise;
expect(response.ok()).toBeTruthy();
```

Create the wait **before** the action. Prefer a specific endpoint/method over generic `networkidle`, which may be unsuitable for polling or long-lived connections.

Tune timeouts deliberately. Increasing every timeout can hide slow services and delay feedback.

# 9. Forms, Dialogs, Frames, Files, and Popups

## Forms

```typescript
await page.getByLabel('Full name').fill('Asha Sharma');
await page.getByLabel('Email').fill('asha@example.test');
await page.getByLabel('Country').selectOption('IN');
await page.getByLabel('I agree to the terms').check();
await page.getByRole('button', { name: 'Create account' }).click();
await expect(page.getByText('Account created')).toBeVisible();
```

## Dialogs

Register the event handler before the action:

```typescript
page.once('dialog', async dialog => {
  expect(dialog.type()).toBe('confirm');
  await dialog.accept();
});
await page.getByRole('button', { name: 'Delete record' }).click();
```

Use `dismiss()` when the scenario should cancel.

## Iframes

```typescript
const frame = page.frameLocator('iframe[title="Secure payment"]');
await frame.getByLabel('Card number').fill('4111111111111111');
```

## File upload

```typescript
await page.getByLabel('Upload document').setInputFiles('fixtures/sample.pdf');
```

For a file chooser event:

```typescript
const chooserPromise = page.waitForEvent('filechooser');
await page.getByRole('button', { name: 'Choose file' }).click();
const chooser = await chooserPromise;
await chooser.setFiles('fixtures/sample.pdf');
```

## Download

```typescript
const downloadPromise = page.waitForEvent('download');
await page.getByRole('link', { name: 'Export CSV' }).click();
const download = await downloadPromise;
expect(download.suggestedFilename()).toMatch(/\.csv$/);
await download.saveAs(`test-results/${download.suggestedFilename()}`);
```

Validate file contents when data correctness is part of the requirement.

## Popup/new tab

```typescript
const popupPromise = page.waitForEvent('popup');
await page.getByRole('link', { name: 'Open report' }).click();
const popup = await popupPromise;
await expect(popup).toHaveURL(/\/report/);
```

# 10. API Testing and Network Mocking

## Mock a response

```typescript
test('shows empty orders state', async ({ page }) => {
  await page.route('**/api/orders', route => route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify({ items: [] }),
  }));

  await page.goto('/orders');
  await expect(page.getByText('No orders found')).toBeVisible();
});
```

Mocking is useful for deterministic UI states and rare errors. It does not prove the real integration works; keep real API/contract tests as well.

## Simulate an error

```typescript
await page.route('**/api/profile', route => route.fulfill({
  status: 503,
  contentType: 'application/json',
  body: JSON.stringify({ message: 'Service unavailable' }),
}));
```

Assert the UI presents an actionable error and supports recovery where required.

## API tests

```typescript
test('creates an order through the API', async ({ request }) => {
  const response = await request.post('/api/orders', {
    data: { sku: 'SKU-100', quantity: 2 },
  });

  expect(response.status()).toBe(201);
  const body = await response.json();
  expect(body).toMatchObject({ sku: 'SKU-100', quantity: 2 });
});
```

Configure `baseURL` to use relative paths. Validate negative cases, authorization, boundaries, headers, schema, idempotency, and business invariants—not just the status code.

## API setup + UI validation

```typescript
test('displays an order created by API', async ({ request, page }) => {
  const response = await request.post('/api/orders', {
    data: { sku: 'SKU-100', quantity: 1 },
  });
  expect(response.ok()).toBeTruthy();

  const order = await response.json();
  await page.goto(`/orders/${order.id}`);
  await expect(page.getByTestId('order-sku')).toHaveText('SKU-100');
});
```

Use API setup to reduce expensive preconditions, but keep end-to-end tests for the critical UI creation journey.

**Important:** TypeScript types do not validate JSON at runtime. Use JSON Schema, Zod, or explicit validation for external data. Contract testing tools such as Pact can help verify consumer/provider expectations.

# 11. Authentication and Test Data

Playwright can save browser storage state and reuse it in tests. State files may contain session tokens and must not be committed.

Example config:

```typescript
use: {
  baseURL: process.env.BASE_URL ?? 'https://qa.example.test',
  storageState: 'playwright/.auth/user.json',
}
```

A setup test can log in and save state:

```typescript
import { test as setup, expect } from '@playwright/test';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('navigation')).toBeVisible();
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

This assumes the app stores reusable auth state in browser storage. Adapt for server-side sessions or MFA. Protect generated state as a credential.

### Good test data is
- Deterministic and synthetic.
- Unique per test or worker.
- Traceable through a run ID or correlation ID.
- Disposable or safely cleaned up.
- Independent of real customer data.

Use APIs for setup when appropriate, unique namespaces, dedicated accounts, or disposable environments. Cleanup must be scoped to resources the test owns; do not silently swallow cleanup failures.

# 12. Page Object Model and Design Patterns

## Page Object Model

```typescript
// pages/LoginPage.ts
import { type Locator, type Page, expect } from '@playwright/test';

export class LoginPage {
  readonly email: Locator;
  readonly password: Locator;
  readonly signIn: Locator;
  readonly error: Locator;

  constructor(private readonly page: Page) {
    this.email = page.getByLabel('Email');
    this.password = page.getByLabel('Password');
    this.signIn = page.getByRole('button', { name: 'Sign in' });
    this.error = page.getByRole('alert');
  }

  async goto(): Promise<void> {
    await this.page.goto('/login');
  }

  async login(email: string, password: string): Promise<void> {
    await this.email.fill(email);
    await this.password.fill(password);
    await this.signIn.click();
  }

  async expectError(message: string): Promise<void> {
    await expect(this.error).toHaveText(message);
  }
}
```

Test:

```typescript
test('rejects invalid credentials', async ({ page }) => {
  const login = new LoginPage(page);
  await login.goto();
  await login.login('invalid@example.test', 'wrong-password');
  await login.expectError('Invalid email or password');
  await expect(page).toHaveURL(/\/login$/);
});
```

### POM guidelines
- Keep objects focused on a page or coherent component.
- Avoid one giant object for the entire application.
- Avoid wrapping every low-level operation without adding value.
- Use readable business operations such as `login()`.
- Keep test intent visible and assertions consistent with team conventions.

### Patterns worth knowing

| Pattern | Use | Caution |
|---|---|---|
| Page Object | Page behavior and locators | Avoid oversized objects |
| Component Object | Reusable UI widgets | Keep boundaries clear |
| Factory | Create users/orders | Avoid needless abstraction |
| Builder | Complex test data variants | Validate required fields |
| Strategy | Vary behavior by environment | Configuration may be enough |
| Adapter | Normalize external API clients | Avoid abstraction for its own sake |
| Facade | Concise business workflow API | Do not hide side effects |
| Dependency Injection | Explicit dependencies | Avoid unnecessary complexity |
| Fixture | Setup/teardown lifecycle | Avoid hidden workflow chains |

Builder example:

```typescript
type OrderInput = { sku: string; quantity: number; couponCode?: string };

class OrderBuilder {
  private data: Partial<OrderInput> = { quantity: 1 };

  withSku(sku: string): this { this.data.sku = sku; return this; }
  withQuantity(quantity: number): this { this.data.quantity = quantity; return this; }
  withCoupon(code: string): this { this.data.couponCode = code; return this; }

  build(): OrderInput {
    if (!this.data.sku) throw new Error('SKU is required');
    if (!Number.isInteger(this.data.quantity) || this.data.quantity! <= 0) {
      throw new Error('Quantity must be a positive integer');
    }
    return this.data as OrderInput;
  }
}
```

Use a builder for complex variants; a simple factory function is usually better for a small object.

# 13. Framework Architecture

Suggested structure:

```text
playwright-framework/
├── tests/
│   ├── ui/
│   ├── api/
│   └── accessibility/
├── pages/
├── components/
├── fixtures/
├── api/
├── data/
├── schemas/
├── utils/
├── playwright/.auth/       # generated; do not commit
├── test-results/            # generated
├── playwright.config.ts
├── package.json
└── README.md
```

Do not create every folder in a small project. Add abstractions when they reduce meaningful duplication or clarify a useful contract.

Example configuration:

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: Boolean(process.env.CI),
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  timeout: 30_000,
  expect: { timeout: 5_000 },
  reporter: process.env.CI
    ? [['list'], ['html', { open: 'never' }]]
    : [['list'], ['html', { open: 'never' }]],
  use: {
    baseURL: process.env.BASE_URL ?? 'https://qa.example.test',
    trace: 'on-first-retry',
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

Check version-specific options against the installed Playwright version. Fail fast if required environment variables are absent. Do not hardcode secrets.

Suggested scripts:

```json
{
  "scripts": {
    "test": "playwright test",
    "test:headed": "playwright test --headed",
    "test:ui": "playwright test --ui",
    "test:debug": "playwright test --debug",
    "test:chromium": "playwright test --project=chromium",
    "test:smoke": "playwright test --grep @smoke",
    "report": "playwright show-report"
  }
}
```

# 14. Parallelism, Retries, and Flakiness

Workers run tests concurrently. Sharding divides the suite across separate CI jobs; they can be combined. More concurrency is not always faster because databases, browsers, APIs, and test environments have finite capacity.

### Common parallel failures
- Shared mutable accounts or records.
- Tests relying on execution order.
- Cleanup deleting another test's data.
- Global state shared between tests.
- Rate limits or overloaded dependencies.
- Reused authentication state with mutable server-side state.

### Reliability practices
- Make each test independently runnable.
- Create unique data per test/worker.
- Scope cleanup to owned data.
- Avoid shared mutable global state.
- Use serial mode only when a real dependency cannot reasonably be removed.
- Benchmark worker count rather than guessing.

### Retry policy
1. Keep retries low and explicit.
2. Track first-attempt failures separately from final pass rate.
3. Use retry artifacts to diagnose transient failures.
4. Assign owners and deadlines to flaky tests.
5. Do not treat a retry pass as proof that a test is healthy.

### Failure triage
Ask:
1. Did the application return the wrong result?
2. Was the test data correct and unique?
3. Did the locator identify the intended element?
4. Did the test wait for the right condition?
5. Was the environment healthy?
6. Is the failure reproducible in isolation?
7. Does the trace indicate a product race or test race?

Do not automatically label every intermittent failure a “flaky test”; race conditions can be genuine product defects.

# 15. Configuration, Projects, and Environments

Projects can represent browser engines, devices, environments, or authenticated roles. Tags can select useful groups:

- `@smoke`: critical journeys
- `@regression`: broader business behavior
- `@api`: API-focused tests
- `@accessibility`: accessibility checks
- `@slow`: expensive scenarios

Example:

```typescript
test('user can sign in @smoke', async ({ page }) => {
  // ...
});
```

```bash
npx playwright test --grep "@smoke"
npx playwright test --grep-invert "@slow"
npx playwright test --project=chromium
```

| Environment | Goal | Typical suite |
|---|---|---|
| Local | Developer feedback | Focused tests |
| Pull request | Prevent regressions | Smoke + changed-area tests |
| Staging | Validate release candidate | Broader regression |
| Scheduled | Wide coverage | Cross-browser and long suite |
| Production synthetic | Safe availability checks | Read-only/controlled journeys |

Never run destructive tests against production without explicit safeguards and authorization.

# 16. CI/CD and Reporting

Typical pipeline:

```text
Pull request
  -> install pinned dependencies and matching browsers
  -> lint/type-check
  -> run targeted tests
  -> publish report and failure artifacts
  -> apply risk-based quality gate
  -> merge/deploy
  -> run staging smoke tests
```

GitHub Actions example:

```yaml
name: Playwright Tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
        env:
          BASE_URL: ${{ vars.BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-results
          path: |
            playwright-report/
            test-results/
          retention-days: 14
```

Adapt versions and secret handling to organizational policy. Do not expose secrets to untrusted pull-request code.

### Reporting metrics
Track:
- First-attempt pass rate
- Final pass rate after retries
- Flaky test rate
- Mean and p95 test duration
- Failure categories by product area
- Defect leakage
- Time to diagnose and resolve failures
- Critical journey coverage
- Maintenance effort and release feedback time

A high pass rate can mislead if retries hide widespread instability or critical failures are underrepresented.

# 17. Debugging and Observability

Useful commands:

```bash
npx playwright test --debug
npx playwright test --headed
npx playwright test --ui
npx playwright show-report
npx playwright test --trace on
```

A practical debugging workflow:
1. Reproduce with the same data and environment.
2. Open the trace and find the first unexpected event.
3. Inspect locator resolution and actionability.
4. Check the relevant request/response.
5. Inspect console errors.
6. Verify setup and cleanup.
7. Fix the root cause and rerun in isolation and parallel.

Log useful identifiers (test run ID, order ID, role, endpoint, correlation ID), not every low-level action. Redact tokens, passwords, personal data, and payment details.

Failure triage template:

```text
Test:
Environment/build:
Browser/project:
Failure category:
First failing action/assertion:
Expected vs actual:
Reproducible alone?:
Reproducible in parallel?:
Trace/report:
Relevant API/application logs:
Test data created:
Owner and next action:
```

# 18. Accessibility, Visual, and Cross-Browser Testing

Role-based locators encourage accessible interaction but do not prove full accessibility compliance. Combine automated scans with keyboard navigation, semantic review, and manual evaluation of complex interactions.

Screenshot assertion:

```typescript
await expect(page).toHaveScreenshot('dashboard.png');
```

Visual tests require stable data, fonts, viewport, browser/OS, and controlled animations. Review diffs before accepting a baseline.

Do not run every test in every browser on every pull request by default. Use risk-based selection:
- Critical journeys on required browser projects.
- Broader cross-browser suite on a schedule or before release.
- Browser-specific scenarios on the relevant engine.
- Responsive checks for important viewport sizes.

WebKit testing is valuable but is not identical to every Safari/platform combination; validate supported platform requirements.

# 19. Performance and Security

Playwright can measure user-observed timings and detect obvious regressions, but it is not a substitute for load testing. Use JMeter, k6, or another suitable tool for throughput, concurrency, soak, and capacity tests.

Potential security checks:
- Protected pages deny unauthenticated access.
- Users cannot read other users' records by changing IDs.
- Session expiry behaves correctly.
- Errors do not expose sensitive implementation details.
- Secrets are not exposed in URLs, logs, or reports.
- Upload validation rejects disallowed types and sizes.

These complement security review, threat modeling, penetration testing, and API authorization testing.

# 20. SDET Lead Framework Design

Reference architecture:

```text
CI/CD + Quality Gates
        |
Playwright Test Runner (projects/tags/retries)
        |
  +-----+-----------+
  |                 |
UI Tests          API Tests       Accessibility/Visual
  |                 |                   |
Page/Component    API Clients         Scanners/Snapshots
Objects           + Schemas
  +-----------------+-------------------+
                    |
       Fixtures / Test Data / Auth / Config
                    |
              App + Services
```

### Design principles
- Keep test intent separate from technical mechanics.
- Make data and environment configuration explicit.
- Use deterministic setup and cleanup.
- Add traceable identifiers for logs and test records.
- Avoid shared mutable data across parallel workers.
- Categorize failures rather than reporting only red/green.
- Make common patterns easy without blocking unusual cases.
- Keep abstractions small, typed, documented, and testable.

### How to execute 2,000 tests efficiently
1. Profile durations and environment utilization.
2. Remove redundant scenarios and UI-heavy setup.
3. Move narrow business rules to unit/API layers where appropriate.
4. Make data independent and parallel-safe.
5. Set worker count based on measured CPU, memory, and backend capacity.
6. Shard across CI jobs and balance by duration.
7. Run fast smoke tests for PRs and broader suites on schedules/pre-release.
8. Preserve diagnostic artifacts mainly for failures.
9. Track first-attempt pass rate and flake rate as well as runtime.

### Leadership interview themes
- Test selection by risk and ROI
- Framework governance and code review
- Flaky-test ownership and remediation
- Quality gates and release decisions
- Coaching engineers
- Coverage vs runtime trade-offs
- Metrics tied to customer and business outcomes

# 21. Interview Questions and Answers

## Beginner

### 1. What is Playwright?
A browser automation framework for Chromium, Firefox, and WebKit. With `@playwright/test`, it includes fixtures, assertions, parallel execution, retries, projects, reporting, and debugging. It supports UI automation and API testing.

### 2. Playwright vs Selenium?
Both automate browsers. Playwright offers built-in auto-waiting, contexts, network routing, tracing, and an integrated test runner. Selenium has a mature WebDriver ecosystem, broad language support, and established grid integrations. Choose based on language, browser/device needs, infrastructure, and existing investments.

### 3. What is `@playwright/test`?
The Playwright test runner package. It provides `test`, `expect`, fixtures, hooks, projects, parallelism, retries, and reporters. The lower-level Playwright library can be used without this runner.

### 4. What is a locator?
A reusable query for elements that resolves at action/assertion time and supports re-resolution and auto-waiting. Prefer semantic locators where they express the intended interaction.

### 5. What is auto-waiting?
Playwright waits for actionability before actions and retries web-first assertions until they pass or time out. It reduces arbitrary sleeps but does not remove the need to wait for the correct business condition.

### 6. What is a browser context?
An isolated session with its own cookies, local storage, and related session state. Multiple roles can be tested in separate contexts.

### 7. What is a fixture?
A dependency such as a page, context, API client, or custom object that Playwright sets up and tears down with a defined lifecycle.

### 8. `getByRole()` vs `locator()`?
Both return locators. `getByRole()` targets semantic role and accessible name. `locator()` supports CSS and other selectors and is useful when semantic options do not fit.

### 9. `toBeVisible()` vs `isVisible()`?
`toBeVisible()` retries as an assertion. `isVisible()` checks the current state and returns a boolean without waiting for it to become visible.

### 10. How do you run a specific test?
Use `npx playwright test tests/login.spec.ts` for a file or `npx playwright test -g "login"` to filter by title.

## Intermediate

### 11. What is POM?
It encapsulates page-specific locators and operations. It reduces duplication but should not hide test intent behind excessive generic wrappers.

### 12. How do you handle dynamic elements?
Use stable semantic locators, scope them, assert uniqueness, and wait for the expected condition using web-first assertions. Avoid brittle positional selectors.

### 13. How do you handle dropdowns?
Use `selectOption()` for native `<select>` controls. For custom dropdowns, click and choose an option by role/name, then assert the resulting value or behavior.

### 14. How do you upload a file?
Use `setInputFiles()` on the file input, or wait for the `filechooser` event before clicking and then call `setFiles()`.

### 15. How do you handle a popup?
Register `page.waitForEvent('popup')` before the action, then assert the popup URL/content.

### 16. How do you mock an API?
Use `page.route()` to intercept and fulfill a browser request. Mocks help deterministic UI tests but do not replace real integration or contract tests.

### 17. `page.route()` vs `request` fixture?
`page.route()` intercepts browser-originated requests. `request` sends HTTP requests directly for API tests and test-data setup.

### 18. How do you manage authentication?
Authenticate in a setup step and save storage state for dependent tests. Protect it like a credential and use separate states/accounts for different roles or mutable user data.

### 19. How do you test negative cases?
Cover invalid input, authorization failures, missing fields, boundaries, duplicate submissions, service errors, and recovery. Assert user-visible behavior and backend invariants where relevant.

### 20. What are projects?
Configurations for different browsers, devices, roles, or environments that can run the same or selected tests.

### 21. What is a trace?
A diagnostic recording that can show actions, DOM snapshots, network activity, and other evidence. It is useful for CI-only and intermittent failures.

### 22. How do you handle flaky tests?
Reproduce, inspect trace, verify data isolation, locators, waits, environment health, and app timing. Track root cause and ownership; do not simply add sleeps or retries.

### 23. How do you configure retries?
Set `retries` in the Playwright config or supported CLI mechanism. Keep them low and report first-attempt failures separately.

### 24. How do you run tests in parallel?
Configure workers and optionally shard across CI jobs. Ensure independent tests, unique data, and sufficient service capacity.

### 25. Test timeout vs assertion timeout?
The test timeout limits total test duration; assertion timeout limits how long a retrying assertion waits. Tune based on evidence.

## Advanced

### 26. How would you design a scalable framework?
Separate tests, page/component objects, API clients, fixtures, data factories, configuration, and reporting. Use strict TypeScript, runtime validation for external data, unique test data, and useful traceable logs. Add only valuable abstractions.

### 27. How do you prevent data collisions?
Generate unique data per test/worker, avoid shared mutable records, scope cleanup, and use dedicated accounts/namespaces. API setup can be faster than UI setup but must not bypass the behavior under test.

### 28. How do you choose worker count?
Benchmark runtime, CPU/memory, browser overhead, service capacity, database load, and failure rate. Increase gradually; there is no universal ideal.

### 29. How do you reduce runtime without reducing confidence?
Remove redundant scenarios, move narrow rules to unit/API layers, parallelize independent tests, use API setup, split smoke/regression, optimize fixtures, and profile long tests.

### 30. Product defect or test defect?
Use expected/actual behavior, trace, request/response evidence, logs, and reproducibility. Verify data and environment. A valid scenario violating a requirement suggests a product defect; wrong locators or setup suggest a test defect.

### 31. What is a good retry policy?
Low retry count, first-attempt failure reporting, clear triage, ownership, and a deadline to fix flakes. Retry pass is not proof of health.

### 32. How do you validate API schemas?
Use runtime schema validation or JSON Schema as TypeScript types disappear at runtime. Validate status, headers, fields, and business invariants.

### 33. How do you test eventual consistency?
Trigger the operation and assert/poll the target state within a documented time budget. Avoid arbitrary sleeps and capture diagnostic evidence on timeout.

### 34. How do you test role-based access?
Use separate roles, test permitted and denied actions at UI/API layers, test direct URL access, and validate object-level authorization. Hidden buttons are not a security control.

### 35. How would you test payments?
Cover success, invalid input, insufficient funds, duplicate submission, idempotency, timeout, decline, async state changes, and reconciliation. Use synthetic accounts and controlled services.

### 36. How do you integrate Playwright into CI?
Install pinned dependencies and matching browsers, run type checks and tests, publish reports/traces even on failure, and use risk-based gates. Separate PR smoke tests from broader suites.

### 37. How do you protect secrets?
Use CI secret stores, redact headers/logs, avoid logging credentials, protect storage-state files, and restrict artifact access/retention. Traces can contain sensitive page/network data.

### 38. Why use API setup?
It creates preconditions quickly without navigating many screens. Keep dedicated end-to-end tests for the creation journey and ensure setup does not bypass the behavior being tested.

### 39. How do you test third-party integrations?
Use mocks for deterministic edge cases, contract tests for expected interaction, and a small number of real integration tests in controlled environments.

### 40. How do you measure automation ROI?
Measure feedback time, manual effort avoided, escaped defects, critical-path coverage, diagnosis time, flake rate, maintenance, and release confidence. Test count alone is not ROI.

## TypeScript

### 41. Why TypeScript?
Static checking, editor support, safer refactoring, and clearer contracts. It does not replace runtime validation.

### 42. `any` vs `unknown`?
`any` disables type checks; `unknown` requires narrowing/validation. Prefer `unknown` for untrusted data.

### 43. What is a generic?
A way to preserve type information across reusable functions/classes, useful for clients and factories. Avoid abstractions that make simple code harder.

### 44. What is a discriminated union?
A union with a shared literal property that lets TypeScript narrow safely:

```typescript
type Result =
  | { status: 'ok'; value: string }
  | { status: 'error'; message: string };

function display(result: Result): string {
  return result.status === 'ok' ? result.value : result.message;
}
```

### 45. Do interfaces validate JSON at runtime?
No. Types are compile-time constructs. Validate external data with a runtime schema or explicit checks.

## Senior / Lead

### 46. What do you automate first?
Prioritize business-critical, high-risk, frequently repeated, stable scenarios. Consider user impact, defect history, frequency, and maintenance cost.

### 47. How do you standardize automation across a team?
Provide a reference framework, conventions, locator/data policies, review checklist, CI templates, failure ownership, and examples. Keep standards practical and consistently reviewed.

### 48. How do you resolve architecture disagreements?
Clarify requirements and constraints, compare options for maintainability/runtime/skills/CI/migration cost, prototype risky assumptions, document the decision, and revisit with evidence.

### 49. How do you coach brittle test authors?
Review a failing example together, explain the failure mode, pair on a semantic locator and meaningful assertion, and track reliability improvements.

### 50. What should block a release?
Use agreed risk gates: critical journey failures, high-severity defects, security issues, or unacceptable quality signals. A retry should not erase a critical failure.

# 22. Scenario-Based Interview Questions

Use this answer structure: **Clarify impact → gather evidence → isolate variables → fix root cause → prevent recurrence → measure improvement.**

### Scenario 1: Passes locally, fails in CI
Compare versions, environment variables, viewport, dependencies, and data. Inspect trace, console, network, and resource contention. Reproduce in a CI-like environment and fix the root cause rather than globally increasing timeouts.

### Scenario 2: Locator matches three Edit buttons
Scope the locator to the row/card using a stable record identifier, assert uniqueness, then click. Do not blindly use `nth()`.

### Scenario 3: Click sometimes does nothing
Inspect actionability, overlays, event handling, request failures, and application errors. Use trace and assert the expected outcome; do not replace the assertion with a sleep.

### Scenario 4: 2,000 tests take 90 minutes
Profile duration, eliminate redundant UI setup, move narrow rules to faster layers, parallelize independent tests, shard CI jobs, balance long tests, and verify backend capacity before increasing workers.

### Scenario 5: Failures only occur in parallel
Check shared accounts, records, global state, cleanup races, order dependencies, and rate limits. Add data ownership and reproduce concurrently. Serial mode may diagnose but should not hide the dependency.

### Scenario 6: Endpoint intermittently returns 500
Capture request/correlation ID, timestamp, response, environment, and trace. Compare service/dependency logs and determine whether it is a product defect, infrastructure issue, or test-induced load. Do not blindly retry it away.

### Scenario 7: Third-party dependency is slow
Mock it for deterministic UI states, use contract tests for expected integration, and retain a small controlled set of real integration tests.

### Scenario 8: Order remains Processing
Treat it as an async state transition. Assert/poll for the expected state within a documented time budget and gather API/event/log evidence on timeout.

### Scenario 9: Test passes after retry
Record a first-attempt failure, compare traces, inspect data and environment, assign ownership, and track recurrence. Do not count it as fully healthy just because the final status is green.

### Scenario 10: Every test logs in through the UI
Consider setup authentication and saved storage state, with separate role states. Protect the state and retain dedicated login journey tests.

### Scenario 11: UI is correct but persisted data is wrong
Validate the UI and backend invariant independently, correlate with a unique ID, and check stale state, caching, and service persistence. Browser success alone does not prove data correctness.

### Scenario 12: Business demands 100% UI automation
Explain that test count is not quality. Propose risk-based coverage across unit, API, UI, integration, accessibility, and performance layers. Measure escaped defects, critical coverage, feedback time, and maintenance.

### Scenario 13: Visual snapshot changes after font update
Confirm expected behavior and baseline environment, review the diff, verify fonts/viewport/browser, and update only after approval. Do not auto-accept all diffs.

### Scenario 14: Test two roles simultaneously
Use separate contexts or authenticated states and verify permissions and state transitions. Separate contexts model simultaneous sessions better than repeatedly logging into one context.

### Scenario 15: CI green but customers find defects
Compare incidents against test coverage, identify missing risks and unrealistic data/environment assumptions, and add targeted tests at the cheapest effective layer. Improve observability, not just test count.

### Scenario 16: Upload is rejected
Check file content, MIME type, extension, size, and server response. Test both valid and invalid files and verify backend processing.

### Scenario 17: Long-running export
Wait for a completion state/download event, validate metadata and contents, and use a realistic timeout. Poll documented status for asynchronous workflows instead of sleeping.

### Scenario 18: Team uses retries to inflate pass rate
Publish first-attempt and final results, track flakes, assign owners and remediation dates, and base release gates on agreed risk rather than final green status alone.

### Scenario 19: API returns 200 but business operation failed
Validate response body, business status, schema, and resulting state. HTTP success does not guarantee business success.

### Scenario 20: Reduce manual regression by 50%
Baseline effort, identify repeated critical scenarios, automate at appropriate layers, improve data setup, integrate CI, then measure hours saved and escaped defects. Do not promise a percentage before measurement.

# 23. Coding Exercises

Attempt each before looking for a solution.

1. **Login:** Valid login reaches dashboard; invalid login shows an error.
2. **Unique row:** Find order `ORD-1001`, click View in that row, and verify the details page.
3. **Mock 503:** Mock `/api/profile` as unavailable and verify recovery UI.
4. **API setup:** Create a record through API, verify its UI details, then clean up only that record.
5. **Parallel safety:** Run identical tests concurrently with unique records.
6. **CSV download:** Verify the download filename and expected column header.
7. **Type-safe factory:** Build an order with required SKU, default quantity one, and optional coupon.
8. **Eventual consistency:** Assert payment reaches COMPLETED within 30 seconds without a fixed sleep.
9. **Role-based access:** An ordinary user cannot access an admin route or another user's order.
10. **Framework review:** Identify risks in code with repeated login, CSS paths, fixed sleeps, shared accounts, and swallowed exceptions.

A strong solution demonstrates semantic locators, meaningful assertions, unique data, cleanup, runtime validation, diagnostics, and appropriate test-layer choices.

# 24. Best Practices and Cheat Sheet

| Mistake | Risk | Better practice |
|---|---|---|
| Fixed sleeps | Slow and flaky | Wait for a meaningful condition |
| Brittle CSS paths | Break on UI refactoring | Prefer semantic locators |
| `nth()` to hide ambiguity | Wrong target | Scope and assert uniqueness |
| Shared mutable data | Parallel collisions | Unique test data |
| Giant page object | Hard to maintain | Page/component boundaries |
| Blanket retries | Hides reliability issues | Track first-attempt failures |
| Everything through UI | Slow/redundant suite | Use appropriate test layers |
| No CI traces | Hard to diagnose | Capture failure artifacts |
| Secrets in reports | Security risk | Secret stores and redaction |
| Type casts for API data | No runtime validation | Validate external data |
| Serial suite by default | Hides dependencies | Make tests independent |
| Happy path only | Misses important risks | Negative and boundary tests |

### Commands

```bash
npx playwright test
npx playwright test tests/login.spec.ts
npx playwright test -g "login"
npx playwright test --project=chromium
npx playwright test --headed
npx playwright test --ui
npx playwright test --debug
npx playwright show-report
npx playwright install
```

### Locator examples

```typescript
page.getByRole('button', { name: 'Save' });
page.getByLabel('Email');
page.getByPlaceholder('Search');
page.getByText('Success');
page.getByTestId('order-status');
page.locator('[data-state="open"]');
```

### Assertion examples

```typescript
await expect(locator).toBeVisible();
await expect(locator).toBeEnabled();
await expect(locator).toHaveText('Completed');
await expect(locator).toHaveCount(3);
await expect(page).toHaveURL(/\/dashboard/);
await expect(page).toHaveTitle(/Portal/);
```

### Code review checklist
- [ ] Is the title behavior-focused?
- [ ] Is the locator stable, semantic, and unique?
- [ ] Are async conditions checked with retrying assertions?
- [ ] Is the test safe to run independently and in parallel?
- [ ] Is test data synthetic, unique, and cleaned up?
- [ ] Are event waits registered before triggering actions?
- [ ] Are errors visible rather than swallowed?
- [ ] Are secrets excluded from logs and artifacts?
- [ ] Are API responses validated beyond status code?
- [ ] Will failure evidence make diagnosis possible?
- [ ] Is UI the right layer for this scenario?

### Senior interview answer formula
1. **Risk:** What business behavior must be protected?
2. **Design:** Which layer, fixtures, data, and environment?
3. **Reliability:** How will the test stay deterministic?
4. **Scale:** How will it run in parallel and CI?
5. **Observability:** What evidence is available on failure?
6. **Measurement:** How will success be evaluated?

# 25. Readiness Checklist

## Core
- [ ] Explain browser, context, page, locator, and fixture.
- [ ] Write tests using semantic locators and web-first assertions.
- [ ] Handle forms, dialogs, frames, popups, uploads, and downloads.
- [ ] Explain promises and `async`/`await`.
- [ ] Explain why fixed sleeps are unreliable.

## Framework
- [ ] Design page/component objects and fixtures.
- [ ] Build API clients and validate response shapes.
- [ ] Manage configuration and authentication state safely.
- [ ] Create unique test data and clean it up.
- [ ] Keep tests independent and parallel-safe.

## Delivery and reliability
- [ ] Diagnose failures with traces and logs.
- [ ] Explain workers, sharding, retries, and isolation.
- [ ] Integrate tests into CI and publish artifacts.
- [ ] Design smoke, regression, and cross-browser suites.
- [ ] Track first-attempt pass rate and flaky-test trends.

## Lead
- [ ] Choose UI vs API vs unit tests based on risk.
- [ ] Define strategy, review standards, ownership, and release gates.
- [ ] Reduce runtime using evidence rather than guesswork.
- [ ] Connect automation metrics to release confidence and customer outcomes.

## Practice project

Build an online shopping suite with login, role-based access, search/filtering, cart validation, checkout happy/negative paths, API-created orders, mocked payment failures, CSV export, cross-browser smoke tests, and CI reports/traces. Include a positive, negative, and boundary/resilience scenario for each critical feature.

## Final advice

Master Playwright by building a reliable suite, not by memorizing API syntax. Senior/lead interviews test whether you can explain why a design is maintainable, diagnosable, secure, and appropriately fast. Be ready to show code, explain a failure investigation, discuss trade-offs, and connect engineering choices to product risk.

**Official reference:** https://playwright.dev/docs/intro

*Documentation changes over time. Verify version-specific APIs and configuration against the official Playwright documentation and the version installed in your project.*
