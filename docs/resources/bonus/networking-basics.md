# Networking Basics

A quick refresher on networking concepts essential for understanding Kubernetes networking.

## OSI Model (Simplified)

Kubernetes operates primarily at these layers:

| Layer | Name        | Examples        | Relevance to K8s                 |
| ----- | ----------- | --------------- | -------------------------------- |
| **7** | Application | HTTP, DNS, gRPC | Ingress, Service mesh            |
| **4** | Transport   | TCP, UDP        | Service types, port mappings     |
| **3** | Network     | IP              | Pod IPs, Service IPs (ClusterIP) |
| **2** | Data Link   | Ethernet        | CNI plugins (Flannel, Calico)    |

## IP Addresses

### IPv4 Format

- Format: `xxx.xxx.xxx.xxx` (e.g., `192.168.1.10`)
- Range: `0.0.0.0` to `255.255.255.255`
- **In Kubernetes:** Pods and Services get IP addresses

### Private IP Ranges (RFC 1918)

Commonly used in Kubernetes and Docker:

```text
10.0.0.0    - 10.255.255.255    (10.0.0.0/8)      # Most flexible
172.16.0.0  - 172.31.255.255    (172.16.0.0/12)   # Docker default
192.168.0.0 - 192.168.255.255   (192.168.0.0/16)  # Home networks
```

### Special Addresses

```yaml
0.0.0.0         # "All interfaces" or "any address"
127.0.0.1       # Localhost (loopback)
localhost       # DNS name for 127.0.0.1
```

**In Kubernetes:**

- Binding to `0.0.0.0` means "listen on all interfaces"
- Binding to `127.0.0.1` means "only accept connections from this Pod"

## 🔌 Ports

### Port Numbers

- Range: `0-65535`
- Well-known ports: `0-1023` (require privileges)
- Registered ports: `1024-49151`
- Dynamic/Private: `49152-65535`

### Common Ports

| Port  | Protocol | Service                          |
| ----- | -------- | -------------------------------- |
| 22    | TCP      | SSH                              |
| 80    | TCP      | HTTP                             |
| 443   | TCP      | HTTPS                            |
| 3000  | TCP      | Common dev servers (React, etc.) |
| 5432  | TCP      | PostgreSQL                       |
| 6379  | TCP      | Redis                            |
| 8080  | TCP      | Alternative HTTP                 |
| 27017 | TCP      | MongoDB                          |

### Port Terminology

```yaml
# In Kubernetes manifests:
spec:
  containers:
    - name: web
      ports:
        - containerPort: 80    # Port inside the container
  ---
spec:
  ports:
    - port: 80                # Port exposed by the Service
      targetPort: 8080        # Port on the Pod (container)
      nodePort: 30000         # Port on the Node (for NodePort services)
```

## TCP vs UDP

| TCP (Transmission Control Protocol) | UDP (User Datagram Protocol) |
| ----------------------------------- | ---------------------------- |
| **Connection-oriented**             | **Connectionless**           |
| Reliable (guarantees delivery)      | Unreliable (best effort)     |
| Ordered packets                     | Unordered packets            |
| Slower (overhead)                   | Faster (low overhead)        |
| HTTP, SSH, databases                | DNS, video streaming, gaming |

**In Kubernetes:**

```yaml
# Services can use either protocol
spec:
  ports:
    - port: 80
      protocol: TCP    # or UDP
```

## DNS (Domain Name System)

### What is DNS?

DNS translates domain names to IP addresses.

```text
google.com  →  (DNS lookup)  →  142.250.185.46
```

### Kubernetes DNS

Every Kubernetes cluster has an internal DNS service. Resources can communicate using names instead of IPs:

```bash
# Format: <service-name>.<namespace>.svc.cluster.local

# Same namespace (short form)
http://web-service

# Different namespace
http://web-service.production

# Fully qualified domain name (FQDN)
http://web-service.production.svc.cluster.local
```

**Example:**

```yaml
# Service in namespace 'default'
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: default

# Pods can reach it via:
# - api-service (from same namespace)
# - api-service.default (from any namespace)
# - api-service.default.svc.cluster.local (FQDN)
```

## Routing & Subnets

### CIDR Notation

**CIDR** (Classless Inter-Domain Routing) describes IP ranges:

```text
10.0.0.0/8     # 16,777,216 addresses (10.0.0.0 - 10.255.255.255)
10.0.0.0/16    # 65,536 addresses     (10.0.0.0 - 10.0.255.255)
10.0.0.0/24    # 256 addresses        (10.0.0.0 - 10.0.0.255)
10.0.0.0/32    # 1 address            (10.0.0.0 only)
```

**The number after `/` is the number of fixed bits.**

- `/8` = 8 bits fixed, 24 bits variable = 2^24 = 16M addresses
- `/24` = 24 bits fixed, 8 bits variable = 2^8 = 256 addresses

**In Kubernetes:**

```yaml
# Pod network CIDR (all Pod IPs come from this range)
podCIDR: 10.244.0.0/16

# Service network CIDR (all Service IPs come from this range)
serviceCIDR: 10.96.0.0/12
```

