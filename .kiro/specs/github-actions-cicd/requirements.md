# Requirements Document

## Introduction

This feature implements a GitHub Actions CI/CD pipeline for the retail store sample application following industry-standard DevOps and GitOps practices. The pipeline automates three core responsibilities: detecting which services have changed on a push to the `main` branch, building and pushing Docker images for changed services to Amazon ECR, and updating the `image.tag` and `image.repository` values in each service's Helm chart `values.yaml` so that ArgoCD can detect the drift and automatically redeploy only the affected services.

The application consists of five microservices — **cart**, **catalog**, **checkout**, **orders**, and **ui** — each with its own source directory under `src/` and its own Helm chart under `src/<service>/chart/`. ArgoCD watches each service's chart directory on the `main` branch (`targetRevision: main`) and syncs automatically when values change.

**GitOps branch model**: This pipeline uses a single-branch GitOps model. The `main` branch is both the application source branch and the GitOps state branch. Developers push code to `main`; the pipeline builds images and commits updated `values.yaml` files back to `main`. ArgoCD reads from `main` and syncs automatically.

---

## Glossary

- **CI_CD_Pipeline**: The GitHub Actions workflow defined in `.github/workflows/ci-cd.yml` that orchestrates change detection, image builds, and Helm chart updates.
- **Change_Detector**: The job in the CI_CD_Pipeline responsible for determining which services have source code changes in a given push.
- **Image_Builder**: The matrix job in the CI_CD_Pipeline responsible for building Docker images and pushing them to Amazon ECR.
- **Helm_Updater**: The job in the CI_CD_Pipeline responsible for modifying `image.repository` and `image.tag` fields in a service's `values.yaml` and committing the result back to `main`.
- **Service**: One of the five application microservices: `cart`, `catalog`, `checkout`, `orders`, or `ui`.
- **ECR**: Amazon Elastic Container Registry — the private Docker image registry used to store built images.
- **Image_Tag**: A 7-character short Git commit SHA used to uniquely identify a built Docker image.
- **Helm_Chart**: The Helm chart directory located at `src/<service>/chart/` for each Service.
- **values.yaml**: The Helm values file at `src/<service>/chart/values.yaml` that contains `image.repository` and `image.tag` fields consumed by ArgoCD.
- **ArgoCD**: The GitOps continuous delivery tool that monitors the `main` branch and syncs Kubernetes deployments when Helm chart values change.
- **Infrastructure_Image**: A Docker image for a supporting service (e.g., DynamoDB Local) that is not managed by the CI_CD_Pipeline and must not be modified.
- **OIDC**: OpenID Connect — the AWS authentication mechanism used instead of long-lived IAM access keys.
- **IAM Role**: The AWS Identity and Access Management role assumed by the pipeline via OIDC federation, scoped to the minimum permissions required.
- **Docker Layer Cache**: Cached Docker build layers stored in GitHub Actions cache to speed up subsequent builds.

---

## Requirements

### Requirement 1: Pipeline Trigger and Branch Strategy

**User Story:** As a developer, I want the CI/CD pipeline to run automatically when I push code changes to the `main` branch, so that deployments are triggered without manual intervention and the GitOps state is always in sync with the source.

#### Acceptance Criteria

1. WHEN a push event occurs on the `main` branch, THE CI_CD_Pipeline SHALL start execution.
2. WHEN a `workflow_dispatch` event is triggered manually, THE CI_CD_Pipeline SHALL start execution and treat all five Services (`cart`, `catalog`, `checkout`, `orders`, `ui`) as changed.
3. WHEN a push event is a merge commit from a pull request, THE CI_CD_Pipeline SHALL execute normally, treating the merged files as the changed set.
4. THE CI_CD_Pipeline SHALL use concurrency groups scoped to `main` so that only one pipeline run executes at a time per branch, and a newer run cancels any in-progress run for the same branch.
5. THE CI_CD_Pipeline SHALL set `permissions: id-token: write` and `contents: write` at the workflow level to enable OIDC authentication and git push operations.

