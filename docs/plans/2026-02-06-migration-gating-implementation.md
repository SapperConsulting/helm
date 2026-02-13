# Migration Gating Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Move migration orchestration out of ArgoCD into GitHub Actions + in-cluster Jobs, using ECR tag conventions (`stage-migrated-*`) as a gate for ArgoCD Image Updater. Staging only.

**Architecture:** GitHub Actions builds the image and submits a migration Job via kubectl. The Job runs `alembic upgrade head` in-cluster, then retags the ECR image with a `stage-migrated-` prefix. ArgoCD Image Updater is configured to only match `stage-migrated-*` tags, so pods never deploy until migration succeeds.

**Tech Stack:** GitHub Actions, AWS ECR, EKS, kubectl, Helm, ArgoCD Image Updater, Alembic, Python/FastAPI

---

## Pre-requisites (manual, outside this plan)

These must be done by hand in AWS/Kubernetes before the code changes take effect:

1. **Create IAM role `migration-runner`** with this policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "arn:aws:ecr:us-east-1:620167017830:repository/udab-server"
    },
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    }
  ]
}
```
Add a trust policy allowing the EKS OIDC provider to assume this role for the `migration-runner` service account.

2. **Create Kubernetes ServiceAccount:**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: migration-runner
  namespace: <your-namespace>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::620167017830:role/migration-runner
EOF
```

