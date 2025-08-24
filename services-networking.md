# Services & Networking (Week 3)

**Complete guide to Kubernetes services, networking, and traffic management**

---

## Table of Contents

1. [Kubernetes Networking Model](#kubernetes-networking-model)
2. [Services Overview](#services-overview)
3. [Service Types](#service-types)
4. [Ingress](#ingress)
5. [Gateway API](#gateway-api)
6. [Network Policies](#network-policies)
7. [DNS and Service Discovery](#dns-and-service-discovery)
8. [CNI (Container Network Interface)](#cni-container-network-interface)
9. [Practice Labs](#practice-labs)
10. [Hands-on Projects](#hands-on-projects)
11. [Troubleshooting Guide](#troubleshooting-guide)

---

## Kubernetes Networking Model

### Networking Fundamentals
The Kubernetes networking model is built on four core principles:

1. **Pod-to-Pod Communication**: All pods can communicate with all other pods without NAT
2. **Node-to-Pod Communication**: Nodes can communicate with all pods without NAT
3. **Pod Identity**: The IP that a pod sees itself as is the same IP others see it as
4. **Container Communication**: Containers within a pod share the network namespace

### Network Architecture Overview
```
┌─────────────────────────────────────────┐
│                 Internet                │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│          Load Balancer/Ingress          │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│              Services                   │
│    (ClusterIP, NodePort, LoadBalancer)  │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│               Pods                      │
│         (Pod Network/CNI)               │
└─────────────────────────────────────────┘
```

### IP Address Ranges
Kubernetes requires three distinct IP ranges:

| Component | IP Range | Purpose |
|-----------|----------|---------|
| **Pod CIDR** | 10.244.0.0/16 | Pod-to-pod communication |
| **Service CIDR** | 10.96.0.0/12 | Service virtual IPs |
| **Node CIDR** | 192.168.1.0/24 | Node-to-node communication |

### Network Layers
```
┌─────────────────────────────────────────┐
│        Layer 7 (Application)            │
│     HTTP, gRPC, Custom Protocols        │
├─────────────────────────────────────────┤
│         Layer 4 (Transport)             │
│           TCP, UDP, SCTP                │
├─────────────────────────────────────────┤
│         Layer 3 (Network)               │
│        IP, ICMP, Routing                │
├─────────────────────────────────────────┤
│       Layer 2 (Data Link)               │
│      Ethernet, VLAN, ARP                │
└─────────────────────────────────────────┘
```

---

## Services Overview

### What is a Service?
A **Service** in Kubernetes is an abstraction that defines a logical set of pods and provides a stable network endpoint to access them. Services enable loose coupling between dependent components.

### Why Services are Needed
- **Pod IP Instability**: Pod IPs change when pods are recreated
- **Load Balancing**: Distribute traffic across multiple pod replicas  
- **Service Discovery**: Provide stable DNS names and IPs
- **Decoupling**: Applications don't need to know about pod locations

### Service Components
```yaml
# Complete Service specification
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
  labels:
    app: web-service
    environment: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: ClusterIP              # Service type
  selector:                    # Pod selector
    app: web-app
    version: v1.0
  ports:                       # Port mappings
  - name: http
    port: 80                   # Service port
    targetPort: 8080           # Container port
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  sessionAffinity: None        # Session stickiness
  # sessionAffinityConfig:     # Session affinity configuration
  #   clientIP:
  #     timeoutSeconds: 300
status:
  loadBalancer: {}             # Status (system-populated)
```

### Service Discovery Methods
1. **Environment Variables**: Automatic injection of service info
2. **DNS**: Cluster DNS resolution (recommended)
3. **API**: Direct Kubernetes API queries

#### DNS-Based Service Discovery
```bash
# Service DNS format
<service-name>.<namespace>.svc.cluster.local

# Examples
web-service.default.svc.cluster.local
database.backend.svc.cluster.local
redis.cache.svc.cluster.local

# Short forms (from same namespace)
web-service
web-service.default
```

---

## Service Types

### 1. ClusterIP (Default)

#### Purpose
Exposes the service on an internal cluster IP, making it reachable only from within the cluster.

#### Use Cases
- **Internal microservices**: Backend APIs, databases
- **Inter-service communication**: Service-to-service calls
- **Internal load balancing**: Distribute traffic among replicas

#### Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP              # Default type
  selector:
    app: backend
  ports:
  - port: 80                   # Service port
    targetPort: 8080           # Pod port
    protocol: TCP
  # clusterIP: 10.96.240.100  # Optional: specify IP
  # clusterIP: None           # Headless service
```

#### Headless Services
```yaml
# Headless service (no load balancing)
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  type: ClusterIP
  clusterIP: None              # Makes it headless
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

**Headless Service Characteristics:**
- **No Virtual IP**: DNS returns pod IPs directly
- **StatefulSet Integration**: Each pod gets unique DNS record
- **Service Discovery**: Applications can discover all pod IPs
- **Direct Connection**: Clients connect directly to pods

### 2. NodePort

#### Purpose
Exposes the service on each node's IP at a static port, making it accessible from outside the cluster.

#### How It Works
```
External Client → Node IP:NodePort → Service → Pods
```

#### Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80                   # Service port
    targetPort: 8080           # Pod port
    nodePort: 30080            # Node port (30000-32767)
    protocol: TCP
```

#### NodePort Characteristics
- **Port Range**: 30000-32767 (configurable)
- **All Nodes**: Service accessible on all node IPs
- **External Access**: Can be accessed from outside cluster
- **No Load Balancer**: Need external load balancer for production

#### NodePort Example
```bash
# Access service from outside cluster
curl http://<node-ip>:30080
curl http://192.168.1.10:30080
curl http://192.168.1.11:30080

# Multiple ways to reach same service
for node in 192.168.1.{10,11,12}; do
  curl http://$node:30080
done
```

### 3. LoadBalancer

#### Purpose
Exposes the service externally using a cloud provider's load balancer.

#### How It Works
```
Internet → Cloud Load Balancer → NodePort → Service → Pods
```

#### Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  annotations:
    # AWS Load Balancer annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    # Azure Load Balancer annotations  
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
    # GCP Load Balancer annotations
    service.beta.kubernetes.io/gce-neg: '{"ingress": true}'
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  # loadBalancerIP: 203.0.113.100  # Request specific IP
  # loadBalancerSourceRanges:      # Restrict source IPs
  # - 203.0.113.0/24
  # - 198.51.100.0/24
```

#### LoadBalancer Features
- **Cloud Integration**: Automatically provisions cloud load balancer
- **External IP**: Gets public IP address
- **Health Checks**: Cloud provider handles health checking
- **SSL Termination**: Can terminate SSL at load balancer level

#### Multi-Cloud LoadBalancer Examples
```yaml
# AWS Network Load Balancer
apiVersion: v1
kind: Service
metadata:
  name: aws-nlb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 443
    targetPort: 8443
    protocol: TCP

---
# Google Cloud Load Balancer
apiVersion: v1
kind: Service
metadata:
  name: gcp-lb-service
  annotations:
    cloud.google.com/load-balancer-type: "External"
    service.beta.kubernetes.io/gce-neg: '{"ingress": true}'
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080

---
# Azure Load Balancer
apiVersion: v1
kind: Service
metadata:
  name: azure-lb-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/health"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

### 4. ExternalName

#### Purpose
Maps the service to the contents of the externalName field by returning a CNAME record.

#### Use Cases
- **External Services**: Integrate external databases or APIs
- **Migration**: Gradually move services to/from cluster
- **Service Aliasing**: Create aliases for services

#### Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.example.com
  ports:
  - port: 5432
    protocol: TCP
```

#### ExternalName Examples
```yaml
# External API service
apiVersion: v1
kind: Service
metadata:
  name: payment-api
spec:
  type: ExternalName
  externalName: api.stripe.com
  ports:
  - port: 443

---
# External database
apiVersion: v1
kind: Service
metadata:
  name: user-database  
spec:
  type: ExternalName
  externalName: users.abc123.us-west-2.rds.amazonaws.com
  ports:
  - port: 5432

---
# Service migration example
apiVersion: v1
kind: Service
metadata:
  name: legacy-service
spec:
  type: ExternalName
  externalName: legacy.internal.company.com
  ports:
  - port: 80
```

### Service Endpoints

#### Understanding Endpoints
```bash
# View service endpoints
kubectl get endpoints
kubectl describe endpoints web-service

# Service endpoints are automatically managed
kubectl get pods -l app=web-app -o wide
kubectl get endpoints web-service -o yaml
```

#### Manual Endpoint Management
```yaml
# Service without selector
apiVersion: v1
kind: Service
metadata:
  name: manual-service
spec:
  ports:
  - port: 80
    targetPort: 8080
# No selector = no automatic endpoints

---
# Manual endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: manual-service    # Must match service name
subsets:
- addresses:
  - ip: 192.168.1.100
    hostname: server1
  - ip: 192.168.1.101
    hostname: server2
  ports:
  - port: 8080
    protocol: TCP
```

---

## Ingress

### What is Ingress?
**Ingress** manages external access to services in a cluster, typically HTTP/HTTPS. It provides load balancing, SSL termination, and name-based virtual hosting.

### Ingress Components
- **Ingress Resource**: Defines routing rules
- **Ingress Controller**: Implements the rules (nginx, traefik, etc.)
- **Ingress Class**: Specifies which controller handles the resource

### Basic Ingress Example
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx    # Specify ingress controller
  tls:                       # TLS configuration
  - hosts:
    - app.example.com
    secretName: app-tls-secret
  rules:
  - host: app.example.com    # Domain-based routing
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix  
        backend:
          service:
            name: backend-service
            port:
              number: 8080
```

### Advanced Ingress Configuration

#### Multi-Host Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app1.example.com
    - app2.example.com
    secretName: wildcard-tls-secret
  rules:
  # First application
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  
  # Second application  
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: app2-admin-service
            port:
              number: 8080
```

#### Path-Based Routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      # Frontend routes
      - path: /(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      
      # API routes  
      - path: /api/v1/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
              
      # API v2 routes
      - path: /api/v2/(.*)
        pathType: ImplementationSpecific  
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080

      # Static assets
      - path: /static/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: static-service
            port:
              number: 80
```

### Ingress Path Types
- **Exact**: Matches exact path only
- **Prefix**: Matches based on URL path prefix
- **ImplementationSpecific**: Depends on ingress controller

### Ingress Controllers

#### Popular Ingress Controllers
| Controller | Features | Use Cases |
|------------|----------|-----------|
| **NGINX** | High performance, mature | Production web apps |
| **Traefik** | Auto-discovery, modern UI | Cloud-native apps |
| **HAProxy** | Advanced load balancing | High-performance needs |
| **Istio** | Service mesh integration | Microservices |
| **Contour** | Envoy-based, API-driven | Cloud-native platforms |

#### Installing NGINX Ingress Controller
```bash
# Install via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Install via kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Check installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### TLS/SSL Configuration
```yaml
# TLS secret
apiVersion: v1
kind: Secret
metadata:
  name: app-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: app-tls-secret
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-app-service
            port:
              number: 80
```

---

## Gateway API

### What is Gateway API?
**Gateway API** is the next generation of Kubernetes traffic management APIs, designed to replace and extend Ingress capabilities with more flexibility and functionality.

### Gateway API vs Ingress
| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| **Protocol Support** | HTTP/HTTPS only | HTTP, HTTPS, TCP, UDP, TLS |
| **Role Separation** | Limited | Infrastructure, Gateway, Route |
| **Extensibility** | Annotations | Native API extensions |
| **Cross-Namespace** | No | Yes |
| **Traffic Splitting** | Controller-specific | Native support |
| **Header Manipulation** | Annotations | Built-in |

### Gateway API Architecture
```
┌─────────────────────────────────────────┐
│         GatewayClass                    │
│    (Infrastructure Provider)           │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│            Gateway                      │
│       (Network Entry Point)            │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│           Routes                        │
│    (HTTPRoute, TCPRoute, etc.)          │
└─────────────────────────────────────────┘
```

### Gateway API Resources

#### 1. GatewayClass
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: nginx-gateway-class
spec:
  controllerName: nginx.org/gateway-controller
  description: "NGINX Gateway Controller"
  parametersRef:
    group: gateway.nginx.org
    kind: NginxGateway
    name: nginx-config
```

#### 2. Gateway
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: web-gateway
  namespace: gateway-system
spec:
  gatewayClassName: nginx-gateway-class
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: https  
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: web-tls-secret
    allowedRoutes:
      namespaces:
        from: All
```

#### 3. HTTPRoute
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: default
spec:
  parentRefs:
  - name: web-gateway
    namespace: gateway-system
  hostnames:
  - app.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/v1
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: x-api-version
          value: v1
    backendRefs:
    - name: api-v1-service
      port: 8080
      weight: 100
  
  - matches:
    - path:
        type: PathPrefix  
        value: /api/v2
    filters:
    - type: RequestRedirect
      requestRedirect:
        hostname: v2.api.example.com
        statusCode: 302
    
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
      weight: 90
    - name: frontend-canary-service
      port: 80
      weight: 10    # 10% canary traffic
```

### Advanced Gateway API Features

#### Traffic Splitting (Canary Deployments)
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - canary.example.com
  rules:
  - matches:
    - headers:
      - name: canary-user
        value: "true"
    backendRefs:
    - name: app-canary-service
      port: 80
      weight: 100
  
  - backendRefs:
    - name: app-stable-service  
      port: 80
      weight: 95
    - name: app-canary-service
      port: 80
      weight: 5     # 5% canary traffic for regular users
```

#### Cross-Namespace Routing
```yaml
# Gateway in gateway-system namespace
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-system
spec:
  gatewayClassName: nginx-gateway-class
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"

---
# HTTPRoute in app namespace
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: app-namespace
spec:
  parentRefs:
  - name: shared-gateway
    namespace: gateway-system
  hostnames:
  - app.example.com
  rules:
  - backendRefs:
    - name: app-service
      port: 80
```

### Gateway API Status (August 2025)
- **Standard Channel**: Gateway, GatewayClass, HTTPRoute
- **Experimental**: TCPRoute, UDPRoute, GRPCRoute, TLSRoute
- **Implementation Status**: 20+ implementations available
- **Graduation**: Moving towards GA in Kubernetes 1.30+

---

## Network Policies

### What are Network Policies?
**Network Policies** are Kubernetes resources that control network traffic flow at the IP address or port level (Layer 3/4) for TCP, UDP, and SCTP protocols.

### Network Policy Model
- **Default Behavior**: All pods can communicate with all pods
- **Policy Application**: Policies create restrictions (whitelist model)
- **Additive Nature**: Multiple policies affecting same pod are combined
- **Directionality**: Separate ingress and egress rules

### Prerequisites
Network policies require a CNI plugin that supports them:
- **Calico** ✅
- **Cilium** ✅  
- **Weave Net** ✅
- **Antrea** ✅
- **Flannel** ❌ (no network policy support)

### Basic Network Policy Structure
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-network-policy
  namespace: default
spec:
  # Target pods (empty = no pods selected)
  podSelector:
    matchLabels:
      app: web-server
  
  # Policy types
  policyTypes:
  - Ingress    # Control incoming traffic
  - Egress     # Control outgoing traffic
  
  # Ingress rules (incoming traffic)
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: trusted
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.1.0.0/16
    ports:
    - protocol: TCP
      port: 8080
  
  # Egress rules (outgoing traffic)  
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Common Network Policy Patterns

#### 1. Deny All Traffic (Default Deny)
```yaml
# Deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}    # Select all pods in namespace
  policyTypes:
  - Ingress         # Empty ingress rules = deny all

---
# Deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
spec:
  podSelector: {}    # Select all pods in namespace
  policyTypes:
  - Egress          # Empty egress rules = deny all

---
# Deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}    # Select all pods in namespace
  policyTypes:
  - Ingress
  - Egress
  # No rules specified = deny all
```

#### 2. Allow Specific Pod-to-Pod Communication
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend      # Target: backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # Source: frontend pods only
    ports:
    - protocol: TCP
      port: 8080
```

#### 3. Namespace Isolation
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}           # All pods in production namespace
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production  # Only from production namespace
    - namespaceSelector:
        matchLabels:
          name: monitoring  # And monitoring namespace
  
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
  - to:
    - namespaceSelector:
        matchLabels:
          name: shared-services
  - to: []                  # Allow DNS
    ports:
    - protocol: UDP
      port: 53
```

#### 4. Database Access Control
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-access
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  
  ingress:
  # Allow backend services
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow admin access from admin namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: admin
    - podSelector:
        matchLabels:
          role: database-admin
    ports:
    - protocol: TCP
      port: 5432
```

#### 5. External Traffic Control
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: external-api-access
spec:
  podSelector:
    matchLabels:
      app: api-service
  policyTypes:
  - Egress
  
  egress:
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  
  # Allow access to external APIs
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8      # Block internal networks
        - 192.168.0.0/16
        - 172.16.0.0/12
    ports:
    - protocol: TCP
      port: 443          # HTTPS only
  
  # Allow access to internal services
  - to:
    - podSelector:
        matchLabels:
          app: internal-service
    ports:
    - protocol: TCP
      port: 8080
```

### Advanced Network Policy Features

#### Named Ports
```yaml
# Pod with named ports
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
      name: http
    - containerPort: 443
      name: https

---
# Network policy using named ports
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-server-policy
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: http        # Reference named port
    - protocol: TCP
      port: https
```

#### Complex Selectors
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: complex-selector-policy
spec:
  podSelector:
    matchExpressions:
    - key: app
      operator: In
      values: [web, api]
    - key: version
      operator: NotIn
      values: [v1.0.0]
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchExpressions:
        - key: tier
          operator: In
          values: [frontend, proxy]
    - namespaceSelector:
        matchExpressions:
        - key: environment
          operator: NotIn
          values: [test, development]
    ports:
    - protocol: TCP
      port: 8080
```

---

## DNS and Service Discovery

### Kubernetes DNS (CoreDNS)

#### DNS Architecture
```
┌─────────────────────────────────────────┐
│               CoreDNS                   │
│         (kube-system namespace)         │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│          DNS Resolution                 │
│                                         │
│  service.namespace.svc.cluster.local    │
│  pod-ip.namespace.pod.cluster.local     │
│  node.cluster.local                     │
└─────────────────────────────────────────┘
```

#### DNS Record Types

**Service Records:**
```bash
# ClusterIP service
my-service.default.svc.cluster.local → 10.96.240.100

# Headless service  
my-headless.default.svc.cluster.local → 10.244.1.5,10.244.2.6,10.244.3.7

# Service ports (SRV records)
_http._tcp.my-service.default.svc.cluster.local → 80
_https._tcp.my-service.default.svc.cluster.local → 443
```

**Pod Records:**
```bash
# Pod A record (requires hostname and subdomain)
pod-hostname.my-subdomain.default.svc.cluster.local → 10.244.1.5

# Pod IP-based record
10-244-1-5.default.pod.cluster.local → 10.244.1.5
```

### CoreDNS Configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4 {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    
    # Custom domain resolution
    example.com:53 {
        errors
        cache 30
        forward . 1.1.1.1 1.0.0.1
    }
```

### Service Discovery Examples

#### DNS-Based Discovery
```yaml
# Application using DNS service discovery
apiVersion: v1
kind: Pod
metadata:
  name: client-app
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      # Different ways to resolve services
      nslookup web-service                           # Same namespace
      nslookup web-service.default                   # Specify namespace  
      nslookup web-service.default.svc               # Include svc
      nslookup web-service.default.svc.cluster.local # Full FQDN
      
      # Connect to service
      wget -O- http://web-service/api/health
      wget -O- http://web-service.backend.svc.cluster.local:8080/
```

#### Headless Service Discovery
```yaml
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: database-cluster
spec:
  clusterIP: None
  selector:
    app: database
  ports:
  - port: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: database-cluster
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        # Each pod gets unique hostname:
        # postgres-0.database-cluster.default.svc.cluster.local
        # postgres-1.database-cluster.default.svc.cluster.local  
        # postgres-2.database-cluster.default.svc.cluster.local
```

#### Custom DNS Configuration
```yaml
# Pod with custom DNS settings
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None          # Override default DNS
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 1.1.1.1
    searches:
    - my.dns.search.suffix
    - another.search.domain
    options:
    - name: ndots
      value: "2"
    - name: edns0
  containers:
  - name: app
    image: nginx

---
# Pod using cluster DNS + custom settings
apiVersion: v1
kind: Pod  
metadata:
  name: mixed-dns-pod
spec:
  dnsPolicy: ClusterFirst   # Use cluster DNS
  dnsConfig:
    nameservers:           # Additional nameservers
    - 8.8.8.8
    searches:              # Additional search domains
    - custom.domain.com
    options:
    - name: timeout
      value: "2"
  containers:
  - name: app
    image: nginx
```

---

## CNI (Container Network Interface)

### What is CNI?
**CNI** is a specification and library for configuring network interfaces in Linux containers. It's used by Kubernetes to set up pod networking.

### CNI Components
- **CNI Plugin**: Executable that configures network interfaces
- **CNI Configuration**: JSON configuration for network setup
- **Runtime**: Container runtime that invokes CNI plugins

### CNI Plugin Categories

#### IPAM Plugins (IP Address Management)
- **host-local**: Allocates IPs from local ranges
- **dhcp**: Uses DHCP for IP allocation
- **static**: Assigns static IP addresses

#### Interface Plugins
- **bridge**: Creates Linux bridge
- **vlan**: VLAN configuration
- **ipvlan**: IP-level virtual networking
- **macvlan**: MAC-level virtual networking

#### Meta Plugins
- **flannel**: Overlay networking
- **calico**: Policy-based networking
- **cilium**: eBPF-based networking
- **weave**: Mesh networking

### Popular CNI Solutions

#### 1. Flannel
```yaml
# Flannel CNI configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

**Flannel Characteristics:**
- **Simple**: Easy to set up and configure
- **Overlay**: Uses VXLAN for pod communication
- **No Network Policies**: Doesn't support network policies
- **Performance**: Good for basic networking needs

#### 2. Calico
```yaml
# Calico configuration example
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never

---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-all
spec:
  selector: all()
  types:
  - Ingress
  - Egress
```

**Calico Features:**
- **Network Policies**: Full support for Kubernetes network policies
- **BGP**: Native BGP routing support
- **Scalability**: Handles large clusters efficiently
- **Security**: Advanced security features
- **Flexibility**: Multiple networking modes

#### 3. Cilium
```yaml
# Cilium configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
  namespace: kube-system
data:
  enable-ipv4: "true"
  enable-ipv6: "false"
  cluster-pool-ipv4-cidr: "10.244.0.0/16"
  enable-bpf-masquerade: "true"
  enable-host-reachable-services: "true"
```

**Cilium Features:**
- **eBPF**: High-performance networking using eBPF
- **Observability**: Advanced network monitoring
- **Security**: Identity-based security policies
- **L7 Policies**: Application-layer policy enforcement
- **Service Mesh**: Native service mesh capabilities

#### 4. Weave Net
```bash
# Install Weave Net
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

**Weave Features:**
- **Simplicity**: Zero-configuration networking
- **Encryption**: Built-in network encryption
- **Multicast**: Supports multicast traffic
- **Network Policies**: Supports Kubernetes network policies
- **Resilience**: Automatic network healing

### CNI Configuration
```json
{
  "cniVersion": "0.4.0",
  "name": "bridge-network",
  "type": "bridge",
  "bridge": "cni-bridge",
  "isGateway": true,
  "ipMasq": true,
  "hairpinMode": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [
        {
          "subnet": "10.244.0.0/16",
          "gateway": "10.244.0.1"
        }
      ]
    ]
  }
}
```

### CNI Troubleshooting
```bash
# Check CNI plugin installation
ls -la /opt/cni/bin/
ls -la /etc/cni/net.d/

# View CNI configuration
cat /etc/cni/net.d/10-flannel.conflist

# Check kubelet CNI settings
ps aux | grep kubelet | grep cni

# View network interfaces
ip addr show
ip route show

# Check pod networking
kubectl exec -it <pod> -- ip addr
kubectl exec -it <pod> -- ip route
```

---

## Practice Labs

### Lab 1: Service Types and Discovery

#### Objective
Understand different service types and service discovery mechanisms.

#### Setup
```bash
# Create test application
kubectl create deployment web-app --image=nginx:1.21 --replicas=3
kubectl create deployment api-app --image=httpd:2.4 --replicas=2

# Label the pods
kubectl label deployment web-app app=web tier=frontend
kubectl label deployment api-app app=api tier=backend
```

#### Service Creation and Testing
```bash
# 1. Create ClusterIP service
kubectl expose deployment web-app --name=web-clusterip --port=80 --type=ClusterIP

# 2. Create NodePort service
kubectl expose deployment web-app --name=web-nodeport --port=80 --type=NodePort

# 3. Create LoadBalancer service (if supported)
kubectl expose deployment web-app --name=web-loadbalancer --port=80 --type=LoadBalancer

# 4. Test ClusterIP service
kubectl run test-pod --image=busybox --rm -it --restart=Never -- /bin/sh
# Inside the pod:
nslookup web-clusterip
wget -qO- http://web-clusterip
exit

# 5. Test NodePort service
kubectl get svc web-nodeport
kubectl get nodes -o wide
# Access via node IP and NodePort

# 6. Test service discovery
kubectl run dns-test --image=busybox --rm -it --restart=Never -- /bin/sh
# Inside the pod:
nslookup web-clusterip
nslookup web-clusterip.default
nslookup web-clusterip.default.svc.cluster.local
```

#### Headless Service Testing
```yaml
# Create headless service
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
```

```bash
# Apply headless service
kubectl apply -f headless-service.yaml

# Test headless service resolution
kubectl run headless-test --image=busybox --rm -it --restart=Never -- /bin/sh
# Inside the pod:
nslookup web-headless
# Should return multiple IP addresses
```

### Lab 2: Ingress Configuration

#### Objective
Set up and configure Ingress for HTTP routing.

#### Prerequisites
```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

#### Setup Test Applications
```yaml
# Frontend application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80

---
# API application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: httpd:2.4
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
```

#### Ingress Configuration
```yaml
# Basic ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

#### Testing Commands
```bash
# Apply applications and ingress
kubectl apply -f test-apps.yaml
kubectl apply -f basic-ingress.yaml

# Get ingress information
kubectl get ingress
kubectl describe ingress web-ingress

# Test ingress (add app.local to /etc/hosts pointing to ingress IP)
curl http://app.local/
curl http://app.local/api/

# Test with port forwarding if no external IP
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
curl -H "Host: app.local" http://localhost:8080/
curl -H "Host: app.local" http://localhost:8080/api/
```

### Lab 3: Network Policies

#### Objective
Implement and test network policies for traffic control.

#### Prerequisites
```bash
# Ensure you have a CNI that supports network policies
# This example assumes Calico is installed
kubectl get pods -n kube-system -l k8s-app=calico-node
```

#### Setup Test Environment
```yaml
# Create namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: frontend
  labels:
    tier: frontend
---
apiVersion: v1
kind: Namespace
metadata:
  name: backend
  labels:
    tier: backend
---
apiVersion: v1
kind: Namespace
metadata:
  name: database
  labels:
    tier: database

---
# Frontend app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:1.21
        ports:
        - containerPort: 80

---
# Backend app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: httpd:2.4
        ports:
        - containerPort: 80

---
# Database app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-app
  namespace: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: database
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
        ports:
        - containerPort: 5432
```

#### Network Policy Implementation
```yaml
# 1. Default deny all in database namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 2. Allow backend to database communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432

---
# 3. Allow frontend to backend communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80

---
# 4. Allow DNS resolution from database namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

#### Testing Network Policies
```bash
# Apply the test environment
kubectl apply -f network-policy-test.yaml

# Wait for pods to be ready
kubectl wait --for=condition=Ready pod -l app=frontend -n frontend --timeout=60s
kubectl wait --for=condition=Ready pod -l app=backend -n backend --timeout=60s
kubectl wait --for=condition=Ready pod -l app=database -n database --timeout=60s

# Test connectivity before policies
kubectl exec -it -n frontend $(kubectl get pod -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- wget -qO- --timeout=5 http://backend-app.backend
kubectl exec -it -n backend $(kubectl get pod -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}') -- nc -zv database-app.database 5432

# Apply network policies
kubectl apply -f network-policies.yaml

# Test connectivity after policies
# This should work (frontend -> backend)
kubectl exec -it -n frontend $(kubectl get pod -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- wget -qO- --timeout=5 http://backend-app.backend

# This should work (backend -> database)
kubectl exec -it -n backend $(kubectl get pod -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}') -- nc -zv database-app.database 5432

# This should fail (frontend -> database)
kubectl exec -it -n frontend $(kubectl get pod -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- nc -zv database-app.database 5432

# View network policy status
kubectl describe networkpolicy -A
```

---

## Hands-on Projects

### Project 1: Multi-Tier Application with Service Mesh

#### Objective
Deploy a complete e-commerce application with frontend, API, and database tiers, implementing proper service communication and security.

#### Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   API Gateway   │    │   Microservices │
│   (React SPA)   │───▶│   (Ingress)     │───▶│   (Products,    │
│                 │    │                 │    │    Users, etc.) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                              │
                       ┌─────────────────┐    │
                       │   Database      │◀───┘
                       │   (PostgreSQL)  │
                       └─────────────────┘
```

#### Implementation

**Step 1: Database Layer**
```yaml
# PostgreSQL StatefulSet with persistent storage
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: ecommerce
type: Opaque
data:
  username: cG9zdGdyZXM=  # postgres
  password: c2VjdXJlcGFzc3dvcmQ=  # securepassword

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: ecommerce
data:
  init.sql: |
    CREATE DATABASE IF NOT EXISTS products;
    CREATE DATABASE IF NOT EXISTS users;
    CREATE DATABASE IF NOT EXISTS orders;
    
    \c products;
    CREATE TABLE IF NOT EXISTS products (
      id SERIAL PRIMARY KEY,
      name VARCHAR(255) NOT NULL,
      price DECIMAL(10,2) NOT NULL,
      description TEXT
    );
    
    \c users;
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      username VARCHAR(50) UNIQUE NOT NULL,
      email VARCHAR(255) UNIQUE NOT NULL
    );

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: ecommerce
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: init-scripts
          mountPath: /docker-entrypoint-initdb.d
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"
      volumes:
      - name: init-scripts
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi

---
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: ecommerce
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432

---
# ClusterIP service for applications
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: ecommerce
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

**Step 2: Microservices Layer**
```yaml
# Product Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
        tier: backend
        service: products
    spec:
      containers:
      - name: product-service
        image: node:16-alpine
        command: ['sh', '-c']
        args:
        - |
          cat > server.js << 'EOF'
          const express = require('express');
          const app = express();
          const port = 3001;
          
          app.get('/health', (req, res) => {
            res.json({ service: 'product-service', status: 'healthy' });
          });
          
          app.get('/products', (req, res) => {
            res.json([
              { id: 1, name: 'Laptop', price: 999.99 },
              { id: 2, name: 'Phone', price: 499.99 }
            ]);
          });
          
          app.listen(port, () => {
            console.log(`Product service running on port ${port}`);
          });
          EOF
          
          npm init -y && npm install express
          node server.js
        ports:
        - containerPort: 3001
        env:
        - name: DB_HOST
          value: postgres-service
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "100m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: ecommerce
spec:
  selector:
    app: product-service
  ports:
  - port: 3001
    targetPort: 3001

---
# User Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        tier: backend
        service: users
    spec:
      containers:
      - name: user-service
        image: node:16-alpine
        command: ['sh', '-c']
        args:
        - |
          cat > server.js << 'EOF'
          const express = require('express');
          const app = express();
          const port = 3002;
          
          app.get('/health', (req, res) => {
            res.json({ service: 'user-service', status: 'healthy' });
          });
          
          app.get('/users', (req, res) => {
            res.json([
              { id: 1, username: 'john_doe', email: 'john@example.com' },
              { id: 2, username: 'jane_smith', email: 'jane@example.com' }
            ]);
          });
          
          app.listen(port, () => {
            console.log(`User service running on port ${port}`);
          });
          EOF
          
          npm init -y && npm install express
          node server.js
        ports:
        - containerPort: 3002
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "100m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3002
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: ecommerce
spec:
  selector:
    app: user-service
  ports:
  - port: 3002
    targetPort: 3002
```

**Step 3: API Gateway**
```yaml
# API Gateway Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
        tier: api
    spec:
      containers:
      - name: api-gateway
        image: node:16-alpine
        command: ['sh', '-c']
        args:
        - |
          cat > gateway.js << 'EOF'
          const express = require('express');
          const http = require('http');
          const app = express();
          const port = 3000;
          
          app.use(express.json());
          
          // Proxy requests to microservices
          app.get('/api/products', (req, res) => {
            const options = {
              hostname: 'product-service',
              port: 3001,
              path: '/products',
              method: 'GET'
            };
            
            const proxyReq = http.request(options, (proxyRes) => {
              let data = '';
              proxyRes.on('data', chunk => data += chunk);
              proxyRes.on('end', () => res.json(JSON.parse(data)));
            });
            
            proxyReq.on('error', (err) => {
              res.status(500).json({ error: 'Service unavailable' });
            });
            
            proxyReq.end();
          });
          
          app.get('/api/users', (req, res) => {
            const options = {
              hostname: 'user-service',
              port: 3002,
              path: '/users',
              method: 'GET'
            };
            
            const proxyReq = http.request(options, (proxyRes) => {
              let data = '';
              proxyRes.on('data', chunk => data += chunk);
              proxyRes.on('end', () => res.json(JSON.parse(data)));
            });
            
            proxyReq.end();
          });
          
          app.get('/health', (req, res) => {
            res.json({ service: 'api-gateway', status: 'healthy' });
          });
          
          app.listen(port, () => {
            console.log(`API Gateway running on port ${port}`);
          });
          EOF
          
          npm init -y && npm install express
          node gateway.js
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "100m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service
  namespace: ecommerce
spec:
  selector:
    app: api-gateway
  ports:
  - port: 3000
    targetPort: 3000
```

**Step 4: Frontend & Ingress**
```yaml
# Frontend Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [frontend]
              topologyKey: kubernetes.io/hostname
      containers:
      - name: frontend
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: static-content
          mountPath: /usr/share/nginx/html
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      volumes:
      - name: nginx-config
        configMap:
          name: frontend-nginx-config
      - name: static-content
        configMap:
          name: frontend-content

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-nginx-config
  namespace: ecommerce
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
        
        location /api/ {
            proxy_pass http://api-gateway-service:3000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-content
  namespace: ecommerce
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>E-Commerce App</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            .container { max-width: 800px; margin: 0 auto; }
            button { padding: 10px 20px; margin: 10px; }
            .results { margin-top: 20px; padding: 20px; background: #f5f5f5; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>E-Commerce Application</h1>
            <button onclick="fetchProducts()">Load Products</button>
            <button onclick="fetchUsers()">Load Users</button>
            <div id="results" class="results"></div>
        </div>
        
        <script>
        async function fetchProducts() {
            try {
                const response = await fetch('/api/products');
                const data = await response.json();
                document.getElementById('results').innerHTML = 
                    '<h3>Products:</h3><pre>' + JSON.stringify(data, null, 2) + '</pre>';
            } catch (error) {
                document.getElementById('results').innerHTML = 
                    '<p style="color: red;">Error loading products: ' + error.message + '</p>';
            }
        }
        
        async function fetchUsers() {
            try {
                const response = await fetch('/api/users');
                const data = await response.json();
                document.getElementById('results').innerHTML = 
                    '<h3>Users:</h3><pre>' + JSON.stringify(data, null, 2) + '</pre>';
            } catch (error) {
                document.getElementById('results').innerHTML = 
                    '<p style="color: red;">Error loading users: ' + error.message + '</p>';
            }
        }
        </script>
    </body>
    </html>

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: ecommerce
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80

---
# Ingress configuration
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  rules:
  - host: ecommerce.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Step 5: Network Policies for Security**
```yaml
# Database isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-isolation
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Allow backend services to access database
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  
  egress:
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53

---
# Backend isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-isolation
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Allow API gateway to access backend services
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 3001
    - protocol: TCP
      port: 3002
  
  egress:
  # Allow access to database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53

---
# API Gateway policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-gateway-policy
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Allow frontend to access API gateway
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 3000
  
  egress:
  # Allow access to backend services
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 3001
    - protocol: TCP
      port: 3002
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

#### Deployment and Testing
```bash
# 1. Create namespace
kubectl create namespace ecommerce

# 2. Deploy all components
kubectl apply -f database-layer.yaml
kubectl apply -f microservices-layer.yaml
kubectl apply -f api-gateway.yaml
kubectl apply -f frontend-ingress.yaml
kubectl apply -f network-policies.yaml

# 3. Wait for all deployments
kubectl wait --for=condition=Ready pod -l app=postgres -n ecommerce --timeout=120s
kubectl wait --for=condition=Ready pod -l tier=backend -n ecommerce --timeout=120s
kubectl wait --for=condition=Ready pod -l tier=api -n ecommerce --timeout=120s
kubectl wait --for=condition=Ready pod -l tier=frontend -n ecommerce --timeout=120s

# 4. Check all services
kubectl get all -n ecommerce

# 5. Test the application
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
# Add to /etc/hosts: 127.0.0.1 ecommerce.local
# Open browser to http://ecommerce.local:8080

# 6. Test API endpoints directly
kubectl port-forward -n ecommerce svc/api-gateway-service 3000:3000
curl http://localhost:3000/api/products
curl http://localhost:3000/api/users

# 7. Test network policies
kubectl exec -it -n ecommerce $(kubectl get pod -n ecommerce -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- wget -qO- --timeout=5 http://api-gateway-service:3000/health

# This should fail (frontend cannot access database directly)
kubectl exec -it -n ecommerce $(kubectl get pod -n ecommerce -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- nc -zv postgres-service 5432
```

---

## Troubleshooting Guide

### Service Issues

#### 1. Service Not Accessible

**Symptoms:**
```bash
# Connection refused or timeouts
kubectl exec -it test-pod -- wget -qO- service-name
# wget: can't connect to remote host: Connection refused
```

**Diagnosis:**
```bash
# Check service configuration
kubectl get svc service-name
kubectl describe svc service-name

# Check endpoints
kubectl get endpoints service-name
kubectl describe endpoints service-name

# Check pod labels and selectors
kubectl get pods -l app=service-label --show-labels
kubectl describe svc service-name | grep Selector
```

**Common Solutions:**
- **Label Mismatch**: Ensure service selector matches pod labels
- **Port Configuration**: Verify port and targetPort settings
- **Pod Readiness**: Check if pods pass readiness probes

#### 2. LoadBalancer External IP Pending

**Symptoms:**
```bash
kubectl get svc
# NAME         TYPE           EXTERNAL-IP    PORT(S)
# my-service   LoadBalancer   <pending>      80:32000/TCP
```

**Diagnosis:**
```bash
# Check cloud provider integration
kubectl describe svc my-service
kubectl get events --field-selector involvedObject.name=my-service

# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

**Solutions:**
- **Cloud Provider**: Ensure cluster has cloud controller manager
- **Quotas**: Check cloud provider load balancer quotas
- **Permissions**: Verify cloud provider permissions

### Ingress Issues

#### 1. Ingress Not Working

**Diagnosis Steps:**
```bash
# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Check ingress resource
kubectl get ingress
kubectl describe ingress my-ingress

# Check ingress class
kubectl get ingressclass
kubectl describe ingressclass nginx
```

#### 2. SSL/TLS Issues

**Common Problems:**
```bash
# Check TLS secret
kubectl get secret tls-secret -o yaml
kubectl describe secret tls-secret

# Verify certificate
kubectl get secret tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

### Network Policy Issues

#### 1. Network Policy Not Enforcing

**Verification Steps:**
```bash
# Check CNI plugin supports network policies
kubectl get pods -n kube-system -l k8s-app=calico-node

# Check network policy status
kubectl get networkpolicy
kubectl describe networkpolicy my-policy

# Test connectivity
kubectl exec -it source-pod -- nc -zv target-service 80
```

#### 2. Pods Cannot Communicate After Policy

**Debugging:**
```bash
# Check DNS is allowed
kubectl exec -it pod-name -- nslookup kubernetes.default

# Verify policy selectors
kubectl get pods --show-labels
kubectl describe networkpolicy policy-name
```

### DNS Issues

#### 1. Service Discovery Failures

**Diagnosis:**
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution
kubectl exec -it test-pod -- nslookup kubernetes.default
kubectl exec -it test-pod -- nslookup my-service.default.svc.cluster.local

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml
```

#### 2. Custom DNS Configuration

**Testing:**
```bash
# Check pod DNS configuration
kubectl exec -it pod-name -- cat /etc/resolv.conf

# Test external DNS
kubectl exec -it pod-name -- nslookup google.com
```

### CNI Issues

#### 1. Pod Networking Problems

**Diagnosis:**
```bash
# Check CNI plugin installation
ls -la /opt/cni/bin/
ls -la /etc/cni/net.d/

# Check pod IP assignment
kubectl get pods -o wide
kubectl describe pod problem-pod

# Check node networking
kubectl get nodes
kubectl describe node node-name
```

#### 2. Cross-Node Pod Communication

**Testing:**
```bash
# Check routing tables
ip route show
kubectl exec -it pod1 -- traceroute pod2-ip

# Check CNI plugin logs
kubectl logs -n kube-system -l app=calico-node
kubectl logs -n kube-system -l app=flannel
```

---

This comprehensive guide covers all aspects of Kubernetes Services and Networking. Master these concepts to effectively manage network communication, traffic routing, and security in your Kubernetes clusters.