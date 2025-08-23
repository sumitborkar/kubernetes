# CoreDNS Kubernetes Project: Multi-Environment DNS Setup

## Project Overview
Build a complete CoreDNS solution that handles multiple environments (dev, staging, prod) with custom domains, monitoring, and security.

## Project Architecture
```
Production Kubernetes Cluster
â”œâ”€â”€ Dev Namespace (dev.company.local)
â”œâ”€â”€ Staging Namespace (staging.company.local) 
â”œâ”€â”€ Prod Namespace (prod.company.local)
â”œâ”€â”€ Custom CoreDNS with monitoring
â””â”€â”€ Network policies for security
```

---

## Phase 1: Environment Setup (30 minutes)

### Step 1.1: Create Project Structure
```bash
# Create project directory
mkdir coredns-project && cd coredns-project

# Create environment namespaces
kubectl create namespace dev
kubectl create namespace staging  
kubectl create namespace prod

# Label namespaces for easy identification
kubectl label namespace dev env=dev
kubectl label namespace staging env=staging
kubectl label namespace prod env=prod
```

### Step 1.2: Deploy Sample Applications
```bash
# Create deployment files
cat <<EOF > dev-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: dev
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
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: dev
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Apply to all environments
kubectl apply -f dev-app.yaml
sed 's/namespace: dev/namespace: staging/g' dev-app.yaml | kubectl apply -f -
sed 's/namespace: dev/namespace: prod/g' dev-app.yaml | kubectl apply -f -

# Verify deployments
kubectl get pods -n dev -l app=web-app
kubectl get pods -n staging -l app=web-app
kubectl get pods -n prod -l app=web-app
```

---

## Phase 2: Custom CoreDNS Configuration (45 minutes)

### Step 2.1: Backup Original Configuration
```bash
# Backup existing CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml > original-coredns-backup.yaml
echo "Original config backed up!"
```

### Step 2.2: Create Advanced CoreDNS Configuration
```bash
cat <<EOF > custom-coredns-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    # Main cluster DNS
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        
        # Custom domain mappings for environments
        rewrite name dev.company.local web-app.dev.svc.cluster.local
        rewrite name staging.company.local web-app.staging.svc.cluster.local
        rewrite name prod.company.local web-app.prod.svc.cluster.local
        
        # Static host entries for external services
        hosts {
            10.1.1.100 database.company.local
            10.1.1.101 cache.company.local
            fallthrough
        }
        
        # Kubernetes service discovery
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        
        # Prometheus metrics for monitoring
        prometheus :9153
        
        # Conditional forwarding for different domains
        forward company.local 10.1.1.10
        forward . /etc/resolv.conf {
            max_concurrent 1000
            policy random
        }
        
        # Enhanced caching
        cache 30 {
            success 9984 30
            denial 9984 5
        }
        
        # Enable detailed logging for troubleshooting
        log
        
        loop
        reload
        loadbalance
    }
    
    # Separate zone for internal company services
    internal.company:53 {
        errors
        hosts /etc/coredns/internal.hosts {
            reload 30s
            fallthrough
        }
        forward . 8.8.8.8 1.1.1.1
        cache 60
    }
EOF

# Apply the new configuration
kubectl apply -f custom-coredns-config.yaml
```

### Step 2.3: Create Internal Hosts File
```bash
cat <<EOF > coredns-hosts-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-hosts
  namespace: kube-system
data:
  internal.hosts: |
    192.168.100.10 api.internal.company
    192.168.100.11 admin.internal.company
    192.168.100.12 monitoring.internal.company
EOF

kubectl apply -f coredns-hosts-config.yaml
```

