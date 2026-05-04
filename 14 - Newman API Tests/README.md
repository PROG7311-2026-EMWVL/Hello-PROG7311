# Adding Newman API Tests to the MathStack Pipeline

## Objective

In this guide, you will add automated API tests to the MathStack CI pipeline using Newman.

Newman is the command-line runner for Postman collections. It allows us to run exported Postman tests from the terminal and from CI pipelines such as GitHub Actions.

By the end of this guide, your pipeline will:
- Build and publish the DB, API and client images
- Pull and run the published images in GitHub Actions
- Wait for the DB, API and client to start
- Run API tests against the live API container
- Fail the pipeline if any API test fails

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
├── postman/
│   ├── mathstack-api.postman_collection.json
│   └── mathstack-ci.postman_environment.json
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

## What We Are Building

The Newman test flow will work like this:

1. GitHub Actions starts the Docker Compose stack.
2. The API becomes available on the GitHub Actions runner.
3. Newman runs the Postman collection.
4. The collection sends real HTTP requests to the API.
5. The tests check the API responses.
6. If any request or assertion fails, the GitHub Actions pipeline fails.

The API will be tested through this base URL:

```text
http://localhost:8081
```

This is because `compose.ci.yml` maps the API container port to the runner like this:

```yaml
ports:
  - "8081:8080"
```

This means:

- Inside Docker, the API runs on port `8080`
- From the GitHub Actions runner, the API is accessed on port `8081`

## Step 1: Create the Postman Folder

In the root of the project, create a folder named:

```text
postman
```

This folder will store the exported Postman collection and environment files.

The folder should be here:

```text
MathStack/postman/
```

## Step 2: Create the Postman Environment File

Inside the `postman` folder, create this file:

```text
postman/mathstack-ci.postman_environment.json
```

Add the following content:

```json
{
  "id": "mathstack-ci-environment",
  "name": "MathStack CI Environment",
  "values": [
    {
      "key": "baseUrl",
      "value": "http://localhost:8081",
      "type": "default",
      "enabled": true
    },
    {
      "key": "email",
      "value": "mathstack-ci-user@example.com",
      "type": "default",
      "enabled": true
    },
    {
      "key": "password",
      "value": "Password123!",
      "type": "secret",
      "enabled": true
    },
    {
      "key": "token",
      "value": "",
      "type": "secret",
      "enabled": true
    },
    {
      "key": "userId",
      "value": "",
      "type": "default",
      "enabled": true
    }
  ],
  "_postman_variable_scope": "environment",
  "_postman_exported_using": "Postman"
}
```

### What this does

This file stores reusable test variables.

The most important variable is:

```text
baseUrl=http://localhost:8081
```

The tests will use this value when calling the API.

For example, this:

```text
{{baseUrl}}/api/Auth/Register
```

will become:

```text
http://localhost:8081/api/Auth/Register
```

## Step 3: Create the Postman Collection File

Inside the `postman` folder, create this file:

```text
postman/mathstack-api.postman_collection.json
```

Add the following content:

