# Continuous Integration (CI)

This document describes the Continuous Integration implementation developed for the **Loan Default Predictor** project.

The objective was to connect source control, automated validation, application building, containerization, and container image publishing into a single CI workflow.

The implementation uses GitHub, Jenkins, Docker, Docker Hub, GitHub Webhooks, and ngrok.

---

## CI Architecture

```text
                    Developer
                        │
                        │ git push
                        ▼
                   ┌─────────┐
                   │ GitHub  │
                   └────┬────┘
                        │
                        │ Webhook
                        ▼
                   ┌─────────┐
                   │  ngrok  │
                   └────┬────┘
                        │
                        ▼
                   ┌─────────┐
                   │ Jenkins │
                   └────┬────┘
                        │
            ┌───────────┼────────────┐
            │           │            │
            ▼           ▼            ▼
        Checkout      Build        Test
                                      │
                                      ▼
                              Docker Image Build
                                      │
                                      ▼
                                 Image Tagging
                                      │
                                      ▼
                                Docker Hub Push
                                      │
                                      ▼
                              Versioned Image
```

The resulting container image becomes the input for the Continuous Deployment stage.

---

# 1. Source Control with Git

Git was used to manage the application's source code and track changes throughout development.

GitHub was used as the remote repository.

### Common Git operations

```bash
git status
git add <files>
git commit -m "<message>"
git push <remote> <branch>
git pull
git branch
git checkout <branch>
```

The Jenkins pipeline was configured to work with the project's `main` branch.

The important CI concept here is:

```text
Local Change
     ↓
Git Commit
     ↓
Git Push
     ↓
GitHub
```

A push to GitHub becomes the starting point for the automated CI workflow.

---

# 2. GitHub Webhook Integration

A GitHub Webhook was configured to notify Jenkins when changes were pushed to the repository.

The workflow is:

```text
Developer
    ↓
git push
    ↓
GitHub
    ↓
Webhook Event
    ↓
Jenkins Pipeline Trigger
```

This removes the need to manually start Jenkins for every source-code change.

The webhook was configured for push events and connected to the Jenkins webhook endpoint.

---

# 3. Local Jenkins Access with ngrok

Jenkins was running locally during development.

Since GitHub needs to communicate with Jenkins from outside the local machine, **ngrok** was used to provide a temporary public endpoint.

General usage:

```bash
ngrok http <jenkins-port>
```

The resulting endpoint was configured as the GitHub Webhook destination.

The relationship is:

```text
GitHub
   │
   │ Webhook
   ▼
ngrok public endpoint
   │
   ▼
Local Jenkins
```

ngrok was used specifically to bridge GitHub's webhook communication with the locally hosted Jenkins instance.

---

# 4. Jenkins

Jenkins acts as the CI automation server.

The pipeline was implemented using a **Jenkins Declarative Pipeline** written in Groovy.

The pipeline was divided into independent stages:

```text
Checkout
   ↓
Build
   ↓
Test
   ↓
Docker Build
   ↓
Docker Tag
   ↓
Docker Push
```

Each stage represents a specific part of the CI process.

---

# 5. Jenkins Workspace

Jenkins maintains a workspace where the repository is checked out and the CI operations are performed.

The pipeline cleans the workspace before obtaining the latest source code.

Conceptually:

```text
Previous Workspace
        ↓
   Workspace Cleanup
        ↓
   Fresh Checkout
        ↓
   Current Source Code
```

This helps avoid stale files from previous builds affecting the current build.

---

# 6. Source Checkout

The first Jenkins stage obtains the latest source code from GitHub.

The pipeline checks out the configured branch into the Jenkins workspace.

Conceptually:

```text
GitHub Repository
       │
       ▼
Jenkins Checkout
       │
       ▼
Jenkins Workspace
```

This ensures that subsequent stages operate on the source code associated with the triggered build.

---

# 7. Dependency Installation

The project uses npm for dependency management.

The CI pipeline installs dependencies using the project's lock file.

General command:

```bash
npm ci
```

Using a lock file helps provide a reproducible dependency installation during automated builds.

---

# 8. Application Build

After dependencies are installed, the application is built.

General command:

```bash
npm run build
```

The purpose of this stage is to verify that the application can successfully produce its production build output.

The flow is:

```text
Source Code
     ↓
Dependencies
     ↓
Application Build
     ↓
Production Build Output
```

A failed build prevents the pipeline from progressing to the containerization stages.

---

# 9. Code Quality Validation

The pipeline includes a quality-check stage using ESLint.

General command:

```bash
npm run lint
```

The purpose of this stage is to identify code-quality problems before the application is packaged and published.

The CI flow therefore becomes:

```text
Checkout
   ↓
Dependency Installation
   ↓
Build
   ↓
Lint / Quality Check
   ↓
Docker
```

Only after the application passes the CI checks does the pipeline proceed to containerization.

---

# 10. Docker Containerization

After the application passes the build and quality checks, Docker is used to package the application into a container image.

The project contains a Dockerfile defining the application runtime environment.

General Docker image creation:

```bash
docker build -t <image-name>:<tag> .
```

Conceptually:

```text
Application
     +
Dependencies
     +
Runtime Environment
     ↓
Docker Image
```

The resulting image contains the application and the environment required to run it.

---

# 11. Docker Image Tagging

The CI pipeline generates a versioned Docker image.

Instead of relying on one fixed image version, the Jenkins build identifier is used as part of the image tag.

Conceptually:

```text
Jenkins Build
      ↓
Build Number
      ↓
Docker Image Tag
```

