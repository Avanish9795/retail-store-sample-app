# Design Document: GitHub Actions CI/CD Pipeline (DevOps & GitOps Best Practices)

## Overview

This document describes the technical design for a GitHub Actions CI/CD pipeline that follows industry-standard DevOps and GitOps practices. It automates building, tagging, and deploying the five retail store microservices (`cart`, `catalog`, `checkout`, `orders`, `ui`) to Amazon ECR and updating Helm chart `values.yaml` files so that ArgoCD detects drift and redeploys only affected services.

**Branch model**: Single-branch GitOps. The `main` branch is both the application source and the GitOps state. The pipeline triggers on push to `main`, builds images, and commits updated `values.yaml` back to `main`. ArgoCD reads `main` (`targetRevision: main`) and syncs automatically.

### Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| AWS auth | OIDC federation (no static keys) | Eliminates long-lived credential exposure; tokens expire after 1 hour |
| Action pinning | Commit SHA pins | Prevents supply chain attacks via mutable tag mutation |
| Build caching | `docker buildx` + GitHub Actions cache (`type=gha`) | Reduces build time by reusing unchanged layers |
| YAML surgery | `mikefarah/yq` | Surgically updates only root-level `image.*` keys; leaves infra image fields untouched |
| Parallel builds | Matrix job with `fail-fast: false` | Builds all changed services in parallel; one failure doesn't cancel others |
| Loop prevention | `[skip ci]` in commit message | Prevents the `values.yaml` commit from re-triggering the pipeline |
| Image tagging | SHA7 + `latest` | SHA7 for traceability; `latest` for convenience |
| Supply chain | SBOM + provenance attestations | Satisfies SLSA Level 2 requirements |
| Concurrency | `concurrency` group per branch | Cancels stale runs; only latest run proceeds |


---

## Architecture

### End-to-End Flow

```
Developer pushes to main
        │
        ▼
GitHub Actions: ci-cd.yml triggered
        │
        ▼
[Job 1] detect_changes
  ├─ git diff HEAD^ HEAD (exclude src/*/chart/)
  ├─ outputs: changed_services JSON array
  └─ e.g. ["cart","ui"]
        │
        ├─ changed_services == [] → pipeline exits (no-op)
        │
        ▼
[Job 2] build_and_push  (matrix, parallel, fail-fast: false)
  ├─ OIDC → assume IAM role → ECR login
  ├─ Create ECR repo (scanOnPush: true) if absent
  ├─ Check: image SHA7 already in ECR? → skip if yes
  ├─ docker buildx build --cache-from/to type=gha
  ├─ Push :SHA7 and :latest tags
  ├─ aws ecr describe-images (verify manifest)
  └─ Attach SBOM + provenance attestations
        │
        ▼  (only if ALL matrix jobs succeed)
[Job 3] update_helm_values
  ├─ git config bot identity
  ├─ For each changed service:
  │    ├─ Check: values.yaml tag == SHA7? → skip if yes
  │    ├─ yq update .image.repository + .image.tag
  │    └─ git add src/<service>/chart/values.yaml
  ├─ git commit "ci: update image tags for <services> to <SHA7> [skip ci]"
  ├─ git push origin main
  └─ On failure: git pull --rebase, retry once
        │
        ▼
ArgoCD detects diff on main → syncs only changed services → EKS
```

### Branch Strategy

```
main branch  ──push──►  GitHub Actions pipeline
                               │
                               ▼
                        builds images → ECR
                               │
                               ▼
                        commits values.yaml → main (same branch)
                               │
                               ▼
                        ArgoCD watches main → syncs EKS
```


---

## Workflow File Structure

```
.github/
└── workflows/
    └── ci-cd.yml          # Single workflow file — all three jobs
```

### Full Workflow Skeleton

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  workflow_dispatch:

# Prevent concurrent runs on the same branch; cancel stale runs
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Minimum permissions at workflow level
permissions:
  id-token: write    # OIDC token exchange
  contents: write    # git push values.yaml updates

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}

