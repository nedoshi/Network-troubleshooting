# ARO Private Cluster Networking Troubleshooting Guide

## Overview

Azure Red Hat OpenShift (ARO) private clusters require careful network configuration to ensure proper connectivity between cluster components, Azure services, and external resources. This guide provides systematic troubleshooting steps for common networking issues.

## Prerequisites

Before troubleshooting, ensure you have:
- Azure CLI installed and authenticated
- OpenShift CLI (oc) installed
- kubectl installed
- Appropriate RBAC permissions for ARO cluster and Azure resources
- Access to Azure portal or Azure CLI for network diagnostics

## Common Networking Issues

### 1. Cluster Access Problems

#### Symptoms
- Unable to connect to OpenShift console
- `oc login` commands fail
- API server unreachable

#### Troubleshooting Steps

**Check API Server Visibility**
```bash
# Verify API server endpoint
az aro show --name <cluster-name> --resource-group <rg-name> --query apiserverProfile

# Test connectivity to API server
curl -k https://<api-server-url>:6443/healthz
```

**Verify Network Security Groups (NSGs)**
```bash
# List NSG rules for master subnet
az network nsg rule list --nsg-name <master-nsg-name> --resource-group <rg-name> --output table

# Check for required inbound rules:
# - Port 6443 (API server) from allowed sources
# - Port 22 (SSH) for management
```

**Validate Private DNS Resolution**
```bash
# Check if private DNS zone exists
az network private-dns zone list --resource-group <rg-name>

# Verify DNS records
az network private-dns record-set a list --zone-name <private-dns-zone> --resource-group <rg-name>
```

### 2. Pod-to-Pod Communication Issues

#### Symptoms
- Pods cannot communicate across nodes
- Service discovery failures
- Intermittent connection timeouts

#### Troubleshooting Steps

**Check Network Plugin Status**
```bash
# Verify SDN pods are running
oc get pods -n openshift-sdn

# Check network operator status
oc get clusteroperator network
```

**Examine Pod Network Configuration**
```bash
# Get pod details including IP addresses
oc get pods -o wide --all-namespaces

# Check if pods have correct subnet assignments
oc describe pod <pod-name> -n <namespace>
```

**Test Inter-Node Connectivity**
```bash
# Create test pods on different nodes
oc run test-pod-1 --image=busybox --command -- sleep 3600
oc run test-pod-2 --image=busybox --command -- sleep 3600

# Test connectivity between pods
oc exec test-pod-1 -- ping <test-pod-2-ip>
```

### 3. External Service Connectivity

#### Symptoms
- Pods cannot reach external services
- DNS resolution failures
- Egress traffic blocked

#### Troubleshooting Steps

**Verify Egress Rules**
```bash
# Check worker node NSG rules
az network nsg rule list --nsg-name <worker-nsg-name> --resource-group <rg-name>

# Ensure required outbound rules exist:
# - Port 443 (HTTPS) for container registries
# - Port 80 (HTTP) for package repositories
# - DNS ports (53) for name resolution
```

**Test DNS Resolution**
```bash
# Create test pod for network diagnostics
oc run network-test --image=busybox --rm -it -- sh

# Inside the pod, test DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup google.com
```

**Check Route Tables**
```bash
# Verify route tables for subnets
az network route-table list --resource-group <rg-name>

# Check routes for worker subnet
az network route-table route list --route-table-name <route-table-name> --resource-group <rg-name>
```

### 4. Load Balancer and Ingress Issues

#### Symptoms
- Services with LoadBalancer type don't get external IPs
- Ingress routes are unreachable
- Router pods failing

#### Troubleshooting Steps

**Check Router Pods**
```bash
# Verify router pods are running
oc get pods -n openshift-ingress

# Check router pod logs
oc logs -n openshift-ingress <router-pod-name>
```

**Examine Load Balancer Configuration**
```bash
# Check Azure Load Balancer resources
az network lb list --resource-group <node-rg-name>

# Verify load balancer rules and health probes
az network lb rule list --lb-name <lb-name> --resource-group <node-rg-name>
```

**Test Service Endpoints**
```bash
# Check service endpoints
oc get endpoints -n <namespace>

# Verify service configuration
oc describe service <service-name> -n <namespace>
```

### 5. Container Registry Access

#### Symptoms
- Image pull failures
- Authentication errors to Azure Container Registry
- Timeout errors during image pulls

#### Troubleshooting Steps

**Verify Registry Configuration**
```bash
# Check image pull secrets
oc get secrets --all-namespaces | grep pull-secret

# Verify registry configuration
oc get image.config.openshift.io/cluster -o yaml
```

