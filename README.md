# Jenkins → GitHub Actions Migration

A demonstration of migrating a CI/CD pipeline from Jenkins to GitHub Actions.

## Source and Attribution

This project builds on the microservices used in my
[AWS DevOps portfolio project](https://gitlab.com/burcudonmez-devops/services/microservices-demo),
which itself uses the source code from Google's open-source
[Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo)
demo project (attribution already noted there). The `.github/workflows/ci-*.yaml`
files that ship with the original Online Boutique repo are unrelated to
this migration and they run GCP-specific test pipelines and were not touched
here.

The **Jenkinsfile** in each microservice was my own work, written for the
GitLab portfolio project to build and deploy these services on AWS
(EKS/ECR). That Jenkins pipelines is the actual source of this migration.

## Scope

Three services were selected for migration, deliberately chosen to
represent three different runtimes:

| Service | Language | Status |
|---|---|---|
| `cartservice` | .NET / C# | ✅ Migrated and tested |
| `productcatalogservice` | Go | 🔜 Planned (same pattern) |
| `recommendationservice` | Python | 🔜 Planned (same pattern) |

This is intentionally a subset, not the full application. The goal is to
demonstrate the migration approach, not to run the complete Online Boutique
stack. `productcatalogservice` and `recommendationservice` will reuse the
same reusable-workflow pattern described below; each only needs a new
~20-line caller file.

## Architecture

Each service has its own lightweight "caller" workflow (e.g.
`cartservice-ci.yaml`) that:
1. Runs the service's own unit tests (language-specific: `dotnet test`,
   `go test`, `pytest`)
2. Calls a shared **reusable workflow**
   (`reusable-build-scan-push.yml`) that builds the Docker image, scans it
   with Trivy, and pushes it to Docker Hub

The reusable workflow is parameterized by `service_name` and
`dockerfile_context`, so adding a new service only requires a new caller
file and no duplication of the build/scan/push logic.

Image tagging follows the event that triggered the run:
- Pull request → `pr-<number>` (for testing the change before merge)
- Push to `main` → `dev-latest`
- Push of a `v*` tag → the tag itself (release candidate)

You can see an example of a successful GitHub Actions run below:
<img width="1347" height="743" alt="GithubActions_workflow" src="https://github.com/user-attachments/assets/76c8ca12-a6c3-4171-aeae-12dc0e36c42f" />

### Caching

Two layers of caching were added to keep the pipeline fast on repeated runs:
- **NuGet packages** (`actions/cache@v4`), keyed on a hash of the
  `.csproj` files and restores are skipped when dependencies haven't changed.
- **Docker layers** (`cache-from`/`cache-to: type=gha`, `mode=max`), so the
  multi-stage build only re-runs layers affected by the actual code change,
  instead of rebuilding the image from scratch every time.

## Jenkins → GitHub Actions Mapping

| Jenkins concept | GitHub Actions equivalent |
|---|---|
| Jenkinsfile (Groovy) | Workflow YAML |
| Shared library function | Reusable workflow (`workflow_call`) |
| Stage | Job / Step |
| `when { environment ... }` conditional | `if:` condition |
| Credentials (Jenkins credential store) | GitHub Secrets |
| Agent / node | `runs-on: ubuntu-latest` |

## Security

Trivy scans each image for CRITICAL/HIGH vulnerabilities before it is
allowed to be pushed. Results are also uploaded as a SARIF report to
GitHub's **Security tab**, so findings are visible outside the workflow
logs.

Vulnerabilities found by Trivy are visible in the repo's Security tab:
<img width="1328" height="708" alt="SecurityTab_Trivy_scan" src="https://github.com/user-attachments/assets/65438eee-5d6f-46fd-bbb2-989d9420f755" />

While setting this up, the scan surfaced two real issues in the existing
codebase that were fixed as part of this work:
- **Npgsql 7.0.4** had a known SQL injection vulnerability
  ([GHSA-x9vc-6hfv-hg8c](https://github.com/advisories/GHSA-x9vc-6hfv-hg8c)) and
  bumped to 7.0.7, which contains the patch.
- The Alpine-based runtime image had several outdated OS packages with
  known CVEs and added an `apk upgrade` step to the Dockerfile's final stage
  so the image picks up current security patches at build time.

## Deliberately out of scope

- **GitOps/deployment step**: the original Jenkinsfile ends by committing
  an image tag update to a separate GitOps repo, which ArgoCD then picks up
  to deploy. Reproducing that here would require write access to another
  repo and an ArgoCD instance, so this migration stops at "image pushed to
  registry" which the same boundary Jenkins itself had before the GitOps
  handoff.
- **Registry choice**: the original pipeline pushed to AWS ECR. This
  version pushes to Docker Hub instead, since the AWS free-tier credits
  used for the original portfolio project have been exhausted.