For example:

```text
application:20
```

The exact number changes with each Jenkins build.

This provides traceability between:

```text
Jenkins Build
      ↕
Docker Image
```

A specific container image can therefore be associated with the CI build that produced it.

---

# 12. Docker Hub

Docker Hub is used as the container registry.

After the Docker image is built and tagged, it is pushed to the registry.

General command:

```bash
docker push <registry-user>/<image>:<tag>
```

The resulting flow is:

```text
Jenkins
   ↓
Docker Build
   ↓
Docker Tag
   ↓
Docker Push
   ↓
Docker Hub
```

Docker Hub acts as the storage and distribution point for the versioned container images.

---

# 13. Jenkins Credentials

Authentication for Docker Hub was handled through the Jenkins Credentials system.

The pipeline references stored credentials rather than embedding a password or access token directly in the source code.

Conceptually:

```text
Jenkins Credentials
        │
        ▼
Pipeline Authentication
        │
        ▼
Docker Registry
```

Sensitive authentication information is not committed to the GitHub repository.

---

# 14. Jenkins Declarative Pipeline

The CI implementation uses a Jenkins Declarative Pipeline.

The following represents the structure used in the project without exposing account-specific configuration:

```groovy
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                // Obtain source code from Git repository
            }
        }

        stage('Build') {
            steps {
                // Install dependencies
                // Build application
            }
        }

        stage('Test') {
            steps {
                // Run code-quality / validation checks
            }
        }

        stage('Docker Build') {
            steps {
                // Build Docker image using Jenkins build identifier
            }
        }

        stage('Docker Tag') {
            steps {
                // Tag image for container registry
            }
        }

        stage('Docker Push') {
            steps {
                // Authenticate using Jenkins Credentials
                // Push image to container registry
            }
        }
    }
}
```

The Kubernetes deployment stage is intentionally documented separately as part of **Continuous Deployment**.

---

# 15. CI Pipeline Flow

The complete CI flow implemented in the project is:

```text
                    GitHub
                       │
                       │ Push
                       ▼
                GitHub Webhook
                       │
                       ▼
                     ngrok
                       │
                       ▼
                    Jenkins
                       │
                       ▼
                   Checkout
                       │
                       ▼
             Dependency Installation
                       │
                       ▼
                     Build
                       │
                       ▼
               Quality Validation
                       │
                       ▼
                 Docker Build
                       │
                       ▼
                 Docker Tag
                       │
                       ▼
                Docker Push
                       │
                       ▼
                  Docker Hub
```

At the end of this process, a versioned container image is available in the registry.

---

# 16. Build Versioning

One of the important implementation decisions was using the Jenkins build number to version Docker images.

For example:

```text
Build #18
    ↓
Image :18

Build #19
    ↓
Image :19

Build #20
    ↓
Image :20
```

This provides a simple relationship between the CI pipeline and the generated container images.

It also allows a deployment to identify exactly which CI build produced the image.

---

# 17. CI Validation

The CI pipeline was tested through multiple Jenkins builds.

During development, the pipeline encountered issues involving:

- Source checkout configuration
- Line-ending consistency
- Docker image availability
- Formatting and lint validation
- Local tool integration

These issues were resolved during implementation and the pipeline was subsequently executed successfully.

The purpose of documenting these issues here is to show that the CI workflow was developed and validated iteratively rather than treating the final pipeline as a static configuration.

Detailed troubleshooting steps are intentionally omitted from this documentation.

---

# 18. Jenkins Stage View

Jenkins Stage View was used to observe the execution of the CI/CD pipeline.

The final successful pipeline shows the progression through the major stages.

### Stage View — Successful Pipeline

<!-- PASTE JENKINS STAGE VIEW SCREENSHOT HERE -->

The Stage View provides visual evidence of the successful execution of the pipeline stages.

---

### Additional Stage View

<!-- PASTE JENKINS STAGE VIEW SCREENSHOT HERE -->

---

### Pipeline History

<!-- PASTE JENKINS BUILD HISTORY SCREENSHOT HERE -->

The build history demonstrates the iterative development and validation of the pipeline.

---

# 19. CI Result

The completed CI workflow provides an automated path from source-code changes to a versioned container image.

```text
Source Code
     ↓
GitHub
     ↓
Webhook
     ↓
Jenkins
     ↓
Build
     ↓
Quality Check
     ↓
Docker Image
     ↓
Versioned Image
     ↓
Docker Hub
```

The resulting image is then consumed by the Continuous Deployment workflow.

---

# 20. CI → CD Boundary

The boundary between Continuous Integration and Continuous Deployment in this project is the container image stored in Docker Hub.

```text
             CONTINUOUS INTEGRATION
                     │
                     ▼
              Versioned Image
                     │
                     ▼
                 Docker Hub
                     │
                     │
             CI / CD Boundary
                     │
                     ▼
             CONTINUOUS DEPLOYMENT
```

CI is responsible for producing and publishing the deployment artifact.

CD is responsible for taking that artifact and updating the running Kubernetes application.

---

# 21. Key Concepts Demonstrated

This implementation provided practical experience with:

- Git-based source control
- GitHub Webhooks
- Jenkins automation
- Declarative Jenkins Pipelines
- Groovy pipeline syntax
- Automated application builds
- Code-quality validation
- Docker containerization
- Docker image tagging
- Container registries
- Jenkins credential management
- CI build versioning
- CI/CD workflow integration

The primary learning objective was understanding how individual DevOps tools connect together to form a complete automated software delivery workflow.

---
