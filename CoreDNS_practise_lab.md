# CoreDNS Kubernetes Practice Labs - Step by Step

## Prerequisites
- A running Kubernetes cluster (minikube, kind, or cloud cluster)
- kubectl configured and working
- Basic understanding of Kubernetes pods and services

---

## Lab 1: Basic CoreDNS Setup and Health Check

### Step 1.1: Check if CoreDNS is Running
```bash
# Check CoreDNS pods status
kubectl get pods -n kube-system | grep coredns

# Expected output: You should see 2 CoreDNS pods in Running state
# coredns-76f75df574-abc12   1/1     Running   0          5h
# coredns-76f75df574-def34   1/1     Running   0          5h
```

### Step 1.2: View CoreDNS Logs
```bash
# Check CoreDNS logs for any errors
kubectl logs -n kube-system -l k8s-app=kube-dns

# Look for error messages or repeated DNS failures
```

### Step 1.3: Examine CoreDNS Configuration
```bash
# View the current CoreDNS configuration (Corefile)
kubectl get configmap coredns -n kube-system -o yaml

# You'll see the Corefile with plugins like:
# - errors (error logging)
# - health (health checks)
# - kubernetes (service discovery)
# - forward (external DNS)
# - cache (DNS caching)
```

**What You Should Learn:**
- CoreDNS runs as pods in kube-system namespace
- Configuration is stored in a ConfigMap called "coredns"
- The Corefile defines which plugins are active

---

## Lab 2: DNS Resolution Testing

### Step 2.1: Create a Test Pod
```bash
# Create a busybox pod for DNS testing
kubectl run dns-test --image=busybox:1.28 --restart=Never -- sleep 3600

# Wait for pod to be ready
kubectl get pod dns-test
```

### Step 2.2: Test Kubernetes Internal DNS
```bash
# Test resolving the kubernetes service
kubectl exec dns-test -- nslookup kubernetes.default

# Expected output:
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# Name:      kubernetes.default
# Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

# Test resolving CoreDNS service itself
kubectl exec dns-test -- nslookup kube-dns.kube-system.svc.cluster.local
```

### Step 2.3: Test External DNS Resolution
```bash
# Test external domain resolution (should work via forwarding)
kubectl exec dns-test -- nslookup google.com

# If this fails, there might be issues with external DNS forwarding
```

**What You Should Learn:**
- Internal services are resolved using cluster.local domain
- External queries are forwarded to upstream DNS servers
- DNS server IP is typically 10.96.0.10 (the kube-dns service IP)

---

## Lab 3: Create and Test Service DNS Resolution

### Step 3.1: Create a Test Service
```bash
# Create a simple nginx deployment
kubectl create deployment test-nginx --image=nginx

# Expose it as a service
kubectl expose deployment test-nginx --port=80 --target-port=80 --name=my-test-service

# Check the service
kubectl get svc my-test-service
```

### Step 3.2: Test Service DNS Resolution
```bash
# From our test pod, resolve the new service
kubectl exec dns-test -- nslookup my-test-service.default.svc.cluster.local

# Test short name resolution (should also work)
kubectl exec dns-test -- nslookup my-test-service

# Test from different namespace
kubectl create namespace test-ns
kubectl run dns-test2 -n test-ns --image=busybox:1.28 --restart=Never -- sleep 3600

# From test-ns namespace, test cross-namespace resolution
kubectl exec -n test-ns dns-test2 -- nslookup my-test-service.default.svc.cluster.local
```

**What You Should Learn:**
- Services get DNS names: `<service>.<namespace>.svc.cluster.local`
- Short names work within the same namespace
- Cross-namespace requires full DNS name

---

## Lab 4: CoreDNS Configuration Customization

### Step 4.1: View Current CoreDNS Config
```bash
# Save current config for backup
kubectl get configmap coredns -n kube-system -o yaml > coredns-backup.yaml

# View the Corefile section
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
```

### Step 4.2: Add Custom DNS Entry
```bash
# Edit CoreDNS configuration
kubectl edit configmap coredns -n kube-system

# Add a hosts plugin section before the kubernetes plugin:
# hosts {
#   192.168.1.100 myapp.local
#   fallthrough
# }

# The Corefile should look like:
# .:53 {
#     errors
#     health {
#         lameduck 5s
#     }
#     ready
#     hosts {
#         192.168.1.100 myapp.local
#         fallthrough
#     }
#     kubernetes cluster.local in-addr.arpa ip6.arpa {
#         pods insecure
#         fallthrough in-addr.arpa ip6.arpa
#         ttl 30
#     }
#     prometheus :9153
#     forward . /etc/resolv.conf
#     cache 30
#     loop
#     reload
#     loadbalance
# }
```

