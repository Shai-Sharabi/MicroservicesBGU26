# CI/CD Pipeline for YoloService with GitHub Actions

CI/CD (Continuous Integration and Continuous Deployment) is a methodology that automates the build, test, and deployment process of a software project.

Previously, deploying a new version meant manually SSHing into the server, stopping the old container, pulling new code, rebuilding the image, and starting everything again.
With CI/CD, every push to the `main` branch triggers an automated pipeline that does all of this for you.
This is called **Continuous Deployment** — on every code change, a new version is automatically built and deployed.

We will use **GitHub Actions** as our automation platform — it is built into GitHub and runs workflows directly from your repository.

---

## How the pipeline works

The pipeline we are building does the following every time you push code to `main`:

```
Push to main
     │
     ▼
①  Build Docker image from the Dockerfile
     │
     ▼
②  Tag image with the Git commit SHA  (e.g. alonithuji/yolo-service:a3f1c9e)
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

---

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
  workflow_dispatch:   # allows manual runs from the GitHub UI

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # ── 1. Check out the repository code ──────────────────────────────────
      - name: Checkout code
        uses: actions/checkout@v4

      # ── 2. Log in to DockerHub ─────────────────────────────────────────────
      - name: Log in to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # ── 3. Build and push the image ────────────────────────────────────────
      #   The image is tagged with the short Git commit SHA so every build
      #   produces a unique, traceable tag (e.g. abc1234).
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/yolo-service:${{ github.sha }}

      # ── 4. Configure SSH ───────────────────────────────────────────────────
      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          cat >> ~/.ssh/config << EOF
          Host ec2
              HostName ${{ secrets.EC2_HOST }}
              User ${{ secrets.EC2_USERNAME }}
              IdentityFile ~/.ssh/id_rsa
              StrictHostKeyChecking no
          EOF

      # ── 5. Deploy on EC2 via docker compose ───────────────────────────────
      #   We pass the new image tag as an environment variable.
      #   docker compose reads it through the IMAGE_TAG variable.
      - name: Deploy on EC2
        run: |
          ssh ec2 "
            cd ~/YoloService &&
            git pull origin main &&
            IMAGE_TAG=${{ github.sha }} \
            DOCKERHUB_USERNAME=${{ secrets.DOCKERHUB_USERNAME }} \
            docker compose up -d --pull always
          "
```

---

## The `docker-compose.yml` on the EC2 instance

The compose file on your EC2 instance must reference the image using the `IMAGE_TAG` and `DOCKERHUB_USERNAME` environment variables so the workflow can inject the correct tag at deploy time:

```yaml
# docker-compose.yml  (lives in ~/YoloService on the EC2 instance)
networks:
  yolo-net:
  monitoring-net:

volumes:
  yolo-uploads:
  yolo-db:
  prometheus-data:
  grafana-data:

services:
  yolo-service:
    image: ${DOCKERHUB_USERNAME}/yolo-service:${IMAGE_TAG:-latest}
    networks:
      - yolo-net
    volumes:
      - yolo-uploads:/app/uploads/
      - yolo-db:/app/predictions.db

  # ... rest of your services (frontend, prometheus, grafana)
```

The `:-latest` fallback means the file still works when you run `docker compose up` manually without setting `IMAGE_TAG`.

---

## Storing secrets in GitHub

GitHub Actions can access sensitive values (keys, passwords, tokens) through **secrets** — encrypted values that are never visible in logs or to other users.

You need to store five secrets for this pipeline:

| Secret name | What to put in it |
|---|---|
| `EC2_SSH_KEY` | The full contents of your `.pem` key file |
| `EC2_HOST` | The public IP address of your EC2 instance |
| `EC2_USERNAME` | The SSH username (usually `ubuntu` for Ubuntu AMIs) |
| `DOCKERHUB_USERNAME` | Your DockerHub username |
| `DOCKERHUB_TOKEN` | A DockerHub personal access token (not your password) |

### How to create a DockerHub access token

