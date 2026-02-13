# Migration Gating via ECR Tag Convention

## Problem

ArgoCD SyncWave-based migration Jobs are unreliable:
- Migrations run too many times (static Job name + TTL cleanup = ArgoCD re-creates them)
- Pods launch before migrations finish (SyncWave has weak ordering guarantees for long-running Jobs)
- `run_remote.sh` also runs `alembic upgrade head` on every pod start, causing redundant migration runs
- Migrations can take seconds to hours, which is fundamentally incompatible with ArgoCD's sync model

## Solution

Move migration orchestration out of ArgoCD. Use ECR image tags as a gate so ArgoCD Image Updater only deploys images that have been successfully migrated.

## Flow

```
Push to stage branch
       |
       v
GitHub Actions: build-and-migrate workflow
  1. Build image
  2. Push to ECR as stage-{sha}
  3. kubectl apply migration Job using stage-{sha}
  4. Workflow ends (~5 min)
       |
       v
Kubernetes: Migration Job (seconds to hours, zero GHA minutes)
  1. Runs alembic upgrade head
  2. On success: retags ECR image as stage-migrated-{sha}
  3. On failure: nothing happens, no tag, no deployment
       |
       v
ArgoCD Image Updater (configured to match only stage-migrated-*)
  1. Detects new stage-migrated-{sha} tag
  2. Updates Deployment image
  3. ArgoCD syncs, pods deploy with already-migrated database
```

## Changes Required

### udab-server repo

#### 1. run_remote.sh — Remove migration from pod startup

Before:
```bash
#!/bin/bash
set -e
alembic upgrade head
uvicorn app.main:app --host 0.0.0.0
```

After:
```bash
#!/bin/bash
set -e
uvicorn app.main:app --host 0.0.0.0
```

#### 2. migrate.sh — Add ECR retag on success

```bash
#!/bin/bash
set -e

alembic upgrade head

# Tag image as migrated so ArgoCD Image Updater can pick it up
MANIFEST=$(aws ecr batch-get-image \
  --repository-name udab-server \
  --image-ids imageTag=stage-$IMAGE_SHA \
  --query 'images[0].imageManifest' \
  --output text)

aws ecr put-image \
  --repository-name udab-server \
  --image-tag "stage-migrated-$IMAGE_SHA" \
  --image-manifest "$MANIFEST"

echo "Migration complete. Image tagged as stage-migrated-$IMAGE_SHA."
```

#### 3. Dockerfile — Add AWS CLI

Add to Dockerfile:
```dockerfile
RUN pip install awscli
```

#### 4. .github/workflows/stage.yaml — Submit migration Job after build

Add a step after image push:

```yaml
- name: Configure kubectl
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::620167017830:role/github-oidc
    aws-region: us-east-1

- name: Update kubeconfig
  run: aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-1

- name: Submit migration Job
  run: |
    SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
    IMAGE="620167017830.dkr.ecr.us-east-1.amazonaws.com/udab-server:stage-${{ github.sha }}"

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
            image: $IMAGE
            command: ["/migrate.sh"]
            env:
            - name: IMAGE_SHA
              value: "${{ github.sha }}"
            envFrom:
            - secretRef:
                name: $SECRET_NAME
    EOF
```

### helm repo

#### 5. Remove migration Job from Helm chart

- Delete `templates/job.yaml`
- Remove `jobs:` section from `values.yaml`
- Remove `syncWave` from `values.yaml` (no longer needed)

#### 6. ArgoCD Image Updater — Match only migrated tags

On the ArgoCD Application resource (wherever it lives):

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=620167017830.dkr.ecr.us-east-1.amazonaws.com/udab-server
    argocd-image-updater.argoproj.io/app.update-strategy: regexp
    argocd-image-updater.argoproj.io/app.allow-tags: regexp:^stage-migrated-
```

### AWS / Kubernetes

#### 7. Create IRSA ServiceAccount for migration Jobs

IAM Role (`migration-runner`):
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

Kubernetes ServiceAccount:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: migration-runner
  namespace: <namespace>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::620167017830:role/migration-runner
```

## Rollout Plan (Staging First)

1. Create the IAM role and k8s ServiceAccount
2. Update `Dockerfile` to include `awscli`
3. Update `migrate.sh` with ECR retag logic
4. Update `run_remote.sh` to remove `alembic upgrade head`
5. Update `stage.yaml` workflow to submit migration Job
6. Update ArgoCD Image Updater to match `stage-migrated-*` only
7. Remove `job.yaml` and `jobs:` from Helm chart
8. Test by pushing to stage — verify migration runs, retag happens, Image Updater picks it up, pods deploy

## Why This Works

- **No race condition**: Pods cannot deploy until the migrated tag exists, which only happens after migration succeeds
- **No GHA minute burn**: Migration runs on cluster infrastructure
- **No redundant migrations**: `run_remote.sh` no longer runs migrations; only the Job does
- **Failure is safe**: Migration failure = no tag = no deployment
- **ArgoCD stays in control**: Image Updater is still the mechanism that triggers deployments
- **Variable duration is fine**: There's no timeout pressure — the Job runs as long as it needs
- **Idempotent fallback**: If something goes wrong and migration runs twice, it's harmless

## Production Rollout

After validating on staging, replicate with `prod-migrated-*` tag convention and update the prod workflow similarly.
