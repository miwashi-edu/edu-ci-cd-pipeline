# edu-ci-cd-pipeline

## Instructions

```bash
cd ^
cd ws
cd git clone https://github.com/miwashi-edu/edu-ci-cd-pipeline
cd edu-ci-cd-pipeline
mkdir -p .github/workflows
```

## Orchestration 

```bash

# ---------------------------------------------------------------- orchestrator
cat > .github/workflows/ci-cd.yaml <<'EOF'
# Dummy CI/CD pipeline – orchestrates one reusable workflow per stage.
name: ci-cd-pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  bootstrap:
    uses: ./.github/workflows/00-bootstrap.yaml

  lint:
    needs: bootstrap
    uses: ./.github/workflows/01-lint.yaml

  unit-test:
    needs: bootstrap
    uses: ./.github/workflows/02-unit-test.yaml

  security-scan:
    needs: bootstrap
    uses: ./.github/workflows/03-security-scan.yaml

  integration-test:
    needs: [lint, unit-test, security-scan]
    uses: ./.github/workflows/04-integration-test.yaml

  package:
    needs: integration-test
    uses: ./.github/workflows/05-package.yaml
    with:
      app_name: demo-service
      version: 1.0.${{ github.run_number }}
      registry: registry.example.com

  deploy-staging:
    needs: package
    uses: ./.github/workflows/06-deploy-staging.yaml
    with:
      app_name: demo-service
      version: 1.0.${{ github.run_number }}

  approval:
    needs: deploy-staging
    uses: ./.github/workflows/07-approval.yaml

  deploy-production:
    needs: approval
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/08-deploy-production.yaml
    with:
      app_name: demo-service
      version: 1.0.${{ github.run_number }}

  notify:
    needs: deploy-production
    if: always()
    uses: ./.github/workflows/09-notify.yaml
    with:
      version: 1.0.${{ github.run_number }}
EOF
```

## Bootstrap step

```bash
# ---------------------------------------------------------------- 0
cat > .github/workflows/00-bootstrap.yaml <<'EOF'
name: "0. Bootstrap – gradle build (tooling + app)"
on:
  workflow_call:

jobs:
  bootstrap:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        run: echo "[dummy] git checkout ${GITHUB_SHA:-HEAD}"
      - name: Set up JDK
        run: echo "[dummy] installing JDK 21"
      - name: Gradle build (app + all pipeline tools)
        run: |
          echo "[dummy] ./gradlew clean build"
          echo "[dummy]   :app:build            -> build/libs/demo-service.jar"
          echo "[dummy]   :tools:linter:build   -> build/tools/linter"
          echo "[dummy]   :tools:scanner:build  -> build/tools/scanner"
          echo "[dummy]   :tools:packager:build -> build/tools/packager"
          echo "[dummy]   :tools:deployer:build -> build/tools/deployer"
          echo "[dummy]   :tools:smoke:build    -> build/tools/smoke"
      - name: Upload build artifacts
        run: echo "[dummy] uploading build/ as artifact 'build-output'"
EOF
```

## Lint step

```bash
# ---------------------------------------------------------------- 1
cat > .github/workflows/01-lint.yaml <<'EOF'
name: "1. Static analysis / lint"
on:
  workflow_call:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifacts
        run: echo "[dummy] downloading artifact 'build-output'"
      - name: Run linter
        run: echo "[dummy] build/tools/linter --src src/ --fail-on error"
EOF

```

## Unit test step

```bash
# ---------------------------------------------------------------- 2
cat > .github/workflows/02-unit-test.yaml <<'EOF'
name: "2. Unit tests"
on:
  workflow_call:

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifacts
        run: echo "[dummy] downloading artifact 'build-output'"
      - name: Run unit tests
        run: echo "[dummy] ./gradlew test -> 128 passed, 0 failed"
      - name: Publish coverage
        run: echo "[dummy] coverage 87% (threshold 80%) ✔"
EOF

```

## Security test step

