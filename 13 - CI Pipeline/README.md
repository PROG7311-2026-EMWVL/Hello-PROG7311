# Pipelining the MathStack Containers with GitHub Actions
## Objective

In this guide, you will set up a GitHub Actions pipeline for the full MathStack solution so that:

* the SQL Server database is packaged as its own Docker image
* the API is packaged as its own Docker image
* the client is packaged as its own Docker image
* the images are published to GitHub Container Registry (GHCR)
* the published images remain private in GHCR
* the pipeline pulls and runs the published images inside GitHub Actions
* the pipeline confirms that the containers start successfully

Repo to refer to: https://github.com/PROG7311-2026-EMWVL/MathStack

## Overall Project Structure
### File Structure

Your solution should look similar to this:

    MathStack/
    │
    ├── .github/
    │   └── workflows/
    │       └── ci.yml
    │
    ├── compose.ci.yml
    ├── db/
    │   ├── Dockerfile
    │   ├── entrypoint.sh
    │   └── init.sql
    │
    ├── MathAPI/
    │   ├── Controllers/
    │   │   └── HealthController.cs
    │   ├── Dockerfile
    │   ├── Program.cs
    │   └── ...
    │
    └── MathAPIClient/
        ├── Controllers/
        │   └── HealthController.cs
        ├── Dockerfile
        ├── Program.cs
        └── ...

### What we are building

The pipeline will follow this flow:

1. GitHub Actions checks out the repository.
2. GitHub Actions builds the DB, API and client images.
3. GitHub Actions pushes those images to private GHCR.
4. GitHub Actions pulls the same published images.
5. GitHub Actions starts the full stack using Docker Compose.
6. GitHub Actions waits for the DB, API and client to come up.
7. GitHub Actions stops the stack.

This means we are running the actual published images, not rebuilding them again in a separate test step.

## Step 1: Prepare your GitHub repository
### Add the required secrets

Open your GitHub repository and go to:

    Settings -> Secrets and variables -> Actions

Add these three repository secrets:

    SA_PASSWORD=your_strong_sql_password_here
    FirebaseMathApp=your_firebase_api_key_here
    MathAppJwtKey=your_long_random_jwt_secret_here

### Notes

* `SA_PASSWORD` is used by the SQL Server container.
* `FirebaseMathApp` is read by the API - please use the same on from local.
* `MathAppJwtKey` is used by the API for JWT authentication.
* You can reuse the existing secrets from your environment variables locally.
* These values should not be committed into the repository.

## Step 2: Package the database as its own image

At the moment, the database is initialised using a script, but it should also have its own Docker image so that the CI pipeline can publish and run it like the other services.

### Create `db/Dockerfile`

Create a file named `Dockerfile` inside the `db` folder.

    FROM mcr.microsoft.com/mssql/server:2022-latest

    USER root

    RUN apt-get update \
        && apt-get install -y curl apt-transport-https gnupg ca-certificates \
        && curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
        && curl https://packages.microsoft.com/config/ubuntu/22.04/prod.list > /etc/apt/sources.list.d/mssql-release.list \
        && apt-get update \
        && ACCEPT_EULA=Y apt-get install -y mssql-tools18 unixodbc-dev \
        && ln -s /opt/mssql-tools18/bin/sqlcmd /usr/local/bin/sqlcmd \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*

    COPY init.sql /docker-entrypoint-initdb.d/init.sql
    COPY entrypoint.sh /entrypoint.sh

    RUN chmod +x /entrypoint.sh

    ENV ACCEPT_EULA=Y

    ENTRYPOINT ["/entrypoint.sh"]

### Create `db/entrypoint.sh`

Create a file named `entrypoint.sh` inside the `db` folder.

    #!/bin/bash
    set -e

    /opt/mssql/bin/sqlservr &
    sql_pid=$!

    echo "Waiting for SQL Server to accept connections..."
    until sqlcmd -S localhost -U sa -P "$MSSQL_SA_PASSWORD" -C -Q "SELECT 1" >/dev/null 2>&1
    do
      sleep 2
    done

    echo "Running init.sql..."
    sqlcmd -S localhost -U sa -P "$MSSQL_SA_PASSWORD" -C -i /docker-entrypoint-initdb.d/init.sql

    echo "Database initialized."
    wait $sql_pid

#### What this does

* starts SQL Server inside the container
* waits until SQL Server is ready to accept connections
* runs `init.sql` automatically
* keeps the SQL Server process running as the main container process

## Step 3: Add a health endpoint to the API