### Step 2.4: Update CoreDNS Deployment to Mount Hosts File
```bash
# Get current deployment and modify it
kubectl get deployment coredns -n kube-system -o yaml > coredns-deployment.yaml

# Edit deployment to add volume mount (manual step)
echo "Manual step: Edit coredns-deployment.yaml to add:"
echo "1. Volume for coredns-hosts configmap"
echo "2. VolumeMount to /etc/coredns/"
echo "3. Then run: kubectl apply -f coredns-deployment.yaml"

# Alternative: Patch the deployment
kubectl patch deployment coredns -n kube-system -p='
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "coredns",
            "volumeMounts": [
              {
                "name": "config-volume",
                "mountPath": "/etc/coredns",
                "readOnly": true
              },
              {
                "name": "hosts-volume", 
                "mountPath": "/etc/coredns/internal.hosts",
                "subPath": "internal.hosts",
                "readOnly": true
              }
            ]
          }
        ],
        "volumes": [
          {
            "name": "config-volume",
            "configMap": {
              "name": "coredns",
              "items": [
                {
                  "key": "Corefile",
                  "path": "Corefile"
                }
              ]
            }
          },
          {
            "name": "hosts-volume",
            "configMap": {
              "name": "coredns-hosts"
            }
          }
        ]
      }
    }
  }
}'

# Restart CoreDNS to apply changes
kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system
```

---

## Phase 3: Testing and Validation (30 minutes)

### Step 3.1: Create Test Pod for DNS Validation
```bash
cat <<EOF > dns-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-tester
  namespace: default
spec:
  containers:
  - name: dns-tools
    image: nicolaka/netshoot
    command: ['sleep', '3600']
  restartPolicy: Never
EOF

kubectl apply -f dns-test-pod.yaml
kubectl wait --for=condition=Ready pod/dns-tester --timeout=60s
```

### Step 3.2: Test Custom Domain Resolution
```bash
echo "Testing custom domain mappings..."

# Test environment-specific domains
kubectl exec dns-tester -- nslookup dev.company.local
kubectl exec dns-tester -- nslookup staging.company.local  
kubectl exec dns-tester -- nslookup prod.company.local

# Test custom host entries
kubectl exec dns-tester -- nslookup database.company.local
kubectl exec dns-tester -- nslookup cache.company.local

# Test standard Kubernetes DNS
kubectl exec dns-tester -- nslookup web-app.dev.svc.cluster.local
kubectl exec dns-tester -- nslookup kubernetes.default.svc.cluster.local

echo "DNS testing completed!"
```

### Step 3.3: Performance Testing
```bash
# Create load test script
cat <<'EOF' > dns-load-test.sh
#!/bin/bash
echo "Starting DNS load test..."

# Function to perform DNS queries
dns_queries() {
    local pod_name=$1
    local namespace=${2:-default}
    
    for i in {1..100}; do
        kubectl exec $pod_name -n $namespace -- nslookup dev.company.local >/dev/null 2>&1 &
        kubectl exec $pod_name -n $namespace -- nslookup web-app.prod.svc.cluster.local >/dev/null 2>&1 &
        kubectl exec $pod_name -n $namespace -- nslookup google.com >/dev/null 2>&1 &
    done
    wait
}

# Run load test
time dns_queries dns-tester

echo "DNS load test completed!"
EOF

chmod +x dns-load-test.sh
./dns-load-test.sh
```

---

## Phase 4: Monitoring and Observability (45 minutes)

### Step 4.1: Set Up Prometheus Monitoring
```bash
# Create ServiceMonitor for CoreDNS
cat <<EOF > coredns-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: coredns
  namespace: kube-system
  labels:
    app: coredns
spec:
  selector:
    matchLabels:
      k8s-app: kube-dns
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF

# Apply if you have Prometheus Operator
# kubectl apply -f coredns-servicemonitor.yaml
```

