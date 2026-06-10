# ArgoCD and GitOps for Kubernetes

ArgoCD is a continuous delivery tool for Kubernetes applications.

ArgoCD monitors a Git repo where your Kubernetes YAML manifests live, and keeps the cluster in sync with whatever is committed there.
Your Git repo becomes the **single source of truth** for the desired state of your application - this pattern is called **GitOps**.

## How it fits together

```
Developer pushes code
        │
        ▼
① GitHub Actions builds a new Docker image and pushes it to DockerHub
        │
        ▼
② GitHub Actions updates the image tag in `infra/k8s`
        │
        ▼
③ ArgoCD detects the change in the repo and syncs the cluster
        │
        ▼
④ New pods are rolled out with the updated image
```

## Fork the YoloService repo

Fork [https://github.com/alonitac/YoloService](https://github.com/alonitac/YoloService) into your own GitHub account.
This repo contains both the application source code and the Kubernetes manifests under `infra/k8s/`.


## Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Access the ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0
```

The username is `admin`. Retrieve the initial password with:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```

## Create the ArgoCD Application

In the ArgoCD UI, click **+ New App** and fill in:

1. In the **General** section, set:
   - For **Application Name**, enter `yolo-service`.
   - For **Project**, select `default`.
   - For **Sync Policy**, select **Automatic**.
2. In the **Source** section, set:
   - For **Repository URL**, enter the URL of your forked `YoloService` GitHub repo.
   - For **Path**, enter the path to the Kubernetes manifests `infra/k8s`.
3. In the **Destination** section, set:
   - For **Cluster URL**, enter `https://kubernetes.default.svc` (this is the in-cluster API server URL).
   - For **Namespace**, enter `default`.
4. Click **Create**. ArgoCD will immediately sync and deploy the YoloService into your cluster.

## Test manual GitOps

Via GitHub, edit the `infra/k8s/deployment.yaml` file in your forked repo. For example, change the number of replicas from `1` to `2`.
Click **Commit changes** to save your changes. Watch ArgoCD detect the change and roll out the update automatically.

## CI/CD pipeline

### Add secrets to the YoloService repo

The workflow needs credentials to push images to DockerHub

Add the following secrets to your **forked YoloService repo**:

1. In your forked `YoloService` repo on GitHub, go to **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret** and add each of the following:

   **`DOCKERHUB_USERNAME`**
   - Your DockerHub username (e.g. `johndoe`).

   **`DOCKERHUB_TOKEN`**
   - A DockerHub access token (not your password).
   - To create one: log in to [hub.docker.com](https://hub.docker.com) → **Account Settings** → **Personal access tokens** → **Generate new token**.
   - Copy the token immediately - it won't be shown again.

### Watch the workflow run

Push any change to the `main` branch of your forked `YoloService` repo to trigger the pipeline.

Go to the **Actions** tab of your forked repo to watch the workflow run in real time:
- The first job builds and pushes a new Docker image to DockerHub.
- The second job updates the image tag in `infra/k8s/` and commits the change back to the same repo.

Once the workflow completes, switch to ArgoCD - it will detect the new commit and automatically roll out the updated image, completing the full GitOps loop.
