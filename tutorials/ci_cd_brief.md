# CI/CD Pipeline for YoloService with GitHub Actions

CI/CD (Continuous Integration and Continuous Deployment) is a methodology that automates the build, test, and deployment process of a software project.

Previously, deploying a new version meant manually SSHing into the server, stopping the old container, pulling new code, rebuilding the image, and starting everything again.
With CI/CD, every push of code in GitHub triggers an automated pipeline that does all of this for you.

This is called **Continuous Deployment** - on every code change, a new version is automatically built and deployed.

We will use **GitHub Actions** as our automation platform - it is built into GitHub and runs workflows directly from your repository.


## How the pipeline works

The pipeline we are building does the following every time you push code to GitHub:

```
Push to main branch
     │
     ▼
①  Build Docker image from the Dockerfile
     │
     ▼
②  Tag image with the Git commit id
     │
     ▼
③  Push the image to DockerHub
     │
     ▼
④  SSH into the EC2 instance
     │
     ▼
⑤  Pass the new image tag to docker compose and bring the stack up
```

This guarantees that exactly the code you pushed is what runs in production, and old images are never silently overwritten.


## Workflow file

Workflows are YAML files stored in `.github/workflows/` inside your repository.
GitHub discovers and runs them automatically.

Here is the complete workflow for the YoloService pipeline:

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy YoloService

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # Check out the repository code
      - name: Checkout code
        uses: actions/checkout@v4

      # Log in to DockerHub
      - name: Log in to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # Build and push the image
      #   The image is tagged with the Git commit SHA so every build
      #   produces a unique, traceable tag (e.g. abc1234).
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/yolo-service:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    needs: build

    steps:
      # SSH into EC2 and deploy
      - name: Deploy on EC2
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ~/YoloService
            git pull
            YOLO_SERVICE_IMAGE_TAG=${{ github.sha }} \
            DOCKERHUB_USERNAME=${{ secrets.DOCKERHUB_USERNAME }} \
            docker compose up -d
```


## The `docker-compose.yml` on the EC2 instance

The compose file on your EC2 instance must reference the image using the `YOLO_SERVICE_IMAGE_TAG` and `DOCKERHUB_USERNAME` environment variables so the workflow can inject the correct tag at deploy time:

```yaml
# docker-compose.yml  (lives in ~/YoloService on the EC2 instance)

services:
  yolo-service:
    image: ${DOCKERHUB_USERNAME}/yolo-service:${YOLO_SERVICE_IMAGE_TAG}

  # ... rest of your services (frontend, prometheus, grafana)
```


## Storing secrets in GitHub

GitHub Actions can access sensitive values (keys, passwords, tokens) through **secrets** - encrypted values that are never visible in logs or to other users.

You need to store four secrets for this pipeline:

| Secret name | What to put in it |
|---|---|
| `EC2_SSH_KEY` | The full contents of your `.pem` key file |
| `EC2_HOST` | The public IP address of your EC2 instance |
| `DOCKERHUB_USERNAME` | Your DockerHub username |
| `DOCKERHUB_TOKEN` | A DockerHub personal access token (not your password) |


### How to add secrets to your GitHub repository

1. Open your repository on GitHub (e.g. `https://github.com/<your-username>/YoloService`).
2. Click the **Settings** tab at the top of the page (not your account settings - the one inside the repo).
3. In the left sidebar, expand **Secrets and variables** and click **Actions**.
4. Click the green **New repository secret** button.
5. Fill in the **Name** field (e.g. `EC2_HOST`) and paste the value into the **Secret** field.
6. Click **Add secret**.
7. Repeat steps 4–6 for each of the five secrets in the table above.

### How to store the `.pem` key as a secret

The `.pem` file is a plain text file. You need to copy its **entire contents** - including the header and footer lines - and paste it as the secret value.

1. On your local machine, print the file contents:
   ```bash
   cat ~/Downloads/my-key.pem
   ```
2. The output will look like this - select and copy **everything**:
   ```
   -----BEGIN RSA PRIVATE KEY-----
   MIIEowIBAAKCAQEA3Dp...
   ...many lines...
   -----END RSA PRIVATE KEY-----
   ```