The pipeline needs a reliable endpoint to check if the API is really running.

### Create `MathAPI/Controllers/HealthController.cs`

    using Microsoft.AspNetCore.Mvc;

    namespace MathAPI.Controllers
    {
        [ApiController]
        [Route("api/[controller]")]
        public class HealthController : ControllerBase
        {
            [HttpGet]
            public IActionResult Get()
            {
                return Ok(new
                {
                    status = "ok",
                    service = "MathAPI"
                });
            }
        }
    }

#### What this does

* adds a simple endpoint at `/api/health`
* returns a successful response when the API is running
* gives GitHub Actions a stable URL to check

## Step 4: Add a health endpoint to the client

The pipeline also needs a simple way to confirm that the client container has started correctly.

### Create `MathAPIClient/Controllers/HealthController.cs`

    using Microsoft.AspNetCore.Mvc;

    namespace MathAPIClient.Controllers
    {
        public class HealthController : Controller
        {
            [HttpGet("/health")]
            public IActionResult Index()
            {
                return Content("ok");
            }
        }
    }

#### What this does

* adds a simple endpoint at `/health`
* returns plain text when the client is running
* gives GitHub Actions a stable URL to check

## Step 5: Update `Program.cs` for container-friendly startup

When the containers run inside CI, we will access them over HTTP. The projects should not force HTTPS redirection in Development, otherwise the readiness checks can fail.

### Update `MathAPI/Program.cs`

Replace this:

    app.UseHttpsRedirection();

with this:

    if (!app.Environment.IsDevelopment())
    {
        app.UseHttpsRedirection();
    }

### Update `MathAPIClient/Program.cs`

Replace this:

    app.UseHttpsRedirection();

with this:

    if (!app.Environment.IsDevelopment())
    {
        app.UseHttpsRedirection();
    }

#### Why this matters

* in CI, the stack will be reached over plain HTTP
* skipping HTTPS redirection in Development avoids unnecessary redirects
* the containers still use HTTPS redirection outside Development if needed

## Step 6: Add a Compose file for CI

We will keep CI separate from local development.

The CI Compose file should pull the already-published images from GHCR rather than building them locally.

### Create `compose.ci.yml`

    services:
      math-db:
        image: ghcr.io/${GITHUB_OWNER_LC}/mathstack-db:${IMAGE_TAG}
        container_name: math-db
        environment:
          ACCEPT_EULA: "Y"
          MSSQL_SA_PASSWORD: "${SA_PASSWORD}"
        ports:
          - "1433:1433"

      math-api:
        image: ghcr.io/${GITHUB_OWNER_LC}/mathstack-api:${IMAGE_TAG}
        container_name: math-api
        depends_on:
          - math-db
        environment:
          ASPNETCORE_ENVIRONMENT: Development
          FirebaseMathApp: "${FirebaseMathApp}"
          MathAppJwtKey: "${MathAppJwtKey}"
          Math_DB: "Server=math-db,1433;Database=MathDB;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=True;"
        ports:
          - "8081:8080"

      math-client:
        image: ghcr.io/${GITHUB_OWNER_LC}/mathstack-client:${IMAGE_TAG}
        container_name: math-client
        depends_on:
          - math-api
        environment:
          ASPNETCORE_ENVIRONMENT: Development
          ApiSettings__BaseUrl: "http://math-api:8080"
        ports:
          - "8082:8080"
        volumes:
          - client-dataprotection-keys:/keys

    volumes:
      client-dataprotection-keys:

#### What this does

* pulls the DB, API and client images from GHCR
* injects the required environment variables from GitHub Actions secrets
* starts all three containers together in the same Docker network
* maps the API to port `8081` and the client to port `8082` on the runner

## Step 7: Add the GitHub Actions pipeline

We will use one workflow with two jobs:

* `publish` builds and pushes the images
* `run-stack` pulls those images and starts the containers

