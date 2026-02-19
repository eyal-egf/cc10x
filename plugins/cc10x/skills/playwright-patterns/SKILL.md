---
name: playwright-patterns
description: "Playwright E2E testing: POM, locators, fixtures, CI/CD. Complementary skill invoked via CLAUDE.md skill table."
allowed-tools: Read, Grep, Glob, Bash, LSP
---

# Playwright Patterns

## Overview

E2E tests exist to verify that the application works from the user's perspective. Every test should simulate real user behavior with real assertions. If a test passes but the user can't complete their task, the test is wrong.

**Core principle:** Test what the user sees and does, not what the code does internally.

**Violating the letter of this process is violating the spirit of E2E testing.**

## Focus Areas (Reference Pattern)

- **Test structure** (describe/test organization, fixtures, hooks)
- **Locator strategies** (getByRole, getByText, getByTestId)
- **Assertions** (toBeVisible, toHaveURL, toContainText)
- **Page Object Model** (encapsulation, reusability)
- **Network handling** (mocking, intercepting, waiting)
- **Reliability** (auto-waiting, retry logic, flaky test prevention)
- **CI/CD integration** (Docker, parallel execution, artifacts)

## The Iron Law

```
NO E2E TEST WITHOUT EXPLICIT USER-VISIBLE ASSERTIONS
```

Checking internal state, network calls, or DOM structure is not E2E testing. If the assertion doesn't verify something the user can see or interact with, it's the wrong kind of test.

## When to Use

**Always when working with:**
- Playwright test files (`*.spec.ts`, `*.test.ts`)
- Playwright configuration (`playwright.config.ts`)
- E2E test infrastructure (fixtures, page objects, helpers)
- CI pipeline E2E test stages
- Cross-browser testing requirements

**Relationship with TDD skill:** The `test-driven-development` skill covers unit/integration testing (Vitest, Jest). Playwright is for end-to-end user-flow verification. They complement each other: use TDD for business logic, use Playwright for critical user journeys.

## Universal Questions (Answer First)

1. **What user flow are we testing?** Map the click-by-click journey.
2. **What's the assertion?** What does the user see when it works?
3. **What can go wrong?** Network failures, slow responses, race conditions.
4. **Is this E2E or should it be a unit/integration test?** E2E is expensive. Only test critical user paths.
5. **Is this test independent?** Can it run alone, in any order, without other tests?

## Project Setup

### Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html', { open: 'never' }],
    ['list'],
    ...(process.env.CI ? [['github'] as const] : []),
  ],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
    { name: 'mobile-safari', use: { ...devices['iPhone 13'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
})
```

**Configuration rules:**
- `fullyParallel: true` for speed
- `forbidOnly: !!process.env.CI` prevents `.only` from reaching CI
- `retries: 2` in CI to handle genuine flakiness
- Always capture traces, screenshots, and video on failure
- Define a `webServer` block so tests auto-start the app

## Locator Strategy (Priority Order)

| Priority | Locator | When to Use | Example |
|----------|---------|-------------|---------|
| 1 | `getByRole` | Interactive elements with accessible names | `getByRole('button', { name: 'Submit' })` |
| 2 | `getByText` | Static text content | `getByText('Welcome back')` |
| 3 | `getByLabel` | Form inputs | `getByLabel('Email address')` |
| 4 | `getByPlaceholder` | Inputs when label is not available | `getByPlaceholder('Search...')` |
| 5 | `getByTestId` | When semantic locators don't work | `getByTestId('order-summary')` |
| LAST | CSS/XPath | Almost never - fragile | `page.locator('.btn-primary')` |

```typescript
// GOOD: Semantic locators (resilient to UI changes)
await page.getByRole('button', { name: 'Add to cart' }).click()
await page.getByLabel('Email').fill('user@example.com')
await expect(page.getByRole('heading', { name: 'Order Confirmation' })).toBeVisible()

// ACCEPTABLE: Test IDs for complex elements
await expect(page.getByTestId('shopping-cart-count')).toHaveText('3')

// BAD: CSS selectors (break when styles change)
await page.locator('.btn-primary').click()
await page.locator('#email-input').fill('user@example.com')
await expect(page.locator('div > span.count')).toHaveText('3')
```

**Locator rules:**
- Prefer `getByRole` for anything interactive (buttons, links, inputs, checkboxes)
- Use `getByLabel` for form fields (tests accessibility simultaneously)
- Reserve `getByTestId` for elements without good semantic identifiers
- Never use CSS class selectors (they change with styling)

## Test Structure

### Basic Test

```typescript
import { test, expect } from '@playwright/test'

test.describe('Shopping Cart', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/shop')
  })

  test('user can add item to cart', async ({ page }) => {
    // Arrange - find the product
    const product = page.getByRole('article').filter({ hasText: 'Blue T-Shirt' })

    // Act - add to cart
    await product.getByRole('button', { name: 'Add to cart' }).click()

    // Assert - verify user-visible feedback
    await expect(page.getByRole('status')).toContainText('Added to cart')
    await expect(page.getByTestId('cart-count')).toHaveText('1')
  })

  test('user can remove item from cart', async ({ page }) => {
    // Setup: add item first
    await page.getByRole('button', { name: 'Add to cart' }).first().click()
    await page.getByRole('link', { name: 'Cart' }).click()

    // Act
    await page.getByRole('button', { name: 'Remove' }).click()

    // Assert
    await expect(page.getByText('Your cart is empty')).toBeVisible()
    await expect(page.getByTestId('cart-count')).toHaveText('0')
  })
})
```

### Test Isolation

```typescript
// GOOD: Each test is independent
test('user can search for products', async ({ page }) => {
  await page.goto('/shop')
  await page.getByLabel('Search').fill('headphones')
  await page.getByLabel('Search').press('Enter')
  await expect(page.getByRole('article')).toHaveCount(3)
})