```bash
# ---------------------------------------------------------------- 3
cat > .github/workflows/03-security-scan.yaml <<'EOF'
name: "3. Security scan (SAST + dependencies)"
on:
  workflow_call:

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifacts
        run: echo "[dummy] downloading artifact 'build-output'"
      - name: Dependency scan
        run: echo "[dummy] build/tools/scanner deps --severity high -> 0 findings"
      - name: SAST scan
        run: echo "[dummy] build/tools/scanner sast src/ -> 0 findings"
EOF

```

## Integration tetst step

```bash
# ---------------------------------------------------------------- 4
cat > .github/workflows/04-integration-test.yaml <<'EOF'
name: "4. Integration tests"
on:
  workflow_call:

jobs:
  integration-test:
    runs-on: ubuntu-latest
    steps:
      - name: Start test dependencies
        run: echo "[dummy] starting postgres + redis containers"
      - name: Run integration tests
        run: echo "[dummy] ./gradlew integrationTest -> 42 passed"
      - name: Tear down
        run: echo "[dummy] stopping test containers"
EOF

```

## Publish Step

```bash
# ---------------------------------------------------------------- 5
cat > .github/workflows/05-package.yaml <<'EOF'
name: "5. Package & publish image"
on:
  workflow_call:
    inputs:
      app_name: { type: string, required: true }
      version:  { type: string, required: true }
      registry: { type: string, required: true }

jobs:
  package:
    runs-on: ubuntu-latest
    env:
      IMAGE: ${{ inputs.registry }}/${{ inputs.app_name }}:${{ inputs.version }}
    steps:
      - name: Build container image
        run: echo "[dummy] build/tools/packager image --tag $IMAGE"
      - name: Sign image
        run: echo "[dummy] signing $IMAGE"
      - name: Push image
        run: echo "[dummy] pushing $IMAGE"
EOF

```

## Staging step

```bash
# ---------------------------------------------------------------- 6
cat > .github/workflows/06-deploy-staging.yaml <<'EOF'
name: "6. Deploy to staging"
on:
  workflow_call:
    inputs:
      app_name: { type: string, required: true }
      version:  { type: string, required: true }

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy
        run: echo "[dummy] build/tools/deployer --env staging --image ${{ inputs.app_name }}:${{ inputs.version }}"
      - name: Smoke test
        run: echo "[dummy] build/tools/smoke https://staging.example.com/health -> 200 OK"
EOF

```

## Promote to prod step

```bash
# ---------------------------------------------------------------- 7
cat > .github/workflows/07-approval.yaml <<'EOF'
name: "7. Manual approval gate"
on:
  workflow_call:

jobs:
  approval:
    runs-on: ubuntu-latest
    environment: production-approval
    steps:
      - name: Await approval
        run: echo "[dummy] waiting for release manager approval... approved ✔"
EOF

```

## Publish step

```bash
# ---------------------------------------------------------------- 8
cat > .github/workflows/08-deploy-production.yaml <<'EOF'
name: "8. Deploy to production"
on:
  workflow_call:
    inputs:
      app_name: { type: string, required: true }
      version:  { type: string, required: true }

jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Canary deploy (10%)
        run: echo "[dummy] build/tools/deployer --env prod --image ${{ inputs.app_name }}:${{ inputs.version }} --strategy canary --weight 10"
      - name: Verify canary
        run: echo "[dummy] error rate 0.01%, p99 latency 120ms ✔"
      - name: Full rollout
        run: echo "[dummy] build/tools/deployer --env prod --strategy canary --weight 100"
      - name: Post-deploy smoke test
        run: echo "[dummy] build/tools/smoke https://www.example.com/health -> 200 OK"
EOF

```

## Notify step

```bash
# ---------------------------------------------------------------- 9
cat > .github/workflows/09-notify.yaml <<'EOF'
name: "9. Notify"
on:
  workflow_call:
    inputs:
      version: { type: string, required: true }

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Send notification
        run: echo "[dummy] posting release ${{ inputs.version }} status to #deployments"
EOF
```
echo "Created:"; ls -1 .github/workflows
```
