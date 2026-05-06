# Adding Playwright Integration Tests to the MathStack Pipeline

## Objective

In this guide, you will add automated browser-based integration tests to the MathStack CI pipeline using Playwright.

Playwright is a browser automation testing tool. It allows us to test the ASP.NET Core MVC client through a real browser. This is different from Newman, which tests the API directly.

By the end of this guide, your pipeline will:

- Build and publish the DB, API and client images
- Pull and run the published images in GitHub Actions
- Wait for the DB, API and client to start
- Run Newman API tests against the live API container
- Run Playwright integration tests against the live MVC client container
- Fail the pipeline if any API or browser test fails

Repo to refer to: https://github.com/PROG7311-2026-EMWVL/MathStack

## Overall Project Structure

Your solution should look similar to this:

```text
MathStack/
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
├── tests/
│   ├── newman/
│   │   ├── mathstack-api.postman_collection.json
│   │   └── mathstack-ci.postman_environment.json
│   │
│   └── playwright/
│       ├── playwright.config.js
│       └── mathstack-client.spec.js
│
├── db/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   └── init.sql
│
├── MathAPI/
│   ├── Controllers/
│   ├── Dockerfile
│   ├── Program.cs
│   └── ...
│
├── MathAPIClient/
│   ├── Controllers/
│   ├── Dockerfile
│   ├── Program.cs
│   └── ...
│
├── compose.ci.yml
├── compose.yml
├── .gitignore
├── package.json
└── package-lock.json
```

---

## What We Are Building

The Playwright test flow will work like this:

1. GitHub Actions starts the Docker Compose stack.
2. The MVC client becomes available on the GitHub Actions runner.
3. Playwright opens a browser.
4. Playwright visits the MVC client.
5. The tests interact with the actual web pages.
6. The MVC client talks to the API and database.
7. If any page, form, route or assertion fails, the GitHub Actions pipeline fails.

The MVC client will be tested through this base URL:

```text
http://localhost:8082
```

This is because `compose.ci.yml` maps the client container port to the runner like this:

```yaml
ports:
  - "8082:8080"
```

This means:
- Inside Docker, the MVC client runs on port `8080`
- From the GitHub Actions runner, the MVC client is accessed on port `8082`

The API is still tested through this base URL:

```text
http://localhost:8081
```

## Step 1: Create the Test Folders

In the root of the project, create a folder named `tests`

Inside it, create two folders:
```text
tests/newman
tests/playwright
```

The folder should be here: `MathStack/tests/`

### What this does

This keeps all test-related files together:
* The `newman` folder stores API tests.
* The `playwright` folder stores browser integration tests.

## Step 2: Move the Newman Files

If your Newman files are currently in a `postman` folder, move them into `tests/newman`.

Your Newman files should now be here:
```text
tests/newman/mathstack-api.postman_collection.json
tests/newman/mathstack-ci.postman_environment.json
```

## Step 3: Install Playwright

Playwright runs through Node.js, so we install it as an npm development dependency.

In the root of the project, run:

```bash
npm install -D @playwright/test
```

This updates these files:

```text
package.json
package-lock.json
```

#### Install the browser locally

Playwright needs a browser to run tests.

For this project, install Chromium only:

```bash
npx playwright install chromium
```

## Step 4: Update `package.json`

Open this file in the root of the project:

```text
package.json
```

Update it to include separate scripts for Newman and Playwright:

```json
{
  "scripts": {
    "test:api": "newman run tests/newman/mathstack-api.postman_collection.json -e tests/newman/mathstack-ci.postman_environment.json",
    "test:playwright": "playwright test --config=tests/playwright/playwright.config.js"
  },
  "devDependencies": {
    "@playwright/test": "^1.56.0",
    "newman": "^6.2.1"
  }
}
```

### What this does

This gives us two test commands:

```text
npm run test:api
npm run test:playwright
```

* The first command runs the Newman API tests.
* The second command runs the Playwright browser tests.


## Step 5: Create the Playwright Config File

Inside the `tests/playwright` folder, create this file:

```text
tests/playwright/playwright.config.js
```

Add the following content:

```javascript
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: '.',
  timeout: 30 * 1000,

  use: {
    baseURL: process.env.CLIENT_BASE_URL || 'http://localhost:8082',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },

  reporter: [
    ['list'],
    ['html', { outputFolder: '../../playwright-report', open: 'never' }]
  ],

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    }
  ]
});
```