### Create `.github/workflows/ci.yml`

    name: CI

    on:
      push:
        branches: [main]
      workflow_dispatch:

    permissions:
      contents: read
      packages: write

    jobs:
      publish:
        runs-on: ubuntu-latest
        outputs:
          owner_lc: ${{ steps.vars.outputs.owner_lc }}

        steps:
          - name: Checkout
            uses: actions/checkout@v4

          - name: Lowercase owner
            id: vars
            run: echo "owner_lc=${GITHUB_REPOSITORY_OWNER,,}" >> "$GITHUB_OUTPUT"

          - name: Set up Docker Buildx
            uses: docker/setup-buildx-action@v3

          - name: Log in to GHCR
            uses: docker/login-action@v3
            with:
              registry: ghcr.io
              username: ${{ github.actor }}
              password: ${{ secrets.GITHUB_TOKEN }}

          - name: Build and push DB
            uses: docker/build-push-action@v6
            with:
              context: ./db
              file: ./db/Dockerfile
              push: true
              tags: ghcr.io/${{ steps.vars.outputs.owner_lc }}/mathstack-db:latest

          - name: Build and push API
            uses: docker/build-push-action@v6
            with:
              context: ./MathAPI
              file: ./MathAPI/Dockerfile
              push: true
              tags: ghcr.io/${{ steps.vars.outputs.owner_lc }}/mathstack-api:latest

          - name: Build and push Client
            uses: docker/build-push-action@v6
            with:
              context: ./MathAPIClient
              file: ./MathAPIClient/Dockerfile
              push: true
              tags: ghcr.io/${{ steps.vars.outputs.owner_lc }}/mathstack-client:latest

      run-stack:
        runs-on: ubuntu-latest
        needs: publish
        permissions:
          contents: read
          packages: read

        env:
          IMAGE_TAG: latest
          GITHUB_OWNER_LC: ${{ needs.publish.outputs.owner_lc }}
          SA_PASSWORD: ${{ secrets.SA_PASSWORD }}
          FirebaseMathApp: ${{ secrets.FirebaseMathApp }}
          MathAppJwtKey: ${{ secrets.MathAppJwtKey }}

        steps:
          - name: Checkout
            uses: actions/checkout@v4

          - name: Log in to GHCR
            uses: docker/login-action@v3
            with:
              registry: ghcr.io
              username: ${{ github.actor }}
              password: ${{ secrets.GITHUB_TOKEN }}

          - name: Pull images
            run: docker compose -f compose.ci.yml pull

          - name: Start stack
            run: docker compose -f compose.ci.yml up -d

          - name: Show running containers
            run: docker ps

          - name: Wait for database
            run: |
              for i in {1..36}; do
                if docker exec math-db /bin/bash -c "sqlcmd -S localhost -U sa -P '$SA_PASSWORD' -C -Q 'SELECT 1'" >/dev/null 2>&1; then
                  echo "Database is ready"
                  exit 0
                fi
                echo "Waiting for database..."
                sleep 5
              done
              echo "Database did not become ready"
              docker logs math-db || true
              exit 1

          - name: Wait for API
            run: |
              for i in {1..36}; do
                if curl -fsS http://localhost:8081/api/health >/dev/null 2>&1; then
                  echo "API is ready"
                  exit 0
                fi
                echo "Waiting for API..."
                sleep 5
              done
              echo "API did not respond"
              docker logs math-api || true
              exit 1

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

          - name: Show logs if failure
            if: failure()
            run: |
              echo "===== DB LOGS ====="
              docker logs math-db || true
              echo "===== API LOGS ====="
              docker logs math-api || true
              echo "===== CLIENT LOGS ====="
              docker logs math-client || true

          - name: Stop stack
            if: always()
            run: docker compose -f compose.ci.yml down -v

#### What this does

* builds all three images on every push to `main`
* pushes the images to private GHCR
* pulls the same images back from GHCR
* starts the full stack in GitHub Actions
* waits for the DB, API and client to become ready
* stops the stack when the job is done

## Step 8: Keep the GHCR images private

Once the pipeline runs for the first time, GitHub will create three packages in GHCR.

These should be:

* `mathstack-db`
* `mathstack-api`
* `mathstack-client`

Leave them as private packages.

### Check package access

For each package, make sure this repository has access to pull it in GitHub Actions.

If the images were created by this same repository, this will usually already be correct.

## Step 9: Commit and push your changes

Once all files are added, commit and push your changes.

    git add .
    git commit -m "Add GitHub Actions container pipeline for MathStack"
    git push

## Step 10: What should happen after you push

When you push to `main`, GitHub Actions should:

1. build the DB image
2. build the API image
3. build the client image
4. push all three images to private GHCR
5. pull those images back inside the workflow
6. start the containers using `compose.ci.yml`
7. wait for the DB, API and client to become ready
8. stop the containers

If all of that succeeds, the container pipeline is working.

## Summary

At this point, your pipeline should be able to:

* publish the DB, API and client images to private GHCR
* pull those images inside GitHub Actions
* run the full MathStack stack in the pipeline
* confirm that the containers start successfully

The next step after this would be to add API tests and frontend integration tests, but that will be done later.