3. **Ensure the GitHub OIDC role** (`arn:aws:iam::620167017830:role/github-oidc`) has permissions to run `eks:DescribeCluster` and create Jobs in the target namespace (it may already have this if you've used kubectl from GHA before).

4. **Know these values** (needed as GitHub Actions secrets/variables):
   - `EKS_CLUSTER_NAME` — the name of your EKS cluster
   - `NAMESPACE` — the Kubernetes namespace for the staging app
   - `SECRET_NAME` — the k8s secret name used by the staging migration Job (e.g., `udab-server-stage-dist` or the app secret)

---

## Task 1: Add AWS CLI to Dockerfile

**Repo:** `udab-server`
**Files:**
- Modify: `/home/ubuntu/projects/clientwork/sapper/udab-server/Dockerfile`

**Step 1: Add awscli install to Dockerfile**

After the line `RUN pip install alembic`, add `awscli` to the same install:

```dockerfile
RUN pip install alembic awscli
```

This replaces the existing `RUN pip install alembic` line. Keeps it as one layer.

**Step 2: Verify Dockerfile syntax**

Run:
```bash
cd /home/ubuntu/projects/clientwork/sapper/udab-server && docker build --no-cache --progress=plain -t udab-test . 2>&1 | tail -5
```
Expected: Build succeeds (or skip if no Docker available locally — CI will validate).

**Step 3: Commit**

```bash
cd /home/ubuntu/projects/clientwork/sapper/udab-server
git add Dockerfile
git commit -m "feat: add awscli to Docker image for ECR retag in migrations"
```

---

## Task 2: Update migrate.sh with ECR retag logic

**Repo:** `udab-server`
**Files:**
- Modify: `/home/ubuntu/projects/clientwork/sapper/udab-server/migrate.sh`

**Step 1: Replace migrate.sh contents**

The script now runs the migration, then retags the image in ECR. The `IMAGE_TAG` and `ECR_REPOSITORY` env vars will be passed by the Job spec (Task 4). A retry loop around the ECR retag handles transient network failures.

```bash
#!/bin/bash
set -e

# Run the database migration
alembic upgrade head

# If IMAGE_TAG is set, retag the image as migrated in ECR
# This signals ArgoCD Image Updater that this image is safe to deploy
if [ -n "$IMAGE_TAG" ]; then
  echo "Migration succeeded. Retagging image as migrated..."

  REGION="${AWS_REGION:-us-east-1}"
  REPO="${ECR_REPOSITORY:-udab-server}"

  for i in 1 2 3; do
    MANIFEST=$(aws ecr batch-get-image \
      --region "$REGION" \
      --repository-name "$REPO" \
      --image-ids imageTag="$IMAGE_TAG" \
      --query 'images[0].imageManifest' \
      --output text)

    aws ecr put-image \
      --region "$REGION" \
      --repository-name "$REPO" \
      --image-tag "${IMAGE_TAG/stage-/stage-migrated-}" \
      --image-manifest "$MANIFEST" \
      && break

    echo "ECR retag attempt $i failed. Retrying in 10s..."
    sleep 10
  done

  echo "Image retagged as ${IMAGE_TAG/stage-/stage-migrated-}"
else
  echo "No IMAGE_TAG set. Skipping ECR retag (local/dev mode)."
fi
```

Key details:
- `IMAGE_TAG` will be `stage-{sha}` — the bash substitution `${IMAGE_TAG/stage-/stage-migrated-}` converts it to `stage-migrated-{sha}`
- When `IMAGE_TAG` is not set (local dev, other contexts), it just runs alembic and exits — backward compatible
- Retry loop (3 attempts, 10s backoff) handles transient ECR API failures

**Step 2: Commit**

```bash
cd /home/ubuntu/projects/clientwork/sapper/udab-server
git add migrate.sh
git commit -m "feat: add ECR retag to migrate.sh for migration gating"
```

---

## Task 3: Remove alembic from run_remote.sh

**Repo:** `udab-server`
**Files:**
- Modify: `/home/ubuntu/projects/clientwork/sapper/udab-server/run_remote.sh`

**Step 1: Remove the alembic line**

Replace the file contents with:

```bash
#!/bin/bash
set -e

uvicorn app.main:app --host 0.0.0.0
```

This stops pods from running migrations at startup. Migrations are now handled exclusively by the migration Job.

**Step 2: Commit**

```bash
cd /home/ubuntu/projects/clientwork/sapper/udab-server
git add run_remote.sh
git commit -m "feat: remove migration from pod startup, migrations now run via dedicated Job"
```

---

## Task 4: Update stage.yaml workflow to submit migration Job

**Repo:** `udab-server`
**Files:**
- Modify: `/home/ubuntu/projects/clientwork/sapper/udab-server/.github/workflows/stage.yaml`

**Step 1: Replace the stage workflow**

The workflow now: builds the image, pushes it with `stage-{sha}` tag, then submits a migration Job to the cluster. Note: we stop pushing the mutable `stage` tag since Image Updater will now match `stage-migrated-*`.

```yaml
on:
  push:
    branches:
      - "stage"

permissions:
  contents: read
  id-token: write

env:
  ECR_REGISTRY: 620167017830.dkr.ecr.us-east-1.amazonaws.com
  ECR_REPOSITORY: udab-server

jobs:
  Build:
    name: Build & Push Image to ECR
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup AWS ECR Details
        uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: arn:aws:iam::620167017830:role/github-oidc
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-aws-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and push the tagged docker image to Amazon ECR
        env:
          IMAGE_TAG: stage-${{ github.sha }}
        run: |
          set -e
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name ${{ vars.EKS_CLUSTER_NAME }} --region us-east-1

      - name: Submit migration Job
        env:
          IMAGE_TAG: stage-${{ github.sha }}
          NAMESPACE: ${{ vars.STAGE_NAMESPACE }}
          SECRET_NAME: ${{ vars.STAGE_SECRET_NAME }}
          DIST_SECRET_NAME: ${{ vars.STAGE_DIST_SECRET_NAME }}
        run: |
          SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)

          kubectl apply -f - <<EOF
          apiVersion: batch/v1
          kind: Job
          metadata:
            name: db-migration-$SHORT_SHA
            namespace: $NAMESPACE
          spec:
            ttlSecondsAfterFinished: 3600
            backoffLimit: 3
            template:
              spec:
                serviceAccountName: migration-runner
                restartPolicy: OnFailure
                nodeSelector:
                  node_group_type: on_demand_main
                tolerations:
                - key: node_group_type
                  operator: Equal
                  value: on_demand_main
                  effect: NoSchedule
                containers:
                - name: migrate
                  image: $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
                  command: ["/migrate.sh"]
                  env:
                  - name: IMAGE_TAG
                    value: "$IMAGE_TAG"
                  - name: ECR_REPOSITORY
                    value: "$ECR_REPOSITORY"
                  - name: AWS_REGION
                    value: "us-east-1"
                  envFrom:
                  - secretRef:
                      name: $DIST_SECRET_NAME
                  - secretRef:
                      name: $SECRET_NAME
          EOF

          echo "Migration Job db-migration-$SHORT_SHA submitted."
```

Key details:
- Image is now tagged only as `stage-{sha}` (no more mutable `stage` tag)
- `EKS_CLUSTER_NAME`, `STAGE_NAMESPACE`, `STAGE_SECRET_NAME`, and `STAGE_DIST_SECRET_NAME` should be set as GitHub repository variables
- The Job passes `IMAGE_TAG`, `ECR_REPOSITORY`, and `AWS_REGION` as env vars so `migrate.sh` can do the retag
- Both `secretName` and `distSecretName` are mounted via `envFrom` so the migration has DB credentials

**Step 2: Commit**

```bash
cd /home/ubuntu/projects/clientwork/sapper/udab-server
git add .github/workflows/stage.yaml
git commit -m "feat: stage workflow now submits migration Job to cluster instead of relying on ArgoCD"
```

---

## Task 5: Remove migration Job and SyncWave from Helm chart

**Repo:** `helm`
**Files:**
- Delete: `/home/ubuntu/projects/clientwork/sapper/helm/charts/application/templates/job.yaml`
- Modify: `/home/ubuntu/projects/clientwork/sapper/helm/charts/application/values.yaml`
- Modify: `/home/ubuntu/projects/clientwork/sapper/helm/charts/application/templates/deployment.yaml`

**Step 1: Delete job.yaml template**

```bash
cd /home/ubuntu/projects/clientwork/sapper/helm
rm charts/application/templates/job.yaml
```

**Step 2: Remove jobs and syncWave from values.yaml**

Replace contents of `values.yaml` with:

```yaml
app:
  pod:
    port:
    healthCheckPath:
  service:
    name:
    protocol: TCP
    port:
  resources:
    requests:
      memory: "256Mi"
      cpu: "125m"
    limits:
      memory: "256Mi"
      cpu: "125m"
  secretName:
  distSecretName:
  annotations:
  is_client: false

image:
  name:
  tag:
  url:

iam:
  service_account_role: ""
  service_account_name: ""
```

Removed: `syncWave` section and `jobs` section.

**Step 3: Remove syncWave annotation from deployment.yaml**

In `deployment.yaml`, remove the sync-wave annotation line:
```
    argocd.argoproj.io/sync-wave: "{{ .Values.syncWave.deployment | default 2 }}"
```

The deployment no longer needs SyncWave ordering since migrations happen before ArgoCD ever sees the new image.

**Step 4: Commit**

```bash
cd /home/ubuntu/projects/clientwork/sapper/helm
git add -A
git commit -m "feat: remove migration Job and SyncWave from Helm chart, migrations now handled by CI"
```

---

## Task 6: Update ArgoCD Image Updater configuration

**Repo:** Wherever your ArgoCD Application manifest lives (may not be in either of these repos).

**Step 1: Update Image Updater annotations on the ArgoCD Application**

Add/update these annotations on the ArgoCD Application resource for the staging app:

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=620167017830.dkr.ecr.us-east-1.amazonaws.com/udab-server
    argocd-image-updater.argoproj.io/app.update-strategy: regexp
    argocd-image-updater.argoproj.io/app.allow-tags: regexp:^stage-migrated-
```

This tells Image Updater to only consider tags matching `stage-migrated-*`. Unmigrated images (`stage-{sha}`) are invisible to it.

If the ArgoCD Application is managed via the ArgoCD UI or CLI rather than a manifest file, apply these annotations manually:

```bash
kubectl annotate application <app-name> -n argocd \
  argocd-image-updater.argoproj.io/image-list="app=620167017830.dkr.ecr.us-east-1.amazonaws.com/udab-server" \
  argocd-image-updater.argoproj.io/app.update-strategy="regexp" \
  argocd-image-updater.argoproj.io/app.allow-tags="regexp:^stage-migrated-" \
  --overwrite
```

**Step 2: Commit (if in a manifest file)**

Commit wherever the Application manifest lives.

---

## Task 7: End-to-end validation on staging

**Step 1: Verify pre-requisites are in place**

```bash
# Check ServiceAccount exists
kubectl get serviceaccount migration-runner -n <namespace>

# Check GitHub repo variables are set
gh variable list
# Should show: EKS_CLUSTER_NAME, STAGE_NAMESPACE, STAGE_SECRET_NAME, STAGE_DIST_SECRET_NAME
```

**Step 2: Push to stage and monitor**

```bash
git push origin stage
```

Watch the workflow in GitHub Actions. After the build step, verify the migration Job was submitted:

```bash
kubectl get jobs -n <namespace> | grep db-migration
```

**Step 3: Monitor migration Job completion**

```bash
kubectl logs -f job/db-migration-<short-sha> -n <namespace>
```

Expected output ends with: `Image retagged as stage-migrated-{sha}`

**Step 4: Verify ECR tag exists**

```bash
aws ecr describe-images --repository-name udab-server \
  --query 'imageDetails[?contains(imageTags, `stage-migrated-`) == `true`].imageTags' \
  --output text
```

**Step 5: Verify ArgoCD Image Updater picks up the new tag**

Check ArgoCD UI or:
```bash
kubectl logs -f deployment/argocd-image-updater -n argocd | grep udab-server
```

**Step 6: Verify pods are running the new image**

```bash
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].image}'
```

Should show the `stage-migrated-{sha}` tagged image.

---

## Summary of changes by repo

| Repo | File | Action |
|------|------|--------|
| udab-server | `Dockerfile` | Add `awscli` |
| udab-server | `migrate.sh` | Add ECR retag logic with retry |
| udab-server | `run_remote.sh` | Remove `alembic upgrade head` |
| udab-server | `.github/workflows/stage.yaml` | Add kubectl Job submission, change tag format |
| helm | `charts/application/templates/job.yaml` | Delete |
| helm | `charts/application/values.yaml` | Remove `jobs:` and `syncWave:` |
| helm | `charts/application/templates/deployment.yaml` | Remove sync-wave annotation |
| ArgoCD | Application resource annotations | Configure Image Updater for `stage-migrated-*` |