### What this does

This tells Playwright:
- The tests are in the same folder as the config file
- The MVC client is available at `http://localhost:8082`
- Only Chromium should be used
- A report should be generated in `playwright-report`
- Screenshots and videos should be kept only when failures occur


## Step 6: Create the Playwright Test File

Inside the `tests/playwright` folder, create this file:

```text
tests/playwright/mathstack-client.spec.js
```

Add the following content:

```javascript
const { test, expect } = require('@playwright/test');

const uniqueId = Date.now();

const testUser = {
  email: `playwright.${uniqueId}@mathstack.test`,
  testPassword: 'NotASecret-UsedOnlyForPlaywrightTests123!'
};

test('health endpoint returns ok', async ({ request }) => {
  const response = await request.get('/health');

  expect(response.ok()).toBeTruthy();
  expect(await response.text()).toBe('ok');
});

/**
test('intentional failing test to view Playwright report', async ({ page }) => {
  await page.goto('/');

  await expect(page.getByRole('heading', { name: 'This heading does not exist' })).toBeVisible();
});
*/

test('home page loads successfully', async ({ page }) => {
  await page.goto('/');

  await expect(page).toHaveTitle(/Home Page - MathAPIClient/);
  await expect(page.getByRole('heading', { name: 'Welcome' })).toBeVisible();
});

test('main navigation links are visible before login', async ({ page }) => {
  await page.goto('/');

  const nav = page.locator('header nav');

  await expect(nav.getByRole('link', { name: 'MathAPIClient' })).toBeVisible();
  await expect(nav.getByRole('link', { name: 'Calculate' })).toBeVisible();
  await expect(nav.getByRole('link', { name: 'History' })).toBeVisible();
  await expect(nav.getByRole('link', { name: 'Login' })).toBeVisible();
});

test('login page loads and links to register', async ({ page }) => {
  await page.goto('/Auth/Login');

  await expect(page).toHaveTitle(/Login - MathAPIClient/);
  await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();
  await expect(page.getByLabel('Email')).toBeVisible();
  await expect(page.getByLabel('Password')).toBeVisible();
  await expect(page.getByRole('button', { name: 'Login' })).toBeVisible();
  await expect(page.getByRole('link', { name: 'Register' })).toBeVisible();
});

test('register page loads and links back to login', async ({ page }) => {
  await page.goto('/Auth/Register');

  await expect(page).toHaveTitle(/Register - MathAPIClient/);
  await expect(page.getByRole('heading', { name: 'Register' })).toBeVisible();
  await expect(page.getByLabel('Email')).toBeVisible();
  await expect(page.getByLabel('Password')).toBeVisible();
  await expect(page.getByRole('button', { name: 'Register' })).toBeVisible();
  await expect(page.getByRole('link', { name: /Already have an account\? Login/i })).toBeVisible();
});

test('calculate page redirects unauthenticated user to login', async ({ page }) => {
  await page.goto('/Math/Calculate');

  await expect(page).toHaveURL(/\/Auth\/Login/i);
  await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();
});

test('history page redirects unauthenticated user to login', async ({ page }) => {
  await page.goto('/Math/History');

  await expect(page).toHaveURL(/\/Auth\/Login/i);
  await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();
});

test.describe.serial('authenticated user features', () => {
  test.beforeAll(async ({ browser }) => {
    const page = await browser.newPage();

    await page.goto('/Auth/Register');

    await page.getByLabel('Email').fill(testUser.email);
    await page.getByLabel('Password').fill(testUser.testPassword);
    await page.getByRole('button', { name: 'Register' }).click();

    await expect(page).toHaveURL(/\/Math\/Calculate/i);
    await expect(page.getByRole('heading', { name: /Welcome to the Calculator/i })).toBeVisible();

    await page.close();
  });

  test.beforeEach(async ({ page }) => {
    await page.goto('/Auth/Login');

    await page.getByLabel('Email').fill(testUser.email);
    await page.getByLabel('Password').fill(testUser.testPassword);
    await page.getByRole('button', { name: 'Login' }).click();

    await expect(page).toHaveURL(/\/Math\/Calculate/i);
    await expect(page.getByRole('heading', { name: /Welcome to the Calculator/i })).toBeVisible();
  });

  test('registered user can access calculate page', async ({ page }) => {
    await page.goto('/Math/Calculate');

    await expect(page).toHaveURL(/\/Math\/Calculate/i);
    await expect(page.getByRole('heading', { name: /Welcome to the Calculator/i })).toBeVisible();
  });

  test('registered user can perform a calculation', async ({ page }) => {
    await page.goto('/Math/Calculate');

    await page.locator('input[name="FirstNumber"]').fill('10');
    await page.locator('input[name="SecondNumber"]').fill('5');
    await page.locator('select[name="Operation"]').selectOption('1');
    await page.getByRole('button', { name: 'Calculate' }).click();

    await expect(page.getByText(/Result is 15/i)).toBeVisible();
  });

  test('registered user can view calculation history', async ({ page }) => {
    await page.goto('/Math/Calculate');

    await page.locator('input[name="FirstNumber"]').fill('10');
    await page.locator('input[name="SecondNumber"]').fill('5');
    await page.locator('select[name="Operation"]').selectOption('1');
    await page.getByRole('button', { name: 'Calculate' }).click();

    await expect(page.getByText(/Result is 15/i)).toBeVisible();

    await page.goto('/Math/History');

    await expect(page).toHaveTitle(/History - MathAPIClient/);
    await expect(page.getByRole('heading', { name: 'History' })).toBeVisible();

    const table = page.locator('table');

    await expect(table).toContainText('10');
    await expect(table).toContainText('+');
    await expect(table).toContainText('5');
    await expect(table).toContainText('15');
  });

  test('registered user can clear calculation history', async ({ page }) => {
    await page.goto('/Math/Calculate');

    await page.locator('input[name="FirstNumber"]').fill('10');
    await page.locator('input[name="SecondNumber"]').fill('5');
    await page.locator('select[name="Operation"]').selectOption('1');
    await page.getByRole('button', { name: 'Calculate' }).click();

    await expect(page.getByText(/Result is 15/i)).toBeVisible();

    await page.goto('/Math/History');
    await page.getByRole('button', { name: 'Clear' }).click();

    await expect(page).toHaveURL(/\/Math\/History/i);
    await expect(page.getByRole('heading', { name: 'History' })).toBeVisible();
    await expect(page.locator('body')).toContainText('History');
  });

  test('registered user can log out', async ({ page }) => {
    await page.goto('/Auth/LogOut');

    await expect(page).toHaveURL(/\/Auth\/Login/i);
    await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();
  });
});
```

