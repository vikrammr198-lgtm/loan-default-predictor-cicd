# Continuous Integration — Loan Default Predictor

## 1. Overview

This project uses **Jenkins** to implement Continuous Integration (CI).

The CI pipeline automatically performs the following steps whenever a change is pushed to the GitHub repository:

```text
Developer
    ↓
GitHub
    ↓
GitHub Webhook
    ↓
Jenkins
    ↓
Checkout
    ↓
Build
    ↓
Test / Lint
    ↓
Docker Build
    ↓
Docker Tag
    ↓
Docker Push
    ↓
Docker Hub
```

The purpose of the CI pipeline is to ensure that every new change is:

- Retrieved from the correct Git branch
- Successfully built
- Checked using linting
- Packaged into a Docker image
- Versioned using the Jenkins build number
- Published to Docker Hub

The Kubernetes deployment is handled as the **CD stage**, which is triggered after the CI stages complete successfully.

---

# 2. Technologies Used

| Technology | Purpose |
|---|---|
| Git | Version control |
| GitHub | Remote source-code repository |
| GitHub Webhook | Notifies Jenkins when code is pushed |
| ngrok | Exposes local Jenkins temporarily for webhook testing |
| Jenkins | Automates the CI/CD pipeline |
| Node.js / npm | Installs dependencies and builds the application |
| ESLint | Performs code-quality checks |
| Docker | Packages the application into a container image |
| Docker Hub | Stores Docker images |
| Kubernetes | Handles deployment and container orchestration |

---

# 3. Repository

The application source code is maintained in GitHub.

The repository used for this project is:

**Repository:** `loan-default-predictor-cicd`

The original application repository was used as the source for the CI/CD implementation.

---

# 4. Git Workflow

Git is used to track changes made to the application.

The general workflow is:

```text
Modify Application
       ↓
git status
       ↓
git add .
       ↓
git commit
       ↓
git push
       ↓
GitHub
```

Basic commands used during development:

```bash
git status
git add .
git commit -m "Description of change"
git push origin main
```

The important point is that a push to the `main` branch becomes the trigger for the automated pipeline.

---

# 5. GitHub Webhook

Jenkins needs to know when a new commit is pushed to GitHub.

For this purpose, a **GitHub Webhook** is configured.

The general flow is:

```text
Git Push
   ↓
GitHub
   ↓
Webhook
   ↓
Jenkins
```

The webhook sends an HTTP request to Jenkins whenever the selected GitHub event occurs.

For this project, the webhook is configured for **push events**.

### Webhook endpoint

The Jenkins webhook endpoint follows this format:

```text
<jenkins-public-url>/github-webhook/
```

When Jenkins is running locally, it is not directly reachable from GitHub.

Therefore, during development, **ngrok** is used to expose the local Jenkins server temporarily.

---

<!-- IMAGE 01: Paste GitHub Webhook configuration and successful delivery screenshot here -->

## GitHub Webhook Screenshot

<img width="1917" height="895" alt="Screenshot 2026-09-22 174404" src="https://github.com/user-attachments/assets/8e0ea979-196b-452e-9713-ee6e0c497069" />

---

# 6. ngrok

Because Jenkins is running locally, GitHub cannot directly send webhook requests to:

```text
http://localhost:8080
```

ngrok provides a temporary public endpoint that forwards requests to the local Jenkins server.

General flow:

```text
GitHub
   ↓
Public ngrok URL
   ↓
Local Jenkins
   ↓
localhost:8080
```

A tunnel is started using:

```bash
ngrok http 8080
```

The generated public URL is then used as part of the GitHub webhook URL.

For example:

```text
https://<ngrok-domain>/github-webhook/
```

ngrok is mainly used here for **local development/testing**. In a production environment, Jenkins would normally be hosted on infrastructure that is directly reachable through an appropriate secure endpoint.

<img width="1917" height="708" alt="Screenshot 2026-09-22 175418" src="https://github.com/user-attachments/assets/497a9bbf-561d-4287-88f6-fd7bda9e4327" />

---

# 7. Jenkins Pipeline