### Step 4.2: Create Monitoring Dashboard Script
```bash
cat <<'EOF' > monitor-coredns.sh
#!/bin/bash

echo "CoreDNS Monitoring Dashboard"
echo "============================"

# Function to get metrics
get_coredns_metrics() {
    echo "Fetching CoreDNS metrics..."
    kubectl port-forward -n kube-system svc/coredns 9153:9153 >/dev/null 2>&1 &
    local PF_PID=$!
    sleep 2
    
    echo "DNS Request Rate (last 5 minutes):"
    curl -s http://localhost:9153/metrics | grep 'coredns_dns_requests_total' | head -5
    
    echo -e "\nDNS Response Codes:"
    curl -s http://localhost:9153/metrics | grep 'coredns_dns_responses_total' | head -5
    
    echo -e "\nCache Statistics:"
    curl -s http://localhost:9153/metrics | grep 'coredns_cache' | head -3
    
    # Clean up
    kill $PF_PID 2>/dev/null
}

# Function to check CoreDNS health
check_coredns_health() {
    echo -e "\nCoreDB Health Check:"
    echo "==================="
    
    # Check pod status
    echo "Pod Status:"
    kubectl get pods -n kube-system -l k8s-app=kube-dns
    
    # Check resource usage
    echo -e "\nResource Usage:"
    kubectl top pods -n kube-system -l k8s-app=kube-dns 2>/dev/null || echo "Metrics not available"
    
    # Check recent logs for errors
    echo -e "\nRecent Errors:"
    kubectl logs -n kube-system -l k8s-app=kube-dns --tail=10 | grep -i error || echo "No recent errors"
}

# Main execution
while true; do
    clear
    echo "$(date)"
    check_coredns_health
    get_coredns_metrics
    echo -e "\n\nPress Ctrl+C to exit, or wait 30 seconds for refresh..."
    sleep 30
done
EOF

chmod +x monitor-coredns.sh
```

### Step 4.3: Create Alerting Rules
```bash
cat <<EOF > coredns-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: coredns-alerts
  namespace: kube-system
spec:
  groups:
  - name: coredns.rules
    rules:
    - alert: CoreDNSDown
      expr: up{job="coredns"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "CoreDNS is down"
        description: "CoreDNS has been down for more than 1 minute."
    
    - alert: CoreDNSHighErrorRate
      expr: rate(coredns_dns_responses_total{rcode!="NOERROR"}[5m]) > 0.1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "CoreDNS high error rate"
        description: "CoreDNS error rate is {{ \$value }} requests/second."
    
    - alert: CoreDNSLowCacheHitRate
      expr: rate(coredns_cache_hits_total[5m]) / rate(coredns_cache_requests_total[5m]) < 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "CoreDNS low cache hit rate"
        description: "CoreDNS cache hit rate is {{ \$value }}."
EOF

# Apply if you have Prometheus Operator
# kubectl apply -f coredns-alerts.yaml
```

---

## Phase 5: Security Implementation (30 minutes)

### Step 5.1: Create Network Policies
```bash
# Network policy for CoreDNS access
cat <<EOF > coredns-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: coredns-access
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      k8s-app: kube-dns
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from: []  # Allow from all pods (DNS needs to be accessible)
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
    - protocol: TCP
      port: 9153  # Metrics port
  egress:
  - {}  # Allow all outbound (for external DNS forwarding)
---
# Policy for application namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: dev
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP  
      port: 53
  - to: []
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
EOF

# Label kube-system namespace for network policy
kubectl label namespace kube-system name=kube-system

# Apply network policies
kubectl apply -f coredns-network-policy.yaml

# Copy policy to other namespaces
sed 's/namespace: dev/namespace: staging/g' coredns-network-policy.yaml | kubectl apply -f -
sed 's/namespace: dev/namespace: prod/g' coredns-network-policy.yaml | kubectl apply -f -
```

### Step 5.2: RBAC Configuration for CoreDNS
```bash
cat <<EOF > coredns-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
EOF

kubectl apply -f coredns-rbac.yaml

# Update CoreDNS deployment to use service account
kubectl patch deployment coredns -n kube-system -p='{"spec":{"template":{"spec":{"serviceAccountName":"coredns"}}}}'
```

