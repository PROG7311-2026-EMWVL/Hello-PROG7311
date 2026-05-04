# Building Pipeline Resilience in GitHub Actions

## Objective


In this guide, you will update the MathStack CI pipeline so that it can still run the Docker Compose stack and Newman API tests when the image build stage needs to be skipped.

This is useful when an external service causes the image build to fail. For example, a Dockerfile may need to download packages using `apt-get`. If the Ubuntu package servers are unavailable or timing out, the image build can fail even when there is nothing wrong with your application code.

Read more on what happened that prompted this addition:
- https://www.tomshardware.com/tech-industry/cyber-security/canonical-under-sustained-ddos-attack-as-ubuntu-26-releases-iranian-group-313-team-claims-responsibility?utm_source=chatgpt.com
- https://www.reddit.com/r/Ubuntu/comments/1t0mmop/active_incident_massive_ddos_attack_on_ubuntu/

## What We Are Changing

We are making four small changes:

```text
1. Add a manual skip_build input
2. Skip the publish job when skip_build is true
3. Allow run-stack to continue when publish is skipped
4. Recreate GITHUB_OWNER_LC inside run-stack
```

## Step 1: Add the `skip_build` Input

Find the top of the workflow file.

The original workflow had:

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

Update it to:

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      skip_build:
        description: "Skip image build and use existing GHCR images"
        required: true
        default: false
        type: boolean
```

### Explanation

This adds a manual option when running the workflow from GitHub Actions.

When the workflow is run manually, you can now choose:

```text
skip_build: false
```

or:

```text
skip_build: true
```

The default is `false`, so normal manual runs still build the images unless you choose to skip them.

## Step 2: Add a Condition to the `publish` Job

Find the start of the `publish` job:

```yaml
publish:
  runs-on: ubuntu-latest
```

Update it to:

```yaml
publish:
  runs-on: ubuntu-latest

  if: ${{ github.event_name != 'workflow_dispatch' || inputs.skip_build == false }}
```

### Explanation

The `publish` job builds and pushes the Docker images.

This condition means:

```text
Push to main                 → publish runs
Manual run, skip_build false → publish runs
Manual run, skip_build true  → publish is skipped
```

So the normal CI process is unchanged for pushes to `main`.

## Step 3: Allow `run-stack` to Continue When `publish` Is Skipped

Find the start of the `run-stack` job:

```yaml
run-stack:
  runs-on: ubuntu-latest
  needs: publish
```

Update it to:

```yaml
run-stack:
  runs-on: ubuntu-latest
  needs: publish

  if: ${{ always() && (needs.publish.result == 'success' || needs.publish.result == 'skipped') }}
```

### Explanation

The `run-stack` job still depends on `publish`, but now it can continue if `publish` was intentionally skipped.

It will run when:

```text
publish succeeded
publish was skipped
```

It will not continue if `publish` actually failed.

## Step 4: Remove `GITHUB_OWNER_LC` from the Job-Level Environment

In the `run-stack` job, find the `env` section.

Remove this line:

```yaml
GITHUB_OWNER_LC: ${{ needs.publish.outputs.owner_lc }}
```

The section should now look like this:

```yaml
env:
  IMAGE_TAG: latest
  SA_PASSWORD: ${{ secrets.SA_PASSWORD }}
  FirebaseMathApp: ${{ secrets.FirebaseMathApp }}
  MathAppJwtKey: ${{ secrets.MathAppJwtKey }}
```

### Explanation

When `publish` is skipped, this output may not exist:

```yaml
${{ needs.publish.outputs.owner_lc }}
```

So we should not depend on it in the `run-stack` environment.

## Step 5: Add `Lowercase owner` Inside `run-stack`

Inside the `run-stack` steps, find this section:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4

  - name: Log in to GHCR
    uses: docker/login-action@v3
```

Add this new step after `Checkout` and before `Log in to GHCR`:

```yaml
  - name: Lowercase owner
    run: echo "GITHUB_OWNER_LC=${GITHUB_REPOSITORY_OWNER,,}" >> "$GITHUB_ENV"
```

The section should now look like this:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4

  - name: Lowercase owner
    run: echo "GITHUB_OWNER_LC=${GITHUB_REPOSITORY_OWNER,,}" >> "$GITHUB_ENV"

  - name: Log in to GHCR
    uses: docker/login-action@v3
```

### Explanation

This recreates `GITHUB_OWNER_LC` inside the `run-stack` job.

That value is needed by `compose.ci.yml` when pulling images from GHCR.

For example:

```text
ghcr.io/prog7311-2026-emwvl/mathstack-api:latest
```


## Step 6: How to Use the New Option

Go to:

```text
GitHub → Actions → CI → Run workflow
```

To run the pipeline normally, leave:

```text
skip_build: false
```

To skip the image build, set:

```text
skip_build: true
```

Then run the workflow.

When `skip_build` is `true`, the workflow skips the image build and uses the existing `latest` images from GHCR.

## Important Note

The skip-build option only works if these images already exist in your GHCR:

```text
ghcr.io/.../mathstack-db:latest
ghcr.io/.../mathstack-api:latest
ghcr.io/.../mathstack-client:latest
```

If the images do not exist, the `Pull images` step will fail.

## Summary

This update adds a small recovery path to the pipeline.

Normal runs still build and push images.

Manual runs can now skip the image build and use existing GHCR images by setting:

```
skip_build: true
```

This is useful when external build dependencies are temporarily unavailable, but previously built images are already available.