```json
{
  "info": {
    "name": "MathStack API Tests",
    "_postman_id": "mathstack-api-tests",
    "description": "Newman API tests for the MathStack API running in Docker Compose CI.",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "01 - Health check",
      "request": {
        "method": "GET",
        "header": [],
        "url": {
          "raw": "{{baseUrl}}/api/Health",
          "host": ["{{baseUrl}}"],
          "path": ["api", "Health"]
        }
      },
      "event": [
        {
          "listen": "test",
          "script": {
            "type": "text/javascript",
            "exec": [
              "pm.test('Health endpoint returns 200', function () {",
              "    pm.response.to.have.status(200);",
              "});",
              "",
              "pm.test('Health endpoint returns MathAPI service status', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json.status).to.eql('ok');",
              "    pm.expect(json.service).to.eql('MathAPI');",
              "});"
            ]
          }
        }
      ]
    },
    {
      "name": "02 - Register user",
      "event": [
        {
          "listen": "prerequest",
          "script": {
            "type": "text/javascript",
            "exec": [
              "const uniqueEmail = `mathstack-ci-${Date.now()}@example.com`;",
              "pm.environment.set('email', uniqueEmail);"
            ]
          }
        },
        {
          "listen": "test",
          "script": {
            "type": "text/javascript",
            "exec": [
              "pm.test('Register returns 200', function () {",
              "    pm.response.to.have.status(200);",
              "});",
              "",
              "pm.test('Register returns token and userId', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json).to.have.property('token');",
              "    pm.expect(json).to.have.property('userId');",
              "    pm.expect(json.token).to.not.be.empty;",
              "    pm.expect(json.userId).to.not.be.empty;",
              "    pm.environment.set('token', json.token);",
              "    pm.environment.set('userId', json.userId);",
              "});"
            ]
          }
        }
      ],
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"email\": \"{{email}}\",\n  \"password\": \"{{password}}\"\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/api/Auth/Register",
          "host": ["{{baseUrl}}"],
          "path": ["api", "Auth", "Register"]
        }
      }
    },
    {
      "name": "03 - Login user",
      "event": [
        {
          "listen": "test",
          "script": {
            "type": "text/javascript",
            "exec": [
              "pm.test('Login returns 200', function () {",
              "    pm.response.to.have.status(200);",
              "});",
              "",
              "pm.test('Login returns token and userId', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json).to.have.property('token');",
              "    pm.expect(json).to.have.property('userId');",
              "    pm.expect(json.token).to.not.be.empty;",
              "    pm.expect(json.userId).to.not.be.empty;",
              "    pm.environment.set('token', json.token);",
              "    pm.environment.set('userId', json.userId);",
              "});"
            ]
          }
        }
      ],
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"email\": \"{{email}}\",\n  \"password\": \"{{password}}\"\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/api/Auth/Login",
          "host": ["{{baseUrl}}"],
          "path": ["api", "Auth", "Login"]
        }
      }
    },
    {
      "name": "04 - Add calculation",
      "event": [
        {
          "listen": "test",
          "script": {
            "type": "text/javascript",
            "exec": [
              "pm.test('PostCalculate returns 201', function () {",
              "    pm.response.to.have.status(201);",
              "});",
              "",
              "pm.test('Addition result is 15', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json.firstNumber).to.eql(10);",
              "    pm.expect(json.secondNumber).to.eql(5);",
              "    pm.expect(json.operation).to.eql(1);",
              "    pm.expect(json.result).to.eql(15);",
              "});",
              "",
              "pm.test('Calculation has database id', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json.calculationId).to.be.above(0);",
              "});"
            ]
          }
        }
      ],
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          },
          {
            "key": "Authorization",
            "value": "Bearer {{token}}"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"firstNumber\": 10,\n  \"secondNumber\": 5,\n  \"operation\": 1\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/api/Math/PostCalculate",
          "host": ["{{baseUrl}}"],
          "path": ["api", "Math", "PostCalculate"]
        }
      }
    },
    {
      "name": "05 - Get calculation history",
      "event": [
        {
          "listen": "test",
          "script": {
            "type": "text/javascript",
            "exec": [
              "pm.test('GetHistory returns 200', function () {",
              "    pm.response.to.have.status(200);",
              "});",
              "",
              "pm.test('History contains at least one calculation', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json).to.be.an('array');",
              "    pm.expect(json.length).to.be.above(0);",
              "});",
              "",
              "pm.test('History contains the calculation for this authenticated user', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json[0]).to.have.property('firebaseUuid');",
              "    pm.expect(json[0].firebaseUuid).to.eql(pm.environment.get('userId'));",
              "});"
            ]
          }
        }
      ],
      "request": {
        "method": "GET",
        "header": [
          {
            "key": "Authorization",
            "value": "Bearer {{token}}"
          }
        ],
        "url": {
          "raw": "{{baseUrl}}/api/Math/GetHistory",
          "host": ["{{baseUrl}}"],
          "path": ["api", "Math", "GetHistory"]
        }
      }
    },
    {
      "name": "06 - Delete calculation history",
      "event": [
        {
          "listen": "test",
          "script": {
            "type": "text/javascript",
            "exec": [
              "pm.test('DeleteHistory returns 200', function () {",
              "    pm.response.to.have.status(200);",
              "});",
              "",
              "pm.test('DeleteHistory returns deleted calculation list', function () {",
              "    const json = pm.response.json();",
              "    pm.expect(json).to.be.an('array');",
              "    pm.expect(json.length).to.be.above(0);",
              "});"
            ]
          }
        }
      ],
      "request": {
        "method": "DELETE",
        "header": [
          {
            "key": "Authorization",
            "value": "Bearer {{token}}"
          }
        ],
        "url": {
          "raw": "{{baseUrl}}/api/Math/DeleteHistory",
          "host": ["{{baseUrl}}"],
          "path": ["api", "Math", "DeleteHistory"]
        }
      }
    }
  ]
}
```

### What this collection tests

The collection sends six API requests:

```text
GET     /api/Health
POST    /api/Auth/Register
POST    /api/Auth/Login
POST    /api/Math/PostCalculate
GET     /api/Math/GetHistory
DELETE  /api/Math/DeleteHistory
```

The registration request creates a unique email address each time the tests run:

```javascript
const uniqueEmail = `mathstack-ci-${Date.now()}@example.com`;
pm.environment.set('email', uniqueEmail);
```

This avoids test failures caused by trying to register the same user more than once.

The register and login tests save these values:

```javascript
pm.environment.set('token', json.token);
pm.environment.set('userId', json.userId);
```