## Step 7: Add Generated Files to `.gitignore`

When you run Playwright, it creates report and result folders. These must not be committed.

Open the `.gitignore` file in the root of the project:
```
.env
bin/
obj/
.vs/
node_modules/
playwright-report/
test-results/
```

### Why this matters

The `node_modules` folder can be very large.
The `playwright-report` and `test-results` folders are generated every time tests run.

## Step 8: Test Locally Before Updating GitHub Actions

Start the Docker Compose stack:

```bash
docker compose up -d --build
```
Wait for the containers to start.

Then run this in the root of your project:

```bash
npm ci
```

Then run:

```bash
npx playwright install chromium
```

Then run:

```bash
npm run test:playwright
```


Once it runs, Playwright should show output similar to this:

```text
12 passed
```

## Step 9: Intentionally Run a Failing Test

Before pushing the tests, run one intentional failure so that you can learn how the Playwright report works.

Open this file:

```text
tests/playwright/mathstack-client.spec.js
```

Find this commented-out test:

```javascript
/**
test('intentional failing test to view Playwright report', async ({ page }) => {
  await page.goto('/');

  await expect(page.getByRole('heading', { name: 'This heading does not exist' })).toBeVisible();
});
*/
```

Uncomment it so that it looks like this:

```javascript
test('intentional failing test to view Playwright report', async ({ page }) => {
  await page.goto('/');

  await expect(page.getByRole('heading', { name: 'This heading does not exist' })).toBeVisible();
});
```

Run Playwright again:

```bash
npm run test:playwright
```

This test should fail.

It fails because the home page does not contain this heading:

```text
This heading does not exist
```

This failure is expected.

## Step 10: View the Failing Playwright Report

After the failing test runs, open the Playwright report:

```bash
npx playwright show-report playwright-report
```

