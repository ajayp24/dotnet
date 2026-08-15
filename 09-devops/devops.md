  # DevOps — Lead .NET Engineer Interview Prep

> **Audience:** Lead .NET Software Engineer interview preparation  
> **Topics:** Git strategies, CI/CD pipelines, quality gates, deployment patterns, IaC

---

## Table of Contents

1. [Git Branch Strategies](#1-git-branch-strategies)
2. [CI/CD with GitHub Actions](#2-cicd-with-github-actions)
3. [CI/CD with Azure DevOps](#3-cicd-with-azure-devops)
4. [Harness CI/CD](#4-harness-cicd)
5. [SonarQube & Code Quality](#5-sonarqube--code-quality)
6. [Code Coverage](#6-code-coverage)
7. [Artifact Repositories](#7-artifact-repositories)
8. [Feature Flags](#8-feature-flags)
9. [Blue/Green Deployment](#9-bluegreen-deployment)
10. [Canary Deployment](#10-canary-deployment)
11. [Infrastructure as Code](#11-infrastructure-as-code)
12. [Interview Q&A Cheat Sheet](#12-interview-qa-cheat-sheet)

---

## 1. Git Branch Strategies

### GitFlow

Long-lived branches for larger, scheduled-release teams.

```
main         ← production-ready, tagged releases
develop      ← integration branch, nightly builds
feature/*    ← feature/JIRA-123-user-login (off develop)
release/*    ← release/1.4.0 — stabilization, bug fixes only
hotfix/*     ← hotfix/1.3.1 — off main, merged back to both main + develop
```

**Workflow:**
1. Cut `feature/JIRA-456` off `develop`
2. Open PR → `develop`
3. When ready for release: cut `release/2.1.0` off `develop`
4. QA + fixes on release branch
5. Merge `release/2.1.0` → `main` (tag `v2.1.0`) AND back → `develop`
6. Production bug? Cut `hotfix/2.1.1` off `main`, fix, merge both ways

**Pros:** Clear separation, good for regulated industries with release gates  
**Cons:** Complex, long-lived branches cause merge hell, slow feedback loops

---

### GitHub Flow

Simpler, web-style continuous delivery.

```
main         ← always deployable, protected
feature/*    ← short-lived (1–3 days), off main, PR → main → deploy
```

**Workflow:**
1. `git checkout -b feature/add-payment-api`
2. Small commits, push, open PR
3. CI runs, code review
4. Merge to `main` → auto-deploy to production

**Pros:** Simple, fast, encourages small PRs  
**Cons:** Requires feature flags for incomplete large features, harder for multi-version products

---

### Trunk-Based Development (TBD)

Highest maturity. Teams commit to `main` (trunk) multiple times per day.

```
main         ← the only long-lived branch
feature/*    ← optional, max 1–2 days old, deleted immediately after merge
```

**Key practices:**
- Feature flags hide incomplete work
- CI runs on every commit — must pass in < 10 min
- Pair programming / mob programming reduces long branches
- Strict code review culture

**Pros:** Fastest feedback, no merge conflicts, true continuous integration  
**Cons:** Requires high test coverage, discipline, and feature flag infrastructure

---

### Comparison Table

| Factor | GitFlow | GitHub Flow | Trunk-Based |
|--------|---------|-------------|-------------|
| Release cadence | Scheduled (biweekly/monthly) | Continuous | Multiple/day |
| Branch lifetime | Weeks–months | Days | Hours–1 day |
| Complexity | High | Low | Low–Medium |
| Merge conflicts | High risk | Medium | Low |
| Feature flags needed | Optional | Sometimes | Always |
| Best for | Enterprise, regulated | SaaS startups | High-maturity teams |
| Rollback strategy | `git revert` on release branch | Revert merge commit | Feature flag off |

---

### Interview Q: "What branching strategy does your team use and why?"

> **Strong answer:**  
> "We use GitHub Flow with elements of trunk-based development. Our branches live no longer than 2–3 days. We use feature flags via LaunchDarkly to deploy incomplete features safely to production without releasing them to users. This gives us continuous integration without the overhead of GitFlow's long-lived release branches. We protect `main` with required PR reviews and passing CI — nobody force-pushes. For hotfixes we still cut a short branch off main, fix it, PR, merge, and tag — identical process, just labeled differently."

**Pitfall to avoid:** Don't say "we just commit to main" without mentioning feature flags — the interviewer will press on incomplete features.

---

## 2. CI/CD with GitHub Actions

### Trigger Types

```yaml
on:
  push:
    branches: [main, develop]
    tags: ['v*.*.*']          # version tags
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
  schedule:
    - cron: '0 2 * * 1'      # Monday 2am UTC — weekly full test suite
  workflow_dispatch:          # manual trigger with optional inputs
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
```

---

### Complete GitHub Actions Pipeline — .NET → ECR → ECS

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp-api
  ECS_SERVICE: myapp-api-service
  ECS_CLUSTER: myapp-cluster
  CONTAINER_NAME: myapp-api
  IMAGE_TAG: ${{ github.sha }}

jobs:
  # ─────────────────────────────────────────────────────
  # JOB 1: Build, Test, Coverage, SonarQube
  # ─────────────────────────────────────────────────────
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write  # for PR comments

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0    # SonarQube needs full history

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      # Cache NuGet packages — key on lock file hash
      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ runner.os }}-${{ hashFiles('**/*.csproj', '**/packages.lock.json') }}
          restore-keys: |
            nuget-${{ runner.os }}-

      - name: Restore dependencies
        run: dotnet restore MyApp.sln --locked-mode

      - name: Build
        run: |
          dotnet build MyApp.sln \
            --configuration Release \
            --no-restore \
            /p:TreatWarningsAsErrors=true

      # Install Coverlet + ReportGenerator globally
      - name: Install coverage tools
        run: |
          dotnet tool install --global dotnet-reportgenerator-globaltool
          dotnet tool install --global coverlet.console

      - name: Run tests with code coverage
        run: |
          dotnet test MyApp.sln \
            --configuration Release \
            --no-build \
            --verbosity normal \
            --collect:"XPlat Code Coverage" \
            --results-directory ./coverage \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Generate coverage report
        run: |
          reportgenerator \
            -reports:"coverage/**/coverage.opencover.xml" \
            -targetdir:"coverage/report" \
            -reporttypes:"HtmlInline_AzurePipelines;Cobertura;TextSummary"
          cat coverage/report/Summary.txt

      - name: Upload coverage artifacts
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/report/

      # SonarQube scan
      - name: Set up JDK 17 (required by SonarScanner)
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Install dotnet-sonarscanner
        run: dotnet tool install --global dotnet-sonarscanner

      - name: SonarQube Begin
        run: |
          dotnet sonarscanner begin \
            /k:"myapp-api" \
            /d:sonar.host.url="${{ secrets.SONAR_HOST_URL }}" \
            /d:sonar.login="${{ secrets.SONAR_TOKEN }}" \
            /d:sonar.cs.opencover.reportsPaths="coverage/**/coverage.opencover.xml" \
            /d:sonar.coverage.exclusions="**/*Tests*/**,**/Migrations/**,**/Program.cs" \
            /d:sonar.qualitygate.wait=true

      - name: Build for SonarQube
        run: dotnet build MyApp.sln --configuration Release --no-restore

      - name: SonarQube End
        run: dotnet sonarscanner end /d:sonar.login="${{ secrets.SONAR_TOKEN }}"

      # Post coverage summary to PR
      - name: Post coverage comment to PR
        if: github.event_name == 'pull_request'
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          path: coverage/report/Summary.txt

  # ─────────────────────────────────────────────────────
  # JOB 2: Docker Build & Push to ECR
  # ─────────────────────────────────────────────────────
  docker:
    name: Docker Build & Push
    needs: build-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    outputs:
      image-uri: ${{ steps.build-push.outputs.image-uri }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx (multi-platform + layer cache)
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./src/MyApp.Api/Dockerfile
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
          # BuildKit layer cache in ECR for fast rebuilds
          cache-from: type=registry,ref=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:buildcache
          cache-to: type=registry,ref=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:buildcache,mode=max
          build-args: |
            BUILD_VERSION=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}

      - name: Output image URI
        run: |
          IMAGE_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}"
          echo "image-uri=$IMAGE_URI" >> $GITHUB_OUTPUT
          echo "Built image: $IMAGE_URI"

  # ─────────────────────────────────────────────────────
  # JOB 3: Deploy to ECS
  # ─────────────────────────────────────────────────────
  deploy:
    name: Deploy to ECS
    needs: docker
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.myapp.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Download current ECS task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition myapp-api-task \
            --query taskDefinition \
            > task-definition.json

      - name: Update task definition with new image
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.docker.outputs.image-uri }}
          environment-variables: |
            APP_VERSION=${{ github.sha }}

      - name: Deploy to ECS (rolling update)
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          wait-for-minutes: 10

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "Deployment successful! :rocket: `${{ env.ECR_REPOSITORY }}` @ `${{ github.sha }}`"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": ":x: Deployment FAILED for `${{ env.ECR_REPOSITORY }}` @ `${{ github.sha }}`"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### Secrets Management in GitHub Actions

```yaml
# Repository secrets (Settings → Secrets → Actions):
# AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
# SONAR_TOKEN, SONAR_HOST_URL
# SLACK_WEBHOOK_URL

# Reference in YAML:
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}

# Use OIDC (preferred — no long-lived keys):
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1
```

> **Tip:** Never hardcode secrets. Use OIDC federation instead of static AWS keys — the token is short-lived and requires no secret rotation.

---

### NuGet Cache Strategy

```yaml
- name: Cache NuGet
  uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    # Cache busts when any .csproj or lock file changes
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj', '**/packages.lock.json') }}
    restore-keys: |
      ${{ runner.os }}-nuget-

- name: Restore
  # --locked-mode enforces packages.lock.json — prevents supply chain drift
  run: dotnet restore --locked-mode
```

Generate the lock file once: `dotnet restore --use-lock-file`

---

## 3. CI/CD with Azure DevOps

### Pipeline: YAML vs Classic

| | YAML Pipeline | Classic (GUI) Pipeline |
|-|--------------|----------------------|
| Version controlled | Yes — in repo | No — stored in ADO |
| Code review | PR on pipeline changes | Not possible |
| Templates | Yes (`extends`, `template`) | Limited |
| Recommended | Yes (2024+) | Legacy only |

---

### Complete Azure DevOps YAML Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    exclude:
      - '**/*.md'
      - 'docs/**'

pr:
  branches:
    include:
      - main

variables:
  - group: myapp-secrets          # Variable group from Library
  - name: dotnetVersion
    value: '8.0.x'
  - name: buildConfiguration
    value: 'Release'
  - name: imageRepository
    value: 'myapp-api'

pool:
  vmImage: 'ubuntu-latest'

stages:
  # ── STAGE 1: Build & Test ───────────────────────────
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildAndTest
        displayName: 'Restore → Build → Test → Sonar'
        steps:
          - task: UseDotNet@2
            displayName: 'Use .NET $(dotnetVersion)'
            inputs:
              version: $(dotnetVersion)
              includePreviewVersions: false

          - task: Cache@2
            displayName: 'Cache NuGet packages'
            inputs:
              key: 'nuget | "$(Agent.OS)" | **/*.csproj'
              restoreKeys: |
                nuget | "$(Agent.OS)"
              path: $(NUGET_PACKAGES)

          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: 'restore'
              projects: '**/*.sln'
              feedsToUse: 'config'
              nugetConfigPath: 'nuget.config'

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              projects: '**/*.sln'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: SonarQubePrepare@5
            displayName: 'SonarQube: Prepare'
            inputs:
              SonarQube: 'SonarQube-ServiceConnection'
              scannerMode: 'MSBuild'
              projectKey: 'myapp-api'
              projectName: 'MyApp API'
              extraProperties: |
                sonar.cs.opencover.reportsPaths=$(Agent.TempDirectory)/**/coverage.opencover.xml
                sonar.coverage.exclusions=**/Migrations/**,**/*Tests*/**

          - task: DotNetCoreCLI@2
            displayName: 'Test with Coverage'
            inputs:
              command: 'test'
              projects: '**/*Tests.csproj'
              arguments: >
                --configuration $(buildConfiguration)
                --no-build
                --collect:"XPlat Code Coverage"
                --results-directory $(Agent.TempDirectory)/coverage
              publishTestResults: true

          - task: SonarQubeAnalyze@5
            displayName: 'SonarQube: Analyze'

          - task: SonarQubePublish@5
            displayName: 'SonarQube: Publish Quality Gate'
            inputs:
              pollingTimeoutSec: '300'

          - task: PublishCodeCoverageResults@2
            displayName: 'Publish coverage to ADO'
            inputs:
              summaryFileLocation: '$(Agent.TempDirectory)/coverage/**/*.xml'

          - task: DotNetCoreCLI@2
            displayName: 'Publish artifact'
            inputs:
              command: 'publish'
              publishWebProjects: true
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
              zipAfterPublish: true

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'app-drop'

  # ── STAGE 2: Docker Build & Push ────────────────────
  - stage: DockerBuildPush
    displayName: 'Docker: Build & Push'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: Docker
        steps:
          - task: Docker@2
            displayName: 'Build and push to ACR'
            inputs:
              command: 'buildAndPush'
              repository: $(imageRepository)
              dockerfile: 'src/MyApp.Api/Dockerfile'
              containerRegistry: 'ACR-ServiceConnection'
              tags: |
                $(Build.BuildId)
                latest

  # ── STAGE 3: Deploy to Dev ───────────────────────────
  - stage: DeployDev
    displayName: 'Deploy → Dev'
    dependsOn: DockerBuildPush
    variables:
      - group: myapp-dev-config
    jobs:
      - deployment: DeployToDev
        displayName: 'Deploy to Dev AKS'
        environment: 'dev'          # ADO Environment (tracks deployments)
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: 'Deploy to AKS (dev)'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'AKS-Dev-ServiceConnection'
                    namespace: 'myapp-dev'
                    manifests: 'k8s/deployment.yaml'
                    containers: 'myacr.azurecr.io/myapp-api:$(Build.BuildId)'

  # ── STAGE 4: Deploy to Staging (with approval) ───────
  - stage: DeployStaging
    displayName: 'Deploy → Staging'
    dependsOn: DeployDev
    jobs:
      - deployment: DeployToStaging
        displayName: 'Deploy to Staging'
        environment: 'staging'      # Approval gate configured on this ADO Environment
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureRmWebAppDeployment@4
                  inputs:
                    ConnectionType: 'AzureRM'
                    azureSubscription: 'Azure-ServiceConnection'
                    appType: 'webApp'
                    WebAppName: 'myapp-staging'
                    packageForLinux: '$(Pipeline.Workspace)/app-drop/*.zip'

  # ── STAGE 5: Deploy to Production (with approval) ────
  - stage: DeployProd
    displayName: 'Deploy → Production'
    dependsOn: DeployStaging
    jobs:
      - deployment: DeployToProd
        displayName: 'Deploy to Production'
        environment: 'production'   # Required approvers set in ADO Environment
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureRmWebAppDeployment@4
                  inputs:
                    ConnectionType: 'AzureRM'
                    azureSubscription: 'Azure-ServiceConnection'
                    appType: 'webApp'
                    WebAppName: 'myapp-prod'
                    packageForLinux: '$(Pipeline.Workspace)/app-drop/*.zip'
```

### Variable Groups & Secrets

```yaml
# Library → Variable Groups → myapp-secrets
# Variables: ConnectionStrings__DefaultConnection, JwtSecret, ApiKey

# Reference in pipeline:
variables:
  - group: myapp-secrets   # maps all group variables to pipeline
  - name: myLocalVar
    value: 'something'

# In steps, use as: $(JwtSecret)
# Mark sensitive variables as "Secret" in ADO — they appear as ***
```

### Approvals and Gates

In Azure DevOps, approvals are configured on **Environments** (not in YAML directly):

- ADO → Environments → `production` → Approvals & Checks
- Add "Approvals" check: require 1+ named approvers
- Add "Business Hours" gate: only deploy Mon–Fri 9am–5pm
- Add "Query Work Items" gate: no open P1 bugs before deploy

---

## 4. Harness CI/CD

### What is Harness?

Enterprise-grade CI/CD platform. Key differentiators:
- **AI-driven** deployment verification (ML on metrics, auto-rollback)
- Built-in support for all major deployment strategies
- Governance: approval workflows, OPA policies
- Feature flags as a first-class feature
- DORA metrics dashboard out-of-the-box

### Harness Pipeline Concepts

```
Account
└── Organization
    └── Project
        └── Pipeline
            ├── Stage: CI (build/test)
            ├── Stage: Deploy Dev
            │   ├── Step: HTTP step (smoke test)
            │   └── Step: Shell script
            ├── Stage: Approval
            │   └── Step: Manual Approval
            └── Stage: Deploy Prod
                └── Step: Rolling / Blue-Green / Canary
```

### Deployment Strategies in Harness

| Strategy | How | Rollback |
|----------|-----|----------|
| Rolling | Replace instances one batch at a time | Redeploy old version |
| Blue/Green | Shift ALB traffic from old to new env | Flip ALB back to blue |
| Canary | Route X% traffic, verify, promote | Shift 100% back to stable |

### Feature Flags in Harness

```
Project → Feature Flags → New Flag
  Type: Boolean / Multivariate
  Targets: environments, segments, individual users
  % Rollout: 0% → 10% → 50% → 100%
  Kill switch: disable instantly without deploy
```

---

## 5. SonarQube & Code Quality

### What SonarQube Measures

| Category | Example |
|----------|---------|
| **Bugs** | Null dereference, unreachable code |
| **Vulnerabilities** | SQL injection, hardcoded credentials |
| **Code Smells** | Long methods, dead code, naming violations |
| **Duplications** | Copy-pasted blocks > N lines |
| **Coverage** | % of lines/branches hit by tests |
| **Security Hotspots** | Areas needing manual review |

### Quality Gate

A quality gate is a pass/fail threshold applied to every analysis:

```
Default Quality Gate "Sonar way":
  ✓ Coverage on new code ≥ 80%
  ✓ Duplicated lines on new code ≤ 3%
  ✓ Maintainability rating on new code = A
  ✓ Reliability rating on new code = A
  ✓ Security rating on new code = A
```

Pipeline fails if quality gate is `ERROR`. The `sonar.qualitygate.wait=true` flag blocks the pipeline until the gate is evaluated.

### .NET Integration (dotnet-sonarscanner)

```bash
# Step 1: Begin (before build)
dotnet sonarscanner begin \
  /k:"project-key" \
  /d:sonar.host.url="https://sonar.company.com" \
  /d:sonar.login="$SONAR_TOKEN" \
  /d:sonar.cs.opencover.reportsPaths="**/coverage.opencover.xml" \
  /d:sonar.exclusions="**/Migrations/**"

# Step 2: Build (sonarscanner instruments the build)
dotnet build MyApp.sln

# Step 3: End (upload analysis to SonarQube server)
dotnet sonarscanner end /d:sonar.login="$SONAR_TOKEN"
```

### Technical Debt

SonarQube estimates remediation time per issue. Debt ratio = (remediation time) / (development time to write the code). A rating of A = < 5% debt ratio.

---

## 6. Code Coverage

### Running Coverage with Coverlet

```bash
dotnet test MyApp.sln \
  --collect:"XPlat Code Coverage" \
  --results-directory ./coverage \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

# Coverlet also works as a CLI tool:
coverlet ./bin/Debug/net8.0/MyApp.Tests.dll \
  --target "dotnet" \
  --targetargs "test --no-build" \
  --format opencover \
  --output ./coverage/coverage.opencover.xml
```

### Generate HTML Report

```bash
dotnet tool install --global dotnet-reportgenerator-globaltool

reportgenerator \
  -reports:"coverage/**/coverage.opencover.xml" \
  -targetdir:"coverage/html" \
  -reporttypes:"Html;TextSummary;Badges;Cobertura"

# Summary output:
# Line coverage: 84.6% (1234 / 1459)
# Branch coverage: 71.2% (412 / 579)
```

### Coverage Formats

| Format | Used by |
|--------|---------|
| `opencover` | SonarQube, ReportGenerator |
| `cobertura` | Azure DevOps, Codecov |
| `lcov` | Codecov, GitHub Actions Summary |
| `json` | Coverlet native |

### What to Cover

```
Target:
  - Business logic / domain services: 90%+
  - Application services / use cases: 80%+
  - Controllers / thin wrappers: 60%+
  - Infrastructure / DB adapters: 50%+ (integration tests)
  - Generated code (migrations, proto): 0% (exclude)
  - Program.cs / startup: 0% (exclude)
```

> **Pitfall:** 100% coverage is not the goal. Meaningful tests on meaningful code. Exclude auto-generated files, migrations, and framework boilerplate from measurement.

---

## 7. Artifact Repositories

### NuGet Package Registries

| Registry | Best For | Auth |
|----------|----------|------|
| NuGet.org | Public OSS packages | API key |
| GitHub Packages | Internal packages, OSS | PAT / GITHUB_TOKEN |
| Azure Artifacts | Enterprise internal | Azure AD / PAT |
| JFrog Artifactory | Enterprise universal | API key / AD |
| Sonatype Nexus | Enterprise universal | LDAP / API key |

```xml
<!-- nuget.config -->
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="company-feed" value="https://pkgs.dev.azure.com/company/_packaging/internal/nuget/v3/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <company-feed>
      <add key="Username" value="PAT" />
      <add key="ClearTextPassword" value="%AZURE_ARTIFACTS_PAT%" />
    </company-feed>
  </packageSourceCredentials>
</configuration>
```

### Docker Image Registries

| Registry | Cloud | Best For |
|----------|-------|----------|
| Amazon ECR | AWS | ECS/EKS workloads |
| Azure Container Registry (ACR) | Azure | AKS workloads |
| GitHub Container Registry (GHCR) | Any | Open-source, GitHub Actions |
| Docker Hub | Any | Public images |
| JFrog Artifactory | Any | Enterprise universal |

### SemVer Versioning Strategy

```
MAJOR.MINOR.PATCH[-prerelease]+[build]

1.0.0          → initial release
1.1.0          → backward-compatible new feature
1.1.1          → bug fix
2.0.0          → breaking change
1.2.0-beta.1   → pre-release
1.2.0-rc.2     → release candidate
```

### Artifact Promotion Pipeline

```
Build → push :1.2.0-dev.42 to dev-registry
  ↓ (integration tests pass)
Promote → retag as :1.2.0-staging, push to staging-registry
  ↓ (staging sign-off)
Promote → retag as :1.2.0, push to prod-registry
```

> Key principle: **promote the artifact, not the build**. The same image binary moves from dev → staging → prod. You never rebuild for staging or production.

---

## 8. Feature Flags

### Why Feature Flags?

Decouple **deployment** (code is running in production) from **release** (users can see the feature). Benefits:

- Deploy incomplete features safely behind a flag
- Gradual rollout / canary without infrastructure changes
- A/B testing with statistical significance
- Kill switch: instant disable without a deploy
- Environment targeting: staging vs production

### .NET — Microsoft.FeatureManagement

```csharp
// Program.cs
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<PercentageFilter>()
    .AddFeatureFilter<TargetingFilter>();

// appsettings.json
{
  "FeatureManagement": {
    "NewPaymentFlow": false,
    "BetaDashboard": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 25 }
        }
      ]
    },
    "AdminPanel": {
      "EnabledFor": [
        {
          "Name": "Targeting",
          "Parameters": {
            "Audience": {
              "Users": ["admin@company.com"],
              "Groups": [{ "Name": "beta-users", "RolloutPercentage": 50 }]
            }
          }
        }
      ]
    }
  }
}
```

```csharp
// In a service / controller
public class PaymentController : ControllerBase
{
    private readonly IFeatureManager _features;

    public PaymentController(IFeatureManager features)
        => _features = features;

    [HttpPost("process")]
    public async Task<IActionResult> ProcessPayment(PaymentRequest request)
    {
        if (await _features.IsEnabledAsync("NewPaymentFlow"))
        {
            return Ok(await _newPaymentService.ProcessAsync(request));
        }
        return Ok(await _legacyPaymentService.ProcessAsync(request));
    }
}
```

### LaunchDarkly Integration

```csharp
// Install: dotnet add package LaunchDarkly.ServerSdk

var config = Configuration.Builder("sdk-key-here")
    .Build();

var client = new LdClient(config);

var user = Context.New("user-abc-123");
bool showNewFlow = client.BoolVariation("new-payment-flow", user, defaultValue: false);
```

### Targeting Strategies

| Strategy | Use case |
|----------|----------|
| % Rollout | Canary-style gradual rollout |
| User list | Beta testers, specific accounts |
| Group/segment | All enterprise users, all EU users |
| Environment | On in staging, off in prod |
| Attribute-based | Users with `plan=premium` |

### Real Example: Gradual Payment Flow Rollout

```
Week 1: 0%  → Internal team only (user targeting)
Week 2: 5%  → Monitor error rates, latency P99
Week 3: 25% → Widen if metrics healthy
Week 4: 50% → Monitor conversion rates
Week 5: 100% → Full release, clean up old code path
```

---

## 9. Blue/Green Deployment

### Concept

Run two identical environments simultaneously:

```
Blue  = currently live (v1.2.0) — receives 100% production traffic
Green = new version (v1.3.0)   — deployed, tested, ready
                                 ↓
                         Switch traffic to Green
                                 ↓
Blue  = idle (kept for instant rollback)
Green = now live (v1.3.0) — receives 100% production traffic
```

### Traffic Switch Mechanisms

**AWS ALB Weighted Target Groups:**
```json
{
  "ListenerArn": "arn:aws:elasticloadbalancing:...",
  "DefaultActions": [
    {
      "Type": "forward",
      "ForwardConfig": {
        "TargetGroups": [
          { "TargetGroupArn": "arn:...:blue-tg",  "Weight": 0   },
          { "TargetGroupArn": "arn:...:green-tg", "Weight": 100 }
        ]
      }
    }
  ]
}
```

**DNS Cutover (Route 53):**
```bash
# Switch CNAME/A record from blue to green
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "green-lb.us-east-1.elb.amazonaws.com"}]
      }
    }]
  }'