---

### Requirement 2: AWS Authentication via OIDC (No Long-Lived Keys)

**User Story:** As a security-conscious platform engineer, I want the pipeline to authenticate with AWS using short-lived OIDC tokens instead of long-lived IAM access keys, so that there are no static credentials stored in GitHub secrets that could be leaked or rotated manually.

#### Acceptance Criteria

1. THE CI_CD_Pipeline SHALL authenticate with AWS using the `aws-actions/configure-aws-credentials` action with `role-to-assume` set to the IAM role ARN stored in the GitHub secret `AWS_ROLE_ARN`.
2. THE CI_CD_Pipeline SHALL NOT use `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` secrets for AWS authentication.
3. THE IAM role assumed by the pipeline SHALL have the minimum required permissions: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:CreateRepository`, `ecr:DescribeRepositories`, `ecr:DescribeImages`.
4. IF the OIDC token exchange fails or the role assumption is denied, THEN THE CI_CD_Pipeline SHALL fail with a non-zero exit code and an error message identifying the authentication failure before attempting any ECR operation.
5. THE GitHub secret `AWS_ACCOUNT_ID` and `AWS_REGION` SHALL be used only for constructing the ECR URI and SHALL NOT be used for authentication.

---

### Requirement 3: Per-Service Change Detection

**User Story:** As a developer, I want the pipeline to detect which services have changed, so that only affected services are rebuilt and redeployed, reducing build time and avoiding unnecessary rollouts.

#### Acceptance Criteria

1. WHEN a push event occurs, THE Change_Detector SHALL compare the files changed in the push using `git diff --name-only HEAD^ HEAD` against the path `src/<service>/` for each Service, excluding changes to `src/<service>/chart/` (Helm chart changes alone do not require a new image build).
2. WHEN files under `src/cart/` (excluding `src/cart/chart/`) are changed, THE Change_Detector SHALL mark the `cart` Service as changed.
3. WHEN files under `src/catalog/` (excluding `src/catalog/chart/`) are changed, THE Change_Detector SHALL mark the `catalog` Service as changed.
4. WHEN files under `src/checkout/` (excluding `src/checkout/chart/`) are changed, THE Change_Detector SHALL mark the `checkout` Service as changed.
5. WHEN files under `src/orders/` (excluding `src/orders/chart/`) are changed, THE Change_Detector SHALL mark the `orders` Service as changed.
6. WHEN files under `src/ui/` (excluding `src/ui/chart/`) are changed, THE Change_Detector SHALL mark the `ui` Service as changed.
7. WHEN no files under `src/<service>/` (excluding chart subdirectory) are changed for a given Service, THE Change_Detector SHALL mark that Service as unchanged, and THE CI_CD_Pipeline SHALL skip the build and Helm update steps for that Service.
8. WHEN a `workflow_dispatch` event is triggered, THE Change_Detector SHALL mark all five Services as changed regardless of file differences.
9. WHEN the push is the first commit on the branch (no previous commit exists, `HEAD^` fails), THE Change_Detector SHALL mark all five Services (`cart`, `catalog`, `checkout`, `orders`, `ui`) as changed.
10. THE Change_Detector SHALL output the changed services list as a JSON array (e.g., `["cart","ui"]`) to `$GITHUB_OUTPUT` for consumption by downstream jobs.

---

### Requirement 4: Docker Image Build with Layer Caching

**User Story:** As a developer, I want Docker images for changed services to be built efficiently using layer caching and pushed to Amazon ECR, so that build times are minimized and the latest code is available for deployment.

#### Acceptance Criteria

1. WHEN a Service is marked as changed, THE Image_Builder SHALL authenticate with ECR using the OIDC-assumed IAM role.
2. WHEN a Service is marked as changed, THE Image_Builder SHALL create the ECR repository named `retail-store-<service>` if it does not already exist, using `--image-scanning-configuration scanOnPush=true` to enable automatic vulnerability scanning.
3. WHEN a Service is marked as changed, THE Image_Builder SHALL build a Docker image using `docker buildx` with GitHub Actions cache (`type=gha`) for layer caching, sourcing the Dockerfile from `src/<service>/`.
4. WHEN a Docker image is built, THE Image_Builder SHALL tag the image with both the 7-character short Git commit SHA (`<IMAGE_TAG>`) and the `latest` tag.
5. WHEN a Docker image is tagged, THE Image_Builder SHALL push both the `<IMAGE_TAG>` and `latest` tags to the ECR repository.
6. IF the Dockerfile does not exist at `src/<service>/Dockerfile`, THEN THE Image_Builder SHALL fail with a non-zero exit code and an error message identifying the missing file path.
7. IF the Docker build fails for a Service, THEN THE Image_Builder SHALL fail that Service's matrix job with a non-zero exit code and SHALL NOT proceed to the Helm update step for that Service. A build failure for one Service SHALL NOT cancel in-progress builds for other Services (`fail-fast: false`).
8. IF the ECR push fails for a Service, THEN THE Image_Builder SHALL fail that Service's matrix job with a non-zero exit code. A push failure for one Service SHALL NOT cancel in-progress builds for other Services.
9. IF the ECR repository creation fails for a Service, THEN THE Image_Builder SHALL fail that Service's matrix job with a non-zero exit code and SHALL NOT proceed to the build step.
10. WHEN a Service is marked as unchanged, THE Image_Builder SHALL skip all build and push steps for that Service.
11. IF the ECR repository for a Service already contains an image with the current Image_Tag, THE Image_Builder SHALL skip the build and push steps for that Service and SHALL NOT fail (idempotency).

---

### Requirement 5: Helm Chart Image Tag Update

**User Story:** As a developer, I want the pipeline to update the Helm chart `values.yaml` with the new ECR image repository and tag after a successful build, so that ArgoCD detects the change and redeploys only the updated service.

#### Acceptance Criteria

1. WHEN a Docker image is successfully pushed to ECR for a Service, THE Helm_Updater SHALL update the `image.repository` field in `src/<service>/chart/values.yaml` to `<AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/retail-store-<service>`.
2. WHEN a Docker image is successfully pushed to ECR for a Service, THE Helm_Updater SHALL update the `image.tag` field in `src/<service>/chart/values.yaml` to the 7-character short Git commit SHA used to tag the image.
3. THE Helm_Updater SHALL update only the `image.repository` and `image.tag` fields nested directly under the root-level `image` key in `values.yaml` and SHALL NOT modify any `image` fields nested under infrastructure service keys (e.g., `dynamodb.image`).
4. WHEN all changed Services have been built and their `values.yaml` files updated, THE Helm_Updater SHALL commit all modified `values.yaml` files to the `main` branch in a single atomic commit with the message `ci: update image tags for <list-of-services> to <IMAGE_TAG>`.
5. WHEN the commit is created, THE Helm_Updater SHALL push the commit to the `main` branch using the `GITHUB_TOKEN` provided by GitHub Actions.
6. IF no Services are marked as changed, THE Helm_Updater SHALL skip the commit and push steps entirely.
7. IF the `values.yaml` file for a Service does not contain an `image.tag` or `image.repository` field, THEN THE Helm_Updater SHALL fail with an error message identifying the missing field and the affected Service name.
8. IF the `image.tag` field in `values.yaml` already equals the current Image_Tag for a Service, THE Helm_Updater SHALL skip the write and commit steps for that Service (idempotency).
9. IF the git push to `main` fails, THE Helm_Updater SHALL pull with rebase and retry the push once. IF the retry also fails, THE Helm_Updater SHALL fail the pipeline with a non-zero exit code.
10. THE Helm_Updater SHALL run only after ALL Image_Builder matrix jobs have succeeded. IF any Image_Builder job fails, THE Helm_Updater SHALL be skipped entirely.

---

### Requirement 6: ArgoCD GitOps Compatibility

**User Story:** As a platform engineer, I want the pipeline's Helm chart updates to be compatible with the existing ArgoCD application definitions, so that ArgoCD can automatically sync and redeploy changed services without manual intervention.

#### Acceptance Criteria

1. THE Helm_Updater SHALL commit updated `values.yaml` files to the `main` branch, which is the `targetRevision: main` monitored by each ArgoCD Application resource.
2. THE CI_CD_Pipeline SHALL NOT delete or rename any file under `src/<service>/chart/`, ensuring that `src/<service>/chart/Chart.yaml` and `src/<service>/chart/values.yaml` exist and are valid after each pipeline run.
3. WHEN the Helm_Updater commits changes, THE CI_CD_Pipeline SHALL configure the Git commit author as a CI bot identity (name: `github-actions[bot]`, email: `41898282+github-actions[bot]@users.noreply.github.com`) to distinguish automated commits from developer commits.
4. THE Image_Builder SHALL verify the pushed image manifest is retrievable from ECR via `aws ecr describe-images` before THE Helm_Updater commits the updated `values.yaml`, ensuring ArgoCD never references a non-existent image.
5. THE CI_CD_Pipeline SHALL add `[skip ci]` to the Helm_Updater commit message to prevent the automated `values.yaml` commit from re-triggering the pipeline in an infinite loop.

---

### Requirement 7: Pipeline Security and Hardening

**User Story:** As a security engineer, I want the pipeline to follow security best practices so that the supply chain is protected and the blast radius of any compromise is minimized.

#### Acceptance Criteria

1. ALL third-party GitHub Actions used in the pipeline SHALL be pinned to a specific commit SHA (e.g., `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`) rather than a mutable tag (e.g., `@v4`), to prevent supply chain attacks via tag mutation.
2. THE CI_CD_Pipeline SHALL set `permissions` at the job level to the minimum required: `id-token: write` for OIDC, `contents: write` for git push, and `packages: none` for all other scopes.
3. THE CI_CD_Pipeline SHALL NOT print or echo the values of any secrets or the assumed role ARN to the workflow logs.
4. ECR repositories created by the pipeline SHALL have image scanning on push enabled (`scanOnPush: true`).
5. THE CI_CD_Pipeline SHALL use `docker buildx` with `--provenance=true` and `--sbom=true` flags to generate and attach SBOM (Software Bill of Materials) and provenance attestations to each pushed image.

---

### Requirement 8: Pipeline Idempotency and Safety

**User Story:** As a developer, I want the pipeline to be safe to re-run without causing unintended side effects, so that retries and manual triggers do not corrupt the deployment state.

#### Acceptance Criteria

1. WHEN the CI_CD_Pipeline is triggered multiple times with the same Git commit SHA, THE Image_Builder SHALL produce an image tagged with the same 7-character SHA.
2. WHEN the CI_CD_Pipeline is triggered multiple times with the same Git commit SHA, THE Helm_Updater SHALL write the same `image.tag` value to `values.yaml`, resulting in no net change to the repository.
3. WHEN the CI_CD_Pipeline runs, THE Helm_Updater SHALL only modify `values.yaml` files for Services that were built in the current run and SHALL NOT revert or alter `values.yaml` files for unchanged Services.
4. IF the ECR repository for a Service already exists, THEN THE Image_Builder SHALL reuse the existing repository and SHALL NOT fail due to a duplicate repository error.
5. IF the ECR repository for a Service already contains an image with the current Image_Tag, THE Image_Builder SHALL skip the build and push steps for that Service and SHALL NOT fail.
6. IF the `image.tag` field in `values.yaml` already equals the current Image_Tag, THE Helm_Updater SHALL skip the write and commit steps for that Service.
