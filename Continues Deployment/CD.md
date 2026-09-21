# Continuous Deployment — Loan Default Predictor

## 1. Overview

This project uses **Kubernetes** to implement Continuous Deployment (CD).

The CD process automatically takes the Docker image produced by the CI pipeline and deploys the new version of the application to the Kubernetes cluster.

The deployment flow is:

```text
Jenkins CI
    ↓
Docker Hub
    ↓
New Docker Image
    ↓
Kubernetes
    ↓
Deployment
    ↓
Rolling Update
    ↓
New Pod
    ↓
Running Application
```

The purpose of the CD pipeline is to automatically deploy a successfully built and published Docker image without manually recreating the application deployment.

---

# 2. Technologies Used

| Technology | Purpose |
|---|---|
| Kubernetes | Container orchestration and deployment |
| Docker Desktop Kubernetes | Local Kubernetes cluster |
| Docker | Container runtime and image creation |
| Docker Hub | Stores application images |
| kubectl | Command-line tool for Kubernetes |
| Jenkins | Triggers and automates deployment |

---

# 3. Kubernetes Environment

For this project, Kubernetes is running locally through **Docker Desktop**.

The Kubernetes cluster uses a single-node development environment.

The general architecture is:

```text
Docker Desktop
      ↓
Kubernetes Cluster
      ↓
Control Plane
      ↓
Application Deployment
      ↓
Application Pod
```

The Kubernetes context used during development is:

```text
docker-desktop
```

The Kubernetes node is verified using:

```bash
kubectl get nodes
```

A healthy cluster should show the node in:

```text
Ready
```

state.

---

# 4. Kubernetes Deployment

A Kubernetes **Deployment** manages the application Pods.

The Deployment is defined in:

**[`deployment.yaml`](./deployment.yaml)**

The Deployment describes:

- Application name
- Number of replicas
- Pod labels
- Container name
- Docker image
- Container port

The basic relationship is:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pod
     ↓
Container
     ↓
Application
```

---

# 5. Deployment Configuration

The Kubernetes Deployment uses the Docker image published by the CI pipeline.

The image follows this format:

```text
vikram220057/loan-default-predictor:<build-number>
```

For example:

```text
vikram220057/loan-default-predictor:20
```

The build number allows the Kubernetes deployment to identify which CI build is being deployed.

The initial Kubernetes configuration is maintained in:

**[`deployment.yaml`](./deployment.yaml)**

---

# 6. Applying the Deployment

The Kubernetes Deployment can be created or updated using:

```bash
kubectl apply -f deployment.yaml
```

This tells Kubernetes to create or update the resources described in the YAML file.

After applying the configuration, the Deployment can be checked using:

```bash
kubectl get deployments
```

A more detailed view can be obtained using:

```bash
kubectl get deployment loan-default-predictor -o wide
```

---

# 7. Kubernetes Pods

A Pod is the smallest deployable unit in Kubernetes.

The application container runs inside a Kubernetes Pod.

The relationship is:

```text
Deployment
     ↓
   Pod
     ↓
Container
     ↓
Loan Default Predictor
```

Pods can be viewed using:

```bash
kubectl get pods
```

A successfully deployed application should show:

```text
READY     1/1
STATUS    Running
```

The exact Pod name changes whenever Kubernetes performs a new rollout.

---

<!-- IMAGE 06: Paste Kubernetes Pod screenshot here -->

## Kubernetes Pod Screenshot

The screenshot should show:

```text
kubectl get pods
```

with the application Pod in:

```text
Running
```

state.

---

# 8. Kubernetes Service

A Kubernetes Service provides a stable network endpoint for the application Pods.

The Service configuration is maintained in:

**[`service.yaml`](./service.yaml)**

The basic flow is:

```text
User
  ↓
Kubernetes Service
  ↓
Pod
  ↓
Container
  ↓
Application
```

The Service selects Pods using their labels.

The application uses port:

```text
3000
```

The Service maps traffic to the application's container port.

---

# 9. Applying the Service

The Service can be created or updated using:

```bash
kubectl apply -f service.yaml
```

The Service can be checked using:

```bash
kubectl get services
```

The Service used in this project is configured as a:

```text
NodePort
```

This allows the Kubernetes Service to expose the application through a node port within the local Kubernetes environment.

---

# 10. Service and Pod Relationship

The Service uses a selector to identify the application Pods.

Conceptually:

```text
Service
   │
   │ selector
   ▼
app=loan-default-predictor
   │
   ▼