### Step 4.3: Restart CoreDNS to Apply Changes
```bash
# Restart CoreDNS deployment to pick up new config
kubectl rollout restart deployment/coredns -n kube-system

# Wait for rollout to complete
kubectl rollout status deployment/coredns -n kube-system

# Test the custom DNS entry
kubectl exec dns-test -- nslookup myapp.local
# Should return 192.168.1.100
```

**What You Should Learn:**
- CoreDNS can be customized by editing the coredns ConfigMap
- Changes require restarting CoreDNS pods
- The hosts plugin allows custom DNS entries

---

## Lab 5: CoreDNS Scaling and Performance

### Step 5.1: Monitor CoreDNS Resources
```bash
# Check current resource usage
kubectl top pods -n kube-system -l k8s-app=kube-dns

# View current replica count
kubectl get deployment coredns -n kube-system
```

### Step 5.2: Scale CoreDNS
```bash
# Scale CoreDNS to 3 replicas for high load
kubectl scale deployment coredns -n kube-system --replicas=3

# Verify scaling
kubectl get pods -n kube-system -l k8s-app=kube-dns

# You should see 3 CoreDNS pods running
```

### Step 5.3: Load Testing (Optional)
```bash
# Create multiple test pods to generate DNS load
for i in {1..5}; do
  kubectl run load-test-$i --image=busybox:1.28 --restart=Never -- sh -c "while true; do nslookup kubernetes.default; sleep 1; done" &
done

# Monitor CoreDNS performance
kubectl top pods -n kube-system -l k8s-app=kube-dns

# Clean up load test
for i in {1..5}; do kubectl delete pod load-test-$i; done
```

**What You Should Learn:**
- CoreDNS can be scaled horizontally by increasing replicas
- More replicas handle higher DNS query load
- Resource monitoring helps determine scaling needs

---

## Lab 6: Troubleshooting DNS Issues

### Step 6.1: Simulate DNS Failure
```bash
# Scale CoreDNS to 0 (simulate failure)
kubectl scale deployment coredns -n kube-system --replicas=0

# Try DNS resolution - should fail
kubectl exec dns-test -- nslookup kubernetes.default
# Expected: server can't find kubernetes.default
```

### Step 6.2: Diagnose DNS Issues
```bash
# Check if CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check kube-dns service (should exist even if pods are down)
kubectl get svc -n kube-system kube-dns

# Check if DNS queries reach the service
kubectl describe svc -n kube-system kube-dns
```

### Step 6.3: Fix DNS Issues
```bash
# Restore CoreDNS replicas
kubectl scale deployment coredns -n kube-system --replicas=2

# Wait for pods to be ready
kubectl wait --for=condition=Ready pod -l k8s-app=kube-dns -n kube-system --timeout=60s

# Test DNS resolution again
kubectl exec dns-test -- nslookup kubernetes.default
# Should work now
```

**What You Should Learn:**
- DNS failures prevent service discovery
- CoreDNS pods must be running for DNS to work
- The kube-dns service acts as a load balancer for CoreDNS pods

---

## Lab 7: Advanced CoreDNS Features

### Step 7.1: Enable DNS Logging
```bash
# Edit CoreDNS config to add logging
kubectl edit configmap coredns -n kube-system

# Add 'log' plugin after 'ready':
# log

# Restart CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# Generate some DNS queries
kubectl exec dns-test -- nslookup my-test-service

# Check logs to see DNS queries
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=10
```

### Step 7.2: Configure External DNS Forwarding
```bash
# Edit CoreDNS config to use custom external DNS
kubectl edit configmap coredns -n kube-system

# Change the forward plugin from:
# forward . /etc/resolv.conf
# to:
# forward . 8.8.8.8 1.1.1.1

# Restart CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# Test external resolution
kubectl exec dns-test -- nslookup google.com
```

### Step 7.3: Add a Custom Domain Zone
```bash
# Edit CoreDNS config to add a custom domain
kubectl edit configmap coredns -n kube-system

# Add a new server block for custom domain:
# company.local:53 {
#     hosts {
#         10.1.1.10 app1.company.local
#         10.1.1.11 app2.company.local
#         fallthrough
#     }
#     forward . 8.8.8.8
# }

# Restart and test
kubectl rollout restart deployment/coredns -n kube-system
kubectl exec dns-test -- nslookup app1.company.local
```

**What You Should Learn:**
- CoreDNS supports multiple server blocks for different domains
- Logging helps debug DNS resolution issues
- External DNS forwarding can be customized per domain

---

## Lab 8: Monitoring CoreDNS

