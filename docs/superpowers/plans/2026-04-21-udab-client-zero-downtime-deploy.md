# udab-client Zero-Downtime Deploy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate 503s during udab-client rolling deploys by fixing pod termination race and capacity gaps.

**Architecture:** Add a `preStop` lifecycle sleep (15s) so nginx stays alive while ALB drains connections. Set `maxUnavailable: 0` so capacity never drops below 4 pods. Drop `successThreshold` from 2→1 so new pods join faster. All three changes go in one Helm template file.

**Tech Stack:** Helm chart YAML template (Go templating), Kubernetes Deployment spec

---

### Task 1: Add rolling update strategy

**Files:**
- Modify: `helm/charts/application/templates/deployment.yaml` — add `strategy` block

**Context:** The deployment currently has no explicit `strategy`, defaulting to `maxUnavailable=1, maxSurge=1`. With 4 replicas this means 1 pod can be killed before its replacement is ready — a 25% capacity drop. Setting `maxUnavailable: 0` prevents any old pod from terminating until a new pod is Ready. `maxSurge: 1` allows spinning up the 5th (replacement) pod first.

- [ ] **Step 1: Open the deployment template**

File: `helm/charts/application/templates/deployment.yaml`

Find the block:
```yaml
spec:
  replicas: 4
  progressDeadlineSeconds: 600
  selector:
```

- [ ] **Step 2: Insert the strategy block**

Add `strategy` after `progressDeadlineSeconds: 600`:

```yaml
spec:
  replicas: 4
  progressDeadlineSeconds: 600
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
```

- [ ] **Step 3: Verify render with helm template**

```bash
cd /workspace/helm
helm template test-release charts/application -f charts/application/test-values.yaml | grep -A 8 "strategy:"
```

Expected output contains:
```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

- [ ] **Step 4: Commit**

```bash
cd /workspace/helm
git add charts/application/templates/deployment.yaml
git commit -m "fix: set maxUnavailable=0 maxSurge=1 to prevent capacity drops during rollout"
```

---

### Task 2: Add preStop lifecycle hook

**Files:**
- Modify: `helm/charts/application/templates/deployment.yaml` — add `lifecycle.preStop` inside container spec

**Context:** When K8s terminates a pod it simultaneously removes the pod from Service endpoints (triggering ALB deregistration) AND starts the preStop hook. ALB drains connections over `deregistration_delay=10s`. The preStop sleep of 15s keeps nginx alive for the entire 10s drain window. At t=15s SIGTERM arrives — nginx is already cut off from ALB traffic. `terminationGracePeriodSeconds: 30` provides 15s of headroom after SIGTERM.

**Critical invariant:** `preStop sleep seconds (15) >= deregistration_delay seconds (10)`. If `deregistration_delay` is ever increased in `dns.tf`, this sleep must increase proportionally.

- [ ] **Step 1: Locate the container spec in the template**

Find this block in `helm/charts/application/templates/deployment.yaml`:

```yaml
      containers:
        - image: "{{ .Values.image.url }}/{{ .Values.image.name }}:{{ .Values.image.tag }}"
          imagePullPolicy: Always
          {{ if .Values.app.is_client }}
          volumeMounts:
```

- [ ] **Step 2: Add lifecycle block after imagePullPolicy**

Insert `lifecycle` immediately after `imagePullPolicy: Always`:

```yaml
      containers:
        - image: "{{ .Values.image.url }}/{{ .Values.image.name }}:{{ .Values.image.tag }}"
          imagePullPolicy: Always
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "15"]
          {{ if .Values.app.is_client }}
          volumeMounts:
```

- [ ] **Step 3: Verify render**

```bash
cd /workspace/helm
helm template test-release charts/application -f charts/application/test-values.yaml | grep -A 5 "lifecycle:"
```

Expected output:
```yaml
          lifecycle:
            preStop:
              exec:
                command:
                - sleep
                - "15"
```

- [ ] **Step 4: Commit**

```bash
cd /workspace/helm
git add charts/application/templates/deployment.yaml
git commit -m "fix: add preStop sleep 15s to outlive ALB deregistration_delay before nginx exits"
```

---

### Task 3: Fix readiness probe successThreshold

**Files:**
- Modify: `helm/charts/application/templates/deployment.yaml` — change `successThreshold: 2` → `successThreshold: 1`

**Context:** `successThreshold: 2` means a pod needs 2 consecutive passing probe checks before K8s marks it Ready. With `periodSeconds` defaulting to 10s and `initialDelaySeconds: 10`, a pod takes ~30s minimum to become Ready (10s wait + 2×10s checks). Dropping to `successThreshold: 1` cuts this to ~20s, reducing the window where the surge pod occupies a slot without serving traffic.

- [ ] **Step 1: Find the readinessProbe block**

In `helm/charts/application/templates/deployment.yaml`, locate:

```yaml
          readinessProbe:
            httpGet:
              path: {{ .Values.app.pod.healthCheckPath }}
              port: {{ .Values.app.pod.port }}
            timeoutSeconds: 10
            initialDelaySeconds: 10
            successThreshold: 2
