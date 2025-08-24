# Kustomize Practice Labs: From Basic to Advanced

**Complete hands-on exercises with step-by-step solutions**

---

## Table of Contents

1. [Lab Setup](#lab-setup)
2. [Lab 1: Basic Kustomization](#lab-1-basic-kustomization)
3. [Lab 2: Environment-Specific Overlays](#lab-2-environment-specific-overlays)
4. [Lab 3: Patches and Transformations](#lab-3-patches-and-transformations)
5. [Lab 4: ConfigMap and Secret Generators](#lab-4-configmap-and-secret-generators)
6. [Lab 5: Advanced Features - Components](#lab-5-advanced-features---components)
7. [Lab 6: JSON Patches and Replacements](#lab-6-json-patches-and-replacements)
8. [Lab 7: Multi-Application Deployment](#lab-7-multi-application-deployment)
9. [Lab 8: CI/CD Pipeline Integration](#lab-8-cicd-pipeline-integration)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Lab Setup

### Prerequisites
- Kubernetes cluster (Docker Desktop, Minikube, or cloud cluster)
- kubectl installed and configured
- kustomize CLI (optional, as it's built into kubectl)

### Verify Setup
```bash
# Check kubectl version (should be 1.14+)
kubectl version --client

# Check kustomize availability
kubectl kustomize --help

# Create practice directory
mkdir kustomize-labs
cd kustomize-labs
```

---

## Lab 1: Basic Kustomization

### Objective
Learn basic kustomization concepts by creating a simple web application deployment.

### Exercise
Create a basic nginx deployment using Kustomize.

### Step 1: Create Directory Structure
```bash
mkdir lab1-basic
cd lab1-basic
```

### Step 2: Create Kubernetes Resources

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Step 3: Create Kustomization File

**kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  name: nginx-app

resources:
- deployment.yaml
- service.yaml

commonLabels:
  app: nginx-webapp
  version: v1.0.0

commonAnnotations:
  managed-by: kustomize
  contact: devops-team@company.com

namePrefix: basic-

namespace: default
```

### Step 4: Build and Deploy
```bash
# Build configuration
kubectl kustomize .

# Apply to cluster
kubectl apply -k .

# Verify deployment
kubectl get pods -l app=nginx-webapp
kubectl get services -l app=nginx-webapp
```

### Expected Output
```
deployment.apps/basic-nginx-deployment created
service/basic-nginx-service created
```

### Verification
```bash
# Check if resources have correct labels and names
kubectl get pods -o wide
kubectl describe deployment basic-nginx-deployment
```

### Solution Files
Your directory should look like:
```
lab1-basic/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

---

## Lab 2: Environment-Specific Overlays

### Objective
Create base configurations and environment-specific overlays for development, staging, and production.

### Exercise
Set up a multi-environment deployment structure.

### Step 1: Create Directory Structure
```bash
cd ..
mkdir -p lab2-overlays/{base,overlays/{dev,staging,prod}}
cd lab2-overlays
```

### Step 2: Create Base Configuration

**base/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        env:
        - name: ENV
          value: "base"
```

**base/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**base/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
data:
  app.properties: |
    server.port=80
    logging.level=INFO
    database.host=localhost
    cache.enabled=false
```

**base/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app: web-application
```

### Step 3: Create Development Overlay

**overlays/dev/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

namePrefix: dev-

namespace: development

commonLabels:
  environment: development

patchesStrategicMerge:
- deployment-patch.yaml
- service-patch.yaml

configMapGenerator:
- name: web-app-config
  behavior: merge
  literals:
  - logging.level=DEBUG
  - cache.enabled=true
```

**overlays/dev/deployment-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: web-app
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        env:
        - name: ENV
          value: "development"
```

**overlays/dev/service-patch.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: NodePort
```

### Step 4: Create Staging Overlay

**overlays/staging/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

namePrefix: staging-

namespace: staging

commonLabels:
  environment: staging

images:
- name: nginx
  newTag: "1.21"

replicas:
- name: web-app
  count: 3

configMapGenerator:
- name: web-app-config
  behavior: merge
  literals:
  - database.host=staging-db.company.com
  - logging.level=WARN
```

### Step 5: Create Production Overlay

**overlays/prod/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

namePrefix: prod-

namespace: production

commonLabels:
  environment: production

images:
- name: nginx
  newTag: "1.21"

replicas:
- name: web-app
  count: 5

patchesStrategicMerge:
- deployment-patch.yaml
- service-patch.yaml

configMapGenerator:
- name: web-app-config
  behavior: merge
  literals:
  - database.host=prod-db.company.com
  - logging.level=ERROR
  - cache.enabled=true
```

**overlays/prod/deployment-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: web-app
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: ENV
          value: "production"
```

**overlays/prod/service-patch.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: LoadBalancer
```

### Step 6: Build and Deploy Each Environment
```bash
# Create namespaces
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Deploy development
kubectl kustomize overlays/dev/
kubectl apply -k overlays/dev/

# Deploy staging
kubectl kustomize overlays/staging/
kubectl apply -k overlays/staging/

# Deploy production
kubectl kustomize overlays/prod/
kubectl apply -k overlays/prod/
```

### Verification
```bash
# Check all deployments
kubectl get pods --all-namespaces -l app=web-application

# Compare configurations
kubectl get configmaps -n development -o yaml
kubectl get configmaps -n production -o yaml

# Check service types
kubectl get services --all-namespaces -l app=web-application
```

---

## Lab 3: Patches and Transformations

### Objective
Master different patching techniques: strategic merge patches, JSON patches, and inline patches.

### Exercise
Modify deployments using various patch strategies.

### Step 1: Create Base Application
```bash
cd ..
mkdir lab3-patches
cd lab3-patches
mkdir base
```

**base/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: httpd:2.4
        ports:
        - containerPort: 80
        env:
        - name: LOG_LEVEL
          value: "info"
        - name: PORT
          value: "80"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
```

**base/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-server
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**base/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
```

### Step 2: Strategic Merge Patches

Create directory for strategic merge example:
```bash
mkdir strategic-merge
```

**strategic-merge/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

patchesStrategicMerge:
- deployment-patch.yaml

commonLabels:
  patch-type: strategic-merge
```

**strategic-merge/deployment-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 4
  template:
    spec:
      containers:
      - name: api
        image: httpd:2.4.51
        env:
        - name: LOG_LEVEL
          value: "debug"
        - name: NEW_ENV_VAR
          value: "added-by-patch"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "400m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 5
```

### Step 3: JSON Patches

Create directory for JSON patch example:
```bash
mkdir json-patch
```

**json-patch/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

patchesJson6902:
- target:
    version: v1
    kind: Deployment
    name: api-server
  path: deployment-json-patch.yaml

commonLabels:
  patch-type: json-patch
```

**json-patch/deployment-json-patch.yaml**
```yaml
# Change replica count
- op: replace
  path: /spec/replicas
  value: 3

# Add new environment variable
- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: PATCH_TYPE
    value: "json-patch"

# Modify existing environment variable
- op: replace
  path: /spec/template/spec/containers/0/env/0/value
  value: "warning"

# Add readiness probe
- op: add
  path: /spec/template/spec/containers/0/readinessProbe
  value:
    httpGet:
      path: /ready
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 3

# Change image tag
- op: replace
  path: /spec/template/spec/containers/0/image
  value: "httpd:2.4.52"
```

### Step 4: Inline Patches

Create directory for inline patch example:
```bash
mkdir inline-patch
```

**inline-patch/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 6
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value:
        name: INLINE_PATCH
        value: "true"
  target:
    kind: Deployment
    name: api-server

- patch: |-
    - op: replace
      path: /spec/type
      value: NodePort
  target:
    kind: Service
    name: api-service

commonLabels:
  patch-type: inline-patch
```

### Step 5: Test Each Patch Type
```bash
# Test strategic merge
echo "=== Strategic Merge Patch ==="
kubectl kustomize strategic-merge/

# Test JSON patch
echo "=== JSON Patch ==="
kubectl kustomize json-patch/

# Test inline patch
echo "=== Inline Patch ==="
kubectl kustomize inline-patch/
```

### Step 6: Apply and Verify
```bash
# Apply strategic merge
kubectl apply -k strategic-merge/
kubectl get deployment api-server -o jsonpath='{.spec.replicas}'
kubectl describe deployment api-server

# Clean up and apply JSON patch
kubectl delete -k strategic-merge/
kubectl apply -k json-patch/
kubectl get deployment api-server -o jsonpath='{.spec.replicas}'

# Clean up and apply inline patch
kubectl delete -k json-patch/
kubectl apply -k inline-patch/
kubectl get services api-service -o jsonpath='{.spec.type}'
```

---

## Lab 4: ConfigMap and Secret Generators

### Objective
Learn to generate ConfigMaps and Secrets from files, literals, and environment files.

### Exercise
Create a web application with database configuration using generators.

### Step 1: Setup Directory Structure
```bash
cd ..
mkdir lab4-generators
cd lab4-generators
mkdir -p configs secrets
```

### Step 2: Create Configuration Files

**configs/app.properties**
```properties
# Application Configuration
server.port=8080
server.name=MyWebApp
logging.level.root=INFO
logging.level.com.myapp=DEBUG

# Database Configuration
database.driver=postgresql
database.port=5432
database.name=myapp_db
database.pool.min=5
database.pool.max=20

# Cache Configuration
cache.enabled=true
cache.ttl=300
cache.provider=redis
```

**configs/nginx.conf**
```nginx
server {
    listen 80;
    server_name localhost;
    
    location / {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /health {
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}
```

**secrets/database.env**
```
DATABASE_HOST=postgres.example.com
DATABASE_USERNAME=myapp_user
DATABASE_PASSWORD=super-secret-password
DATABASE_URL=postgresql://myapp_user:super-secret-password@postgres.example.com:5432/myapp_db
```

**secrets/api-keys.txt**
```
stripe-api-key=sk_test_4eC39HqLyjWDarjtT1zdp7dc
sendgrid-api-key=SG.1234567890abcdef.1234567890abcdef1234567890
jwt-secret=my-super-secret-jwt-signing-key-that-should-be-random
```

### Step 3: Create Deployment

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: nginx:1.20
        ports:
        - containerPort: 80
        env:
        - name: CONFIG_FILE
          value: "/etc/config/app.properties"
        envFrom:
        - secretRef:
            name: database-secrets
        - configMapRef:
            name: app-runtime-config
        volumeMounts:
        - name: app-config
          mountPath: /etc/config
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: api-keys
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: app-config
        configMap:
          name: app-config
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: api-keys
        secret:
          secretName: api-keys
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Step 4: Create Kustomization with Generators

**kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

# ConfigMap from files
configMapGenerator:
- name: app-config
  files:
  - configs/app.properties
- name: nginx-config
  files:
  - default.conf=configs/nginx.conf

# ConfigMap from literals
- name: app-runtime-config
  literals:
  - ENV=production
  - DEBUG=false
  - LOG_FORMAT=json
  - CACHE_SIZE=100MB

# Secret from environment file
secretGenerator:
- name: database-secrets
  envs:
  - secrets/database.env

# Secret from files
- name: api-keys
  files:
  - secrets/api-keys.txt
  type: Opaque

# Secret from literals
- name: app-secrets
  literals:
  - session-secret=my-session-secret-key
  - encryption-key=AES256-encryption-key-here
  type: Opaque

generatorOptions:
  disableNameSuffixHash: false
  labels:
    generated-by: kustomize
  annotations:
    config.kubernetes.io/origin: generator
```

### Step 5: Advanced Generator Configuration

Create environment-specific override:

**overlays/production/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../

# Override ConfigMaps for production
configMapGenerator:
- name: app-runtime-config
  behavior: replace
  literals:
  - ENV=production
  - DEBUG=false
  - LOG_FORMAT=structured
  - CACHE_SIZE=1GB
  - MAX_CONNECTIONS=1000

# Add additional production secrets
secretGenerator:
- name: prod-certificates
  files:
  - tls.crt
  - tls.key
  type: kubernetes.io/tls

# Merge additional database config
- name: database-secrets
  behavior: merge
  literals:
  - DATABASE_SSL_MODE=require
  - DATABASE_TIMEOUT=30

generatorOptions:
  disableNameSuffixHash: false
  labels:
    environment: production
    generated-by: kustomize
```

### Step 6: Build and Inspect
```bash
# Build base configuration
kubectl kustomize .

# Build production overlay
kubectl kustomize overlays/production/

# Apply base
kubectl apply -k .

# Inspect generated resources
kubectl get configmaps
kubectl get secrets
kubectl describe configmap app-config
kubectl describe secret database-secrets
```

### Verification Commands
```bash
# Check ConfigMap contents
kubectl get configmap app-config -o yaml
kubectl get configmap nginx-config -o yaml

# Check Secret contents (base64 encoded)
kubectl get secret database-secrets -o yaml
kubectl get secret api-keys -o yaml

# Verify deployment mounts
kubectl describe deployment webapp
kubectl describe pod -l app=webapp
```

---

## Lab 5: Advanced Features - Components

### Objective
Learn to create reusable components for monitoring, logging, and security configurations.

### Exercise
Build a component-based architecture with shared monitoring and logging components.

### Step 1: Create Component Structure
```bash
cd ..
mkdir -p lab5-components/{base,components/{monitoring,logging,security},overlays/{dev,prod}}
cd lab5-components
```

### Step 2: Create Base Application

**base/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: microservice
  template:
    metadata:
      labels:
        app: microservice
    spec:
      containers:
      - name: app
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

**base/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: microservice
spec:
  selector:
    app: microservice
  ports:
  - port: 80
    targetPort: 80
```

**base/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

commonLabels:
  app: microservice
```

### Step 3: Create Monitoring Component

**components/monitoring/servicemonitor.yaml**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
spec:
  selector:
    matchLabels:
      monitoring: enabled
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

**components/monitoring/prometheusrule.yaml**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
spec:
  groups:
  - name: app.rules
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value }} for {{ $labels.job }}"
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.8
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage"
        description: "Memory usage is above 80%"
```

**components/monitoring/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- servicemonitor.yaml
- prometheusrule.yaml

commonLabels:
  monitoring: enabled

patchesStrategicMerge:
- service-patch.yaml
```

**components/monitoring/service-patch.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: microservice
  labels:
    monitoring: enabled
spec:
  ports:
  - port: 80
    targetPort: 80
    name: http
```

### Step 4: Create Logging Component

**components/logging/sidecar-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice
spec:
  template:
    spec:
      containers:
      - name: log-shipper
        image: fluent/fluent-bit:1.8
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc
      volumes:
      - name: varlog
        emptyDir: {}
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
```

**components/logging/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/*.log
        Parser            docker
        Tag               app.*

    [OUTPUT]
        Name  stdout
        Match *
        
  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L
```

**components/logging/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- configmap.yaml

patchesStrategicMerge:
- sidecar-patch.yaml

commonLabels:
  logging: enabled
```

### Step 5: Create Security Component

**components/security/networkpolicy.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: microservice-netpol
spec:
  podSelector:
    matchLabels:
      app: microservice
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: gateway
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443  # HTTPS
    - protocol: TCP
      port: 53   # DNS
    - protocol: UDP
      port: 53   # DNS
```

**components/security/podsecuritypolicy.yaml**
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: microservice-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

**components/security/security-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: app
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
```

**components/security/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- networkpolicy.yaml
- podsecuritypolicy.yaml

patchesStrategicMerge:
- security-patch.yaml

commonLabels:
  security: hardened
```

### Step 6: Create Environment Overlays

**overlays/dev/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

components:
- ../../components/logging

namePrefix: dev-
namespace: development

commonLabels:
  environment: development

replicas:
- name: microservice
  count: 1
```

**overlays/prod/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

components:
- ../../components/monitoring
- ../../components/logging
- ../../components/security

namePrefix: prod-
namespace: production

commonLabels:
  environment: production

replicas:
- name: microservice
  count: 5

images:
- name: nginx
  newTag: "1.21"
```

### Step 7: Test Component Combinations
```bash
# Build development with only logging
kubectl kustomize overlays/dev/

# Build production with all components
kubectl kustomize overlays/prod/

# Create namespaces
kubectl create namespace development
kubectl create namespace production

# Deploy development
kubectl apply -k overlays/dev/

# Deploy production
kubectl apply -k overlays/prod/
```

### Verification
```bash
# Check development deployment (should have logging sidecar)
kubectl get pods -n development
kubectl describe deployment dev-microservice -n development

# Check production deployment (should have all components)
kubectl get pods -n production
kubectl describe deployment prod-microservice -n production
kubectl get networkpolicies -n production
```

---

## Lab 6: JSON Patches and Replacements

### Objective
Master advanced patching techniques including variable substitution and complex transformations.

### Exercise
Create a complex deployment with dynamic configuration using replacements.

### Step 1: Setup Complex Base
```bash
cd ..
mkdir lab6-advanced
cd lab6-advanced
```

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-app
  annotations:
    app.version: "PLACEHOLDER_VERSION"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complex-app
  template:
    metadata:
      labels:
        app: complex-app
        version: "PLACEHOLDER_VERSION"
    spec:
      containers:
      - name: app
        image: nginx:PLACEHOLDER_VERSION
        env:
        - name: APP_VERSION
          value: "PLACEHOLDER_VERSION"
        - name: DATABASE_HOST
          value: "PLACEHOLDER_DB_HOST"
        - name: REDIS_URL
          value: "redis://PLACEHOLDER_REDIS_HOST:6379"
        - name: API_ENDPOINT
          value: "https://api.PLACEHOLDER_DOMAIN/v1"
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "PLACEHOLDER_MEMORY_REQUEST"
            cpu: "PLACEHOLDER_CPU_REQUEST"
          limits:
            memory: "PLACEHOLDER_MEMORY_LIMIT"
            cpu: "PLACEHOLDER_CPU_LIMIT"
```

**configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.yaml: |
    app:
      name: complex-app
      version: "PLACEHOLDER_VERSION"
      environment: "PLACEHOLDER_ENVIRONMENT"
    
    database:
      host: "PLACEHOLDER_DB_HOST"
      port: 5432
      name: "PLACEHOLDER_DB_NAME"
    
    redis:
      host: "PLACEHOLDER_REDIS_HOST"
      port: 6379
    
    api:
      base_url: "https://api.PLACEHOLDER_DOMAIN"
      timeout: 30
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: complex-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "PLACEHOLDER_SSL_CERT"
spec:
  selector:
    app: complex-app
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

### Step 2: Create Configuration Sources

**config-source.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: deployment-config
data:
  app_version: "1.21.1"
  environment: "production"
  domain: "mycompany.com"
  db_host: "postgres.prod.mycompany.com"
  db_name: "production_db"
  redis_host: "redis.prod.mycompany.com"
  memory_request: "256Mi"
  cpu_request: "200m"
  memory_limit: "512Mi"
  cpu_limit: "500m"
  ssl_cert: "arn:aws:acm:us-west-2:123456789012:certificate/12345678-1234-1234-1234-123456789012"
```

### Step 3: Advanced Kustomization with Replacements

**kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml
- config-source.yaml

# Complex replacements from ConfigMap data
replacements:
# Version replacements
- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.app_version
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - metadata.annotations.[app.version]
    - spec.template.metadata.labels.version
    - spec.template.spec.containers.[name=app].env.[name=APP_VERSION].value
    - spec.template.spec.containers.[name=app].image
    options:
      delimiter: ':'
      index: 1

# Database host replacement
- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.db_host
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - spec.template.spec.containers.[name=app].env.[name=DATABASE_HOST].value
  - select:
      kind: ConfigMap
      name: app-config
    fieldPaths:
    - data.[app.yaml]
    options:
      delimiter: 'PLACEHOLDER_DB_HOST'

# Redis host replacement
- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.redis_host
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - spec.template.spec.containers.[name=app].env.[name=REDIS_URL].value
    options:
      delimiter: 'PLACEHOLDER_REDIS_HOST'
  - select:
      kind: ConfigMap
      name: app-config
    fieldPaths:
    - data.[app.yaml]
    options:
      delimiter: 'PLACEHOLDER_REDIS_HOST'

# Domain replacement
- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.domain
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - spec.template.spec.containers.[name=app].env.[name=API_ENDPOINT].value
    options:
      delimiter: 'PLACEHOLDER_DOMAIN'
  - select:
      kind: ConfigMap
      name: app-config
    fieldPaths:
    - data.[app.yaml]
    options:
      delimiter: 'PLACEHOLDER_DOMAIN'

# Resource limits replacements
- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.memory_request
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - spec.template.spec.containers.[name=app].resources.requests.memory

- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.cpu_request
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - spec.template.spec.containers.[name=app].resources.requests.cpu

- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.memory_limit
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - spec.template.spec.containers.[name=app].resources.limits.memory

- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.cpu_limit
  targets:
  - select:
      kind: Deployment
      name: complex-app
    fieldPaths:
    - spec.template.spec.containers.[name=app].resources.limits.cpu

# SSL certificate replacement
- source:
    kind: ConfigMap
    name: deployment-config
    fieldPath: data.ssl_cert
  targets:
  - select:
      kind: Service
      name: complex-app
    fieldPaths:
    - metadata.annotations.[service.beta.kubernetes.io/aws-load-balancer-ssl-cert]

# Advanced JSON patches for complex modifications
patchesJson6902:
- target:
    version: v1
    kind: Deployment
    name: complex-app
  path: advanced-patches.yaml
```

**advanced-patches.yaml**
```yaml
# Add additional container
- op: add
  path: /spec/template/spec/containers/-
  value:
    name: sidecar-proxy
    image: envoyproxy/envoy:v1.20.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"

# Add init container
- op: add
  path: /spec/template/spec/initContainers
  value:
  - name: migration
    image: migrate/migrate:v4.15.1
    command: ["migrate", "-path", "/migrations", "-database", "postgres://...", "up"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "25m"

# Modify deployment strategy
- op: add
  path: /spec/strategy
  value:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1

# Add volume mounts
- op: add
  path: /spec/template/spec/containers/0/volumeMounts
  value:
  - name: app-config
    mountPath: /etc/config
  - name: cache
    mountPath: /app/cache

# Add volumes
- op: add
  path: /spec/template/spec/volumes
  value:
  - name: app-config
    configMap:
      name: app-config
  - name: cache
    emptyDir:
      sizeLimit: 1Gi

# Add pod disruption budget
- op: add
  path: /spec/template/metadata/annotations
  value:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
```

### Step 4: Environment-Specific Configurations

Create staging configuration:

**staging/config-override.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: deployment-config
data:
  app_version: "1.20.0"
  environment: "staging"
  domain: "staging.mycompany.com"
  db_host: "postgres.staging.mycompany.com"
  db_name: "staging_db"
  redis_host: "redis.staging.mycompany.com"
  memory_request: "128Mi"
  cpu_request: "100m"
  memory_limit: "256Mi"
  cpu_limit: "200m"
  ssl_cert: "arn:aws:acm:us-west-2:123456789012:certificate/staging-cert"
```

**staging/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../

patchesStrategicMerge:
- config-override.yaml

namePrefix: staging-
namespace: staging

replicas:
- name: complex-app
  count: 2
```

### Step 5: Build and Test
```bash
# Build base configuration
kubectl kustomize . > base-output.yaml

# Build staging configuration
kubectl kustomize staging/ > staging-output.yaml

# Compare outputs
echo "=== Base Configuration ==="
grep -A 5 -B 5 "APP_VERSION\|DATABASE_HOST" base-output.yaml

echo "=== Staging Configuration ==="
grep -A 5 -B 5 "APP_VERSION\|DATABASE_HOST" staging-output.yaml

# Apply staging
kubectl create namespace staging
kubectl apply -k staging/

# Verify replacements worked
kubectl get deployment staging-complex-app -n staging -o yaml | grep -A 10 env:
kubectl get configmap staging-app-config -n staging -o yaml
```

---

## Lab 7: Multi-Application Deployment

### Objective
Deploy a complete microservices application stack with frontend, backend, database, and shared services.

### Exercise
Create a full e-commerce application with multiple services and shared components.

### Step 1: Create Project Structure
```bash
cd ..
mkdir -p lab7-multi-app/{applications/{frontend,backend,database,cache},shared/{ingress,monitoring},environments/{dev,staging,prod}}
cd lab7-multi-app
```

### Step 2: Create Frontend Application

**applications/frontend/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: presentation
  template:
    metadata:
      labels:
        app: frontend
        tier: presentation
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        env:
        - name: BACKEND_URL
          value: "http://backend"
        - name: NODE_ENV
          value: "production"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: config
        configMap:
          name: frontend-config
```

**applications/frontend/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**applications/frontend/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  default.conf: |
    server {
        listen 80;
        server_name _;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
        
        location /api {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }
```

**applications/frontend/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  component: frontend
```

### Step 3: Create Backend Application

**applications/backend/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      tier: application
  template:
    metadata:
      labels:
        app: backend
        tier: application
    spec:
      containers:
      - name: app
        image: node:16-alpine
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: DATABASE_URL
          valueFrom:
            secretRef:
              name: database-secrets
              key: url
        - name: REDIS_URL
          value: "redis://cache:6379"
        - name: JWT_SECRET
          valueFrom:
            secretRef:
              name: app-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**applications/backend/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

**applications/backend/hpa.yaml**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**applications/backend/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- hpa.yaml

commonLabels:
  component: backend
```

### Step 4: Create Database Application

**applications/database/statefulset.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: data
  template:
    metadata:
      labels:
        app: database
        tier: data
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "ecommerce"
        - name: POSTGRES_USER
          valueFrom:
            secretRef:
              name: database-secrets
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretRef:
              name: database-secrets
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**applications/database/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
  clusterIP: None  # Headless service for StatefulSet
```

**applications/database/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- statefulset.yaml
- service.yaml

commonLabels:
  component: database
```

### Step 5: Create Cache Application

**applications/cache/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cache
      tier: cache
  template:
    metadata:
      labels:
        app: cache
        tier: cache
    spec:
      containers:
      - name: redis
        image: redis:6-alpine
        ports:
        - containerPort: 6379
        command:
        - redis-server
        - /etc/redis/redis.conf
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        volumeMounts:
        - name: config
          mountPath: /etc/redis
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: redis-config
```

**applications/cache/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: cache
spec:
  selector:
    app: cache
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

**applications/cache/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  redis.conf: |
    bind 0.0.0.0
    port 6379
    timeout 0
    tcp-keepalive 300
    maxmemory 64mb
    maxmemory-policy allkeys-lru
    save ""
    appendonly no
```

**applications/cache/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  component: cache
```

### Step 6: Create Shared Ingress

**shared/ingress/ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.example.com
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
```

**shared/ingress/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ingress.yaml
```

### Step 7: Create Environment Configurations

**environments/dev/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../applications/frontend
- ../../applications/backend
- ../../applications/database
- ../../applications/cache

namePrefix: dev-
namespace: development

commonLabels:
  environment: development

# Development-specific configurations
secretGenerator:
- name: database-secrets
  literals:
  - username=devuser
  - password=devpass123
  - url=postgresql://devuser:devpass123@dev-database:5432/ecommerce

- name: app-secrets
  literals:
  - jwt-secret=dev-jwt-secret-key

# Reduce resources for development
replicas:
- name: frontend
  count: 1
- name: backend
  count: 1

patches:
- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "32Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: "64Mi"
  target:
    kind: Deployment
    name: frontend

# Disable HPA in development
- patch: |-
    $patch: delete
  target:
    kind: HorizontalPodAutoscaler
    name: backend-hpa
```

**environments/prod/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../applications/frontend
- ../../applications/backend
- ../../applications/database
- ../../applications/cache
- ../../shared/ingress

namePrefix: prod-
namespace: production

commonLabels:
  environment: production

# Production secrets (these should come from external secret management)
secretGenerator:
- name: database-secrets
  literals:
  - username=produser
  - password=super-secure-prod-password
  - url=postgresql://produser:super-secure-prod-password@prod-database:5432/ecommerce

- name: app-secrets
  literals:
  - jwt-secret=production-jwt-secret-very-secure-key

# Production scaling
replicas:
- name: frontend
  count: 3
- name: backend
  count: 5
- name: cache
  count: 2

# Production resource limits
patches:
- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "128Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: "256Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "100m"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/cpu
      value: "200m"
  target:
    kind: Deployment
    name: frontend

# Production database with more storage
- patch: |-
    - op: replace
      path: /spec/volumeClaimTemplates/0/spec/resources/requests/storage
      value: "100Gi"
  target:
    kind: StatefulSet
    name: database

# Update ingress for production domain
- patch: |-
    - op: replace
      path: /spec/rules/0/host
      value: "app.mycompany.com"
    - op: replace
      path: /spec/rules/1/host
      value: "api.mycompany.com"
    - op: replace
      path: /spec/tls/0/hosts/0
      value: "api.mycompany.com"
    - op: replace
      path: /spec/tls/0/hosts/1
      value: "app.mycompany.com"
  target:
    kind: Ingress
    name: app-ingress
```

### Step 8: Deploy and Test
```bash
# Create namespaces
kubectl create namespace development
kubectl create namespace production

# Deploy development environment
kubectl apply -k environments/dev/

# Deploy production environment
kubectl apply -k environments/prod/

# Verify deployments
kubectl get all -n development
kubectl get all -n production

# Check services can communicate
kubectl exec -n development -it deployment/dev-backend -- wget -qO- http://dev-database:5432 || echo "Database connection test"
```

### Step 9: Verification Commands
```bash
# Check all applications are running
kubectl get pods -n development -l environment=development
kubectl get pods -n production -l environment=production

# Verify services
kubectl get services -n development
kubectl get services -n production

# Check ingress
kubectl get ingress -n production

# Verify secrets are created
kubectl get secrets -n development
kubectl get secrets -n production

# Test connectivity between services
kubectl exec -n development -it deployment/dev-frontend -- nslookup dev-backend
kubectl exec -n production -it deployment/prod-backend -- nslookup prod-database
```

---

## Lab 8: CI/CD Pipeline Integration

### Objective
Integrate Kustomize with CI/CD pipelines using GitHub Actions and GitOps principles.

### Exercise
Create a complete CI/CD workflow that builds, tests, and deploys applications using Kustomize.

### Step 1: Create GitOps Repository Structure
```bash
cd ..
mkdir lab8-cicd
cd lab8-cicd
mkdir -p {apps/{web-app,api-service},infrastructure/{base,overlays/{staging,production}},scripts,.github/workflows}
```

### Step 2: Create Application Base

**apps/web-app/base/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  annotations:
    image.opencontainers.org/version: "v1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.20
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v1.0.0"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

**apps/web-app/base/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

**apps/web-app/base/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

images:
- name: nginx
  newName: myregistry/web-app
  newTag: latest
```

### Step 3: Create API Service

**apps/api-service/base/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  annotations:
    image.opencontainers.org/version: "v1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api
        image: node:16-alpine
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: VERSION
          value: "v1.0.0"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

**apps/api-service/base/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-service
  ports:
  - port: 80
    targetPort: 3000
```

**apps/api-service/base/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

images:
- name: node
  newName: myregistry/api-service
  newTag: latest
```

### Step 4: Create Environment Overlays

**infrastructure/overlays/staging/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../../apps/web-app/base
- ../../../apps/api-service/base

namePrefix: staging-
namespace: staging

commonLabels:
  environment: staging
  managed-by: kustomize

commonAnnotations:
  deployment.kubernetes.io/revision: "PLACEHOLDER_BUILD_NUMBER"
  app.kubernetes.io/version: "PLACEHOLDER_VERSION"

configMapGenerator:
- name: environment-config
  literals:
  - ENVIRONMENT=staging
  - LOG_LEVEL=debug
  - API_BASE_URL=https://staging-api.example.com

replicas:
- name: web-app
  count: 2
- name: api-service
  count: 2

patches:
- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "32Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: "64Mi"
  target:
    kind: Deployment
    name: web-app
```

**infrastructure/overlays/production/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../../apps/web-app/base
- ../../../apps/api-service/base
- ingress.yaml
- hpa.yaml

namePrefix: prod-
namespace: production

commonLabels:
  environment: production
  managed-by: kustomize

commonAnnotations:
  deployment.kubernetes.io/revision: "PLACEHOLDER_BUILD_NUMBER"
  app.kubernetes.io/version: "PLACEHOLDER_VERSION"

configMapGenerator:
- name: environment-config
  literals:
  - ENVIRONMENT=production
  - LOG_LEVEL=warn
  - API_BASE_URL=https://api.example.com

replicas:
- name: web-app
  count: 5
- name: api-service
  count: 10

patches:
- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "128Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: "256Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "100m"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/cpu
      value: "200m"
  target:
    kind: Deployment
    name: web-app

- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "256Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: "512Mi"
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "200m"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/cpu
      value: "500m"
  target:
    kind: Deployment
    name: api-service
```

**infrastructure/overlays/production/ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "1000"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - app.example.com
    - api.example.com
    secretName: app-tls-cert
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prod-web-app
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prod-api-service
            port:
              number: 80
```

**infrastructure/overlays/production/hpa.yaml**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prod-web-app
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prod-api-service
  minReplicas: 10
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Step 5: Create CI/CD Scripts

**scripts/deploy.sh**
```bash
#!/bin/bash

set -euo pipefail

# Configuration
ENVIRONMENT=${1:-staging}
IMAGE_TAG=${2:-latest}
BUILD_NUMBER=${3:-unknown}

# Validate environment
if [[ ! "$ENVIRONMENT" =~ ^(staging|production)$ ]]; then
    echo "Error: Environment must be 'staging' or 'production'"
    exit 1
fi

echo "Deploying to $ENVIRONMENT with image tag: $IMAGE_TAG"

# Create temporary kustomization
TEMP_DIR=$(mktemp -d)
cp -r infrastructure/overlays/$ENVIRONMENT/* $TEMP_DIR/

# Update image tags
cd $TEMP_DIR
kustomize edit set image myregistry/web-app:$IMAGE_TAG
kustomize edit set image myregistry/api-service:$IMAGE_TAG

# Replace placeholders
sed -i "s/PLACEHOLDER_BUILD_NUMBER/$BUILD_NUMBER/g" kustomization.yaml
sed -i "s/PLACEHOLDER_VERSION/$IMAGE_TAG/g" kustomization.yaml

# Apply to cluster
kubectl apply -k .

# Wait for rollout
kubectl rollout status deployment/${ENVIRONMENT}-web-app -n $ENVIRONMENT --timeout=300s
kubectl rollout status deployment/${ENVIRONMENT}-api-service -n $ENVIRONMENT --timeout=300s

echo "Deployment to $ENVIRONMENT completed successfully"

# Cleanup
rm -rf $TEMP_DIR
```

**scripts/validate.sh**
```bash
#!/bin/bash

set -euo pipefail

echo "Validating Kustomize configurations..."

# Validate each environment
for env in staging production; do
    echo "Validating $env environment..."
    
    # Build configuration
    kubectl kustomize infrastructure/overlays/$env > /tmp/kustomize-$env.yaml
    
    # Validate YAML syntax
    kubectl apply --dry-run=client -f /tmp/kustomize-$env.yaml
    
    # Check for required resources
    if ! grep -q "kind: Deployment" /tmp/kustomize-$env.yaml; then
        echo "Error: No Deployment found in $env"
        exit 1
    fi
    
    if ! grep -q "kind: Service" /tmp/kustomize-$env.yaml; then
        echo "Error: No Service found in $env"
        exit 1
    fi
    
    # Validate resource naming
    if [[ "$env" == "production" ]]; then
        if ! grep -q "name: prod-web-app" /tmp/kustomize-$env.yaml; then
            echo "Error: Production resources not properly prefixed"
            exit 1
        fi
    fi
    
    echo "$env environment validation passed"
done

echo "All validations passed successfully"
```

### Step 6: Create GitHub Actions Workflows

**.github/workflows/ci.yml**
```yaml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.25.0'

    - name: Validate Kustomize configurations
      run: |
        chmod +x scripts/validate.sh
        ./scripts/validate.sh

    - name: Security scan Kustomize configs
      uses: stackrox/kube-linter-action@v1
      with:
        directory: infrastructure/overlays
        format: json
        output-file: kube-linter-results.json

    - name: Upload scan results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: kube-linter-results
        path: kube-linter-results.json

  build:
    needs: validate
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Login to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-

    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.25.0'

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config

    - name: Deploy to staging
      run: |
        chmod +x scripts/deploy.sh
        ./scripts/deploy.sh staging ${{ github.sha }} ${{ github.run_number }}

    - name: Run smoke tests
      run: |
        kubectl wait --for=condition=available --timeout=300s deployment/staging-web-app -n staging
        kubectl wait --for=condition=available --timeout=300s deployment/staging-api-service -n staging
        
        # Test application endpoints
        kubectl port-forward svc/staging-web-app 8080:80 -n staging &
        sleep 5
        curl -f http://localhost:8080/health || exit 1

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.25.0'

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > $HOME/.kube/config

    - name: Deploy to production
      run: |
        chmod +x scripts/deploy.sh
        ./scripts/deploy.sh production ${{ github.sha }} ${{ github.run_number }}

    - name: Verify deployment
      run: |
        kubectl wait --for=condition=available --timeout=600s deployment/prod-web-app -n production
        kubectl wait --for=condition=available --timeout=600s deployment/prod-api-service -n production
        
        # Verify ingress is working
        curl -f https://app.example.com/health || exit 1
        curl -f https://api.example.com/health || exit 1

    - name: Slack notification
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

**.github/workflows/gitops-sync.yml**
```yaml
name: GitOps Sync

on:
  push:
    branches: [ main ]
    paths: [ 'infrastructure/**' ]

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v3
      with:
        repository: company/gitops-configs
        token: ${{ secrets.GITOPS_TOKEN }}
        path: gitops

    - name: Checkout current repo
      uses: actions/checkout@v3
      with:
        path: source

    - name: Update GitOps configurations
      run: |
        # Copy updated configurations to GitOps repo
        cp -r source/infrastructure/* gitops/applications/myapp/
        
        cd gitops
        
        # Check if there are changes
        if git diff --quiet; then
          echo "No changes to sync"
          exit 0
        fi
        
        # Commit and push changes
        git config user.name "GitOps Bot"
        git config user.email "gitops@company.com"
        git add .
        git commit -m "Update myapp configurations from commit ${{ github.sha }}"
        git push
```

### Step 7: Test the Complete Setup

**Create Dockerfile for testing**
```dockerfile
FROM nginx:1.20-alpine

# Copy static files
COPY html/ /usr/share/nginx/html/

# Add health check endpoint
RUN echo '{"status":"healthy","version":"1.0.0"}' > /usr/share/nginx/html/health

EXPOSE 80
```

**Create test HTML**
```bash
mkdir html
echo '<html><body><h1>My Web App</h1><p>Version: 1.0.0</p></body></html>' > html/index.html
```

**Test locally**
```bash
# Make scripts executable
chmod +x scripts/*.sh

# Validate configurations
./scripts/validate.sh

# Test staging deployment (dry run)
kubectl kustomize infrastructure/overlays/staging/

# Test production deployment (dry run)
kubectl kustomize infrastructure/overlays/production/

# Create namespaces for testing
kubectl create namespace staging
kubectl create namespace production

# Deploy to staging
./scripts/deploy.sh staging v1.0.0 123

# Verify deployment
kubectl get pods -n staging
kubectl get services -n staging
kubectl describe deployment staging-web-app -n staging
```

### Verification Commands
```bash
# Check all resources in staging
kubectl get all -n staging

# Check all resources in production
kubectl get all -n production

# Verify image tags are correct
kubectl get deployment staging-web-app -n staging -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check environment-specific configurations
kubectl get configmap staging-environment-config -n staging -o yaml
kubectl get configmap prod-environment-config -n production -o yaml

# Verify ingress configuration
kubectl get ingress -n production

# Check HPA is working
kubectl get hpa -n production
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. Image Pull Errors
```bash
# Check image name and tag
kubectl describe pod <pod-name>

# Verify image exists in registry
docker pull <image-name>

# Check imagePullSecrets if using private registry
kubectl get secrets
```

#### 2. Configuration Not Applied
```bash
# Verify kustomization.yaml syntax
kubectl kustomize . --dry-run

# Check resource references
ls -la  # ensure all referenced files exist

# Validate YAML syntax
kubectl apply --dry-run=client -f <file>
```

#### 3. Patches Not Working
```bash
# Test strategic merge patches
kubectl kustomize . | grep -A 10 -B 10 <expected-change>

# Validate JSON patch syntax
cat patch.yaml | python -m json.tool

# Check target selectors
kubectl kustomize . | grep -B 5 -A 5 "name: <target-name>"
```

#### 4. Environment Variables Not Set
```bash
# Check configmap/secret generation
kubectl get configmaps -o yaml
kubectl get secrets -o yaml

# Verify environment references
kubectl describe pod <pod-name> | grep -A 20 "Environment:"
```

#### 5. Resource Naming Issues
```bash
# Check generated names
kubectl kustomize . | grep "name:"

# Verify namePrefix/nameSuffix application
kubectl get all --show-labels
```

### Debugging Commands

```bash
# Build and inspect output
kubectl kustomize . > output.yaml
cat output.yaml

# Dry run application
kubectl apply --dry-run=client -k .

# Check specific resource generation
kubectl kustomize . | grep -A 20 "kind: Deployment"

# Validate all resource types
kubectl kustomize . | grep "^kind:" | sort | uniq -c

# Check cross-references
kubectl kustomize . | grep -E "(name:|selector:|value:)" | sort
```

This comprehensive lab guide provides hands-on experience with Kustomize from basic concepts to advanced CI/CD integration. Each lab builds upon previous knowledge and includes practical examples that mirror real-world scenarios.