### Step 8.1: Access CoreDNS Metrics
```bash
# Check if prometheus plugin is enabled in Corefile
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}' | grep prometheus

# Port forward to access metrics
kubectl port-forward -n kube-system svc/coredns 9153:9153 &

# View metrics in another terminal
curl http://localhost:9153/metrics | grep coredns_dns

# Stop port forward
pkill -f "port-forward"
```

### Step 8.2: Monitor DNS Query Patterns
```bash
# Generate various DNS queries
kubectl exec dns-test -- nslookup kubernetes.default
kubectl exec dns-test -- nslookup my-test-service
kubectl exec dns-test -- nslookup google.com
kubectl exec dns-test -- nslookup nonexistent.service

# Check metrics again
kubectl port-forward -n kube-system svc/coredns 9153:9153 &
curl http://localhost:9153/metrics | grep -E "(coredns_dns_requests_total|coredns_dns_responses_total)"
```

**What You Should Learn:**
- CoreDNS exposes Prometheus metrics on port 9153
- Metrics show request counts, response codes, and query types
- Monitoring helps identify DNS performance issues

---

## Lab 9: Network Policy and DNS Security

### Step 9.1: Create a Restrictive Network Policy
```bash
# Create a namespace with strict network policy
kubectl create namespace secure-ns

# Create network policy that blocks all traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Create test pod in secure namespace
kubectl run secure-test -n secure-ns --image=busybox:1.28 --restart=Never -- sleep 3600

# Test DNS resolution - should fail
kubectl exec -n secure-ns secure-test -- nslookup kubernetes.default
```

### Step 9.2: Allow DNS Traffic
```bash
# Create policy to allow DNS traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: secure-ns
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
EOF

# Label kube-system namespace
kubectl label namespace kube-system name=kube-system

# Test DNS again - should work now
kubectl exec -n secure-ns secure-test -- nslookup kubernetes.default
```

**What You Should Learn:**
- Network policies can block DNS traffic
- DNS uses both UDP and TCP on port 53
- CoreDNS runs in kube-system namespace

---

## Lab 10: Backup and Disaster Recovery

### Step 10.1: Backup CoreDNS Configuration
```bash
# Create backup of CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml > coredns-config-backup.yaml

# Backup CoreDNS deployment
kubectl get deployment coredns -n kube-system -o yaml > coredns-deployment-backup.yaml

# Backup CoreDNS service
kubectl get service kube-dns -n kube-system -o yaml > coredns-service-backup.yaml

echo "Backup files created:"
ls -la coredns-*-backup.yaml
```

### Step 10.2: Simulate Disaster and Restore
```bash
# Simulate disaster - delete CoreDNS deployment
kubectl delete deployment coredns -n kube-system

# Verify DNS is broken
kubectl exec dns-test -- nslookup kubernetes.default || echo "DNS is broken as expected"

# Restore from backup
kubectl apply -f coredns-deployment-backup.yaml

# Wait for restore
kubectl wait --for=condition=Available deployment/coredns -n kube-system --timeout=60s

# Verify DNS is working
kubectl exec dns-test -- nslookup kubernetes.default
echo "DNS restored successfully!"
```

**What You Should Learn:**
- Regular backups of CoreDNS configuration are important
- CoreDNS deployment, service, and config should all be backed up
- Recovery involves recreating the deployment with saved config

---

## Lab 11: Cleanup
```bash
# Remove test resources
kubectl delete pod dns-test
kubectl delete pod dns-test2 -n test-ns
kubectl delete pod secure-test -n secure-ns
kubectl delete deployment test-nginx
kubectl delete service my-test-service
kubectl delete namespace test-ns secure-ns

# Restore original CoreDNS config if needed
kubectl apply -f coredns-backup.yaml

# Clean up backup files
rm -f coredns-*-backup.yaml

echo "Cleanup completed!"
```

---

## Summary of What You Learned

1. **CoreDNS Basics**: How CoreDNS runs in Kubernetes and provides DNS resolution
2. **DNS Testing**: How to test internal and external DNS resolution
3. **Service Discovery**: How Kubernetes services get DNS names
4. **Configuration**: How to customize CoreDNS via the Corefile
5. **Scaling**: How to scale CoreDNS for performance
6. **Troubleshooting**: How to diagnose and fix DNS issues
7. **Advanced Features**: Logging, custom domains, external forwarding
8. **Monitoring**: How to access CoreDNS metrics
9. **Security**: How network policies affect DNS traffic
10. **Disaster Recovery**: How to backup and restore CoreDNS

Each lab builds on the previous one, giving you hands-on experience with CoreDNS in a real Kubernetes environment.