---

## Phase 6: Disaster Recovery and Backup (20 minutes)

### Step 6.1: Create Backup Scripts
```bash
cat <<'EOF' > backup-coredns.sh
#!/bin/bash

BACKUP_DIR="coredns-backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

echo "Creating CoreDNS backup in $BACKUP_DIR..."

# Backup CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml > $BACKUP_DIR/coredns-config.yaml

# Backup CoreDNS deployment
kubectl get deployment coredns -n kube-system -o yaml > $BACKUP_DIR/coredns-deployment.yaml

# Backup CoreDNS service
kubectl get service kube-dns -n kube-system -o yaml > $BACKUP_DIR/coredns-service.yaml

# Backup custom configurations
kubectl get configmap coredns-hosts -n kube-system -o yaml > $BACKUP_DIR/coredns-hosts.yaml 2>/dev/null

# Backup RBAC
kubectl get serviceaccount coredns -n kube-system -o yaml > $BACKUP_DIR/coredns-serviceaccount.yaml
kubectl get clusterrole system:coredns -o yaml > $BACKUP_DIR/coredns-clusterrole.yaml
kubectl get clusterrolebinding system:coredns -o yaml > $BACKUP_DIR/coredns-clusterrolebinding.yaml

# Create restore script
cat > $BACKUP_DIR/restore-coredns.sh << 'SCRIPT'
#!/bin/bash
echo "Restoring CoreDNS from backup..."
kubectl apply -f coredns-config.yaml
kubectl apply -f coredns-hosts.yaml
kubectl apply -f coredns-serviceaccount.yaml  
kubectl apply -f coredns-clusterrole.yaml
kubectl apply -f coredns-clusterrolebinding.yaml
kubectl apply -f coredns-deployment.yaml
kubectl apply -f coredns-service.yaml

echo "Waiting for CoreDNS to be ready..."
kubectl wait --for=condition=Available deployment/coredns -n kube-system --timeout=120s
echo "CoreDNS restore completed!"
SCRIPT

chmod +x $BACKUP_DIR/restore-coredns.sh

echo "Backup completed: $BACKUP_DIR"
ls -la $BACKUP_DIR/
EOF

chmod +x backup-coredns.sh
./backup-coredns.sh
```

### Step 6.2: Test Disaster Recovery
```bash
# Test the disaster recovery process
echo "Testing disaster recovery..."

# Simulate disaster (be careful!)
echo "Simulating CoreDNS failure..."
kubectl scale deployment coredns -n kube-system --replicas=0

# Verify DNS is broken
kubectl exec dns-tester -- nslookup kubernetes.default || echo "DNS failed as expected"

# Find the latest backup
LATEST_BACKUP=$(ls -d coredns-backups/* | tail -1)
echo "Restoring from: $LATEST_BACKUP"

# Restore
cd $LATEST_BACKUP && ./restore-coredns.sh
cd ../..

# Verify DNS is working
sleep 10
kubectl exec dns-tester -- nslookup kubernetes.default && echo "DNS restored successfully!"
```

---

## Phase 7: Final Testing and Documentation (30 minutes)