Application Pod
```

This means the Service does not depend on a specific Pod name.

When Kubernetes replaces a Pod during a rollout, the Service can continue directing traffic to the appropriate Pod.

---

# 11. Local Application Access

The Kubernetes Service was tested locally using port forwarding.

The general command is:

```bash
kubectl port-forward service/<service-name> <local-port>:<service-port>
```

For this project:

```bash
kubectl port-forward service/loan-default-predictor-service 3000:3000
```

The application can then be accessed through:

```text
http://localhost:3000
```

The request flow becomes:

```text
Browser
   ↓
localhost:3000
   ↓
kubectl port-forward
   ↓
Kubernetes Service
   ↓
Application Pod
   ↓
Container Port 3000
   ↓
Application
```

---

<!-- IMAGE 08: Paste running application screenshot here -->

## Running Application Screenshot

The screenshot should show the Loan Default Predictor application running successfully in the browser after Kubernetes port forwarding.

---

# 12. Continuous Deployment from Jenkins

After the CI pipeline successfully builds and pushes a Docker image to Docker Hub, Jenkins performs the Kubernetes deployment stage.

The deployment stage is defined in:

**[`Jenkinsfile`](../Continuous%20Integration/Jenkinsfile)**

The general flow is:

```text
CI Pipeline
     ↓
Docker Build
     ↓
Docker Push
     ↓
Docker Hub
     ↓
Kubernetes Deploy
```

Jenkins updates the Kubernetes Deployment to use the Docker image corresponding to the current Jenkins build number.

---

# 13. Updating the Kubernetes Image

The Kubernetes Deployment image is updated using:

```bash
kubectl set image deployment/<deployment-name> <container-name>=<image>:<version>
```

For this project, the image version corresponds to:

```text
Jenkins BUILD_NUMBER
```

Conceptually:

```text
Jenkins Build #20
       ↓
Docker Image :20
       ↓
Docker Hub :20
       ↓
Kubernetes Deployment :20
```

This connects the CI build directly to the deployed application version.

---

# 14. Kubernetes Rolling Update

Kubernetes does not simply delete the existing application and start from zero.

When the Deployment image is changed, Kubernetes performs a **rolling update**.

The general process is:

```text
Old Pod
   +
New Image
   ↓
New Pod Created
   ↓
New Pod Becomes Ready
   ↓
Old Pod Terminated
   ↓
New Version Running
```

This allows Kubernetes to update the application in a controlled manner.

---

# 15. Rollout Status

Jenkins verifies that Kubernetes successfully completes the deployment using:

```bash
kubectl rollout status deployment/<deployment-name>
```

For this project:

```bash
kubectl rollout status deployment/loan-default-predictor
```

A successful rollout produces a message indicating that the Deployment has successfully rolled out.

If the rollout does not complete successfully, Jenkins treats the deployment stage as failed.

---

<!-- IMAGE 07: Paste Kubernetes rollout screenshot here -->

## Kubernetes Rollout Screenshot

The screenshot should show the rollout command and successful rollout message.

For example:

```text
deployment "loan-default-predictor" successfully rolled out
```

---

# 16. Verifying the Deployment

After deployment, the running Pods can be checked using:

```bash
kubectl get pods
```

The Deployment can be checked using:

```bash
kubectl get deployment
```

The deployed image can be verified using:

```bash
kubectl get deployment loan-default-predictor -o wide
```

This allows us to verify that Kubernetes is using the expected Docker image version.

---

<!-- IMAGE 05: Paste Kubernetes Deployment screenshot here -->

## Kubernetes Deployment Screenshot

The screenshot should show:

```bash
kubectl get deployment loan-default-predictor -o wide
```

The output should include the deployed Docker image and show the Deployment as available.

---

# 17. Verifying the Deployed Image

The image running inside the Pod can also be checked using:

```bash
kubectl get pod <pod-name> -o wide
```

The container image can be inspected using:

```bash
kubectl get pod <pod-name> -o jsonpath="{.spec.containers[0].image}"
```

For example:

```text
vikram220057/loan-default-predictor:20
```

This confirms that the Pod is using the Docker image produced by the corresponding CI build.

---

# 18. Complete CD Flow

The complete Continuous Deployment flow is:

```text
Jenkins
   │
   │ Successful CI build
   ▼
Docker Hub
   │
   │ Versioned Docker image
   ▼
Kubernetes
   │
   │ Update Deployment image
   ▼
Deployment
   │
   ▼
ReplicaSet
   │
   ▼
New Pod
   │
   │ Rollout
   ▼
Running Application
```

---

# 19. End-to-End CI/CD Flow

The complete project workflow combines Continuous Integration and Continuous Deployment.

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    │ webhook
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
             │ new image
             ▼
      Kubernetes Deploy
             │
             ▼
        Deployment
             │
             ▼
        Rolling Update
             │
             ▼
            Pod
             │
             ▼
        Kubernetes Service
             │
             ▼
       Running Application
```