Jenkins is responsible for executing the automated CI pipeline.

The pipeline is stored separately as:

**[`Jenkinsfile`](./Jenkinsfile)**

The Jenkinsfile contains the complete pipeline definition.

The pipeline is divided into stages so that each stage performs a specific responsibility.

---

# 8. Jenkins Pipeline Flow

The complete pipeline is:

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
   ↓
Kubernetes Deploy
```

The first stages perform the CI work.

The final Kubernetes stage acts as the handoff from CI into Continuous Deployment.

---

# 9. Stage 1 — Checkout

The first stage retrieves the source code from GitHub.

The pipeline explicitly checks out the `main` branch.

Conceptually:

```text
Jenkins
   ↓
GitHub
   ↓
main branch
   ↓
Jenkins workspace
```

The Jenkins workspace is cleaned before checkout so that files from an earlier build do not interfere with the current build.

The general commands involved are:

```bash
git checkout main
```

and workspace cleanup is handled by Jenkins.

The Jenkins pipeline uses the Git repository URL and branch information defined in the Jenkinsfile.

---

# 10. Stage 2 — Build

After the source code is checked out, Jenkins installs the project dependencies and creates a production build.

The project uses npm.

The dependency installation is performed using:

```bash
npm ci
```

The production build is performed using:

```bash
npm run build
```

### Why `npm ci`?

`npm ci` installs dependencies based on the lock file and is intended for clean and repeatable CI environments.

### Why build the application?

The build stage verifies that the source code can be converted into the production application successfully.

If the build fails, Jenkins stops the pipeline and later stages are not executed.

---

# 11. Stage 3 — Test / Code Quality Check

The project currently does not contain a dedicated automated test script.

Therefore, the existing linting configuration is used as the CI quality check.

The command is:

```bash
npm run lint
```

ESLint checks the project source code for configured code-quality and formatting-related issues.

The important CI principle is:

```text
Build successful
      +
Lint successful
      ↓
Continue pipeline
```

If the lint stage fails, Jenkins stops the pipeline before creating and publishing the Docker image.

---

# 12. Line Ending Handling

During the CI setup, the project initially encountered line-ending differences between Windows development and Jenkins.

The issue was caused by differences between:

```text
CRLF
```

and:

```text
LF
```

To make the repository consistently use LF line endings, the following `.gitattributes` file was added:

```text
* text=auto eol=lf
```

This allows Git to maintain consistent line-ending behavior across environments.

The file is stored at the root of the repository:

```text
.gitattributes
```

This is particularly useful when development and CI environments use different operating systems or Git configurations.

---

# 13. Stage 4 — Docker Build

After the application successfully builds and passes the quality check, Jenkins creates a Docker image.

The Dockerfile is stored at the root of the repository:

```text
Dockerfile
```

The Docker image contains the production version of the application.

The general Docker build command is:

```bash
docker build -t <image-name>:<version> .
```

In this project, Jenkins uses the Jenkins build number as the image version.

Conceptually:

```text
Build #20
    ↓
Docker image
    ↓
loan-default-predictor:20
```

This provides a unique version for every successful pipeline execution.

---

# 14. Dockerfile

The Dockerfile defines how the application is packaged.

The project uses Node.js as the base image and performs the following general operations:

```text
Node.js base image
       ↓
Set working directory
       ↓
Copy package files
       ↓
Install dependencies
       ↓
Copy application source
       ↓
Build application
       ↓
Expose port 3000
       ↓
Start production server
```

The Dockerfile is maintained separately at:

```text
Dockerfile
```

---

# 15. Docker Image Versioning

Instead of using only a fixed tag such as:

```text
latest
```

the pipeline uses the Jenkins build number.

For example:

```text
Build #18
→ image version 18

Build #19
→ image version 19

Build #20
→ image version 20
```

This creates traceability between:

```text
Jenkins Build
      ↓
Docker Image
      ↓
Docker Hub
      ↓
Kubernetes Deployment
```

For example:

```text
Jenkins Build #20
        ↓
loan-default-predictor:20
        ↓