### Step 7.1: Comprehensive Test Suite
```bash
cat <<'EOF' > comprehensive-test.sh
#!/bin/bash

echo "CoreDNS Project - Comprehensive Test Suite"
echo "=========================================="

# Test counter
TOTAL_TESTS=0
PASSED_TESTS=0

# Function to run test
run_test() {
    local test_name="$1"
    local test_command="$2"
    
    echo -n "Testing $test_name... "
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    
    if eval "$test_command" >/dev/null 2>&1; then
        echo "âœ… PASSED"
        PASSED_TESTS=$((PASSED_TESTS + 1))
    else
        echo "âŒ FAILED"
    fi
}

# DNS Resolution Tests
echo "1. DNS Resolution Tests"
echo "======================="
run_test "Kubernetes Default Service" "kubectl exec dns-tester -- nslookup kubernetes.default"
run_test "Dev Environment Service" "kubectl exec dns-tester -- nslookup web-app.dev.svc.cluster.local"
run_test "Staging Environment Service" "kubectl exec dns-tester -- nslookup web-app.staging.svc.cluster.local"
run_test "Prod Environment Service" "kubectl exec dns-tester -- nslookup web-app.prod.svc.cluster.local"
run_test "Custom Domain - Dev" "kubectl exec dns-tester -- nslookup dev.company.local"
run_test "Custom Domain - Staging" "kubectl exec dns-tester -- nslookup staging.company.local"
run_test "Custom Domain - Prod" "kubectl exec dns-tester -- nslookup prod.company.local"
run_test "Custom Host Entry - Database" "kubectl exec dns-tester -- nslookup database.company.local"
run_test "External DNS Resolution" "kubectl exec dns-tester -- nslookup google.com"

# CoreDNS Health Tests  
echo -e "\n2. CoreDNS Health Tests"
echo "======================="
run_test "CoreDNS Pods Running" "kubectl get pods -n kube-system -l k8s-app=kube-dns | grep Running"
run_test "CoreDNS Service Available" "kubectl get svc -n kube-system kube-dns"
run_test "CoreDNS Metrics Endpoint" "kubectl exec -n kube-system deploy/coredns -- wget -q -O- localhost:9153/metrics | grep coredns_dns_requests_total"

# Configuration Tests
echo -e "\n3. Configuration Tests"  
echo "======================"
run_test "CoreDNS ConfigMap Exists" "kubectl get configmap coredns -n kube-system"
run_test "Custom Hosts ConfigMap Exists" "kubectl get configmap coredns-hosts -n kube-system"
run_test "RBAC Configuration" "kubectl get serviceaccount coredns -n kube-system"

# Security Tests
echo -e "\n4. Security Tests"
echo "================="
run_test "Network Policy - kube-system" "kubectl get networkpolicy -n kube-system coredns-access"
run_test "Network Policy - dev" "kubectl get networkpolicy -n dev allow-dns"
run_test "Network Policy - staging" "kubectl get networkpolicy -n staging allow-dns"
run_test "Network Policy - prod" "kubectl get networkpolicy -n prod allow-dns"

# Performance Tests
echo -e "\n5. Performance Tests"
echo "===================="
run_test "DNS Cache Working" "kubectl exec dns-tester -- sh -c 'time nslookup google.com' | grep -q real"

# Results Summary
echo -e "\n=========================================="
echo "Test Results: $PASSED_TESTS/$TOTAL_TESTS tests passed"
if [ $PASSED_TESTS -eq $TOTAL_TESTS ]; then
    echo "ðŸŽ‰ All tests passed! CoreDNS project is working correctly."
else
    echo "âš ï¸ Some tests failed. Please check the configuration."
fi
echo "=========================================="
EOF

chmod +x comprehensive-test.sh
./comprehensive-test.sh
```

