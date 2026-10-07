# Jenkins Three-Environment Pipeline

A practical Jenkins CI/CD pipeline demonstrating how an application is promoted through three environments:

**DEV → STAGING → PRODUCTION**

The pipeline follows the **Build Once, Promote the Same Artifact** approach. A Docker image is built once, tested in DEV and STAGING, and the same image version is promoted to production after manual approval.

---

## 1. Project Objective

The objective of this project is to understand how to implement a Jenkins pipeline for multiple environments.

The pipeline demonstrates:

* Source code checkout from GitHub
* Application validation
* Docker image creation
* Versioned Docker images
* DEV deployment
* DEV smoke testing
* STAGING deployment
* STAGING smoke testing
* Manual production approval
* Production deployment
* Production smoke testing

This project is intentionally simple so that the complete CI/CD flow can be implemented and understood in a KodeKloud playground.

---

## 2. CI/CD Flow

```text
                    GitHub
                       |
                       v
                +--------------+
                |    Jenkins   |
                +--------------+
                       |
                       v
                 Code Checkout
                       |
                       v
                    Build
                       |
                       v
                     Test
                       |
                       v
             Build Docker Image
                       |
                       v
              three-env-app:BUILD
                       |
             +---------+---------+
             |                   |
             v                   v
          DEV :8081         STAGING :8082
             |                   |
             v                   v
        Smoke Test           Smoke Test
             |                   |
             +---------+---------+
                       |
                       v
              Manual Approval
                       |
                       v
                  PROD :8083
                       |
                       v
                  Smoke Test
```

---

## 3. Main Principle

The most important concept in this project is:

> **Build Once, Promote the Same Artifact**

The Docker image is built only once.

For example, if Jenkins creates build number `25`:

```text
three-env-app:25
```

The same image is promoted through all environments:

```text
three-env-app:25
       |
       +----> DEV
       |
       +----> STAGING
       |
       +----> PROD
```

We do not rebuild the application separately for DEV, STAGING, and PROD.

This helps ensure that the exact artifact tested in the lower environments is the artifact deployed to production.

---

# 4. Repository Structure

```text
jenkins-three-env-pipeline/
│
├── app/
│   ├── index.html
│   └── Dockerfile
│
└── Jenkinsfile
```

### Files

### `app/index.html`

Simple web application used for the demonstration.

### `app/Dockerfile`

Dockerfile used to create the application image.

### `Jenkinsfile`

Contains the complete Jenkins CI/CD pipeline.

---

# 5. Dockerfile

Example Dockerfile:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

The application runs inside an Nginx container.

The container listens on:

```text
Port 80
```

The Jenkins pipeline maps different host ports for each environment.

---

# 6. Environment Mapping

| Environment | Container     | Host Port | Container Port |
| ----------- | ------------- | --------: | -------------: |
| DEV         | `dev-app`     |    `8081` |           `80` |
| STAGING     | `staging-app` |    `8082` |           `80` |
| PROD        | `prod-app`    |    `8083` |           `80` |

Therefore:

```text
DEV
http://localhost:8081

STAGING
http://localhost:8082

PROD
http://localhost:8083
```

All three containers use port `80` internally.

Different host ports allow all three containers to run simultaneously on the same machine.

---