The token is then used in the protected math requests:

```text
Authorization: Bearer {{token}}
```

## Step 4: Add Newman to the Project

Newman runs through Node.js, so we need a small `package.json` file in the root of the repository.

Create this file in the root of the project:

```text
package.json
```

Add the following content:

```json
{
  "scripts": {
    "test:api": "newman run postman/mathstack-api.postman_collection.json -e postman/mathstack-ci.postman_environment.json"
  },
  "devDependencies": {
    "newman": "^6.2.1"
  }
}
```

### What this does

This adds a script called:

```text
test:api
```

That script runs this Newman command:

```bash
newman run postman/mathstack-api.postman_collection.json -e postman/mathstack-ci.postman_environment.json
```

This means we can run the API tests using:

```bash
npm run test:api
```

## Step 5: Install Newman Locally

In the root of the project, run:

```bash
npm install
```

This will create a file named:

```text
package-lock.json
```

You should commit both files:

```text
package.json
package-lock.json
```

### Why we commit `package-lock.json`

The `package-lock.json` file records the exact dependency versions installed by npm.

This helps the tests run consistently on different machines and in GitHub Actions.

## Step 6: Add `node_modules` to `.gitignore`

When you run `npm install`, a folder named `node_modules` will be created.

This folder must not be committed to GitHub.

Open the `.gitignore` file in the root of the project.

Add this line:

```gitignore
node_modules/
```

If your project already has a `.gitignore`, add it near the other build or dependency folders.

Example:

```gitignore
bin/
obj/
.vs/
.env
node_modules/
```

### Why this matters

The `node_modules` folder can be very large.

It should be recreated using:

```bash
npm install
```

or, in GitHub Actions:

```bash
npm ci
```

It should not be stored in the repository.

## Step 7: Test Locally

Before updating GitHub Actions, first test Newman locally.

Start the Docker Compose stack:

```bash
docker compose up -d --build
```

Wait for the containers to start.

Then run:

```bash
npm run test:api -- --env-var baseUrl=http://localhost:8081
```

You should see output similar to this:

```text
MathStack API Tests

→ 01 - Health check
  GET http://localhost:8081/api/Health [200 OK]

→ 02 - Register user
  POST http://localhost:8081/api/Auth/Register [200 OK]

→ 03 - Login user
  POST http://localhost:8081/api/Auth/Login [200 OK]

→ 04 - Add calculation
  POST http://localhost:8081/api/Math/PostCalculate [201 Created]

→ 05 - Get calculation history
  GET http://localhost:8081/api/Math/GetHistory [200 OK]

→ 06 - Delete calculation history
  DELETE http://localhost:8081/api/Math/DeleteHistory [200 OK]
```

At the end, Newman should show:

```text
requests      6 executed, 0 failed
assertions   14 executed, 0 failed
```

## Step 8: Update the GitHub Actions Pipeline

Open this file:

```text
.github/workflows/ci.yml
```

Find the `run-stack` job.

The Newman steps must be added after the API and client are ready.

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

    echo "Client did not respond"
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

- name: Install Newman
  run: npm ci

- name: Run Newman API tests
  run: npm run test:api -- --env-var baseUrl=http://localhost:8081
```

### What this does

This part of the pipeline:

- Installs Node.js
- Installs Newman from `package-lock.json`
- Runs the Postman collection against the running API container
- Fails the pipeline if any API test fails

## Step 9: Confirm the Final Workflow Order

The `run-stack` job should now follow this order:

```text
1. Checkout
2. Log in to GHCR
3. Pull images
4. Start stack
5. Show running containers
6. Wait for database
7. Wait for API
8. Wait for client
9. Set up Node.js
10. Install Newman
11. Run Newman API tests
12. Show logs if failure
13. Stop stack
```

The important point is that Newman must run after this:

```text
Wait for API
Wait for client
```

Newman cannot test the API until the API container is running.

## Step 10: Commit and Push Your Changes

Once all files are added and tested, commit and push your changes.

## Step 12: What Should Happen After You Push

When you push to `main`, GitHub Actions should:

1. Build the DB image
2. Build the API image
3. Build the client image
4. Push all three images to GHCR
5. Pull the images inside the workflow
6. Start the stack using `compose.ci.yml`
7. Wait for the DB, API and client
8. Install Newman
9. Run the API tests
10. Fail the workflow if any API test fails
11. Stop the containers

If all of that succeeds, the container pipeline is now testing the running API automatically.

## Summary

At this point, your pipeline should be able to:

- Publish the DB, API and client images to GHCR
- Pull those images inside GitHub Actions
- Run the full MathStack stack in the pipeline
- Confirm that the containers start successfully
- Run Newman API tests against the live API
- Fail the pipeline when the API does not behave as expected

The next step after this would be to add frontend integration tests to the pipeline.