// BAD: Test depends on previous test state
test('user sees search results', async ({ page }) => {
  // Assumes previous test already searched!
  await expect(page.getByRole('article')).toHaveCount(3) // Flaky!
})
```

**Test isolation rules:**
- Every test must work independently (no shared state)
- Use `beforeEach` for common setup (navigation, auth)
- Clean state between tests (Playwright does this automatically with new contexts)
- Never rely on test execution order

## Page Object Model (POM)

```typescript
// e2e/pages/LoginPage.ts
import { type Locator, type Page, expect } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly submitButton: Locator
  readonly errorMessage: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByLabel('Email')
    this.passwordInput = page.getByLabel('Password')
    this.submitButton = page.getByRole('button', { name: 'Sign in' })
    this.errorMessage = page.getByRole('alert')
  }

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.submitButton.click()
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message)
  }

  async expectLoggedIn() {
    await expect(this.page).toHaveURL('/dashboard')
  }
}
```

```typescript
// e2e/tests/login.spec.ts
import { test } from '@playwright/test'
import { LoginPage } from '../pages/LoginPage'

test.describe('Login', () => {
  let loginPage: LoginPage

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page)
    await loginPage.goto()
  })

  test('successful login redirects to dashboard', async () => {
    await loginPage.login('user@example.com', 'password123')
    await loginPage.expectLoggedIn()
  })

  test('invalid credentials show error', async () => {
    await loginPage.login('wrong@example.com', 'wrongpassword')
    await loginPage.expectError('Invalid email or password')
  })
})
```

**POM rules:**
- One page object per page/major component
- Locators defined in constructor (single source of truth)
- Methods represent user actions (login, search, addToCart)
- Assertions can live in page objects (`expectError`, `expectLoggedIn`)
- Never expose raw locators outside the page object for complex interactions

## Fixtures (Reusable Test Setup)

```typescript
// e2e/fixtures.ts
import { test as base } from '@playwright/test'
import { LoginPage } from './pages/LoginPage'
import { DashboardPage } from './pages/DashboardPage'

type Fixtures = {
  loginPage: LoginPage
  dashboardPage: DashboardPage
  authenticatedPage: DashboardPage
}

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page)
    await use(loginPage)
  },

  dashboardPage: async ({ page }, use) => {
    const dashboardPage = new DashboardPage(page)
    await use(dashboardPage)
  },

  // Fixture with authentication setup
  authenticatedPage: async ({ page }, use) => {
    // Reuse auth state from storage
    const loginPage = new LoginPage(page)
    await loginPage.goto()
    await loginPage.login('test@example.com', 'password123')
    const dashboardPage = new DashboardPage(page)
    await use(dashboardPage)
  },
})

export { expect } from '@playwright/test'
```

```typescript
// Usage in tests
import { test, expect } from '../fixtures'

test('authenticated user sees dashboard', async ({ authenticatedPage }) => {
  await expect(authenticatedPage.heading).toBeVisible()
})
```

## Authentication State Reuse

```typescript
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test'

const authFile = 'e2e/.auth/user.json'

setup('authenticate', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('Email').fill('test@example.com')
  await page.getByLabel('Password').fill('password123')
  await page.getByRole('button', { name: 'Sign in' }).click()
  await expect(page).toHaveURL('/dashboard')

  // Save auth state
  await page.context().storageState({ path: authFile })
})
```

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    // Setup project runs first
    { name: 'setup', testMatch: /.*\.setup\.ts/ },

    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'e2e/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
})
```

**Auth rules:**
- Run auth setup once, reuse via `storageState`
- Add `e2e/.auth/` to `.gitignore`
- Use separate auth files for different roles (admin, user, guest)