# 7. Jenkins Pipeline

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = "three-env-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Code') {
            steps {
                echo 'Checking out source code'

                git branch: 'main',
                    url: 'https://github.com/ankanidileep/jenkins-three-env-pipeline.git'
            }
        }

        stage('Build') {
            steps {
                echo 'Building application'

                sh 'ls -la'
                sh 'ls -la app'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests'

                sh 'test -f app/index.html'
                sh 'test -f app/Dockerfile'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    ./app
                '''
            }
        }

        stage('Deploy DEV') {
            steps {
                sh '''
                    docker rm -f dev-app || true

                    docker run -d \
                      --name dev-app \
                      -p 8081:80 \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('DEV Smoke Test') {
            steps {
                sh '''
                    echo "Testing DEV..."
                    curl -f http://localhost:8081
                '''
            }
        }

        stage('Deploy STAGING') {
            steps {
                sh '''
                    docker rm -f staging-app || true

                    docker run -d \
                      --name staging-app \
                      -p 8082:80 \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('STAGING Smoke Test') {
            steps {
                sh '''
                    echo "Testing STAGING..."
                    curl -f http://localhost:8082
                '''
            }
        }

        stage('Production Approval') {
            steps {
                input message: 'Deploy this version to Production?',
                      ok: 'Deploy to Production'
            }
        }

        stage('Deploy PROD') {
            steps {
                sh '''
                    docker rm -f prod-app || true

                    docker run -d \
                      --name prod-app \
                      -p 8083:80 \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('PROD Smoke Test') {
            steps {
                sh '''
                    echo "Testing PRODUCTION..."
                    curl -f http://localhost:8083
                '''
            }
        }
    }
}
```

---

# 8. Pipeline Stages

## Stage 1 — Code

```groovy
stage('Code') {
    steps {
        git branch: 'main',
            url: 'https://github.com/ankanidileep/jenkins-three-env-pipeline.git'
    }
}
```

### Purpose

Checks out the application source code from GitHub.

Jenkins creates a workspace and downloads the repository.

---

# 9. Stage 2 — Build

```groovy
stage('Build') {
    steps {
        sh 'ls -la'
        sh 'ls -la app'
    }
}
```

### Purpose

In this simple HTML application there is no Maven or Gradle compilation.

Therefore, the Build stage validates that Jenkins has received the expected application files.

Docker image creation happens separately in the `Build Docker Image` stage.

---

# 10. Stage 3 — Test

```groovy
stage('Test') {
    steps {
        sh 'test -f app/index.html'
        sh 'test -f app/Dockerfile'
    }
}
```

The Linux command:

```bash
test -f filename
```

checks whether the file exists and is a regular file.

If the file does not exist, the command returns a non-zero exit code and Jenkins fails the stage.

---

# 11. Stage 4 — Build Docker Image

```groovy
docker build \
-t ${IMAGE_NAME}:${IMAGE_TAG} \
./app
```

The environment variables are:

```text
IMAGE_NAME = three-env-app
IMAGE_TAG  = BUILD_NUMBER
```

For example:

```text
BUILD_NUMBER = 25
```

creates:

```text
three-env-app:25
```

Verify it with:

```bash
docker images
```

---

# 12. Stage 5 — Deploy DEV

```bash
docker rm -f dev-app || true

docker run -d \
  --name dev-app \
  -p 8081:80 \
  three-env-app:<BUILD_NUMBER>
```

The old DEV container is removed first.

```bash
docker rm -f dev-app || true
```

The `|| true` prevents the pipeline from failing if the container does not already exist.

Then the new container is started.

```text
Host 8081 → Container 80
```

---

# 13. Stage 6 — DEV Smoke Test

```bash
curl -f http://localhost:8081
```

This checks whether the DEV application is reachable.

If the request fails, Jenkins stops the pipeline.

Therefore:

```text
DEV Deployment
      |
      v
DEV Smoke Test
      |
   PASS?
    /  \
  YES   NO
   |     |
   v     STOP
STAGING
```

---

# 14. Stage 7 — Deploy STAGING

The same Docker image is deployed to STAGING.

```bash
docker rm -f staging-app || true

docker run -d \
  --name staging-app \
  -p 8082:80 \
  three-env-app:<BUILD_NUMBER>
```

Notice that there is no:

```bash
docker build
```

again.

This is important.

The same image used in DEV is promoted to STAGING.

---

# 15. Stage 8 — STAGING Smoke Test

```bash
curl -f http://localhost:8082
```

If the STAGING test fails:

```text
Pipeline stops
      |
      X
Production approval is not reached
      |
      X
Production is not deployed
```

This protects production from an unvalidated version.

---

# 16. Stage 9 — Production Approval

```groovy
input message: 'Deploy this version to Production?',
      ok: 'Deploy to Production'
```

Jenkins pauses the pipeline.

A person must approve the deployment.

The flow becomes:

```text
STAGING PASS
     |
     v
MANUAL APPROVAL
     |
   APPROVE
     |
     v
   PROD
```

If the deployment is rejected, the production deployment does not happen.

---

# 17. Stage 10 — Deploy PROD

```bash
docker rm -f prod-app || true

docker run -d \
  --name prod-app \
  -p 8083:80 \
  three-env-app:<BUILD_NUMBER>
```

The exact same Docker image that passed DEV and STAGING is deployed.

```text
three-env-app:25

    DEV
     |
     v
  STAGING
     |
     v
    PROD
```

---

# 18. Stage 11 — PROD Smoke Test

```bash
curl -f http://localhost:8083
```

This validates that the production container is responding.

---

# 19. Complete Pipeline Flow

```text
             GitHub
                |
                v
          Code Checkout
                |
                v
              Build
                |
                v
              Test
                |
                v
       Build Docker Image
                |
                v
       three-env-app:25
                |
                v
          Deploy DEV
                |
                v
        DEV Smoke Test
                |
              PASS
                |
                v
       Deploy STAGING
                |
                v
      STAGING Smoke Test
                |
              PASS
                |
                v
       Manual Approval
                |
             APPROVE
                |
                v
         Deploy PROD
                |
                v
        PROD Smoke Test
                |
                v
             SUCCESS
```

---

# 20. KodeKloud Hands-on Implementation

## Step 1 — Start the Playground

Start your KodeKloud playground.

Make sure you have access to:

```bash
docker
jenkins
git
curl
```

Verify:

```bash
docker --version
git --version
curl --version
```

---

## Step 2 — Verify Docker

```bash
docker ps
```

Then:

```bash
docker images
```

Jenkins must be able to execute Docker commands.

---

## Step 3 — Verify GitHub Repository

```bash
git ls-remote https://github.com/ankanidileep/jenkins-three-env-pipeline.git
```

This verifies that Git can reach the repository.

---

## Step 4 — Create Jenkins Pipeline

Open Jenkins.

Go to:

```text
Jenkins
   |
   +-- New Item
```

Enter:

```text
three-env-pipeline
```

Select:

```text
Pipeline
```

Click:

```text
Create
```

---

## Step 5 — Add Pipeline Script

Go to:

```text
Pipeline
    |
    +-- Definition
```

For the simple lab:

```text
Pipeline script
```

Paste the Jenkinsfile.

Click:

```text
Save
```

---

## Step 6 — Run Pipeline

Click:

```text
Build Now
```

Open:

```text
Console Output
```

You should see the stages executing sequentially.

---

# 21. Verify Docker Image

After the Docker build stage:

```bash
docker images
```

You should see something similar to:

```text
REPOSITORY       TAG
three-env-app    1
three-env-app    2
three-env-app    3
```

The tag corresponds to the Jenkins build number.

---

# 22. Verify DEV

```bash
docker ps
```

You should see:

```text
dev-app
```

Test:

```bash
curl http://localhost:8081
```

---

# 23. Verify STAGING

```bash
docker ps
```

You should see:

```text
staging-app
```

Test:

```bash
curl http://localhost:8082
```

---

# 24. Verify PROD

After approving the production deployment:

```bash
docker ps
```

You should see:

```text
prod-app
```

Test:

```bash
curl http://localhost:8083
```

---

# 25. Troubleshooting

## Check running containers

```bash
docker ps
```

## Check all containers

```bash
docker ps -a
```

## Check images

```bash
docker images
```

## Check DEV logs

```bash
docker logs dev-app
```

## Check STAGING logs

```bash
docker logs staging-app
```

## Check PROD logs

```bash
docker logs prod-app
```

## Inspect DEV container

```bash
docker inspect dev-app
```

## Test all environments

```bash
curl -f http://localhost:8081
curl -f http://localhost:8082
curl -f http://localhost:8083
```

## Check listening ports

```bash
ss -lntp
```

---

# 26. Common Failure Scenarios

### Test fails

Example:

```text
app/index.html does not exist
```

Result:

```text
Test
 |
 X
Pipeline stops
```

---

### Docker build fails

Possible reasons:

* Invalid Dockerfile
* Missing files
* Docker unavailable
* Jenkins does not have Docker permission

Result:

```text
Docker Build
     |
     X
Pipeline stops
```

---

### DEV smoke test fails

```bash
curl -f http://localhost:8081
```

If it fails, STAGING is not deployed.

---

### STAGING smoke test fails

The pipeline stops before production approval.

---

### Production approval rejected

Production deployment does not happen.

---

### Production smoke test fails

Investigate:

```bash
docker ps -a
docker logs prod-app
docker inspect prod-app
```

In a real production environment, follow the organization's rollback procedure.

---

# 27. Interview Scenario

### Interviewer:

> How do you implement a three-environment pipeline in Jenkins?

### Answer:

I implement the pipeline using three environments: DEV, STAGING, and PROD.

First, Jenkins checks out the source code from GitHub. Then I validate the application and build a Docker image with a unique version using the Jenkins build number.

For example:

```text
three-env-app:25
```

I deploy that image to DEV and perform a smoke test.

If DEV passes, I promote the same image to STAGING and perform another smoke test.

After STAGING passes, Jenkins pauses at a manual approval gate.

Once the deployment is approved, Jenkins deploys the exact same image to production and performs a final smoke test.

The main principle is:

> Build once and promote the same artifact across environments.

---

# 28. Why Use Manual Approval?

Production deployment usually requires additional validation and authorization.

The approval stage provides a control point:

```text
DEV
 |
PASS
 |
STAGING
 |
PASS
 |
MANUAL APPROVAL
 |
APPROVE
 |
PROD
```

This prevents Jenkins from automatically deploying every successful build directly to production.

---

# 29. How Would You Implement This in AWS/EKS?

The current project uses Docker containers for learning.

In a real AWS environment, the architecture could become:

```text
Developer
    |
    v
GitHub
    |
    v
Jenkins
    |
    +--> Build
    |
    +--> Test
    |
    +--> Docker Image
    |
    v
Amazon ECR
    |
    +--> DEV EKS
    |
    +--> STAGING EKS
    |
    +--> Approval
    |
    +--> PROD EKS
```

The same immutable image tag would be promoted across the environments.

For example:

```text
ECR
 |
 +-- financial-app:abc123
          |
          +--> DEV
          +--> STAGING
          +--> PROD
```

The deployment target changes, but the promotion principle remains the same.

---

# 30. Production Improvements

For a real production implementation, this lab can be extended with:

* Amazon ECR
* Amazon EKS
* Kubernetes Deployments
* Helm
* Argo CD
* Jenkins Credentials
* AWS IAM roles
* Automated unit/integration testing
* SonarQube
* Trivy
* Prometheus
* Grafana
* CloudWatch
* New Relic
* Deployment notifications
* Rollback strategy
* Production access control

The current Docker pipeline is the foundation for understanding the promotion model.

---

# 31. Important Interview Point

Do not say:

> “I built three separate applications for DEV, STAGING, and PROD.”

Instead say:

> “I built one versioned artifact and promoted the same artifact through DEV, STAGING, and PROD.”

This demonstrates a better understanding of CI/CD artifact promotion.

---

# 32. Useful Jenkins Concepts

| Concept        | Meaning                                      |       |                                                 |
| -------------- | -------------------------------------------- | ----- | ----------------------------------------------- |
| `agent any`    | Runs pipeline on an available Jenkins agent  |       |                                                 |
| `environment`  | Defines reusable environment variables       |       |                                                 |
| `BUILD_NUMBER` | Jenkins build identifier                     |       |                                                 |
| `stage`        | Logical section of the pipeline              |       |                                                 |
| `steps`        | Commands executed inside a stage             |       |                                                 |
| `sh`           | Executes shell commands                      |       |                                                 |
| `input`        | Manual approval                              |       |                                                 |
| `docker build` | Creates Docker image                         |       |                                                 |
| `docker run`   | Starts container                             |       |                                                 |
| `curl -f`      | Performs HTTP check and fails on HTTP errors |       |                                                 |
| `              |                                              | true` | Prevents cleanup command from failing the stage |

---

# 33. Final Mental Model

Remember this sequence:

```text
CHECKOUT
    ↓
BUILD
    ↓
TEST
    ↓
BUILD DOCKER IMAGE
    ↓
DEPLOY DEV
    ↓
DEV SMOKE TEST
    ↓
DEPLOY STAGING
    ↓
STAGING SMOKE TEST
    ↓
MANUAL APPROVAL
    ↓
DEPLOY PROD
    ↓
PROD SMOKE TEST
```

And remember the most important sentence:

> **Build once, test progressively, and promote the same artifact from DEV to STAGING to PROD.**

---

# 34. One-Line Interview Summary

> I implement a three-environment Jenkins pipeline where I build and version the application once, validate it in DEV and STAGING, require approval before production, and promote the same artifact through all environments.

---

## Cleanup

After completing the lab:

```bash
docker rm -f dev-app staging-app prod-app || true
```

Check images:

```bash
docker images | grep three-env-app
```

Remove a specific test image if required:

```bash
docker image rm three-env-app:<BUILD_NUMBER>
```

Finally, close the KodeKloud playground.

---

## Project Outcome

After completing this project, you should be able to explain and demonstrate:

```text
GitHub
   ↓
Jenkins
   ↓
Build
   ↓
Test
   ↓
Docker Image
   ↓
DEV
   ↓
STAGING
   ↓
Production Approval
   ↓
PROD
```

This project demonstrates the fundamental CI/CD concept of:

**Build Once → Validate → Promote → Approve → Release**
