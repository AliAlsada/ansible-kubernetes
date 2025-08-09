# Kubernetes Cluster Startup Issues - Root Cause Analysis and Resolution

## Issue Summary
A Kubernetes master node experienced persistent API server instability with components failing to start properly, causing intermittent cluster unavailability and kubectl connection failures.

## Symptoms Observed
- API server restarting 28+ times with graceful shutdowns
- Intermittent `kubectl` connection failures: "Connection refused" to port 6443
- Control plane components in CrashLoopBackOff state
- DNS nameserver limit exceeded errors
- Node authorization errors: "no relationship found between node"

## Root Cause Analysis

### Primary Issue: Race Condition Between etcd and API Server
**Problem**: API server startup probe was configured to check health before etcd was fully ready to serve requests.

**Timeline of Failure**:
```
1. kubelet starts etcd container
2. kubelet starts API server container  
3. API server begins initialization
4. After 10 seconds: startupProbe begins health checks to 192.168.121.164:6443/livez
5. etcd still initializing (not ready until ~30+ seconds)
6. API server cannot connect to etcd (connection refused to 127.0.0.1:2379)
7. API server health checks fail
8. kubelet kills API server container
9. Process repeats
```

**Evidence**:
```bash
# API server logs showed:
W0804 15:20:31.824493 logging.go:55] grpc: addrConn.createTransport failed to connect to {Addr: "127.0.0.1:2379"}: connection refused

# etcd logs showed it became ready later:
{"level":"info","ts":"2025-08-04T15:20:34.920732Z","msg":"serving client traffic securely","address":"127.0.0.1:2379"}
```

### Secondary Issue: DNS Configuration
**Problem**: Too many DNS nameservers configured, exceeding Linux limit of 3.

**Evidence**:
```bash
# kubelet logs:
E0804 14:57:03.075549 dns.go:153] "Nameserver limits exceeded" err="Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 8.8.8.8 8.8.4.4 4.2.2.1"

# DNS configuration showed 6 total nameservers:
Global DNS Servers: 8.8.8.8, 8.8.4.4
Link 2 (eth0) DNS Servers: 4.2.2.1, 4.2.2.2, 208.67.220.220, 192.168.121.1
```

### 1. Fix DNS Configuration
```bash
# Reduce DNS servers on eth0 interface to prevent limit exceeded
resolvectl revert eth0
resolvectl dns eth0 192.168.121.1
systemctl restart systemd-resolved
```

### 3. Verification Steps
```bash
# Verify configuration changes
grep -A 10 "startupProbe:" /etc/kubernetes/manifests/kube-apiserver.yaml

# Check DNS configuration
resolvectl status

# Monitor API server startup
journalctl -u kubelet -f | grep -E "apiserver|startup|probe"

# Test cluster functionality
kubectl get nodes
kubectl get pods -n kube-system
```

## Prevention Measures

### 1. Proper Cluster Initialization
```bash
# Use proper kubeadm init with all required parameters
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<NODE_IP> \
  --control-plane-endpoint=<NODE_IP>:6443
```

### 2. Pre-Installation Checks
```bash
# Verify DNS configuration before cluster init
resolvectl status | wc -l  # Should show reasonable number of nameservers

# Ensure adequate resources
df -h          # Sufficient disk space
free -h        # Adequate memory
nproc          # Multiple CPU cores recommended
```

## Key Learnings

### Kubernetes Component Dependencies
```
etcd → API Server → Controller Manager → Scheduler → Other Components
```
Startup probes must account for these dependencies with appropriate delays.

### Container Runtime Considerations
- **Docker**: Use `docker logs <container-id>` for debugging
- **containerd**: Use `crictl logs <container-id>` for debugging
- Health checks should use localhost when possible for reliability

### DNS in Container Environments
- Linux limit: maximum 3 nameservers in `/etc/resolv.conf`
- systemd-resolved can aggregate multiple sources
- Container DNS issues can cause pod startup failures

## Monitoring and Alerting Recommendations

### Key Metrics to Monitor
```bash
# API server availability
curl -k https://localhost:6443/healthz

# Component restart counts
kubectl get pods -n kube-system -o wide

# etcd health
ETCDCTL_API=3 etcdctl endpoint health --cluster

# Node conditions
kubectl describe nodes | grep -A 10 Conditions
```

### Log Patterns to Alert On
- `connection refused` to etcd (127.0.0.1:2379)
- `Nameserver limits exceeded`
- High container restart counts in kube-system namespace
- `CrashLoopBackOff` for control plane components

## Tools Used for Diagnosis
- `journalctl -u kubelet` - kubelet service logs
- `crictl ps -a` - container runtime inspection
- `crictl logs <container-id>` - container logs
- `kubectl describe node` - node status and events
- `curl` - API server health endpoint testing
- `resolvectl status` - DNS configuration inspection

## Conclusion
This issue demonstrates the importance of proper timing configuration in Kubernetes startup probes and the cascading effects that DNS and networking misconfigurations can have on cluster stability. The resolution involved addressing the root cause (timing race condition) and contributing factors (DNS limits, suboptimal health check configuration) to achieve a stable, production-ready cluster.

**Estimated Downtime**: ~45 minutes of intermittent availability
**Resolution Time**: ~30 minutes once root cause identified
**Impact**: Development cluster - no production workloads affected