## Network Handling

### API Mocking

```typescript
test('shows error when API fails', async ({ page }) => {
  // Mock API response
  await page.route('**/api/orders', (route) => {
    route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Internal server error' }),
    })
  })

  await page.goto('/orders')

  await expect(page.getByRole('alert')).toContainText('Failed to load orders')
  await expect(page.getByRole('button', { name: 'Try again' })).toBeVisible()
})

test('shows loading state', async ({ page }) => {
  // Delay API response to catch loading state
  await page.route('**/api/orders', async (route) => {
    await new Promise((resolve) => setTimeout(resolve, 2000))
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ data: [] }),
    })
  })

  await page.goto('/orders')

  // Assert loading state is visible
  await expect(page.getByText('Loading orders...')).toBeVisible()
})
```

### Waiting for Network

```typescript
test('form submission waits for API', async ({ page }) => {
  await page.goto('/orders/new')

  // Fill form
  await page.getByLabel('Product').fill('Widget')
  await page.getByLabel('Quantity').fill('5')

  // Wait for API response after submit
  const responsePromise = page.waitForResponse('**/api/orders')
  await page.getByRole('button', { name: 'Place order' }).click()
  const response = await responsePromise

  expect(response.status()).toBe(201)
  await expect(page.getByText('Order placed successfully')).toBeVisible()
})
```