**Test Registry Connectivity**
```bash
# Create test pod to verify registry access
oc run registry-test --image=busybox --rm -it -- sh

# Test connection to Azure Container Registry
wget -q --spider https://<registry-name>.azurecr.io
```

**Check Service Endpoints for Container Registry**
```bash
# Verify private endpoints for ACR if using private link
az network private-endpoint list --resource-group <rg-name>
```

## Network Diagnostic Commands

### Azure Network Diagnostics

```bash
# Network Watcher connection troubleshoot
az network watcher test-connectivity \
  --source-resource <vm-resource-id> \
  --dest-address <destination-ip> \
  --dest-port <port>

# Check effective routes
az network nic show-effective-route-table \
  --name <nic-name> \
  --resource-group <rg-name>

# Verify security rules
az network nic list-effective-nsg \
  --name <nic-name> \
  --resource-group <rg-name>
```

### OpenShift Network Diagnostics

```bash
# Check cluster network configuration
oc get network.config.openshift.io cluster -o yaml

# Verify network policies
oc get networkpolicy --all-namespaces

# Check DNS configuration
oc get dns.operator.openshift.io default -o yaml

# Examine CNI configuration
oc get pods -n openshift-multus
```

## Advanced Troubleshooting

### Packet Capture Analysis

```bash
# Enable packet capture on worker nodes (requires node debugging)
oc debug node/<node-name>

# Inside the debug pod:
chroot /host
tcpdump -i any -w /tmp/capture.pcap host <target-ip>
```

### Network Performance Testing

```bash
# Create iperf3 server pod
oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: iperf3-server
spec:
  containers:
  - name: iperf3-server
    image: networkstatic/iperf3
    command: ["/bin/sh", "-c", "iperf3 -s"]
    ports:
    - containerPort: 5201
EOF

# Create iperf3 client pod and test
oc run iperf3-client --image=networkstatic/iperf3 --rm -it -- iperf3 -c <server-pod-ip>
```

## Monitoring and Logging

### Enable Network Monitoring

```bash
# Check network metrics in Prometheus
oc get prometheus -n openshift-monitoring

# Query network-related metrics
oc exec -n openshift-monitoring prometheus-k8s-0 -- \
  promtool query instant 'container_network_receive_bytes_total'
```

### Collect Network Logs

```bash
# Collect SDN logs
oc logs -n openshift-sdn -l app=sdn --tail=100

# Collect DNS logs
oc logs -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default

# Collect ingress controller logs
oc logs -n openshift-ingress-operator deployment/ingress-operator
```

## Prevention and Best Practices

### Network Security

1. **Implement Network Policies**: Use Kubernetes NetworkPolicies to control pod-to-pod communication
2. **Regular Security Review**: Periodically review NSG rules and remove unnecessary access
3. **Monitor Traffic**: Use Azure Network Watcher and OpenShift monitoring to track network traffic

### Configuration Management

1. **Document Network Architecture**: Maintain up-to-date network diagrams and IP allocation schemes
2. **Version Control**: Store network configurations in version control systems
3. **Automated Testing**: Implement automated network connectivity tests

### Monitoring Setup

```bash
# Create network monitoring dashboard
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: network-monitoring-dashboard
  namespace: openshift-monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "ARO Network Monitoring",
        "panels": [
          {
            "title": "Network Traffic",
            "targets": [
              {
                "expr": "rate(container_network_receive_bytes_total[5m])"
              }
            ]
          }
        ]
      }
    }
EOF
```

## Common Error Codes and Solutions

| Error Code | Description | Solution |
|------------|-------------|----------|
| DNS_PROBE_FINISHED_NXDOMAIN | DNS resolution failure | Check DNS configuration and private DNS zones |
| Connection timeout | Network connectivity issues | Verify NSG rules and route tables |
| Image pull failed | Registry access issues | Check service endpoints and authentication |
| Service unavailable | Load balancer configuration | Verify Azure Load Balancer settings |

## Additional Resources

- [Azure Red Hat OpenShift Documentation](https://docs.microsoft.com/en-us/azure/openshift/)
- [OpenShift Network Troubleshooting](https://docs.openshift.com/container-platform/4.12/networking/troubleshooting-network-issues.html)
- [Azure Network Troubleshooting](https://docs.microsoft.com/en-us/azure/virtual-network/troubleshoot-connectivity-problem-between-vms)

## Conclusion

Network troubleshooting in ARO private clusters requires understanding both Azure networking components and OpenShift network architecture. Use this guide systematically, starting with basic connectivity tests and progressing to more advanced diagnostics as needed. Always document your findings and implement monitoring to prevent future issues.