```

- [ ] **Step 2: Change successThreshold**

```yaml
          readinessProbe:
            httpGet:
              path: {{ .Values.app.pod.healthCheckPath }}
              port: {{ .Values.app.pod.port }}
            timeoutSeconds: 10
            initialDelaySeconds: 10
            successThreshold: 1
```

- [ ] **Step 3: Verify render**

```bash
cd /workspace/helm
helm template test-release charts/application -f charts/application/test-values.yaml | grep -A 8 "readinessProbe:"
```

Expected output contains `successThreshold: 1` (not 2).

- [ ] **Step 4: Full template lint check**

```bash
cd /workspace/helm
helm template test-release charts/application -f charts/application/test-values.yaml > /dev/null && echo "OK"
```

Expected: `OK` with no errors.

- [ ] **Step 5: Commit**

```bash
cd /workspace/helm
git add charts/application/templates/deployment.yaml
git commit -m "fix: drop readiness successThreshold 2->1 to cut pod warm-up from 30s to 20s"
```

---

### Task 4: Verify final rendered deployment

**Files:**
- Read-only: `helm/charts/application/templates/deployment.yaml`

- [ ] **Step 1: Render full deployment and inspect all three changes**

```bash
cd /workspace/helm
helm template test-release charts/application -f charts/application/test-values.yaml
```

Confirm all three are present in the output:

1. `strategy` block with `maxUnavailable: 0` and `maxSurge: 1`
2. `lifecycle.preStop.exec.command: [sleep, "15"]`
3. `readinessProbe.successThreshold: 1`

- [ ] **Step 2: Confirm terminationGracePeriodSeconds still 30**

```bash
cd /workspace/helm
helm template test-release charts/application -f charts/application/test-values.yaml | grep terminationGracePeriodSeconds
```

Expected: `terminationGracePeriodSeconds: 30`

This confirms the invariant: preStop(15s) + nginx shutdown(~1s) = ~16s, well under 30s grace period.

- [ ] **Step 3: Final state of deployment.yaml**

The complete modified file should look like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.service.name }}
  annotations:
    {{- if .Values.app.annotations }}
{{ .Values.app.annotations | toYaml | indent 4 }}
    {{- end }}
spec:
  replicas: 4
  progressDeadlineSeconds: 600
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: {{ .Values.app.service.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.service.name }}
    spec:
      tolerations:
        - key: node_group_type
          operator: Equal
          value: on_demand_main
          effect: NoSchedule
      nodeSelector:
        node_group_type: on_demand_main
      {{- if .Values.iam.service_account_name }}
      serviceAccountName:  {{ .Values.iam.service_account_name }}
      {{ end }}
      terminationGracePeriodSeconds: 30
      containers:
        - image: "{{ .Values.image.url }}/{{ .Values.image.name }}:{{ .Values.image.tag }}"
          imagePullPolicy: Always
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "15"]
          {{ if .Values.app.is_client }}
          volumeMounts:
            - mountPath: /var/cache/nginx
              name: client-nginx-cache
          {{ end }}
          name: {{ .Values.app.service.name }}
          {{- if .Values.app.command }}
          command: {{ .Values.app.command }}
          {{- end }}
          ports:
            - containerPort: {{ .Values.app.pod.port }}
            - containerPort: 8001
          livenessProbe:
            httpGet:
              path: {{ .Values.app.pod.healthCheckPath }}
              port: {{ .Values.app.pod.port }}
            timeoutSeconds: 10
            initialDelaySeconds: 10
          readinessProbe:
            httpGet:
              path: {{ .Values.app.pod.healthCheckPath }}
              port: {{ .Values.app.pod.port }}
            timeoutSeconds: 10
            initialDelaySeconds: 10
            successThreshold: 1
          resources:
            requests:
              memory: {{ .Values.app.resources.requests.memory }}
              cpu: {{ .Values.app.resources.requests.cpu }}
            limits:
              memory: {{ .Values.app.resources.limits.memory }}
              cpu: {{ .Values.app.resources.limits.cpu }}
          envFrom:
            - secretRef:
                name: {{ .Values.app.secretName }}
          {{ if .Values.app.is_client }}
          securityContext:
            allowPrivilegeEscalation: false
            runAsUser: 0
          {{ end }}
      restartPolicy: Always
      {{ if .Values.app.is_client }}
      volumes:
        - name: client-nginx-cache
          emptyDir: { }
      {{ end }}
status: { }
```

---

## Production Verification

After ArgoCD syncs the new chart version to prod:

```bash
# Watch rollout complete without error
kubectl rollout status deployment/udab-client -n prod

# Smoke test during next deploy — run in a terminal while deploy is happening
while true; do
  code=$(curl -o /dev/null -s -w "%{http_code}" https://<your-domain>)
  echo "$(date +%T) $code"
  sleep 1
done
```

Expected: all 200s, zero 503s throughout the rolling update.

Also watch in AWS console: EC2 → Target Groups → `udab-prod-client` → Targets. All targets should stay Healthy (green) throughout the deploy.