**Network rules:**
- Mock external APIs in tests (don't depend on third-party availability)
- Use `page.route()` for consistent test data
- Use `page.waitForResponse()` when you need to verify API calls
- Always test error states (500, 404, timeout)

## Assertions (User-Visible)

```typescript
// Page-level
await expect(page).toHaveTitle('Dashboard')
await expect(page).toHaveURL('/dashboard')
await expect(page).toHaveURL(/\/orders\/\d+/)

// Element visibility
await expect(page.getByText('Welcome')).toBeVisible()
await expect(page.getByText('Error')).not.toBeVisible()
await expect(page.getByRole('dialog')).toBeHidden()

// Text content
await expect(page.getByRole('heading')).toHaveText('My Orders')
await expect(page.getByTestId('total')).toContainText('$99.99')

// Element state
await expect(page.getByRole('button', { name: 'Submit' })).toBeEnabled()
await expect(page.getByRole('button', { name: 'Submit' })).toBeDisabled()
await expect(page.getByRole('checkbox')).toBeChecked()

// Element count
await expect(page.getByRole('listitem')).toHaveCount(5)

// Input values
await expect(page.getByLabel('Email')).toHaveValue('user@example.com')

// CSS
await expect(page.getByTestId('status')).toHaveCSS('color', 'rgb(0, 128, 0)')

// Attribute
await expect(page.getByRole('link')).toHaveAttribute('href', '/about')
```

**Assertion rules:**
- Use `expect(locator)` (auto-waits) not `expect(await locator.textContent())`
- Prefer `toBeVisible()` over `toBeAttached()` (user perspective)
- Use `toContainText()` for partial matches (more resilient)
- All assertions auto-retry by default (Playwright's built-in waiting)

## Handling Flaky Tests

| Flaky Pattern | Root Cause | Fix |
|---------------|-----------|-----|
| Element not found | Race condition | Use Playwright's auto-waiting (don't add manual waits) |
| Wrong text content | Content loads async | Use `await expect(locator).toHaveText()` (auto-retries) |
| Click intercepted | Overlay/animation | `await locator.click({ force: true })` or wait for overlay to disappear |
| Test passes locally, fails in CI | Timing differences | Increase `timeout` in config, use `retries: 2` for CI |
| State leaks between tests | Shared browser state | Ensure test isolation (new context per test is default) |

```typescript
// AVOID: Manual waits (flaky and slow)
await page.waitForTimeout(2000) // BAD

// GOOD: Wait for specific conditions
await expect(page.getByText('Loaded')).toBeVisible()
await page.waitForLoadState('networkidle')
await page.getByRole('button', { name: 'Submit' }).waitFor({ state: 'visible' })
```

**Flaky test rules:**
- Never use `page.waitForTimeout()` (hardcoded waits are always wrong)
- Use Playwright's built-in auto-waiting (locator actions wait automatically)
- Use `expect` with locators (auto-retries assertions)
- If a test is flaky, fix the test or the app — don't add retries as a band-aid

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on: [push, pull_request]
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

**CI rules:**
- Install browser deps: `npx playwright install --with-deps`
- Always upload reports as artifacts (even on success)
- Use `workers: 1` in CI to reduce flakiness on shared runners
- Use `retries: 2` in CI for genuine environment flakiness
- Set `forbidOnly: !!process.env.CI` to prevent `.only` in CI

## Debugging

```typescript
// Pause execution and open inspector
await page.pause()

// Run in headed mode for debugging
// npx playwright test --headed

// Run with Playwright UI mode
// npx playwright test --ui

// Run with trace viewer
// npx playwright test --trace on
// npx playwright show-trace trace.zip

// Debug specific test
// npx playwright test -g "user can login" --debug
```

**Test commands:**
```bash
# Run all E2E tests
npx playwright test

# Run specific file
npx playwright test e2e/tests/login.spec.ts

# Run with specific browser
npx playwright test --project=chromium

# Run in headed mode (see browser)
npx playwright test --headed

# Run with UI mode (interactive)
npx playwright test --ui

# Generate test code
npx playwright codegen http://localhost:3000

# Show last test report
npx playwright show-report
```

## Visual Regression Testing

```typescript
test('homepage matches snapshot', async ({ page }) => {
  await page.goto('/')
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixelRatio: 0.01, // Allow 1% difference
  })
})

test('component matches snapshot', async ({ page }) => {
  await page.goto('/components')
  const card = page.getByTestId('user-card')
  await expect(card).toHaveScreenshot('user-card.png')
})
```

**Visual regression rules:**
- Use `maxDiffPixelRatio` to tolerate rendering differences across platforms
- Update snapshots intentionally: `npx playwright test --update-snapshots`
- Store snapshots in version control
- Run visual tests on a single browser/OS for consistency

## Red Flags - STOP and Reconsider

If you find yourself:

- Using `page.waitForTimeout()` (hardcoded waits)
- Testing implementation details (CSS classes, DOM structure)
- Writing tests that depend on other tests
- Using CSS selectors instead of semantic locators
- Not testing error states and edge cases
- Skipping assertions ("test just needs to not crash")
- Writing E2E tests for logic that should be unit tested
- Ignoring flaky tests ("it passes most of the time")

**STOP. Revisit the test from the user's perspective.**

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "waitForTimeout is fine for this case" | It's never fine. Wait for a condition, not time. |
| "CSS selectors are more specific" | They're more fragile. Use role/label/testid. |
| "This test is flaky but it mostly passes" | Flaky tests erode trust. Fix it or delete it. |
| "E2E tests are too slow" | Only test critical paths E2E. Unit test the rest. |
| "We'll add error testing later" | Error states are where bugs hide. Test them now. |
| "page.pause() is just for debugging" | Remove before committing. Never commit debugging aids. |

## Anti-patterns Blocklist

| Anti-pattern | Why It's Wrong | Fix |
|--------------|----------------|-----|
| `page.waitForTimeout(N)` | Flaky, slow, non-deterministic | Use auto-waiting or `expect` assertions |
| `page.locator('.class-name')` | Breaks when styles change | Use `getByRole`, `getByLabel`, `getByTestId` |
| Tests sharing state | Flaky, order-dependent | Each test starts fresh |
| `test.only()` committed | Skips other tests in CI | Remove before merge, use `forbidOnly` |
| No assertions in test | Test proves nothing | Add user-visible assertions |
| Hardcoded test data URLs | Breaks across environments | Use `baseURL` config |
| `page.evaluate()` for assertions | Testing internals, not UI | Use Playwright assertions |
| `expect(await el.textContent())` | No auto-retry, flaky | Use `await expect(el).toHaveText()` |

## Output Format

```markdown
## Playwright Review: [Test Suite/Feature]

### Test Coverage
- [ ] Happy path tested
- [ ] Error states tested (API failure, validation errors)
- [ ] Loading states tested
- [ ] Empty states tested
- [ ] Authentication flows tested

### Reliability
- [ ] No hardcoded waits (`waitForTimeout`)
- [ ] Semantic locators used (getByRole/getByLabel)
- [ ] Tests are independent (no shared state)
- [ ] Auto-waiting leveraged (no manual sleeps)
- [ ] Flaky tests investigated and fixed

### Infrastructure
- [ ] CI pipeline configured
- [ ] Reports uploaded as artifacts
- [ ] Auth state reused (not logging in per test)
- [ ] Appropriate browser coverage

### Issues
| Severity | Issue | Location | Fix |
|----------|-------|----------|-----|
| [BLOCKS/IMPAIRS/MINOR] | [Issue] | `file:line` | [Fix] |

### Recommendations
1. [Most critical]
2. [Second]
```

## Final Check

Before completing E2E test work:

- [ ] All tests pass locally: `npx playwright test`
- [ ] Tests pass in CI (or CI config is correct)
- [ ] No `waitForTimeout` in test code
- [ ] No `.only` in test code
- [ ] All tests are independent
- [ ] Error and edge cases covered
- [ ] Page objects used for complex pages
- [ ] Auth state reused where possible
- [ ] Traces/screenshots configured for failures
- [ ] No CSS selectors when semantic locators work
