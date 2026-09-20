# Loan Default Predictor

A machine learning-based web application for predicting the likelihood of loan default.

The project was developed as a full-stack application and extended with an automated CI/CD workflow using Jenkins, Docker, Docker Hub, and Kubernetes.

---

## Project Overview

The Loan Default Predictor provides a web interface for submitting relevant loan and applicant information and obtaining a prediction from the trained machine learning model.

The project was also used to understand how a software application moves from source code to a containerized and Kubernetes-deployed application through an automated CI/CD workflow.

---

## Tech Stack

### Application

- React
- TypeScript
- Vite
- Tailwind CSS
- TanStack Start

### DevOps & Deployment

- Git
- GitHub
- Jenkins
- Groovy
- Docker
- Docker Hub
- Kubernetes
- Docker Desktop Kubernetes
- GitHub Webhooks
- ngrok
- npm
- ESLint

---

## CI/CD & DevOps

Implemented an end-to-end CI/CD workflow connecting source control, continuous integration, containerization, image publishing, and Kubernetes deployment.

### CI/CD Workflow

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
     Kubernetes Deploy
            │
            ▼
     Rolling Update
            │
            ▼
      Application Pod
            │
            ▼
       Kubernetes Service
