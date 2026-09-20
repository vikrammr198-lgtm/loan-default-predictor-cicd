# Continuous Deployment (CD)

This document describes the Continuous Deployment implementation developed for the **Loan Default Predictor** project.

The objective of the CD workflow is to take the versioned container image produced during Continuous Integration, deploy it to Kubernetes, perform a rolling update, and verify that the new application version is running successfully.

---

## CD Workflow

```text
Docker Hub
    │
    │ Versioned Docker Image
    ▼
Jenkins
    │
    │ Deployment Update
    ▼
Kubernetes
    │
    ▼
Deployment
    │
    ▼
Pod
    │
    ▼
Service
    │
    ▼
Application
```

---

## Technologies Used

- Jenkins
- Docker
- Docker Hub
- Kubernetes
- kubectl
- Docker Desktop Kubernetes

---

## 1. CI → CD Connection

The container image produced by the CI pipeline acts as the deployment artifact for CD.

```text
              CONTINUOUS INTEGRATION
                       │
                       ▼
                Docker Image
                       │
                       ▼
                  Docker Hub
                       │
                       │
                CI / CD Boundary
                       │
                       ▼
              CONTINUOUS DEPLOYMENT
                       │
                       ▼
                  Kubernetes
```

The CI process is responsible for building, validating, versioning, and publishing the container image.

The CD process takes that image and updates the running Kubernetes application.

---

## 2. Kubernetes

Kubernetes is used as the container orchestration platform for the application.

The project was deployed to a Kubernetes cluster provided through Docker Desktop.

Kubernetes manages:

- Application Pods
- Deployments
- Services
- Container image versions
- Rolling updates
- Application availability

The deployment architecture is:

```text
Kubernetes Cluster
       │
       ▼
  Deployment
       │
       ▼
    ReplicaSet
       │
       ▼
      Pod
       │
       ▼
   Container
```

---

## 3. Kubernetes Deployment

A Kubernetes Deployment defines the desired state of the application.

It specifies concepts such as:

- Application identity
- Number of replicas
- Pod labels
- Container image
- Container port

A simplified Deployment structure is:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <application>
spec:
  replicas: <number>
  selector:
    matchLabels:
      app: <application>
  template:
    metadata:
      labels:
        app: <application>
    spec:
      containers:
        - name: <container>
          image: <registry>/<image>:<tag>
          ports:
            - containerPort: <application-port>
```

The Deployment controller continuously works toward maintaining the desired state.

---

## 4. Kubernetes Service

A Kubernetes Service provides a stable network endpoint for the application Pods.

The relationship is:

```text
Client
  │
  ▼
Service
  │
  ▼
Pod
  │
  ▼
Container
```

The Service selects Pods using labels and forwards traffic to the appropriate container port.

For the local Kubernetes environment, a `NodePort` Service was used.

A simplified Service structure is:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <application-service>
spec:
  selector:
    app: <application>
  ports:
    - protocol: TCP
      port: <service-port>
      targetPort: <container-port>
  type: NodePort
```

The Service provides a stable access point even when Pods are replaced during a deployment.

---

## 5. Kubernetes Image Update

After a new Docker image is successfully published by CI, the Kubernetes Deployment needs to reference the new image version.

A Kubernetes image update can be performed using:

```bash
kubectl set image deployment/<deployment> \
<container>=<registry>/<image>:<tag>
```

Conceptually:

```text
Jenkins Build
     │
     ▼
New Image Tag
     │
     ▼
Docker Hub
     │
     ▼
Kubernetes Deployment
     │
     ▼
Image Updated
```

The image version is associated with the CI build that produced it.

This creates traceability between:

```text
CI Build
   ↓
Docker Image
   ↓
Kubernetes Deployment
```

---

## 6. Jenkins → Kubernetes Integration

Jenkins was configured to interact with the local Kubernetes cluster.

The deployment stage is responsible for:

1. Updating the Kubernetes Deployment with the newly produced image.
2. Waiting for Kubernetes to complete the rollout.
3. Marking the Jenkins deployment stage as successful only after the rollout completes successfully.

The general flow is:

```text
Docker Push
     │
     ▼
Docker Hub
     │
     ▼
Jenkins Kubernetes Deploy Stage
     │
     ▼
Kubernetes Deployment Update
     │
     ▼
Rollout
```

---

## 7. Automated Deployment

The CD stage follows the Docker image publishing stage.

```text
Docker Build
      ↓
Docker Tag
      ↓
Docker Push
      ↓
Docker Hub
      ↓
Kubernetes Image Update
      ↓
Kubernetes Rollout
```

This removes the need to manually update the Kubernetes Deployment for every successful CI build.

---

## 8. Rolling Updates

When the image referenced by a Kubernetes Deployment changes, Kubernetes performs a rolling update.

Conceptually:

```text
Current Version
      │
      │ New Image
      ▼
New Pod Created
      │
      ▼
New Pod Becomes Ready
      │
      ▼
Old Pod Terminated
      │
      ▼
New Version Running
```

During the implementation, Kubernetes successfully replaced the previous Pod with the Pod running the newly generated image.

The Jenkins pipeline waits for this rollout to complete.

---

## 9. Rollout Verification

The rollout can be monitored using:

```bash
kubectl rollout status deployment/<deployment>
```

The command waits for the Deployment to reach its desired state.

Conceptually:

```text
Deployment Update
       │
       ▼
Rollout Started
       │
       ▼
New Pod Created
       │
       ▼
New Pod Ready
       │
       ▼
Old Pod Removed
       │
       ▼
Rollout Successful
```

A successful rollout allows the Jenkins pipeline to continue and finish successfully.

---

## 10. Kubernetes Verification

The Kubernetes environment can be inspected using commands such as:

```bash
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get services
```

These commands are useful for verifying:

- Cluster availability
- Pod status
- Deployment state
- Service configuration

---

## 11. Verifying the Deployed Image

The image currently referenced by a Deployment can be inspected using:

```bash
kubectl get deployment <deployment> -o wide
```

The image used by a running Pod can also be inspected through Kubernetes resource information.

The relationship is:

```text
Jenkins Build
     │
     ▼
Versioned Docker Image
     │
     ▼
Docker Hub
     │
     ▼
Kubernetes Deployment
     │
     ▼
Running Pod
```

This provides traceability from the CI build to the running application version.

---

## 12. Local Application Access

Because the Kubernetes cluster is running locally, port forwarding can be used to access the application.

General form:

```bash
kubectl port-forward service/<service> <local-port>:<service-port>
```

The connection becomes:

```text
Browser
   │
   ▼
localhost
   │
   ▼
Kubernetes Service
   │
   ▼
Application Pod
   │
   ▼
Container
```

This was used to validate the Kubernetes-deployed application locally.

Port forwarding is intended for local development and validation. It does not make the application publicly accessible over the internet.

---

## 13. Complete CD Flow

The complete Continuous Deployment process is:

```text
                 Docker Hub
                      │
                      │ Versioned Image
                      ▼
                   Jenkins
                      │
                      │ Update Image
                      ▼
                Kubernetes
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
                  Service
                      │
                      ▼
                Application
```

---

## 14. Jenkins Kubernetes Deploy Stage

The Jenkins pipeline contains a deployment stage responsible for updating Kubernetes and verifying the rollout.

The general structure is:

```groovy
stage('Kubernetes Deploy') {
    steps {
        // Update Kubernetes Deployment
        // Wait for rollout completion
    }
}
```

The image version used by this stage corresponds to the container image generated by the current CI build.

The relationship is:

```text
Jenkins BUILD_NUMBER
        │
        ▼
Docker Image Tag
        │
        ▼
Docker Hub Image
        │
        ▼
Kubernetes Deployment
        │
        ▼
New Pod
```