3. In GitHub, create a new secret named `EC2_SSH_KEY` and paste the copied text as the value.
4. Click **Add secret**.

> **Tip**: Make sure there are no extra blank lines or spaces at the beginning or end of the pasted value. The key must start with `-----BEGIN` on the very first line.


## Triggering and monitoring the pipeline

1. Make any change to your code (e.g. add a comment to `app.py`), commit and push to `main`.
2. Go to your repository on GitHub and click the **Actions** tab.
3. You will see your workflow listed. Click it to watch each step execute in real time.
4. A green checkmark means success. A red ✗ means a step failed - click it to read the logs.



# Exercises

## Forking the repository

A **fork** is your own personal copy of someone else's GitHub repository.
When you fork, GitHub creates an independent copy of the repo under your account.
You can push changes, add secrets, and configure Actions - without affecting the original repository.

We fork here because the YoloService repository belongs to the course organisation.
You cannot add secrets or push workflows to a repo you do not own, so you need your own fork to build a CI/CD pipeline.

### How to fork

1. Open the YoloService repository on GitHub (the link will be shared by your instructor).
2. Click the **Fork** button near the top-right of the page.
3. Leave all settings as-is and click **Create fork**.
4. GitHub will redirect you to `https://github.com/<your-username>/YoloService` - this is now your copy.

All the exercises below are done in **your fork**, not the original repository.


### :pencil2: Create a CI/CD pipeline for YoloService

Follow the steps in this tutorial to build and deploy the full pipeline:

1. **Add the workflow file** - in your fork, click **Add file → Create new file**.
   - Set the path to `.github/workflows/deploy.yml` (type the slashes; GitHub will create the folders automatically).
   - Paste the full workflow YAML from the [Workflow file](#workflow-file) section above.
   - Scroll down and click **Commit changes**.
2. **Add `docker-compose.yml`** - repeat the same steps to create `docker-compose.yml` at the root of the repo, using the compose file from the [Docker Compose tutorial](docker_compose.md).
3. **Add secrets** - follow the [Storing secrets in GitHub](#storing-secrets-in-github) section to add all four secrets (`EC2_SSH_KEY`, `EC2_HOST`, `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`).
4. **One-time EC2 setup** - SSH into your instance and run the commands from the [Setting up the EC2 instance](#setting-up-the-ec2-instance) section.
5. **Trigger the pipeline** - make any small edit to a file in your fork directly in the GitHub UI (e.g. open `README.md`, click the pencil icon, add a space, then **Commit changes**). This push to `main` will kick off the workflow.
6. Click the **Actions** tab and watch the pipeline run. A green checkmark means success.
7. Verify the deployment: `ssh` into your EC2 and run `docker compose ps` - all services should be running.
8. Run `docker images` on the EC2 and confirm the image tag matches the Git commit SHA shown in the Actions run.

### :pencil2: Image vulnerabilities with Trivy

[Trivy](https://github.com/aquasecurity/trivy) is an open-source vulnerability scanner for container images.
Add it as a step in your workflow **after** the image is built but **before** it is deployed, so a critical vulnerability blocks the deployment automatically.

Add the following step to your `deploy.yml`, between the **Build and push** step and the **Deploy on EC2** step:

```yaml
      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ secrets.DOCKERHUB_USERNAME }}/yolo-service:${{ github.sha }}
          format: table
          exit-code: 1
          severity: CRITICAL,HIGH
```

- `exit-code: 1` causes the workflow to **fail** if any matching vulnerabilities are found, preventing deployment of an insecure image.
- `severity: CRITICAL,HIGH` limits the scan to the most serious findings; adjust as needed.

Push a change to `main` and open the **Actions** tab. You should see a **Scan image for vulnerabilities** step in the run. If vulnerabilities are found, the step will fail and the deploy step will not execute.

> **Tip**: To allow the pipeline to continue even when vulnerabilities are found, set `exit-code: 0`. The scan results will still be printed in the log.
