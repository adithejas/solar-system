# Deep Dive Line-by-Line & Action Guide: Solar System Tests Workflow

This document provides a comprehensive, line-by-line breakdown of every line and third-party action used in the `Solar System Tests` GitHub Actions workflow.

---

## 🧩 Part 1: GitHub Actions Used & Why They Are Used

Here is an inventory of every GitHub Action referenced in this pipeline and its purpose:

| Action Name                      | Version  | Purpose & Why It's Used                                                                                                                                                                                      |
| :------------------------------- | :------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`actions/checkout`**           | `v4`     | Fetches and clones your repository's source code into the virtual runner environment so subsequent build and test commands have access to your files.                                                        |
| **`actions/setup-node`**         | `v4`     | Downloads and installs the specified Node.js runtime version on the runner machine, configuring `node` and `npm` CLI utilities.                                                                              |
| **`actions/cache`**              | `v4`     | Caches dependencies (e.g., `node_modules`) between workflow runs based on a hash key of `package-lock.json`. This dramatically speeds up pipeline execution times by avoiding unnecessary package downloads. |
| **`actions/upload-artifact`**    | `v4`     | Uploads build outputs, test results, or coverage reports to GitHub storage, making them available for download in the GitHub UI after the run finishes.                                                      |
| **`docker/login-action`**        | `v3`     | Authenticates the runner against container registries (e.g., Docker Hub, GitHub Container Registry) using provided secret credentials.                                                                       |
| **`docker/setup-buildx-action`** | `v3`     | Sets up Docker Buildx, an extended Docker tool set that enables advanced features like multi-architecture builds and GitHub Actions cache integration (`type=gha`).                                          |
| **`docker/build-push-action`**   | `v5`     | Compiles a Dockerfile into an image, tags it, caches layers using Buildx, and loads or pushes the built image to registries.                                                                                 |
| **`cschleiden/replace-tokens`**  | `v1.3.0` | Finds placeholder tokens (such as `_{_VARIABLE_NAME_}_`) in configuration files (like Kubernetes YAML manifests) and replaces them with live environment values at runtime.                                  |

---

## 📜 Part 2: Line-by-Line Workflow Explanation

```yaml
name: Solar System Tests
```

- **Line 1:** `name: Solar System Tests`
  - **Why it's used:** Sets the title of the workflow as displayed in the GitHub Actions UI tab.

```yaml
on:
  workflow_dispatch:
  push:
    branches:
      - main
      - "feature/*"
```

- **Line 3:** `on:` — Keyword defining trigger events that cause this workflow to start running.
- **Line 4:** `workflow_dispatch:` — Enables a manual "Run workflow" trigger button inside the GitHub UI.
- **Line 5:** `push:` — Tells GitHub to listen for code push events.
- **Line 6–8:** `branches: [main, "feature/*"]` — Filters push events to run only when commits are pushed to the `main` branch or any branch starting with `feature/` (e.g., `feature/login-page`).

```yaml
env:
  MONGO_URI: ${{ secrets.MONGO_URI }}
  MONGO_USERNAME: ${{ vars.MONGO_USERNAME }}
  MONGO_PASSWORD: ${{ secrets.MONGO_PASSWORD }}
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

- **Line 9:** `env:` — Declares global environment variables available to every job and step in this workflow file.
- **Line 10:** `MONGO_URI: ${{ secrets.MONGO_URI }}` — Safely injects the MongoDB connection URL from repository secrets.
- **Line 11:** `MONGO_USERNAME: ${{ vars.MONGO_USERNAME }}` — Injecting the MongoDB username from repository variables.
- **Line 12:** `MONGO_PASSWORD: ${{ secrets.MONGO_PASSWORD }}` — Injects the sensitive MongoDB password securely from encrypted GitHub Secrets.
- **Line 13:** `DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}` — Injects your Docker Hub username.
- **Line 14:** `DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}` — Injects your Docker Hub access token or password securely.

---

### Job 1: `unit-tests`

```yaml
jobs:
  unit-tests:
    name: Run Unit Tests
    runs-on: ubuntu-latest
```

- **Line 15:** `jobs:` — Top-level key containing all independent tasks (jobs) in the workflow.
- **Line 16:** `unit-tests:` — Unique identifier/ID for the first job.
- **Line 17:** `name: Run Unit Tests` — Human-readable label shown in GitHub Actions logs.
- **Line 18:** `runs-on: ubuntu-latest` — Specifies that this job runs on an isolated Ubuntu Linux virtual machine hosted by GitHub.

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4
```

- **Line 19:** `steps:` — Ordered list of commands/actions executed sequentially within this job.
- **Line 20–21:** Executes `actions/checkout@v4` to copy repository files onto the virtual machine.

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: 20
```

- **Line 22–25:** Uses `actions/setup-node@v4` to install Node.js version 20 on the runner.

```yaml
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
```

- **Line 26–30:** Checks if a cached copy of `node_modules` exists matching the hash of `package-lock.json`. If found, restores it to skip downloading packages again.

```yaml
- name: Install dependencies
  run: npm ci