vikram220057/loan-default-predictor:20
```

This makes it possible to identify which build produced a particular image.

---

# 16. Stage 5 — Docker Tag

After creating the local Docker image, Jenkins tags it with the Docker Hub repository name.

The general command is:

```bash
docker tag <local-image>:<version> <dockerhub-user>/<repository>:<version>
```

For this project, the Docker Hub repository is:

```text
vikram220057/loan-default-predictor
```

Therefore the image follows this structure:

```text
vikram220057/loan-default-predictor:<build-number>
```

---

# 17. Stage 6 — Docker Push

After tagging, Jenkins pushes the image to Docker Hub.

The general command is:

```bash
docker push <dockerhub-user>/<repository>:<version>
```

The pipeline uses Jenkins credentials to authenticate with Docker Hub.

The credentials are stored inside Jenkins rather than directly inside the Jenkinsfile.

The credential ID used by the pipeline is:

```text
dockerhub-credentials
```

The actual Docker Hub password/PAT should never be written directly into the Jenkinsfile or committed to GitHub.

The flow is:

```text
Jenkins
   ↓
Docker Login
   ↓
Docker Hub
   ↓
Push Image
```

---

<!-- IMAGE 04: Paste Docker Hub repository screenshot showing build-number image tags here -->

## Docker Hub Screenshot

The screenshot  show the Docker Hub repository containing multiple build-number tags.
<img width="1917" height="981" alt="Screenshot 2026-09-22 175010" src="https://github.com/user-attachments/assets/b1d35702-6589-4b3c-a48d-6d98699cd263" />
<img width="1917" height="985" alt="Screenshot 2026-09-22 175028" src="https://github.com/user-attachments/assets/91fefa30-2733-4be1-afe4-09a1bdf5c159" />

For example:

```text
1
9
17
19
20
```

The exact tags visible will depend on the builds that have been executed.

---

# 18. Jenkins Credentials

Docker Hub authentication is handled through Jenkins Credentials.

The general approach is:

```text
Docker Hub username
        +
Docker Hub Personal Access Token
        ↓
Jenkins Credentials
        ↓
Jenkins Pipeline
        ↓
Docker Login
        ↓
Docker Push
```

The Personal Access Token is stored securely in Jenkins.

It should not be:

- committed to Git
- written directly inside Jenkinsfile
- placed inside README files
- shared publicly
- included in screenshots

---

# 19. Jenkins Stage View

Jenkins provides a Stage View to visualize the pipeline execution.

The expected successful pipeline looks conceptually like:

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
   ↓
Kubernetes Deploy
```

A successful pipeline shows the stages completing successfully.

---

<!-- IMAGE 02: Paste final Jenkins Stage View screenshot here -->

## Jenkins Stage View Screenshot

<img width="1917" height="1010" alt="Screenshot 2026-09-20 230145" src="https://github.com/user-attachments/assets/ea96f5c6-6917-4765-b926-01114499bb6d" />

The screenshot  show the final successful pipeline with stages similar to:

```text
Checkout → Build → Test → Docker Build → Docker Tag → Docker Push → Kubernetes Deploy
```

All stages should be shown as successful.

---

# 20. Jenkins Build History

Jenkins maintains a history of pipeline executions.

Each execution receives a build number.

For example:

```text
Build #17
Build #18
Build #19
Build #20
```

The build number is also used as the Docker image version.

Therefore:

```text
Jenkins Build #20
        ↓
Docker Image :20
        ↓
Docker Hub Tag :20
        ↓
Kubernetes Deployment :20
```

This provides a simple traceability mechanism across the CI/CD pipeline.

---

<!-- IMAGE 03: Paste Jenkins Build History screenshot here -->

## Jenkins Build History Screenshot

The screenshot show multiple Jenkins builds and their build numbers.

<img width="1917" height="1013" alt="Screenshot 2026-09-20 230035" src="https://github.com/user-attachments/assets/ebf895a3-dd8b-411c-87b6-3ff1faab6428" />
<img width="1917" height="503" alt="Screenshot 2026-09-20 230112" src="https://github.com/user-attachments/assets/659e7535-48fd-40c2-8448-bfd9aa01bebd" />