```

### Pros & Cons

| Pros | Cons |
|------|------|
| Instant rollback (flip back to Blue) | Double infrastructure cost |
| Zero downtime deployment | Database migrations must be backward-compatible |
| Test Green with production traffic shape before full switch | More complex to orchestrate |
| No mixed-version requests during cutover | Session affinity needs careful handling |

### Database Considerations

Blue/Green requires **backward-compatible migrations**:
1. Add new column (nullable, default) — both versions work
2. Deploy Green (writes to new column)
3. Backfill old rows
4. Remove old column in next release (Blue is gone)

---

## 10. Canary Deployment

### Concept

Route a small, gradually increasing percentage of traffic to the new version. Monitor health metrics. Promote or rollback based on signals.

```
Step 1: 5%  → new version | 95% → old version
         Monitor: error rate, p99 latency, business KPIs
Step 2: 25% → new | 75%  → old (if healthy)
Step 3: 50% → new | 50%  → old
Step 4: 100% → new — canary complete, old pods removed
```

### Implementation Options

**Argo Rollouts (Kubernetes):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-api
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: {duration: 5m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      # Automated analysis
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: myapp-api-canary
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.95   # 95% success rate required
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{app="{{args.service-name}}",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{app="{{args.service-name}}"}[5m]))
```