1. Log in at [hub.docker.com](https://hub.docker.com).
2. Click your avatar (top-right) → **Account Settings** → **Personal access tokens**.
3. Click **Generate new token**.
4. Give it a name (e.g. `github-actions`) and set permissions to **Read & Write**.
5. Click **Generate** and **copy the token immediately** — you cannot see it again.

### How to add secrets to your GitHub repository

> If you have never opened repository settings before, follow each step carefully.

1. Open your repository on GitHub (e.g. `https://github.com/<your-username>/YoloService`).
2. Click the **Settings** tab at the top of the page (not your account settings — the one inside the repo).
3. In the left sidebar, expand **Secrets and variables** and click **Actions**.
4. Click the green **New repository secret** button.
5. Fill in the **Name** field (e.g. `EC2_HOST`) and paste the value into the **Secret** field.
6. Click **Add secret**.
7. Repeat steps 4–6 for each of the five secrets in the table above.

### How to store the `.pem` key as a secret

The `.pem` file is a plain text file. You need to copy its **entire contents** — including the header and footer lines — and paste it as the secret value.

1. On your local machine, print the file contents:
   ```bash
   cat ~/Downloads/my-key.pem
   ```
2. The output will look like this — select and copy **everything**:
   ```
   -----BEGIN RSA PRIVATE KEY-----
   MIIEowIBAAKCAQEA3Dp...
   ...many lines...
   -----END RSA PRIVATE KEY-----
   ```
3. In GitHub, create a new secret named `EC2_SSH_KEY` and paste the copied text as the value.
4. Click **Add secret**.

> **Tip**: Make sure there are no extra blank lines or spaces at the beginning or end of the pasted value. The key must start with `-----BEGIN` on the very first line.

---

## Setting up the EC2 instance

The EC2 instance needs Docker and Docker Compose installed, and the repository cloned, before the first deployment.

Connect to your instance and run:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
sudo usermod -aG docker $USER
# Log out and back in so the group change takes effect

# Clone the repository
git clone https://github.com/<your-username>/YoloService.git ~/YoloService
```

After this one-time setup, every subsequent deployment is handled entirely by the GitHub Actions workflow.

---

## Triggering and monitoring the pipeline

1. Make any change to your code (e.g. add a comment to `app.py`), commit and push to `main`.
2. Go to your repository on GitHub and click the **Actions** tab.
3. You will see your workflow listed. Click it to watch each step execute in real time.
4. A green checkmark means success. A red ✗ means a step failed — click it to read the logs.

> **Tip**: To avoid updating the `EC2_HOST` secret every time your EC2 instance restarts and gets a new IP, consider [assigning an Elastic IP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-eips.html) to the instance. Then update `EC2_HOST` once with the static IP.

---

## Git Workflows

**Git workflows** refer to the different strategies teams adopt when using Git for collaboration.

> [!IMPORTANT]
> Don't confuse **Git workflows** with workflows in **GitHub Actions**.
>
> - **Git workflows**: Strategies for managing branches, commits, and collaboration in Git.
> - **GitHub Actions workflows**: Automated YAML-defined pipelines that build, test, and deploy code.

[Gitflow](https://nvie.com/posts/a-successful-git-branching-model/) (by Vincent Driessen) is the most widely known Git workflow. It uses **feature branches** and multiple **primary branches**.

In this course we use a simpler approach: create a feature branch, work on it, and open a **pull request (PR)** to `main`.

Perform the following steps in the **YoloService** repo:

1. Ensure you are on the main branch: `git checkout main`
2. Pull the latest changes: `git pull origin main`
3. Create a feature branch: `git checkout -b feature-branch-name`
4. Make changes and commit.
5. Push the branch: `git push origin feature-branch-name`
6. Open a **Pull Request** on GitHub:
   - Go to your repository on GitHub.
   - You will see a prompt to create a pull request for your recently pushed branch — click **Compare & pull request**.
   - Add a title and description.
   - Ensure the **base branch** is `main` and the compare branch is your feature branch.
   - Click **Create pull request**.
7. Once reviewed, click **Merge pull request**.

### Protecting the `main` branch

To ensure no code is pushed directly to `main` — only through reviewed pull requests:

1. Go to **Settings** → **Branches** in your repository.
2. Click **Add classic branch protection rule**.
3. Set the branch name pattern to `main`.
4. Check **Require a pull request before merging**.
5. Check **Do not allow bypassing the above settings**.
6. Click **Save changes**.

Your `main` branch is now protected. Every change must go through a pull request.

---

# Exercises

### :pencil2: Create a CI/CD pipeline for YoloService

Follow the steps in this tutorial to build and deploy the full pipeline:

1. Add the `.github/workflows/deploy.yml` file to your YoloService repository.
2. Add the `docker-compose.yml` (from the [Docker Compose tutorial](docker_compose.md)) to the repository.
3. Create all five secrets (`EC2_SSH_KEY`, `EC2_HOST`, `EC2_USERNAME`, `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`) in your repository settings.
4. Perform the one-time EC2 setup (install Docker, clone the repo).
5. Push a small change to `main` and watch the **Actions** tab — the pipeline should build the image, push it to DockerHub, SSH into your EC2 instance and bring the compose stack up with the new image tag.
6. Verify the deployment: `ssh` into your EC2 and run `docker compose ps` — all services should be running.
7. Run `docker images` on the EC2 and confirm the image tag matches the Git commit SHA shown in the Actions run.

### :pencil2: Image vulnerabilities with Trivy