```

- **Line 31–32:** Executes `npm ci`, performing a clean, automated dependency installation using `package-lock.json`.

```yaml
- name: Run unit tests
  id: unit-tests
  run: npm test
```

- **Line 33–35:** Runs unit tests via `npm test`. The step ID `unit-tests` allows downstream steps to check the step outcome if needed.

```yaml
- name: Archive test results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-results-${{ github.actor }}
    path: test-results.xml
    retention-days: 1
```

- **Line 36–42:** Uses `actions/upload-artifact@v4` to save `test-results.xml`. The `if: always()` block ensures that test results are captured even if the `npm test` step fails.

---

### Job 2: `Code-coverage`

```yaml
Code-coverage:
  name: Code Coverage
  runs-on: ubuntu-latest
```

- **Line 43–45:** Job definition for code coverage analysis, running on GitHub-hosted Ubuntu.

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Setup Node.js
    uses: actions/setup-node@v4
    with:
      node-version: 20

  - name: Cache node modules
    uses: actions/cache@v4
    with:
      path: node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

  - name: Install dependencies
    run: npm ci
```

- **Line 46–60:** Clones code, installs Node.js v20, restores cached modules, and installs dependencies.

```yaml
- name: Run code coverage
  continue-on-error: true
  run: npm run coverage
```

- **Line 61–63:** Executes `npm run coverage`. `continue-on-error: true` prevents a code coverage check failure from stopping the whole deployment pipeline.

```yaml
- name: Upload coverage report
  uses: actions/upload-artifact@v4
  with:
    name: coverage-report-${{ github.actor }}
    path: coverage
    retention-days: 1
```

- **Line 64–69:** Saves the `coverage/` folder as a downloadable artifact retained for 1 day.

---

### Job 3: `Container-build`

```yaml
Container-build:
  name: Build and Push Docker Image
  needs: [unit-tests, Code-coverage]
  permissions:
    contents: read
    packages: write
  runs-on: ubuntu-latest
```

- **Line 70–72:** Defines container build job. `needs: [unit-tests, Code-coverage]` ensures this job will ONLY start if both test jobs succeed first.
- **Line 73–75:** Configures GITHUB_TOKEN permissions: grants read access to source repository and write access to GitHub Container Registry (GHCR).

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Log in to Docker Hub
    uses: docker/login-action@v3
    with:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}

  - name: Log in to GHCR
    uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
```

- **Line 76–90:** Clones the repo and uses `docker/login-action@v3` to log in to both Docker Hub and GitHub Container Registry (`ghcr.io`).

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
```

- **Line 91–92:** Initializes Docker Buildx builder instance for advanced caching capabilities.

```yaml
- name: Build Docker image
  id: build-image
  uses: docker/build-push-action@v5
  with:
    context: .
    load: true
    tags: ${{ secrets.DOCKER_USERNAME }}/solar-system:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

- **Line 93–102:** Builds the container image tagged with the commit SHA (`${{ github.sha }}`). Uses GitHub Actions cache (`type=gha`) to accelerate docker builds. `load: true` loads the image into local Docker engine for testing.

```yaml
- name: Test Docker image
  run: |
    docker run -d \
      --name solar-system-app \
      --publish 3000:3000 \
      -e MONGO_URI="${{ env.MONGO_URI }}" \
      -e MONGO_USERNAME="${{ vars.MONGO_USERNAME }}" \
      -e MONGO_PASSWORD="${{ secrets.MONGO_PASSWORD }}" \
      ${{ secrets.DOCKER_USERNAME }}/solar-system:${{ github.sha }}

    echo "Waiting for service to start..."
    sleep 5
    curl -s 127.0.0.1:3000/live | grep live
```

- **Line 103–117:** Starts the newly built container locally, passes runtime database environment variables, waits 5 seconds, and pings the `/live` health-check endpoint using `curl`.

```yaml
- name: Push Docker image to Docker Hub and GHCR
  run: |
    docker tag ${{ secrets.DOCKER_USERNAME }}/solar-system:${{ github.sha }} ghcr.io/${{ github.repository_owner }}/solar-system:${{ github.sha }}
    docker push ${{ secrets.DOCKER_USERNAME }}/solar-system:${{ github.sha }}
    docker push ghcr.io/${{ github.repository_owner }}/solar-system:${{ github.sha }}
```

- **Line 118–122:** Tags and pushes the verified container image to both Docker Hub and GHCR.

---

### Job 4 & 5: Development Deployment (`Dev-Deployment` & `Integration-tests-Dev`)

```yaml
Dev-Deployment:
  name: Deploy to Development Environment
  if: github.ref != 'refs/heads/main'
  needs: Container-build
  environment:
    name: development
    url: http://${{ steps.set-app-ingress-url.outputs.APP_INGRESS_HOST }}
  outputs:
    APP_INGRESS_HOST: ${{ steps.set-app-ingress-url.outputs.APP_INGRESS_HOST }}
  runs-on: self-hosted
