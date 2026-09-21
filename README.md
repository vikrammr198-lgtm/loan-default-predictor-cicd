# Loan Default Predictor — CI/CD

## Overview

This project demonstrates an end-to-end CI/CD pipeline for the Loan Default Predictor application.

The pipeline connects source control, continuous integration, containerization, container image distribution, and container orchestration into an automated workflow.

### Technologies Used

- Git
- GitHub
- GitHub Webhooks
- Jenkins
- Docker
- Docker Hub
- Kubernetes
- kubectl
- ngrok

---

## Original Application

The CI/CD pipeline was built around the original Loan Default Predictor application.

**Original Project Repository:**

https://github.com/vikrammr198-lgtm/loan-default-predictor

This repository focuses on the CI/CD and DevOps implementation of that application.

---

## Overall CI/CD Workflow

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    │ Webhook
    ▼
ngrok
    │
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
       Running Application