---

# 20. Deployment Versioning

The project uses Jenkins build numbers as Docker image versions.

For example:

```text
Jenkins Build #19
        ↓
Docker Image :19
        ↓
Kubernetes Deployment :19
```

When the next successful build occurs:

```text
Jenkins Build #20
        ↓
Docker Image :20
        ↓
Kubernetes Deployment :20
```

This means each deployment can be associated with a specific CI build.

---

# 21. Rolling Deployment Example

Suppose the currently running application uses:

```text
vikram220057/loan-default-predictor:19
```

A new GitHub push triggers Jenkins.

The pipeline creates:

```text
vikram220057/loan-default-predictor:20
```

Jenkins then updates Kubernetes:

```text
Version 19
    ↓
Version 20
```

Kubernetes creates a new Pod using version `20`.

After the new Pod becomes ready, the old Pod running version `19` is terminated.

The final state becomes:

```text
Version 20
   ↓
Running Pod
   ↓
Kubernetes Service
   ↓
Application
```

---

# 22. Deployment Verification

A successful deployment can be verified using the following commands:

```bash
kubectl get pods
```

```bash
kubectl get deployment
```

```bash
kubectl get services
```

```bash
kubectl rollout status deployment/loan-default-predictor
```

These commands provide different views of the deployment:

| Command | Purpose |
|---|---|
| `kubectl get pods` | Check running application Pods |
| `kubectl get deployment` | Check Deployment state |
| `kubectl get services` | Check Service |
| `kubectl rollout status` | Check rollout completion |

---

# 23. Failure Handling

Kubernetes deployment is considered successful only when the rollout completes successfully.

The deployment flow is:

```text
Update Image
     ↓
Create New Pod
     ↓
Pod Ready?
   /     \
 No       Yes
 ↓         ↓
Failure   Continue
           ↓
      Old Pod Removed
           ↓
      Rollout Complete
```

Jenkins waits for the rollout to complete.

If Kubernetes cannot successfully deploy the new image, the deployment stage fails.

---

# 24. Local Kubernetes Limitation

The Kubernetes cluster used in this project is running locally through Docker Desktop.

Therefore, the application is not automatically available to users on the public Internet.

Local access is achieved using:

```bash
kubectl port-forward service/loan-default-predictor-service 3000:3000
```

and then:

```text
http://localhost:3000
```

For a production deployment, Kubernetes could instead run on a cloud platform such as AWS EKS, with an appropriate external load balancer or ingress configuration.

---

# 25. Kubernetes Configuration Files

The Kubernetes configuration is maintained separately from the CI pipeline.

### Deployment

**[`deployment.yaml`](./deployment.yaml)**

Defines:

- Deployment
- Replica count
- Pod labels
- Container
- Docker image
- Container port

### Service

**[`service.yaml`](./service.yaml)**

Defines:

- Kubernetes Service
- Pod selector
- Service port
- Target port
- NodePort exposure

---

# 26. Jenkins and Kubernetes Integration

The important integration between Jenkins and Kubernetes is:

```text
Jenkins
   │
   │ kubectl
   ▼
Kubernetes API
   │
   ▼
Deployment
   │
   ▼
Pod
```

Jenkins does not build the Kubernetes cluster.

Instead, Jenkins acts as the automation layer that tells Kubernetes which application image should be deployed.

---

# 27. Final CD Architecture

```text
                  ┌─────────────────────┐
                  │       GitHub        │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │      Jenkins        │
                  │   CI + CD Pipeline  │
                  └──────────┬──────────┘
                             │
                       Docker Image
                             │
                             ▼
                  ┌─────────────────────┐
                  │     Docker Hub      │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │     Kubernetes      │
                  │     Deployment      │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │        Pod          │
                  │  Loan Predictor     │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │      Service        │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │   Running Web App   │
                  └─────────────────────┘
```

---

# 28. Result

The Continuous Deployment implementation successfully connects the CI pipeline with Kubernetes.

A new application change follows this path:

```text
GitHub Push
    ↓
Jenkins
    ↓
Build
    ↓
Lint
    ↓
Docker Image
    ↓
Docker Hub
    ↓
Kubernetes
    ↓
Deployment Update
    ↓
Rolling Update
    ↓
New Pod
    ↓
Service
    ↓
Running Application
```

The deployment is therefore automated from the successful CI build through to the updated Kubernetes application.

---

```

Together, these files document the complete Continuous Integration and Continuous Deployment workflow of the project.