---

# 21. Failure Handling

One of the main advantages of CI is that a failure prevents the pipeline from continuing.

For example:

```text
Checkout
   ↓
Build
   ↓
Test ❌
   ↓
STOP
```

If the build fails:

```text
Build ❌
   ↓
Docker Build
   ↓
Docker Push
```

will not execute.

Similarly, if Docker image creation fails:

```text
Docker Build ❌
   ↓
Docker Tag
   ↓
Docker Push
```

will not execute.

This prevents an invalid build from being automatically published or deployed.

---

# 22. CI/CD Trigger Flow

The complete trigger mechanism is:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    │ webhook
    ▼
ngrok
    │
    │ forwards request
    ▼
Jenkins
    │
    ├── Checkout
    ├── Build
    ├── Test
    ├── Docker Build
    ├── Docker Tag
    └── Docker Push
            │
            ▼
       Docker Hub
            │
            ▼
   Kubernetes Deployment
```

This creates an automated path from source-code changes to deployment.

---

# 23. Jenkinsfile

The complete Jenkins pipeline is maintained separately so that the pipeline definition remains reusable and version-controlled.

Open the Jenkinsfile here:

**[`Jenkinsfile`](./Jenkinsfile)**

The Jenkinsfile contains:

- Git checkout
- Application build
- Linting
- Docker image creation
- Docker image tagging
- Docker Hub authentication
- Docker image push
- Kubernetes deployment trigger
- Kubernetes rollout verification

---

# 24. CI Responsibilities

The CI part of the project is responsible for:

| Stage | Responsibility |
|---|---|
| Checkout | Retrieve source code |
| Build | Install dependencies and compile/build application |
| Test | Run lint/code-quality checks |
| Docker Build | Create container image |
| Docker Tag | Assign repository and build-number tag |
| Docker Push | Publish image to Docker Hub |

The CI pipeline produces a versioned Docker image that can then be deployed.

---

# 25. CI → CD Handoff

The important boundary between CI and CD is:

```text
CI
│
├── Source checkout
├── Build
├── Test
├── Docker Build
├── Docker Tag
└── Docker Push
        │
        ▼
   Docker Hub Image
        │
        ▼
CD
│
└── Kubernetes Deployment
```

After a Docker image is successfully pushed, the CD stage updates Kubernetes to use that newly created image.

The complete deployment process is documented separately in:

**[`CD.md`](../Continuous%20Deployment/CD.md)**

---

# 26. Final CI Flow

The final CI workflow implemented in this project is:

```text
┌──────────────┐
│   Developer  │
└──────┬───────┘
       │
       │ git push
       ▼
┌──────────────┐
│    GitHub    │
└──────┬───────┘
       │
       │ Webhook
       ▼
┌──────────────┐
│    ngrok     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Jenkins   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Checkout  │
└──────┬───────┘
       ▼
┌──────────────┐
│     Build    │
└──────┬───────┘
       ▼
┌──────────────┐
│ Test / Lint  │
└──────┬───────┘
       ▼
┌──────────────┐
│ Docker Build │
└──────┬───────┘
       ▼
┌──────────────┐
│  Docker Tag  │
└──────┬───────┘
       ▼
┌──────────────┐
│ Docker Push  │
└──────┬───────┘
       ▼
┌──────────────┐
│  Docker Hub  │
└──────┬───────┘
       │
       ▼
   CD / Kubernetes
```

---

# 27. Result

The CI implementation successfully automates the process from a GitHub code change to a versioned Docker image.

A successful pipeline performs:

```text
Git Push
   ↓
GitHub Webhook
   ↓
Jenkins
   ↓
Checkout
   ↓
Build
   ↓
Lint
   ↓
Docker Build
   ↓
Docker Tag
   ↓
Docker Push
   ↓
Docker Hub
   ↓
Kubernetes Deployment
```

This demonstrates an automated CI/CD workflow using GitHub, Jenkins, Docker, Docker Hub, and Kubernetes.