jobs:
  detect_changes:
    runs-on: ubuntu-latest
    outputs:
      changed_services: ${{ steps.detect.outputs.changed_services }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          fetch-depth: 2
      - id: detect
        run: |
          # ... change detection script (see Component 1)

  build_and_push:
    needs: detect_changes
    if: needs.detect_changes.outputs.changed_services != '[]'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect_changes.outputs.changed_services) }}
      fail-fast: false
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502  # v4.0.2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      - uses: aws-actions/amazon-ecr-login@062b18b96a7aff071d4dc91bc00c4c1a7945b076  # v2.0.1
        id: ecr-login
      - uses: docker/setup-buildx-action@b5ca514318bd6ebac0fb2aedd5d36ec1b5c232a2  # v3.10.0
      # ... build, push, verify, attest steps (see Component 2)

  update_helm_values:
    needs: [detect_changes, build_and_push]
    if: needs.build_and_push.result == 'success'
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          ref: main
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0
      # ... yq update, git commit with [skip ci], git push with rebase retry
```


---

## Components and Interfaces

### Component 1: Change_Detector

**Responsibility**: Determine which services have source code changes, excluding Helm chart-only changes.

**Key logic**:
```bash
IMAGE_TAG="${GITHUB_SHA:0:7}"
SERVICES=("cart" "catalog" "checkout" "orders" "ui")
CHANGED=()

if [ "$GITHUB_EVENT_NAME" = "workflow_dispatch" ]; then
  CHANGED=("${SERVICES[@]}")
elif ! git rev-parse HEAD^ > /dev/null 2>&1; then
  # First commit — treat all as changed
  CHANGED=("${SERVICES[@]}")
else
  DIFF=$(git diff --name-only HEAD^ HEAD)
  for svc in "${SERVICES[@]}"; do
    # Match src/<svc>/ but NOT src/<svc>/chart/ (chart-only changes skip rebuild)
    if echo "$DIFF" | grep -qE "^src/${svc}/(?!chart/)"; then
      CHANGED+=("$svc")
    fi
  done
fi

# Serialize to JSON array for matrix consumption
JSON=$(printf '%s\n' "${CHANGED[@]}" | jq -R . | jq -sc .)
echo "changed_services=${JSON}" >> "$GITHUB_OUTPUT"
echo "image_tag=${IMAGE_TAG}" >> "$GITHUB_OUTPUT"
```

**Why exclude `src/<svc>/chart/` changes**: A Helm chart-only change (e.g., updating resource limits) does not require a new Docker image. ArgoCD will pick up the chart change directly. Rebuilding the image would be wasteful and would overwrite the existing tag.

---

### Component 2: Image_Builder

**Responsibility**: Build and push Docker images with caching, dual tagging, and supply chain attestations.

**Key steps per matrix service**:

```bash
SERVICE="${{ matrix.service }}"
IMAGE_TAG="${GITHUB_SHA:0:7}"
ECR_REGISTRY="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com"
IMAGE_URI="${ECR_REGISTRY}/retail-store-${SERVICE}"

# 1. Create ECR repo with scan-on-push (idempotent)
aws ecr create-repository \
  --repository-name "retail-store-${SERVICE}" \
  --image-scanning-configuration scanOnPush=true \
  --region "$AWS_REGION" 2>&1 | \
  grep -v "RepositoryAlreadyExistsException" || true

# 2. Skip if image already exists (idempotency)
if aws ecr describe-images \
     --repository-name "retail-store-${SERVICE}" \
     --image-ids imageTag="${IMAGE_TAG}" \
     --region "$AWS_REGION" > /dev/null 2>&1; then
  echo "Image ${IMAGE_TAG} already exists — skipping build."
  exit 0
fi

# 3. Verify Dockerfile exists
[ -f "src/${SERVICE}/Dockerfile" ] || \
  { echo "ERROR: Dockerfile not found at src/${SERVICE}/Dockerfile" >&2; exit 1; }

# 4. Build with layer cache (docker buildx)
docker buildx build \
  --cache-from type=gha,scope=${SERVICE} \
  --cache-to   type=gha,scope=${SERVICE},mode=max \
  --tag "${IMAGE_URI}:${IMAGE_TAG}" \
  --tag "${IMAGE_URI}:latest" \
  --provenance=true \
  --sbom=true \
  --push \
  "src/${SERVICE}/"