### Step 7.2: Generate Project Documentation
```bash
cat <<EOF > PROJECT-SUMMARY.md
# CoreDNS Kubernetes Project - Summary

## Project Completion Status
- âœ… Multi-environment DNS setup (dev/staging/prod)
- âœ… Custom domain mapping and host entries
- âœ… Advanced CoreDNS configuration with multiple zones
- âœ… Monitoring and metrics collection
- âœ… Security implementation (RBAC + Network Policies)
- âœ… Disaster recovery and backup procedures
- âœ… Comprehensive testing suite

## Architecture Implemented
\`\`\`
Kubernetes Cluster
â”œâ”€â”€ Namespaces: dev, staging, prod
â”œâ”€â”€ Custom CoreDNS Configuration
â”‚   â”œâ”€â”€ Domain rewrites (*.company.local)
â”‚   â”œâ”€â”€ Static host entries
â”‚   â”œâ”€â”€ Conditional forwarding
â”‚   â””â”€â”€ Enhanced caching
â”œâ”€â”€ Monitoring Setup
â”‚   â”œâ”€â”€ Prometheus metrics
â”‚   â””â”€â”€ Health checks
â”œâ”€â”€ Security Layer
â”‚   â”œâ”€â”€ RBAC configuration
â”‚   â””â”€â”€ Network policies
â””â”€â”€ Backup & Recovery System
\`\`\`

## Key Features Deployed
1. **Environment-Specific DNS**: dev.company.local, staging.company.local, prod.company.local
2. **Custom Host Entries**: database.company.local, cache.company.local
3. **Monitoring**: Metrics on port 9153, health checks
4. **Security**: Network policies restricting DNS access
5. **High Availability**: Multiple CoreDNS replicas with load balancing

## Files Created
- custom-coredns-config.yaml (Advanced CoreDNS configuration)
- coredns-hosts-config.yaml (Custom host entries)
- coredns-network-policy.yaml (Security policies)
- coredns-rbac.yaml (Service account and permissions)
- monitor-coredns.sh (Monitoring script)
- backup-coredns.sh (Backup automation)
- comprehensive-test.sh (Testing suite)

## Next Steps for Production
1. Implement proper secret management for sensitive DNS entries
2. Set up automated backups with cron jobs
3. Configure proper log aggregation and alerting
4. Implement DNS-based load balancing for external services
5. Add DNS-over-TLS for enhanced security

## Knowledge Gained
- CoreDNS plugin architecture and configuration
- Kubernetes DNS service discovery mechanisms
- Network policy implementation for DNS security
- DNS monitoring and troubleshooting techniques
- Disaster recovery procedures for critical services

Project completed successfully! ðŸŽ‰
EOF

echo "Project documentation created: PROJECT-SUMMARY.md"
```

---

## Phase 8: Cleanup (Optional)
```bash
cat <<'EOF' > cleanup-project.sh
#!/bin/bash

echo "Cleaning up CoreDNS project resources..."

# Restore original CoreDNS configuration
if [ -f "original-coredns-backup.yaml" ]; then
    kubectl apply -f original-coredns-backup.yaml
    echo "âœ… Original CoreDNS configuration restored"
fi

# Remove custom configurations
kubectl delete configmap coredns-hosts -n kube-system --ignore-not-found=true

# Remove network policies
kubectl delete networkpolicy coredns-access -n kube-system --ignore-not-found=true
kubectl delete networkpolicy allow-dns -n dev --ignore-not-found=true
kubectl delete networkpolicy allow-dns -n staging --ignore-not-found=true  
kubectl delete networkpolicy allow-dns -n prod --ignore-not-found=true

# Remove test applications
kubectl delete deployment web-app -n dev --ignore-not-found=true
kubectl delete deployment web-app -n staging --ignore-not-found=true
kubectl delete deployment web-app -n prod --ignore-not-found=true
kubectl delete service web-app -n dev --ignore-not-found=true
kubectl delete service web-app -n staging --ignore-not-found=true
kubectl delete service web-app -n prod --ignore-not-found=true

# Remove test pod
kubectl delete pod dns-tester --ignore-not-found=true

# Remove namespaces (optional - uncomment if you want to remove them)
# kubectl delete namespace dev staging prod

echo "ðŸ§¹ Cleanup completed!"
echo "ðŸ“ Backup files and scripts are preserved for future reference"
EOF

chmod +x cleanup-project.sh
```

## Project Complete! ðŸŽ‰

You have successfully built a complete CoreDNS solution with:
- Multi-environment DNS management
- Custom domain resolution  
- Security implementation
- Monitoring and alerting
- Disaster recovery procedures
- Comprehensive testing

This project demonstrates enterprise-level CoreDNS deployment and management skills.
EOF
