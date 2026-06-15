# Module 8 - Build Automation & CI/CD with Jenkins

**Demo Project:** CI/CD pipeline for a Java Maven application with Jenkins (build, containerize, publish, version automation)

**Technologies used:** Jenkins, Docker, DigitalOcean, Linux, Git, GitHub, Java, Maven, Groovy

**Project Description:**

- Install and run Jenkins on a DigitalOcean Droplet (Jenkins as a Docker container)
- Configure build tools (Maven) and Docker on the Jenkins server
- Create Jenkins credentials for GitHub and Docker Hub
- Implement CI with Pipeline as Code (`Jenkinsfile`): build JAR, build Docker image, push to a private Docker Hub repository
- Explore Freestyle, Pipeline, and Multibranch Pipeline job types
- Integrate a Jenkins Shared Library for reusable build steps
- Trigger builds automatically via GitHub webhook
- Dynamically increment application version, tag images with version + build number, and commit version updates back to Git

*Module 8: Build Automation & CI/CD with Jenkins*

---

## Overview

This repository is the **application and pipeline code** for Module 8 of the DevOps Bootcamp. It contains a small **Spring Boot** Java application (`java-maven-app`) and several **Jenkinsfile** variants that progress from basic Pipeline syntax to a full **CI pipeline** with **dynamic versioning**, **Docker image build/push**, and **Git write-back**.

The work mirrors a typical company setup: a dedicated Jenkins server, credentials for external systems, Pipeline as Code in Git, container images published to a registry, and automation triggered on code changes.