# 5. Verify manifest is retrievable before Helm update
aws ecr describe-images \
  --repository-name "retail-store-${SERVICE}" \
  --image-ids imageTag="${IMAGE_TAG}" \
  --region "$AWS_REGION" > /dev/null
```

**`fail-fast: false`**: A build failure for `cart` does not cancel the in-progress build for `catalog`. Each service is independent. The Helm_Updater only runs if **all** matrix jobs succeed.

---

### Component 3: Helm_Updater

**Responsibility**: Update `values.yaml` for each changed service and commit back to `main` with `[skip ci]` to prevent pipeline loops.

**Key steps**:

```bash
IMAGE_TAG="${GITHUB_SHA:0:7}"
SERVICES=$(echo '${{ needs.detect_changes.outputs.changed_services }}' | jq -r '.[]')
ECR_BASE="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com"
UPDATED=()

git config user.name  "github-actions[bot]"
git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

for SERVICE in $SERVICES; do
  VALUES="src/${SERVICE}/chart/values.yaml"

  # Validate required fields exist
  yq e '.image.tag' "$VALUES" | grep -qv 'null' || \
    { echo "ERROR: .image.tag missing in $VALUES" >&2; exit 1; }
  yq e '.image.repository' "$VALUES" | grep -qv 'null' || \
    { echo "ERROR: .image.repository missing in $VALUES" >&2; exit 1; }

  # Idempotency: skip if tag already matches
  CURRENT_TAG=$(yq e '.image.tag' "$VALUES")
  if [ "$CURRENT_TAG" = "$IMAGE_TAG" ]; then
    echo "values.yaml for ${SERVICE} already at ${IMAGE_TAG} — skipping."
    continue
  fi

  # Update only root-level image fields
  yq e ".image.repository = \"${ECR_BASE}/retail-store-${SERVICE}\"" -i "$VALUES"
  yq e ".image.tag = \"${IMAGE_TAG}\"" -i "$VALUES"

  git add "$VALUES"
  UPDATED+=("$SERVICE")
done

# Commit only if there are staged changes
if [ ${#UPDATED[@]} -gt 0 ]; then
  SERVICES_STR=$(IFS=', '; echo "${UPDATED[*]}")
  # [skip ci] prevents this commit from re-triggering the pipeline
  git commit -m "ci: update image tags for ${SERVICES_STR} to ${IMAGE_TAG} [skip ci]"

  # Push with rebase retry on conflict
  git push origin main || {
    echo "Push failed — pulling with rebase and retrying..."
    git pull --rebase origin main
    git push origin main || { echo "ERROR: git push failed after rebase retry." >&2; exit 1; }
  }
fi
```


---

## Data Models

### `values.yaml` Image Fields (Actual Structure)

Based on `src/cart/chart/values.yaml`, the root-level `image` block is:

```yaml
image:
  repository: public.ecr.aws/aws-containers/retail-store-sample-cart  # ← updated
  pullPolicy: Always
  tag: "1.2.2"                                                          # ← updated
```

Infrastructure sidecar images are nested under their own top-level keys and **must not be touched**:

```yaml
dynamodb:
  image:
    repository: public.ecr.aws/aws-dynamodb-local/aws-dynamodb-local  # ← DO NOT MODIFY
    tag: "1.25.1"
```

The `yq` path `.image.repository` resolves only the root-level `image` mapping. It does **not** match `dynamodb.image.repository` because that path is `.dynamodb.image.repository`.

### ECR Repository Naming

| Service   | ECR Repo Name         | Full URI                                                                    |
|-----------|-----------------------|-----------------------------------------------------------------------------|
| cart      | `retail-store-cart`   | `<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/retail-store-cart:<SHA7>`        |
| catalog   | `retail-store-catalog`| `<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/retail-store-catalog:<SHA7>`     |
| checkout  | `retail-store-checkout`| `<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/retail-store-checkout:<SHA7>`   |
| orders    | `retail-store-orders` | `<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/retail-store-orders:<SHA7>`      |
| ui        | `retail-store-ui`     | `<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/retail-store-ui:<SHA7>`          |

### Secrets and Permissions

| Name              | Source              | Used By                          | Notes                              |
|-------------------|---------------------|----------------------------------|------------------------------------|
| `AWS_ROLE_ARN`    | GitHub secret       | Image_Builder (OIDC)             | Replaces static access keys        |
| `AWS_REGION`      | GitHub secret       | All jobs (ECR URI construction)  |                                    |
| `AWS_ACCOUNT_ID`  | GitHub secret       | All jobs (ECR URI construction)  |                                    |
| `GITHUB_TOKEN`    | Auto-provided       | Helm_Updater (git push)          | Scoped to `contents: write`        |

**No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` are used.** OIDC federation provides short-lived tokens automatically.

### IAM Role Trust Policy (OIDC)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:ref:refs/heads/main"
      }
    }
  }]
}
```

This restricts the role to only be assumable from the `main` branch of your specific repository.

### Job Dependency Graph

```
detect_changes
      │
      ▼
