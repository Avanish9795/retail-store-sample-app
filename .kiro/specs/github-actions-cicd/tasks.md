# Implementation Tasks

## Task 1: Set Up AWS OIDC IAM Role

- [x] 1.1 Create an IAM OIDC identity provider in AWS for `token.actions.githubusercontent.com`
- [ ] 1.2 Create an IAM role `github-actions-retail-store` with the trust policy scoped to `repo:<ORG>/<REPO>:ref:refs/heads/main`
- [ ] 1.3 Attach an inline policy granting minimum ECR permissions: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:CreateRepository`, `ecr:DescribeRepositories`, `ecr:DescribeImages`
- [ ] 1.4 Add GitHub secrets: `AWS_ROLE_ARN`, `AWS_REGION`, `AWS_ACCOUNT_ID`

**Validates**: Requirements 2.1, 2.2, 2.3

---

## Task 2: Create the Workflow File Skeleton

- [x] 2.1 Create `.github/workflows/ci-cd.yml` with `on: push: branches: [main]` and `workflow_dispatch` triggers
- [ ] 2.2 Add `concurrency` group `${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: true`
- [ ] 2.3 Set workflow-level `permissions: id-token: write, contents: write`
- [ ] 2.4 Define `env` block with `AWS_REGION` and `AWS_ACCOUNT_ID` from secrets

**Validates**: Requirements 1.1, 1.2, 1.4, 1.5

---

## Task 3: Implement the detect_changes Job

- [ ] 3.1 Add `detect_changes` job with `actions/checkout@<SHA>` (`fetch-depth: 2`)
- [ ] 3.2 Write the change detection bash script:
  - Handle `workflow_dispatch` → all 5 services
  - Handle first commit (`HEAD^` fails) → all 5 services
  - Handle normal push → `git diff --name-only HEAD^ HEAD`, filter per `src/<svc>/` excluding `src/<svc>/chart/`
- [ ] 3.3 Serialize result to JSON array using `jq` and write to `$GITHUB_OUTPUT` as `changed_services`
- [ ] 3.4 Output `image_tag` as `${GITHUB_SHA:0:7}` to `$GITHUB_OUTPUT`

**Validates**: Requirements 3.1–3.10

---

## Task 4: Implement the build_and_push Matrix Job

- [ ] 4.1 Add `build_and_push` job with `needs: detect_changes` and `if: needs.detect_changes.outputs.changed_services != '[]'`
- [ ] 4.2 Configure matrix: `service: ${{ fromJson(needs.detect_changes.outputs.changed_services) }}` with `fail-fast: false`
- [ ] 4.3 Set job-level `permissions: id-token: write, contents: read`
- [ ] 4.4 Add `aws-actions/configure-aws-credentials@<SHA>` step with `role-to-assume: ${{ secrets.AWS_ROLE_ARN }}`
- [ ] 4.5 Add `aws-actions/amazon-ecr-login@<SHA>` step
- [ ] 4.6 Add `docker/setup-buildx-action@<SHA>` step
- [ ] 4.7 Add ECR repository creation step with `scanOnPush=true` (ignore `RepositoryAlreadyExistsException`)
- [ ] 4.8 Add image existence check — skip build+push if `aws ecr describe-images` returns the current SHA7 tag
- [ ] 4.9 Add Dockerfile existence check — fail with descriptive error if missing
- [ ] 4.10 Add `docker buildx build` step with `--cache-from type=gha,scope=<service>`, `--cache-to type=gha,scope=<service>,mode=max`, `--tag :SHA7`, `--tag :latest`, `--provenance=true`, `--sbom=true`, `--push`
- [ ] 4.11 Add ECR manifest verification step: `aws ecr describe-images` to confirm image is pullable

**Validates**: Requirements 4.1–4.11

---

## Task 5: Implement the update_helm_values Job

- [ ] 5.1 Add `update_helm_values` job with `needs: [detect_changes, build_and_push]` and `if: needs.build_and_push.result == 'success'`
- [ ] 5.2 Set job-level `permissions: contents: write`
- [ ] 5.3 Add `actions/checkout@<SHA>` with `ref: main`, `token: ${{ secrets.GITHUB_TOKEN }}`, `fetch-depth: 0`
- [ ] 5.4 Install `yq` (mikefarah/yq) via the official action or `wget`
- [ ] 5.5 Configure git identity: `github-actions[bot]` / `41898282+github-actions[bot]@users.noreply.github.com`
- [ ] 5.6 Loop over changed services:
  - Validate `image.tag` and `image.repository` fields exist in `values.yaml`
  - Skip service if current tag already equals SHA7 (idempotency)
  - Run `yq e ".image.repository = ..."` and `yq e ".image.tag = ..."` in-place
  - `git add src/<service>/chart/values.yaml`
- [ ] 5.7 Commit with message `ci: update image tags for <services> to <SHA7> [skip ci]`
- [ ] 5.8 Push with rebase retry: `git push origin main || (git pull --rebase origin main && git push origin main)`

**Validates**: Requirements 5.1–5.10, 6.1–6.3

---

## Task 6: Pin All Action SHAs

- [ ] 6.1 Look up and pin `actions/checkout` to its current release commit SHA
- [ ] 6.2 Look up and pin `aws-actions/configure-aws-credentials` to its current release commit SHA
- [ ] 6.3 Look up and pin `aws-actions/amazon-ecr-login` to its current release commit SHA
- [ ] 6.4 Look up and pin `docker/setup-buildx-action` to its current release commit SHA
- [ ] 6.5 Add inline comments with the human-readable version tag next to each SHA pin

**Validates**: Requirement 7.1

---

## Task 7: Validate ArgoCD Compatibility

- [ ] 7.1 Confirm all ArgoCD Application `targetRevision` values are `main` (already confirmed in existing YAML)
- [ ] 7.2 Confirm all ArgoCD Application `path` values match `src/<service>/chart` (already confirmed)
- [ ] 7.3 Verify the `[skip ci]` tag in the commit message prevents pipeline re-trigger (test with a dry run)

**Validates**: Requirements 6.1–6.5