```

- **Line 123–125:** Condition `if: github.ref != 'refs/heads/main'` restricts deployment to non-main branches (feature branches). Depends on `Container-build`.
- **Line 126–128:** Binds to GitHub Environment `development` and outputs deployment URL.
- **Line 129–130:** Exports `APP_INGRESS_HOST` as job output for subsequent integration test jobs.
- **Line 131:** `runs-on: self-hosted` — Executes on private infrastructure inside your network with access to the K8s cluster.

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Fetch ingress controller IP address
    run: |
      echo "INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')" >> $GITHUB_ENV

  - name: Replace tokens in deployment.yaml
    uses: cschleiden/replace-tokens@v1.3.0
    with:
      tokenPrefix: "_{_"
      tokenSuffix: "_}_"
      files: '["kubernetes/deployment.yaml"]'
    env:
      NAMESPACE: ${{ vars.NAMESPACE }}
      REPLICAS: ${{ vars.REPLICAS }}
      IMAGE: ${{ secrets.DOCKER_USERNAME }}/solar-system:${{ github.sha }}
      INGRESS_IP: ${{ env.INGRESS_IP }}
```

- **Line 132–151:** Queries Kubernetes for Ingress IP and replaces placeholders (like `_{_IMAGE_}_`) inside `kubernetes/deployment.yaml` with live build values.

```yaml
- name: Apply Deployment to K8s Cluster
  run: |
    kubectl apply -f kubernetes/deployment.yaml
    kubectl rollout status deployment/solar-system-deployment -n ${{ vars.NAMESPACE }} --timeout=120s

- name: Set app ingress URL
  id: set-app-ingress-url
  run: |
    echo "APP_INGRESS_HOST=$(kubectl -n ${{ vars.NAMESPACE }} get ingress -o jsonpath='{.items[0].spec.rules[0].host}')" >> $GITHUB_OUTPUT
```

- **Line 152–160:** Applies the manifest to Kubernetes, waits up to 120 seconds for rollout completion, and exposes the deployment URL host to `$GITHUB_OUTPUT`.

```yaml
Integration-tests-Dev:
  name: Run Integration Tests (Dev)
  needs: Dev-Deployment
  runs-on: ubuntu-latest
  steps:
    - name: Test Integration
      run: |
        curl -s -k http://${{ needs.Dev-Deployment.outputs.APP_INGRESS_HOST }}/live | grep -i live
```

- **Line 161–169:** Runs integration smoke tests against the live development server URL output from `Dev-Deployment`.

---

### Job 6 & 7: Production Deployment (`Prod-Deployment` & `Integration-tests-Prod`)

```yaml
Prod-Deployment:
  name: Deploy to Production Environment
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  needs: Container-build
  environment:
    name: prod
    url: http://${{ steps.set-app-ingress-url.outputs.APP_INGRESS_HOST }}
  outputs:
    APP_INGRESS_HOST: ${{ steps.set-app-ingress-url.outputs.APP_INGRESS_HOST }}
  runs-on: self-hosted
```

- **Line 170–178:** Production deployment job guarded by `if: github.ref == 'refs/heads/main' && github.event_name == 'push'`, ensuring only pushes to `main` trigger production releases.

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Fetch ingress controller IP address
    run: |
      echo "INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')" >> $GITHUB_ENV

  - name: Replace tokens in production.yaml
    uses: cschleiden/replace-tokens@v1.3.0
    with:
      tokenPrefix: "_{_"
      tokenSuffix: "_}_"
      files: '["kubernetes/production.yaml"]'
    env:
      NAMESPACE: ${{ vars.NAMESPACE }}
      REPLICAS: ${{ vars.REPLICAS }}
      IMAGE: ${{ secrets.DOCKER_USERNAME }}/solar-system:${{ github.sha }}
      INGRESS_IP: ${{ env.INGRESS_IP }}

  - name: Apply Production Configuration to K8s Cluster
    run: |
      kubectl apply -f kubernetes/production.yaml

  - name: Set app ingress URL
    id: set-app-ingress-url
    run: |
      echo "APP_INGRESS_HOST=$(kubectl -n ${{ vars.NAMESPACE }} get ingress -o jsonpath='{.items[0].spec.rules[0].host}')" >> $GITHUB_OUTPUT
```

- **Line 179–205:** Replaces configuration tokens in `kubernetes/production.yaml`, deploys to the Production K8s cluster, and exports the production ingress host.

```yaml
Integration-tests-Prod:
  name: Run Integration Tests (Prod)
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  needs: Prod-Deployment
  runs-on: ubuntu-latest
  steps:
    - name: Test Integration
      run: |
        curl -s -k http://${{ needs.Prod-Deployment.outputs.APP_INGRESS_HOST }}/live | grep -i live
```

- **Line 206–215:** Executes post-deployment integration smoke tests against the live production environment endpoint.