build_and_push (matrix: per changed service, fail-fast: false)
      │  (only if ALL matrix jobs succeed)
      ▼
update_helm_values
```


---

## Security Design

### OIDC Authentication Flow

```
GitHub Actions runner
        │
        │  1. Request OIDC token from GitHub
        ▼
GitHub OIDC provider (token.actions.githubusercontent.com)
        │
        │  2. Return signed JWT (audience: sts.amazonaws.com)
        ▼
AWS STS AssumeRoleWithWebIdentity
        │
        │  3. Validate JWT, check trust policy conditions
        │     (repo match + branch match)
        ▼
Short-lived AWS credentials (1 hour TTL)
        │
        │  4. Use credentials for ECR operations
        ▼
Amazon ECR
```

No static credentials are stored anywhere. The token expires after 1 hour and cannot be reused outside the pipeline run.

### Action SHA Pinning

All third-party actions are pinned to immutable commit SHAs:

```yaml
# Good — pinned to commit SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

# Bad — mutable tag, vulnerable to tag mutation attacks
- uses: actions/checkout@v4
```

### Minimum Permissions

```yaml
# Workflow-level defaults (deny all)
permissions:
  contents: none
  id-token: none

# Per-job overrides (grant only what's needed)
jobs:
  detect_changes:
    permissions:
      contents: read        # checkout only

  build_and_push:
    permissions:
      id-token: write       # OIDC token exchange
      contents: read        # checkout only

  update_helm_values:
    permissions:
      contents: write       # git push values.yaml
```

---

## Error Handling Summary

| Scenario | Behavior |
|---|---|
| OIDC auth failure | Fail immediately with auth error message |
| Missing `AWS_ROLE_ARN` secret | Fail immediately before any AWS call |
| Dockerfile not found | Fail with path in error message |
| ECR repo creation fails | Fail that service's matrix job; others continue |
| Docker build fails | Fail that service's matrix job; others continue |
| ECR push fails | Fail that service's matrix job; others continue |
| Image already in ECR | Skip build+push silently (idempotent) |
| `values.yaml` missing `image.tag` | Fail with field name + service in error message |
| `values.yaml` tag already current | Skip write+commit silently (idempotent) |
| Git push conflict | `git pull --rebase` + retry once; fail on second failure |
| Any Image_Builder fails | Helm_Updater is skipped entirely |
| `[skip ci]` in commit | Pipeline does not re-trigger on the values.yaml commit |

---

## GitOps Compatibility Notes

Your ArgoCD Application resources use:
- `targetRevision: main` — matches the branch the Helm_Updater commits to
- `path: src/<service>/chart` — the pipeline never modifies chart structure, only `values.yaml` content
- `syncPolicy.automated.prune: true` + `selfHeal: true` — ArgoCD will auto-sync within seconds of the commit landing on `main`
- `syncOptions: CreateNamespace=true` — no pipeline changes needed

The `[skip ci]` tag in the commit message prevents the `values.yaml` update from triggering another pipeline run, breaking the potential infinite loop:

```
push code → pipeline → commit values.yaml [skip ci] → ArgoCD syncs → done
                                ↑
                         [skip ci] stops re-trigger here
```

