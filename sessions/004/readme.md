# **004:** Introduction to Helm: Package Manager for Kubernetes — *Basic*

## Overview

In the previous sessions, we've learned how to create and manage Kubernetes resources using YAML manifests. While this works great for learning and small deployments, you've probably noticed some challenges:

- **Repetition**: Deploying multiple similar applications means copy-pasting YAML files and changing values manually
- **Configuration Management**: Managing different configurations for dev/staging/prod environments requires maintaining separate YAML files
- **Versioning**: Tracking which version of your application is deployed where is difficult
- **Updates**: Rolling back a deployment means manually reverting multiple YAML files
- **Sharing**: Distributing a complete application stack requires sharing dozens of YAML files

This is where **Helm** comes in—the package manager for Kubernetes.

## Why Helm?

Think of Helm like package managers you already know:

- **apt/yum** for Linux packages
- **npm** for Node.js packages
- **pip** for Python packages
- **brew** for macOS packages

Helm does for Kubernetes what these do for their respective ecosystems: it packages, distributes, and manages applications with a single command.

### The Problem Helm Solves

Let's say you want to deploy a web application with:

- Frontend deployment + service
- Backend deployment + service
- Database statefulset + service
- ConfigMaps for configuration
- Secrets for credentials
- Gateway for routing

Without Helm, you'd need to:

1. Maintain 10+ separate YAML files
2. Manually replace values for different environments (dev/prod)
3. Apply files in the correct order
4. Track which versions are deployed where
5. Manually rollback each file if something goes wrong

With Helm:

```sh
helm install myapp ./mychart --values envs/eu1/values-prod.yaml
```

Done. And rolling back is just:

```sh
helm rollback myapp
```

## Alternatives to Helm

Helm isn't the only solution, not even the best one, as it has some drawbacks. However, it's the most widely adopted. Let's look at some alternatives:

### Kustomize

- Built into `kubectl` (no installation needed)
- Uses overlays instead of templating
- Better type safety (pure YAML, no Go templates)
- **Downside**: More verbose, no built-in package distribution

### Timoni

- Modern alternative built on CUE (type-safe configuration language)
- Strong type safety and validation at build time
- OCI-native (stores modules in container registries)
- **Downside**: Smaller ecosystem, newer project (2023+)
- <https://timoni.sh/>

### Tanka

- Built on Jsonnet with Kubernetes-specific abstractions
- Type-safe, reusable, and composable configurations
- GitOps-friendly and environment management built-in
- Used at Grafana Labs for production deployments
- **Downside**: Requires learning Jsonnet, steeper learning curve
- <https://tanka.dev/>

### KRO (Kubernetes Resource Orchestrator)

- Controller-based approach for defining resource groups
- Declarative resource orchestration with dependencies
- Built-in status aggregation and lifecycle management
- **Downside**: Requires cluster-side controller, newer project, better suited for the platform team, not application
- <https://kro.run/>

### Plain Kubectl

- No abstraction, full control
- **Downside**: Manual management, no versioning, lots of repetition

### Why Helm Wins for Application Teams (Despite Its Flaws)

**Helm is the most widely adopted** solution because:

1. **Massive Ecosystem**: Thousands of ready-made charts at [artifacthub.io](https://artifacthub.io/)
   - Need Prometheus monitoring? `helm install prometheus prometheus-community/prometheus`
   - Need a Kafka Operator? `helm install my-strimzi-kafka-operator oci://quay.io/strimzi-helm/strimzi-kafka-operator --version 0.49.0-rc2`
   - Need PostgreSQL, ValKey, MySQL? Search Artifact Hub for official charts

2. **Industry Standard**: Most vendors provide official Helm charts
3. **Built-in Versioning & Rollback**: Track deployment history automatically
4. **Large Community**: Extensive documentation and support

## Helm's Disadvantages

Let's be honest about Helm's limitations:

### 1. **No Type Safety**

Typos won't be caught until runtime:

```yaml
# This will fail at runtime, not during development
replicas: {{ .Values.replicaCount | default "oops" }}  # Should be a number
```

### 2. **Template Complexity**

Complex charts become hard to debug:

```yaml
{{- if and .Values.enabled (or .Values.feature1 (not .Values.feature2)) }}
  # Good luck debugging this
{{- end }}
```

### 3. **Hidden Logic**

Templates can hide what's actually being deployed, making troubleshooting harder. A GitOps approach with a visual diff, before deploying is helpful here, see [ArgoCD](https://argo-cd.readthedocs.io/en/stable/user-guide/diff-strategies/).

### 4. **Learning Curve**

You "need" to learn Go template syntax on top of Kubernetes YAML

### 5. **State Management**

Helm stores release state in Kubernetes secrets, which can get messy

**Despite these issues**, Helm's ecosystem and convenience make it the pragmatic choice for most teams.

## Let's Build a Helm Chart from Scratch

We'll create a simple full-stack application where the frontend actually calls the backend:

- **Frontend**: A web app that calls the backend API and displays the response
- **Backend**: An API server that responds to the frontend

We'll use Google's sample images that are designed to work together.

### Prerequisites

Install Helm:

```sh
# macOS
brew install helm

# Or download from https://helm.sh/docs/intro/install/
```

Create a kind cluster if you don't have one:

```sh
CLUSTER_NAME="cloud-native-kubernetes-workshop"
kind create cluster --name $CLUSTER_NAME --config kind-cluster.yaml
```

### Step 1: Create the Chart Structure

```sh
# Create the directory structure
mkdir -p fullstack-app/templates

# Create the Chart.yaml file
cat > fullstack-app/Chart.yaml <<'EOF'
apiVersion: v2
name: fullstack-app
description: A simple full-stack application with frontend and backend
type: application
version: 1.0.0
appVersion: "1.0"
keywords:
  - fullstack
  - demo
  - workshop
maintainers:
  - name: Cloud Native Workshop
EOF
```

### Step 2: Create Our Values File

This is where all configurable values live:

```sh
cat > fullstack-app/values.yaml <<'EOF'
frontend:
  name: frontend
  replicas: 2
  image:
    repository: gcr.io/google-samples/hello-frontend
    tag: "1.0"
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi

backend:
  name: hello
  replicas: 2
  image:
    repository: gcr.io/google-samples/hello-go-gke
    tag: "1.0"
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
  resources:
    requests:
      cpu: 100m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi

# Global values that can be referenced by both components
global:
  environment: development
EOF
```

### Step 3: Create Frontend Templates

Create the frontend deployment:

```sh
cat > fullstack-app/templates/frontend-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.frontend.name }}
  labels:
    app: {{ .Values.frontend.name }}
    environment: {{ .Values.global.environment }}
spec:
  replicas: {{ .Values.frontend.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.frontend.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.frontend.name }}
        environment: {{ .Values.global.environment }}
    spec:
      containers:
      - name: {{ .Values.frontend.name }}
        image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
        imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.frontend.service.port }}
        resources:
          {{- toYaml .Values.frontend.resources | nindent 10 }}
EOF
```

Create the frontend service:

```sh
cat > fullstack-app/templates/frontend-service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.frontend.name }}
  labels:
    app: {{ .Values.frontend.name }}
spec:
  type: {{ .Values.frontend.service.type }}
  ports:
  - port: {{ .Values.frontend.service.port }}
    targetPort: {{ .Values.frontend.service.port }}
    protocol: TCP
  selector:
    app: {{ .Values.frontend.name }}
EOF
```

### Step 4: Create Backend Templates

Create the backend deployment:

```sh
cat > fullstack-app/templates/backend-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.backend.name }}
  labels:
    app: {{ .Values.backend.name }}
    environment: {{ .Values.global.environment }}
spec:
  replicas: {{ .Values.backend.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.backend.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.backend.name }}
        environment: {{ .Values.global.environment }}
    spec:
      containers:
      - name: {{ .Values.backend.name }}
        image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
        imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.backend.service.port }}
        resources:
          {{- toYaml .Values.backend.resources | nindent 10 }}
EOF
```

Create the backend service:

```sh
cat > fullstack-app/templates/backend-service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.backend.name }}
  labels:
    app: {{ .Values.backend.name }}
spec:
  type: {{ .Values.backend.service.type }}
  ports:
  - port: {{ .Values.backend.service.port }}
    targetPort: {{ .Values.backend.service.port }}
    protocol: TCP
  selector:
    app: {{ .Values.backend.name }}
EOF
```

### Step 5: Validate and Preview the Chart

Before installing, let's see what Helm will generate:

```sh
# Check for syntax errors
helm lint fullstack-app

# See the generated YAML without installing
helm template fullstack-app ./fullstack-app

# See what will be installed
helm install --dry-run --debug fullstack-app ./fullstack-app
```

### Step 6: Install the Chart

```sh
# Install the chart
helm install myapp ./fullstack-app

# Check what was created
kubectl get all -l environment=development
helm list
helm status myapp
```

### Step 7: Test the Application

```sh
# Test the frontend (it will call the backend and show the response)
kubectl run tmp-shell --rm -i --image nicolaka/netshoot --restart=Never -- curl http://frontend

# Test the backend directly
kubectl run tmp-shell --rm -i --image nicolaka/netshoot --restart=Never -- curl http://hello

# You can also port-forward to test in your browser
kubectl port-forward svc/frontend 8080:80
# Then visit http://localhost:8080 - you'll see the frontend calling the backend!
```

### Step 8: Customize for Production

Create a production values file:

```sh
cat > fullstack-app/values-prod.yaml <<'EOF'
frontend:
  replicas: 3
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

backend:
  replicas: 3
  resources:
    requests:
      cpu: 200m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

global:
  environment: production
EOF
```

Preview the changes:

```sh
helm template myapp ./fullstack-app --values fullstack-app/values-prod.yaml
```

### Step 9: Upgrade the Release

```sh
# Upgrade with production values
helm upgrade myapp ./fullstack-app --values fullstack-app/values-prod.yaml

# Check the upgrade
helm list
kubectl get all -l environment=production

# View upgrade history
helm history myapp
```

### Step 10: Rollback

```sh
# Rollback to the previous version
helm rollback myapp

# Check it rolled back
kubectl get all -l environment=development

# See history
helm history myapp
```

## Helm Best Practices (Quick Tips)

1. **Use `--dry-run --debug`** before every install/upgrade
2. **Version your charts** in `Chart.yaml`
3. **Document values** with comments in `values.yaml`
4. **Use `helm lint`** to catch issues early
5. **Pin image tags** (avoid `latest` in production)
6. **Use `.helmignore`** to exclude unnecessary files
7. **Test templates** with `helm template` before deploying

or look here for more detail: <https://helm.sh/docs/chart_best_practices/>

## Common Helm Commands

```sh
# List releases
helm list

# Get release status
helm status RELEASE_NAME

# See release values
helm get values RELEASE_NAME

# See all manifests
helm get manifest RELEASE_NAME

# History
helm history RELEASE_NAME

# Uninstall
helm uninstall RELEASE_NAME

# Search charts on Artifact Hub
helm search hub wordpress

# Add a chart repository (example)
helm repo add <repo-name> <repo-url>
helm repo update
```

## Cleanup

```sh
# Uninstall your app
helm uninstall myapp

# Verify cleanup
kubectl get all
helm list

# Delete the cluster
kind delete cluster --name $CLUSTER_NAME
```