## NAT (Network Address Translation)

### What is NAT?

NAT translates private IPs to public IPs (and vice versa).

```text
Private Network          NAT Gateway           Public Internet
10.0.0.5:45321    →→→   203.0.113.10:45321  →→→  142.250.185.46:443
```

**In Kubernetes:**

- Pods have private IPs
- Services of type `LoadBalancer` use NAT to expose Pods externally
- NodePort services use NAT to forward traffic from Node IP to Pod IP

## Firewalls & Network Policies

### Firewall Basics

Firewalls control traffic based on rules:

- **Ingress**: Incoming traffic
- **Egress**: Outgoing traffic

**In Kubernetes:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress    # Control incoming traffic
    - Egress     # Control outgoing traffic
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
```

## Load Balancing

### What is Load Balancing?

Distributing traffic across multiple servers.

```text
                    ┌──→ Pod 1 (10.244.0.5)
Client → Service → ├──→ Pod 2 (10.244.0.6)
                    └──→ Pod 3 (10.244.0.7)
```

### Load Balancing Algorithms

- **Round Robin**: Cycle through servers (1, 2, 3, 1, 2, 3...)
- **Least Connections**: Send to server with fewest connections
- **IP Hash**: Same client always goes to same server

**In Kubernetes:**

- Services automatically load balance across Pods matching their selector
- Default algorithm: Round Robin (configurable with service mesh)

## Kubernetes Networking Model

### Three Types of Communication

1. **Container-to-Container** (inside a Pod)
   - Via `localhost`
   - Ports must be unique

2. **Pod-to-Pod** (within cluster)
   - Every Pod gets a unique IP
   - Pods can communicate directly (flat network)
   - No NAT between Pods

3. **External-to-Service** (outside → cluster)
   - Via Services (ClusterIP, NodePort, LoadBalancer)
   - Via Ingress (HTTP/HTTPS routing)

### Network Diagram

```text
┌─────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                     │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Node (e.g., 192.168.1.10)                       │  │
│  │                                                   │  │
│  │  ┌───────────────┐  ┌───────────────┐           │  │
│  │  │ Pod           │  │ Pod           │           │  │
│  │  │ IP: 10.244.0.5│  │ IP: 10.244.0.6│           │  │
│  │  │               │  │               │           │  │
│  │  │ ┌──────────┐  │  │ ┌──────────┐  │           │  │
│  │  │ │Container │  │  │ │Container │  │           │  │
│  │  │ │ :80      │  │  │ │ :8080    │  │           │  │
│  │  │ └──────────┘  │  │ └──────────┘  │           │  │
│  │  └───────────────┘  └───────────────┘           │  │
│  │         │                    │                   │  │
│  │         └────────┬───────────┘                   │  │
│  │                  │                               │  │
│  │         ┌────────▼────────┐                      │  │
│  │         │ Service         │                      │  │
│  │         │ IP: 10.96.0.10  │                      │  │
│  │         │ Port: 80        │                      │  │
│  │         └─────────────────┘                      │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │   External  │
                    │   Traffic   │
                    └─────────────┘
```

## Common Network Troubleshooting Commands

### Inside a Pod

```bash
# Check if a port is listening
netstat -tlnp

# Test connectivity to another service
curl http://api-service:8080

# DNS lookup
nslookup api-service
dig api-service

# Ping another Pod (if ICMP allowed)
ping 10.244.0.5

# Trace route
traceroute api-service
```

### From kubectl

```bash
# Check Service endpoints
kubectl get endpoints

# Port forward to test locally
kubectl port-forward pod/my-pod 8080:80

# Check Pod networking
kubectl describe pod my-pod | grep IP

# Get Service details
kubectl get svc -o wide
```

## Docker Compose → Kubernetes Networking

### Docker Compose

```yaml
version: '3'
services:
  web:
    image: nginx
    ports:
      - "8080:80"      # Host:Container
    networks:
      - frontend

  api:
    image: api-server
    networks:
      - frontend
      - backend
    # Web can reach this via http://api

networks:
  frontend:
  backend:
```

### Kubernetes Equivalent

```yaml
# No explicit network creation needed!
# Services handle communication via DNS

apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer    # Exposes externally

---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 8080
  type: ClusterIP       # Internal only

# Pods communicate via Service DNS names:
# http://web, http://api, etc.
```

## Key Takeaways

1. **Every Pod gets an IP address** - no port conflicts between Pods
2. **Services provide stable endpoints** - Pod IPs change, Service IPs don't
3. **DNS is built-in** - use Service names, not IPs
4. **Network Policies are opt-in** - by default, all Pods can talk to all Pods
5. **Three Service types for different exposures**:
   - `ClusterIP`: Internal only
   - `NodePort`: Accessible via Node IP
   - `LoadBalancer`: Accessible via external IP

## Additional Resources

- [Kubernetes Networking Model](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Services Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Understanding CNI](https://www.cni.dev/)

---

**Pro Tip:** When debugging networking issues in Kubernetes, start with DNS:

```bash
# From within a Pod
kubectl exec -it my-pod -- nslookup my-service
```