**Related repository:** reusable pipeline steps live in a separate [jenkins-shared-library](https://github.com/ronkaiser/jenkins-shared-library) Git repo (see branch `jenkins-shared-lib` in this project for integration example).

---

## What was implemented

### Infrastructure

- **Jenkins** installed on an **Ubuntu Droplet** on DigitalOcean, run as a **Docker container**
- **Maven** configured in Jenkins (Global Tool Configuration) for Java builds
- **Docker** made available to the Jenkins container (host Docker socket / CLI) for image build and push steps
- **Firewall** rules so Jenkins UI and required ports are reachable as needed for the lab

### CI pipeline (end state on `jenkins-jobs`)

The active pipeline in [java-maven-app/Jenkinsfile](./java-maven-app/Jenkinsfile) on branch **`jenkins-jobs`** runs these stages:

| Stage | Purpose |
|-------|---------|
| **increment version** | Bump the Maven **patch** version in `pom.xml` using `build-helper:parse-version` and `versions:set`; derive `IMAGE_NAME` as `{version}-{BUILD_NUMBER}` |
| **build app** | `mvn clean package` — compile, test, and produce the executable JAR |
| **build image** | `docker build`, `docker login`, `docker push` to **Docker Hub** (`ronkaiser86/demo-app:${IMAGE_NAME}`) |
| **deploy** | Placeholder stage (deployment to a remote host is extended in later modules) |
| **commit version update** | Commit the updated `pom.xml` and push back to Git so the repo records the new version |

### Jenkins job types and learning artifacts

On branch **`main`**, multiple files document the course progression without replacing the live pipeline on `jenkins-jobs`:

| File | Illustrates |
|------|-------------|
| [Jenkinsfile](./java-maven-app/Jenkinsfile) | Full CI pipeline with versioning (evolved alongside `jenkins-jobs`) |
| [Jenkinsfile-pipeline](./java-maven-app/Jenkinsfile-pipeline) | Declarative Pipeline loading [script.groovy](./java-maven-app/script.groovy) for build / image / deploy |
| [Jenkinsfile-syntax](./java-maven-app/Jenkinsfile-syntax) | Parameters, `when` conditions, and user **input** for environment selection |
| [Jenkinsfile-condition](./java-maven-app/Jenkinsfile-condition) | **Multibranch**-style branching: stages gated on `BRANCH_NAME` |
| [freestyle-build.sh](./freestyle-build.sh) | Shell steps used by a **Freestyle** job (build JAR, Docker image, push) |
| Embedded `jenkins-shared-library/` (on `main` only) | Local copy of shared library structure while learning |

Branch **`jenkins-shared-lib`** wires the external shared library into the Pipeline (`buildJar()`, `dockerLogin()`, `dockerPush()`, etc.).

Branch **`feature/payment`** supports **Multibranch Pipeline** demos (separate branch for branch-specific behavior).

### Jenkins Shared Library

Common steps (build JAR, build image, registry login, push) were extracted into [jenkins-shared-library](https://github.com/ronkaiser/jenkins-shared-library) so multiple pipelines can reuse the same Groovy logic. The `jenkins-shared-lib` branch in this repo shows loading that library from GitHub and calling global steps from the Jenkinsfile.

### Automatic triggers

The Jenkins job is configured to run on pushes to the application repository using a **GitHub webhook** (course material uses GitLab; this project uses **GitHub**). Configure the webhook in the GitHub repo settings and the corresponding trigger in the Jenkins job / multibranch project.

To avoid an infinite loop when the pipeline **commits version bumps**, either:

- Push those commits to a dedicated branch (**`jenkins-jobs`**, as implemented), and/or  
- Configure the job to **ignore** commits with message `ci: version bump`, or exclude that branch from automatic builds.

---

## Repository branches

| Branch | Role |
|--------|------|
| **`main`** | Default branch: course exercises, multiple Jenkinsfile variants, and (historically) an embedded shared-library tree for learning |
| **`jenkins-jobs`** | **Primary branch for the live Jenkins job** — contains the Jenkinsfile used day-to-day; receives automated `ci: version bump` commits from the pipeline |
| **`jenkins-shared-lib`** | Pipeline that consumes the external Jenkins Shared Library from GitHub |
| **`feature/payment`** | Feature branch for Multibranch Pipeline demonstrations |

When cloning or configuring Jenkins SCM, point the **Multibranch** or **Pipeline** job at **`jenkins-jobs`** unless you are reproducing a specific exercise from `main`.

---

## Application layout

```text
devops-08-automation-ci-cd-with-jenkins/
├── java-maven-app/
│   ├── src/                    # Spring Boot application (Java 17)
│   ├── pom.xml                 # Maven coordinates; version updated by CI
│   ├── Dockerfile              # Packages target/java-maven-app-*.jar
│   ├── Jenkinsfile             # Pipeline definition (branch-dependent)
│   ├── Jenkinsfile-pipeline    # Pipeline + script.groovy (main)
│   ├── Jenkinsfile-syntax      # Parameters, when, input (main)
│   ├── Jenkinsfile-condition   # Branch conditions (main)
│   └── script.groovy           # Local Groovy helpers (buildJar, buildImage, …)
├── freestyle-build.sh          # Freestyle job helper (repo root on jenkins-jobs)
└── nodesource_setup.sh         # Optional Node.js setup on the agent
```

The Docker image is built from [java-maven-app/Dockerfile](./java-maven-app/Dockerfile) after `mvn package` produces the JAR under `target/`.

---

## Jenkins configuration checklist

Use this when reproducing the setup on a new Jenkins instance or documenting for your team.

### Tools

- **Maven** — name in Jenkinsfile: `Maven` (on `jenkins-jobs`) or `maven-3.9` (on some learning branches); must match **Global Tool Configuration**
- **Docker** — CLI available on the agent; Jenkins container typically needs access to the host Docker daemon

### Credentials (Global scope)

| Credential ID | Type | Used for |
|---------------|------|----------|
| `docker-hub-repo` | Username with password | `docker login` and push to **Docker Hub** |
| `github-pat-devops-08` | Username with password (see below) | Git push in **commit version update** stage on `jenkins-jobs` |
| `github-credentials` | Username with password / PAT | Used on other branches (e.g. `jenkins-shared-lib`) for SCM and library fetch |

### GitHub: Personal Access Token (not account password)

This project uses **GitHub**, not GitLab. GitHub **does not accept account passwords** for Git over HTTPS. For the **commit version update** stage you need a **Personal Access Token (PAT)** (classic or fine-grained with `contents: write` on the repo).

Store it in Jenkins as **Username with password**:

- **Username:** your GitHub username  
- **Password:** the **PAT** (not your GitHub login password)  
- **ID:** `github-pat-devops-08` (must match the Jenkinsfile on `jenkins-jobs`)

### Git write-back stage (GitHub-specific)

On **`jenkins-jobs`**, the final stage pushes the version bump to Git. Implementation details worth knowing for maintenance:

1. **`git remote set-url`** uses a **hardcoded HTTPS repository URL** (no credentials embedded in the remote URL on disk):

   `https://github.com/ronkaiser/devops-08-automation-ci-cd-with-jenkins.git`

2. **`git push`** passes credentials in the push URL and targets **`jenkins-jobs`** explicitly:

   `git push https://${USER}:${PASS}@github.com/ronkaiser/devops-08-automation-ci-cd-with-jenkins.git HEAD:jenkins-jobs`

   This keeps version commits on a branch used for CI output and reduces webhook re-trigger loops on `main`.

3. Commit message is fixed: `ci: version bump`.

If you fork the repo or rename remotes, update **both** the `set-url` and `push` URLs in [java-maven-app/Jenkinsfile](./java-maven-app/Jenkinsfile) on `jenkins-jobs`.

---

## Image tagging and versioning

- **Maven version** in `pom.xml` is incremented on each pipeline run (patch / incremental).
- **Docker tag** format: `{pom-version}-{BUILD_NUMBER}` (e.g. `1.1.6-42`), stored in env `IMAGE_NAME`.
- Images are pushed to: `ronkaiser86/demo-app:${IMAGE_NAME}`.

Adjust registry namespace and image name in the **build image** stage when using your own Docker Hub or private registry.

---

## Running the pipeline locally (without Jenkins)

From `java-maven-app`:

```bash
mvn clean package
docker build -t my-app:local .
```

Run the JAR directly:

```bash
java -jar target/java-maven-app-*.jar
```

---

## Prerequisites

- DigitalOcean Droplet (or other Linux host) for Jenkins
- Docker and Docker Compose on the Jenkins host
- GitHub repository with webhook access to your Jenkins URL
- Docker Hub account (or private registry) for image push
- Java **17** and Maven **3.9+** aligned with [pom.xml](./java-maven-app/pom.xml)

---

## References

- Jenkins: [Pipeline](https://www.jenkins.io/doc/book/pipeline/), [Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/), [Credentials](https://www.jenkins.io/doc/book/using/using-credentials/)
- GitHub: [Managing personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- Docker: [docker build](https://docs.docker.com/reference/cli/docker/build/), [docker push](https://docs.docker.com/reference/cli/docker/push/)
- Jenkins Shared Library repo: [github.com/ronkaiser/jenkins-shared-library](https://github.com/ronkaiser/jenkins-shared-library)