The report should show:

- Which test failed
- Which line failed
- What Playwright was trying to find
- The page state when the failure happened
- Screenshots or traces if they were captured

### Why this matters

Integration tests often fail because:

- The page did not load
- The selector was wrong
- The user was redirected
- The API or database was not ready
- The expected text was different from the actual text

The Playwright report helps you find the reason for the failure.

You can also view screenshots, the report and a video related to that test in the `test-reports` folder.

## Step 11: Comment the Failing Test Out Again

After viewing the report, comment the intentional failing test out again.

Change it back to this or remove it entirely:

```javascript
/**
test('intentional failing test to view Playwright report', async ({ page }) => {
  await page.goto('/');

  await expect(page.getByRole('heading', { name: 'This heading does not exist' })).toBeVisible();
});
*/
```

Run Playwright again:

```bash
npm run test:playwright
```

The tests should pass again.

Then view the report again:

```bash
npx playwright show-report playwright-report
```

You should now see a passing test run.

Tip: You can view the browser in action while tests run using this command:
```bash
npm run test:playwright -- --headed
```

## Step 12: Update the GitHub Actions Pipeline

Open this file:

```text
.github/workflows/ci.yml
```

Find the `run-stack` job.

The Newman and Playwright steps must be added after the API and client are ready.

Find this section:

```yaml
- name: Wait for client
  run: |
    for i in {1..36}; do
      if curl -fsS http://localhost:8082/health >/dev/null 2>&1; then
        echo "Client is ready"
        exit 0
      fi
      echo "Waiting for client..."
      sleep 5
    done
    docker logs math-client || true
    exit 1
```

Immediately after that, add:

```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version: 22
    cache: npm

- name: Install npm dependencies
  run: npm ci

- name: Install Playwright Chromium
  run: npx playwright install chromium --with-deps

- name: Run Newman API tests
  run: npm run test:api -- --env-var baseUrl=http://localhost:8081

- name: Run Playwright integration tests
  run: npm run test:playwright
  env:
    CLIENT_BASE_URL: http://localhost:8082

- name: Upload Playwright report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report
    path: playwright-report
    if-no-files-found: ignore
```

### What this does

This part of the pipeline:

- Installs Node.js
- Installs Newman and Playwright from `package-lock.json`
- Installs Chromium for Playwright
- Runs the Postman collection against the running API container
- Runs browser tests against the running MVC client container
- Uploads the Playwright report as a GitHub Actions artifact
- Fails the pipeline if any API or browser test fails

## Step 13: Confirm the Final Workflow Order

The `run-stack` job should now follow this order:

```text
1. Checkout
2. Log in to GHCR
3. Pull images
4. Start stack
5. Wait for database
6. Wait for API
7. Wait for client
8. Set up Node.js
9. Install npm dependencies
10. Install Playwright Chromium
11. Run Newman API tests
12. Run Playwright integration tests
13. Upload Playwright report
14. Show logs if failure
15. Stop stack
```

The important point is that Newman and Playwright must run after this:

```text
Wait for API
Wait for client
```

Newman cannot test the API until the API container is running.

Playwright cannot test the MVC client until the client container is running.

## Step 14: Commit and Push Your Changes

Once all files are added and tested, commit and push your changes.

When you push to `main`, GitHub Actions should:

1. Build the DB image
2. Build the API image
3. Build the client image
4. Push all three images to GHCR
5. Pull the images inside the workflow
6. Start the stack using `compose.ci.yml`
7. Wait for the DB, API and client
8. Install Newman and Playwright
9. Install Chromium for Playwright
10. Run the API tests
11. Run the browser integration tests
12. Upload the Playwright report
13. Fail the workflow if any test fails
14. Stop the containers

If all of that succeeds, the container pipeline is now testing both the running API and the running MVC client automatically.

## Summary

At this point, your pipeline should be able to:

- Publish the DB, API and client images to GHCR
- Pull those images inside GitHub Actions
- Run the full MathStack stack in the pipeline
- Confirm that the containers start successfully
- Run Newman API tests against the live API
- Run Playwright browser integration tests against the live MVC client
- Upload a Playwright report
- Fail the pipeline when the API or MVC client does not behave as expected

This gives ensures that the full Dockerised MathStack application works correctly, not only at API level, but also from the user interface.