**AWS CodeDeploy (ECS/EC2):**
```json
{
  "deploymentConfigName": "MyCanaryConfig",
  "computePlatform": "ECS",
  "trafficRoutingConfig": {
    "type": "TimeBasedCanary",
    "timeBasedCanary": {
      "canaryPercentage": 10,
      "canaryInterval": 15
    }
  }
}
```

**Flagger (Service Mesh — Istio/Linkerd):**
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp-api
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  progressDeadlineSeconds: 120
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 10
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 1m
```

### Canary vs Blue/Green Decision

| | Blue/Green | Canary |
|-|-----------|--------|
| Rollback speed | Instant (traffic flip) | Graceful (reduce %) |
| Infrastructure cost | 2× (whole env) | Minimal (few extra pods) |
| Risk exposure | All-or-nothing | Gradual |
| Automation needed | Low | High (metrics analysis) |
| Best for | Simple apps, big changes | High-traffic, confidence-building |

---

## 11. Infrastructure as Code

### Terraform

```hcl
# main.tf — ECS Service with ALB
resource "aws_ecs_service" "myapp_api" {
  name            = "myapp-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.myapp_api.arn
  desired_count   = 3

  load_balancer {
    target_group_arn = aws_lb_target_group.myapp_api.arn
    container_name   = "myapp-api"
    container_port   = 8080
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}

# Plan: show changes without applying
# terraform plan -out=tfplan

# Apply: execute the plan
# terraform apply tfplan

# State: stored in S3 + DynamoDB lock (shared team state)
```

### AWS CDK (C#)

```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.ECS;
using Amazon.CDK.AWS.ECS.Patterns;

public class MyAppStack : Stack
{
    public MyAppStack(Construct scope, string id, IStackProps props = null) 
        : base(scope, id, props)
    {
        var cluster = new Cluster(this, "MyCluster");

        new ApplicationLoadBalancedFargateService(this, "MyAppService", 
            new ApplicationLoadBalancedFargateServiceProps
            {
                Cluster = cluster,
                Cpu = 512,
                DesiredCount = 2,
                MemoryLimitMiB = 1024,
                TaskImageOptions = new ApplicationLoadBalancedTaskImageOptions
                {
                    Image = ContainerImage.FromEcrRepository(
                        Repository.FromRepositoryName(this, "Repo", "myapp-api"),
                        "latest")
                }
            });
    }
}
```

### IaC Comparison

| Tool | Language | Cloud | State |
|------|----------|-------|-------|
| Terraform | HCL | Multi-cloud | Remote (S3, Terraform Cloud) |
| AWS CDK | TS, Python, C#, Java | AWS only | CloudFormation |
| Pulumi | TS, Python, C#, Go | Multi-cloud | Pulumi Cloud / S3 |
| CloudFormation | YAML/JSON | AWS only | Managed by AWS |
| Bicep | Bicep DSL | Azure only | Managed by Azure |
| Azure ARM | JSON | Azure only | Managed by Azure |

### Why IaC?

- **Reproducibility:** spin up identical environments in minutes
- **Version control:** infrastructure changes go through PR review
- **Auditability:** know exactly who changed what and when
- **Drift detection:** `terraform plan` shows live vs desired state
- **Disaster recovery:** recreate entire infra from code after an incident

---

## 12. Interview Q&A Cheat Sheet

### CI/CD Questions

**Q: Walk me through your CI/CD pipeline.**

> "Our GitHub Actions pipeline triggers on every PR to main. It runs restore, build, test with coverage collection, and SonarQube analysis in parallel where possible. If the quality gate passes and coverage is above 80%, the Docker image is built using BuildKit with layer caching to ECR. Deployment to ECS uses the Amazon ECS deploy action and waits for service stability. We use OIDC federation to AWS — no static access keys stored as secrets."

**Q: How do you handle secrets in your pipelines?**

> "We use OIDC federation for AWS — the pipeline assumes an IAM role via a temporary JWT token, so there are no long-lived AWS keys anywhere. Application secrets like database passwords and API keys are stored in AWS Secrets Manager, injected as ECS task environment variables at runtime. GitHub repository secrets hold only the OIDC role ARN and non-sensitive configuration like region names."

**Q: What's the difference between canary and blue/green deployments?**

> "Blue/green is a binary switch — you maintain two full environments and flip all traffic at once. It's fast to rollback but costs double. Canary gradually shifts traffic, maybe 5% → 25% → 100%, and uses automated metric analysis to promote or abort. Canary exposes fewer users to risk and is better for high-traffic services where you want confidence before full exposure. Blue/green is simpler and better when your changes are large enough that you want a clean cut."

**Q: How do you enforce code coverage thresholds?**

> "We configure a SonarQube quality gate that fails the pipeline if coverage on new code drops below 80%. The `sonar.qualitygate.wait=true` flag makes the pipeline wait for the server-side evaluation. We also generate a Cobertura XML report that Azure DevOps displays inline on the PR, so reviewers see coverage without leaving GitHub."

**Q: What are feature flags and when do you use them?**

> "Feature flags decouple when you deploy from when users see the feature. We use Microsoft.FeatureManagement in .NET for simple flags backed by Azure App Configuration, and LaunchDarkly for complex targeting. A typical use case: we deploy a new payment flow behind a flag, enable it for our internal team first, then roll out to 5% of users, monitor error rates and conversion metrics, and expand to 100% over two weeks. If something goes wrong, we flip the flag — no deploy, no incident."

**Q: How would you implement a zero-downtime deployment?**

> "For ECS, we use a rolling update with minimum healthy percent of 100% — new tasks must start successfully before old ones are stopped. We define health checks in the load balancer so traffic only shifts to a task once it returns 200 on `/health`. For databases, we use expand/contract migrations: add nullable columns in one release, backfill, then remove the old column in a subsequent release, never breaking the running version."

**Q: What is SonarQube and how does it fit in CI?**

> "SonarQube provides static code analysis — it detects bugs, security vulnerabilities, code smells, and code duplication without running the code. It tracks technical debt over time. In our CI pipeline, dotnet-sonarscanner wraps the build. After tests run and coverage reports are generated, the scanner ships everything to SonarQube. We fail the pipeline if the quality gate is ERROR — typically triggered by coverage below 80% or new critical vulnerabilities. This enforces quality as a hard gate rather than a suggestion."

---

### Pitfalls to Avoid

| Topic | Common Mistake | Better Answer |
|-------|---------------|---------------|
| Branching | "We use GitFlow" without context | Explain why your team's cadence fits that strategy |
| Secrets | "Stored in GitHub secrets" | Mention OIDC or secret rotation policy |
| Coverage | "We aim for 100%" | "80%+ on business logic, exclude generated code" |
| Blue/Green | Forgetting database migrations | Expand/contract pattern, backward-compatible first |
| Feature flags | "We use env vars as feature flags" | Named flag system with targeting and audit trail |
| IaC | "We use CloudFormation" | Can you diff it? Review it? Mention state management |
| Docker | "We cache Docker layers" | Mention BuildKit, cache-from ECR, multi-stage builds |