---

## 15. Deployment Verification

After the Jenkins deployment stage completes, Kubernetes resources can be checked to verify the deployment.

A healthy Pod should show a state similar to:

```text
READY    STATUS
1/1      Running
```

The Deployment should report the expected image version and show the desired number of replicas as available.

The rollout should also complete successfully.

---

## 16. Deployment Result

The final deployment successfully demonstrated the transition from an existing application version to a newly built container image.

The process was:

```text
Previous Image
      │
      ▼
New CI Build
      │
      ▼
New Docker Image
      │
      ▼
Docker Hub
      │
      ▼
Kubernetes Image Update
      │
      ▼
New Pod
      │
      ▼
Old Pod Terminated
      │
      ▼
New Version Running
```

This verified that the CD pipeline could automatically propagate a new container image into the Kubernetes environment.

---

## 17. CD Evidence

### Jenkins Deployment Stage

The Jenkins Stage View provides evidence that the Kubernetes deployment stage completed successfully.

**Paste Jenkins Stage View screenshot here:**

<!-- PASTE JENKINS STAGE VIEW SCREENSHOT HERE -->

---

### Kubernetes Pod

The Kubernetes Pod was verified to be running after the deployment.

**Paste Kubernetes Pod screenshot here:**

<!-- PASTE KUBERNETES POD SCREENSHOT HERE -->

---

### Kubernetes Deployment

The Deployment was verified to be using the expected container image.

**Paste Kubernetes Deployment screenshot here:**

<!-- PASTE KUBERNETES DEPLOYMENT SCREENSHOT HERE -->

---

### Application

The deployed application was accessed locally through the Kubernetes Service.

**Optional application screenshot:**

<!-- PASTE APPLICATION SCREENSHOT HERE -->

---

## 18. CI/CD Integration

The complete CI/CD workflow combines the CI and CD processes:

```text
                         CI
                         │
                         ▼
GitHub → Jenkins → Build → Test → Docker Build
                                      │
                                      ▼
                                 Docker Hub
                                      │
                                      │
                                     CD
                                      │
                                      ▼
                              Kubernetes Deploy
                                      │
                                      ▼
                                Rolling Update
                                      │
                                      ▼
                                     Pod
                                      │
                                      ▼
                                   Service
                                      │
                                      ▼
                                Application
```

The key relationship is:

> **CI produces the deployment artifact. CD deploys that artifact.**

---

## 19. Key Concepts Demonstrated

The CD implementation provided practical experience with:

- Kubernetes Deployments
- Kubernetes Pods
- Kubernetes Services
- Container image versioning
- Kubernetes image updates
- Rolling deployments
- Rollout verification
- Kubernetes networking
- Jenkins-to-Kubernetes integration
- Local Kubernetes deployment

---

## 20. Current Deployment Environment

The current implementation uses a Kubernetes cluster running locally through Docker Desktop.

The application was successfully deployed and verified in this environment.

The local deployment demonstrates the CD workflow without requiring a public cloud environment.

---

## 21. Future Improvements

The current CD implementation can be extended toward a more production-oriented deployment environment.

Possible improvements include:

- AWS EKS deployment
- Cloud Load Balancer
- Public application endpoint
- Kubernetes Ingress
- Readiness probes
- Liveness probes
- Resource requests and limits
- ConfigMaps
- Secrets
- Automated rollback
- Monitoring
- Logging
- Observability

---

## 22. Final CD Outcome

The Continuous Deployment implementation successfully connects the container registry with Kubernetes.

The final flow is:

```text
Versioned Docker Image
        ↓
Docker Hub
        ↓
Jenkins Deployment Stage
        ↓
Kubernetes Deployment
        ↓
Rolling Update
        ↓
New Pod
        ↓
Kubernetes Service
        ↓
Running Application
```

The deployment was verified by confirming that the new container image was successfully deployed and running inside the Kubernetes cluster.
