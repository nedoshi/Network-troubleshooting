# OpenShift on GCP (OSD) Network Troubleshooting Guide

Complete guide for diagnosing and fixing networking issues in OpenShift Dedicated (OSD) clusters on Google Cloud Platform.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Quick Diagnostic Checklist](#quick-diagnostic-checklist)
4. [DNS Resolution Testing](#dns-resolution-testing)
5. [Network Connectivity Testing](#network-connectivity-testing)
6. [GCP Firewall Rules Configuration](#gcp-firewall-rules-configuration)
7. [VPC Network Configuration](#vpc-network-configuration)
8. [Routes and Cloud NAT](#routes-and-cloud-nat)
9. [Load Balancer Health](#load-balancer-health)
10. [Common Issues and Solutions](#common-issues-and-solutions)
11. [Step-by-Step Troubleshooting Process](#step-by-step-troubleshooting-process)
12. [Recommended Fix Actions](#recommended-fix-actions)
13. [Advanced Diagnostics](#advanced-diagnostics)
14. [Reference Commands](#reference-commands)

---

## Overview

OpenShift Dedicated (OSD) on GCP requires proper network configuration to ensure cluster components can communicate with each other, access GCP services, and connect to external resources. This guide provides systematic troubleshooting steps for common networking issues.

### Common Error Example

```
cluster is not reachable: Get "https://api.abc-osd-test.fyaz.p2.openshiftapps.com:6443/?timeout=5s": context deadline exceeded
```

This indicates the Kubernetes API server at port 6443 is not accessible due to networking issues.

---

## Prerequisites

Before troubleshooting, ensure you have:
- Google Cloud SDK (`gcloud`) installed and authenticated
- OpenShift CLI (`oc`) installed
- `kubectl` installed
- Appropriate GCP IAM permissions for VPC, firewall, and compute resources
- Access to GCP Console or gcloud CLI for network diagnostics
- Cluster credentials and kubeconfig file

### Verify Tool Installation

```bash
gcloud --version
oc version
kubectl version --client
```

---

## Quick Diagnostic Checklist

### 1. DNS Resolution

**Check if the API endpoint resolves:**

```bash
# Test DNS resolution
nslookup api.abc-osd-test.fyaz.p2.openshiftapps.com

# Alternative DNS test
dig api.abc-osd-test.fyaz.p2.openshiftapps.com

# Test with specific DNS server
dig @8.8.8.8 api.abc-osd-test.fyaz.p2.openshiftapps.com
```

**Expected:** Should resolve to an IP address (internal or external depending on your setup)

---

### 2. Network Connectivity

**Test basic connectivity:**

```bash
# Test if port 6443 is reachable
telnet api.abc-osd-test.fyaz.p2.openshiftapps.com 6443

# Or using curl with timeout
curl -v --max-time 10 https://api.abc-osd-test.fyaz.p2.openshiftapps.com:6443

# Test with verbose output
curl -v --connect-timeout 10 --max-time 30 https://api.abc-osd-test.fyaz.p2.openshiftapps.com:6443/healthz
```

---

### 3. GCP Firewall Rules

**Check firewall rules allow required traffic:**

```bash
# Set your project
gcloud config set project YOUR_PROJECT_ID

# List firewall rules for your VPC
gcloud compute firewall-rules list --filter="network:YOUR_VPC_NAME" --format="table(name,direction,sourceRanges,allowed)"

# Check specific rule details
gcloud compute firewall-rules describe RULE_NAME

# List all firewall rules
gcloud compute firewall-rules list --format="table(name,network,direction,priority,sourceRanges,allowed)"
```

**Required firewall rules for OpenShift:**
- **Port 6443**: API server (from installer/clients to control plane)
- **Port 22623**: Machine config server (from bootstrap/nodes to control plane)
- **Port 443/80**: Ingress (from external to worker nodes)
- **Port 10250**: Kubelet API (from control plane to nodes)

---

## DNS Resolution Testing

### Test Internal DNS from Pod

```bash
# Create a test pod
oc run dns-test --image=registry.access.redhat.com/ubi8/ubi --rm -it --restart=Never -- bash

# Inside the pod, test DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup api.abc-osd-test.fyaz.p2.openshiftapps.com
dig @172.30.0.10 kubernetes.default.svc.cluster.local
```

### Check CoreDNS Pods

```bash
# Check DNS pods status
oc get pods -n openshift-dns

# View DNS pod logs
oc logs -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default --tail=50

# Check DNS configuration
oc get configmap/dns-default -n openshift-dns -o yaml
```

### Test External DNS Resolution

```bash
# Test external DNS from pod
oc run dns-external-test --image=registry.access.redhat.com/ubi8/ubi --rm -it --restart=Never -- nslookup google.com
```

---

## Network Connectivity Testing

### Test Pod-to-Pod Communication

```bash
# Create test pods
oc run test-pod-1 --image=registry.access.redhat.com/ubi8/ubi -- sleep infinity
oc run test-pod-2 --image=registry.access.redhat.com/ubi8/ubi -- sleep infinity

# Wait for pods to be ready
oc wait --for=condition=Ready pod/test-pod-1 --timeout=60s
oc wait --for=condition=Ready pod/test-pod-2 --timeout=60s

# Get pod IPs
POD1_IP=$(oc get pod test-pod-1 -o jsonpath='{.status.podIP}')
POD2_IP=$(oc get pod test-pod-2 -o jsonpath='{.status.podIP}')

# Test connectivity
oc exec test-pod-1 -- ping -c 3 $POD2_IP

# Clean up
oc delete pod test-pod-1 test-pod-2
```

### Test External Connectivity

```bash
# Test outbound internet connectivity
oc run connectivity-test --image=registry.access.redhat.com/ubi8/ubi --rm -it --restart=Never -- curl -I --connect-timeout 10 https://www.redhat.com

# Test specific ports
oc run port-test --image=registry.access.redhat.com/ubi8/ubi --rm -it --restart=Never -- telnet google.com 443
```

---

## GCP Firewall Rules Configuration

### Essential Firewall Rules

#### Control Plane Access

```bash
# Allow API server access (6443)
gcloud compute firewall-rules create osd-allow-api \
    --network=YOUR_VPC_NAME \
    --allow=tcp:6443 \
    --source-ranges=YOUR_SOURCE_CIDR \
    --target-tags=osd-master \
    --description="Allow API server access"

# Allow machine config server (22623)
gcloud compute firewall-rules create osd-allow-machine-config \
    --network=YOUR_VPC_NAME \
    --allow=tcp:22623 \
    --source-ranges=YOUR_CLUSTER_CIDR \
    --target-tags=osd-master \
    --description="Allow machine config server"

# Allow kubelet API (10250)
gcloud compute firewall-rules create osd-allow-kubelet \
    --network=YOUR_VPC_NAME \
    --allow=tcp:10250 \
    --source-ranges=YOUR_CLUSTER_CIDR \
    --target-tags=osd-master,osd-worker \
    --description="Allow kubelet API"
```

#### Internal Cluster Communication

```bash
# Allow internal cluster traffic
gcloud compute firewall-rules create osd-internal \
    --network=YOUR_VPC_NAME \
    --allow=tcp,udp,icmp \
    --source-ranges=YOUR_CLUSTER_CIDR \
    --description="Allow internal cluster communication"

# Allow all internal traffic within VPC
gcloud compute firewall-rules create osd-internal-vpc \
    --network=YOUR_VPC_NAME \
    --allow=tcp,udp,icmp \
    --source-ranges=YOUR_VPC_CIDR \
    --description="Allow all VPC internal traffic"
```

#### Ingress Access

```bash
# Allow HTTP ingress (80)
gcloud compute firewall-rules create osd-allow-http \
    --network=YOUR_VPC_NAME \
    --allow=tcp:80 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=osd-worker \
    --description="Allow HTTP ingress"

# Allow HTTPS ingress (443)
gcloud compute firewall-rules create osd-allow-https \
    --network=YOUR_VPC_NAME \
    --allow=tcp:443 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=osd-worker \
    --description="Allow HTTPS ingress"
```

### Verify Firewall Rules

```bash
# Export current firewall rules
gcloud compute firewall-rules list \
    --filter="network:YOUR_VPC_NAME" \
    --format=json > current-firewall-rules.json

# Check for required ports
gcloud compute firewall-rules list \
    --filter="network:YOUR_VPC_NAME AND allowed.ports:(6443 OR 22623 OR 443 OR 80)" \
    --format="table(name,allowed,sourceRanges,targetTags)"
```

---

## VPC Network Configuration

### Verify VPC Configuration

**Verify VPC exists and is properly configured:**

```bash
# List VPCs
gcloud compute networks list

# Describe your VPC
gcloud compute networks describe YOUR_VPC_NAME

# List subnets
gcloud compute networks subnets list --network=YOUR_VPC_NAME

# Get detailed subnet information
gcloud compute networks subnets list --network=YOUR_VPC_NAME \
    --format="table(name,region,ipCidrRange,privateIpGoogleAccess)"
```

**Requirements:**
- VPC must have sufficient IP address space for cluster nodes
- Subnets for control plane and compute nodes
- Private Google Access enabled (for restricted networks)
- Secondary IP ranges for pods and services

### Check Subnet Configuration

```bash
# List all subnets with details
gcloud compute networks subnets list \
    --format="table(name,region,network,ipCidrRange,privateIpGoogleAccess,secondaryIpRanges)"

# Describe specific subnet
gcloud compute networks subnets describe SUBNET_NAME --region=REGION
```

### Verify Secondary IP Ranges

```bash
# Check secondary IP ranges (for pods and services)
gcloud compute networks subnets describe SUBNET_NAME --region=REGION \
    --format="get(secondaryIpRanges)"
```

---

## Routes and Cloud NAT

### For Restricted/Private Clusters

**Verify Cloud NAT:**

```bash
# List Cloud Routers
gcloud compute routers list

# Describe router configuration
gcloud compute routers describe YOUR_ROUTER_NAME --region=YOUR_REGION

# Describe NAT configuration
gcloud compute routers nats list --router=YOUR_ROUTER_NAME --region=YOUR_REGION

# Get NAT details
gcloud compute routers nats describe NAT_NAME \
    --router=YOUR_ROUTER_NAME \
    --region=YOUR_REGION
```

**Check routes:**

```bash
# List routes in your VPC
gcloud compute routes list --filter="network:YOUR_VPC_NAME"

# List routes with details
gcloud compute routes list --filter="network:YOUR_VPC_NAME" \
    --format="table(name,destRange,nextHopGateway,nextHopIp,priority)"
```

### Verify NAT Gateway Status

```bash
# Check NAT gateway metrics (if available)
gcloud logging read "resource.type=gce_nat_gateway" \
    --limit=10 \
    --format=json
```

---

## Load Balancer Health

### Check Load Balancers

**Check if load balancers are healthy:**

```bash
# List forwarding rules
gcloud compute forwarding-rules list --filter="name~osd"

# List all forwarding rules
gcloud compute forwarding-rules list \
    --format="table(name,region,IPAddress,IPProtocol,target)"

# Check backend services
gcloud compute backend-services list

# List backend services with details
gcloud compute backend-services list \
    --format="table(name,backends[].group,healthChecks[].name)"

# Check health checks
gcloud compute health-checks list

# Describe specific health check
gcloud compute health-checks describe HEALTH_CHECK_NAME
```

### Verify Backend Health

```bash
# Get backend service details
gcloud compute backend-services get-health BACKEND_SERVICE_NAME \
    --region=REGION

# Check target pools (for network load balancers)
gcloud compute target-pools list
gcloud compute target-pools get-health TARGET_POOL_NAME --region=REGION
```

---

## Common Issues and Solutions

### Issue 1: Firewall Blocking API Access

**Symptom:** Connection timeout to port 6443

**Solution:** Add firewall rule allowing TCP 6443 from your source network

```bash
# Create firewall rule for API access
gcloud compute firewall-rules create osd-allow-api-custom \
    --network=YOUR_VPC_NAME \
    --allow=tcp:6443 \
    --source-ranges=YOUR_SOURCE_IP/32 \
    --target-tags=osd-master \
    --description="Allow API server from specific IP"
```

### Issue 2: DNS Not Resolving

**Symptom:** nslookup fails or returns no address

**Solution:**
- Verify Cloud DNS zone is created
- Check DNS records for api.* endpoint
- Ensure DNS is accessible from installer machine

```bash
# Check Cloud DNS zones
gcloud dns managed-zones list

# List DNS records
gcloud dns record-sets list --zone=YOUR_DNS_ZONE

# Check specific record
gcloud dns record-sets describe api.YOUR_DOMAIN \
    --zone=YOUR_DNS_ZONE \
    --type=A
```

### Issue 3: Private Cluster Without NAT

**Symptom:** Nodes can't reach external registries

**Solution:** Configure Cloud NAT for outbound internet access

```bash
# Create Cloud NAT
gcloud compute routers nats create NAT_NAME \
    --router=YOUR_ROUTER_NAME \
    --region=YOUR_REGION \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

### Issue 4: Incorrect Source Ranges

**Symptom:** Firewall rules exist but connection still fails

**Solution:** Verify source-ranges include the CIDR where you're installing from

```bash
# Check current firewall rule source ranges
gcloud compute firewall-rules describe RULE_NAME \
    --format="get(sourceRanges)"

# Update firewall rule with correct source range
gcloud compute firewall-rules update RULE_NAME \
    --source-ranges=YOUR_CORRECT_CIDR
```

### Issue 5: Load Balancer Not Healthy

**Symptom:** Services unreachable, health checks failing

**Solution:**
```bash
# Check backend health
gcloud compute backend-services get-health BACKEND_SERVICE_NAME --region=REGION

# Check health check configuration
gcloud compute health-checks describe HEALTH_CHECK_NAME

# Verify target instances are healthy
gcloud compute instances list --filter="tags.items~osd-worker"
```

---

## Step-by-Step Troubleshooting Process

### Step 1: Verify Basic Infrastructure

```bash
# Set your project
gcloud config set project YOUR_PROJECT_ID

# Verify VPC exists
gcloud compute networks describe YOUR_VPC_NAME

# Check if subnets exist with proper CIDR ranges
gcloud compute networks subnets list --network=YOUR_VPC_NAME \
    --format="table(name,region,ipCidrRange,privateIpGoogleAccess)"

# Verify project and region
gcloud config list
```

### Step 2: Validate Firewall Configuration

```bash
# Export current firewall rules
gcloud compute firewall-rules list \
    --filter="network:YOUR_VPC_NAME" \
    --format=json > current-firewall-rules.json

# Check for required ports
gcloud compute firewall-rules list \
    --filter="network:YOUR_VPC_NAME AND allowed.ports:(6443 OR 22623)" \
    --format="table(name,allowed,sourceRanges,targetTags)"

# Verify firewall rules are active
gcloud compute firewall-rules list \
    --filter="network:YOUR_VPC_NAME" \
    --format="table(name,direction,priority,sourceRanges,allowed,targetTags)"
```

### Step 3: Test Connectivity

```bash
# From installer machine or bastion host
curl -v --max-time 10 https://api.abc-osd-test.fyaz.p2.openshiftapps.com:6443

# Check if you can reach the expected load balancer IP
LOAD_BALANCER_IP=$(dig +short api.abc-osd-test.fyaz.p2.openshiftapps.com)
ping -c 4 $LOAD_BALANCER_IP

# Test API health endpoint
curl -k https://api.abc-osd-test.fyaz.p2.openshiftapps.com:6443/healthz
```

### Step 4: Review Installation Logs

```bash
# Check installer logs for more details
cat .openshift_install.log | grep -i "error\|fail\|timeout"

# Look for specific network-related errors
cat .openshift_install.log | grep -i "network\|firewall\|connection"

# Check for DNS-related errors
cat .openshift_install.log | grep -i "dns\|resolve"
```

### Step 5: Verify Cluster Status

```bash
# Check cluster operators
oc get clusteroperators

# Check node status
oc get nodes -o wide

# Check network operator
oc get clusteroperator network
oc describe clusteroperator network
```

---

## Recommended Fix Actions

### Before Re-installing

1. **Delete existing partial installation:**
   ```bash
   openshift-install destroy cluster --dir=YOUR_INSTALL_DIR
   ```

2. **Verify all required firewall rules:**
   ```bash
   # Use the firewall rule commands from Section 3 above
   ```

3. **Ensure installer machine has access:**
   - Installer must be able to reach api.* endpoint on port 6443
   - Consider using a bastion host in the same VPC
   - Or add your external IP to firewall source ranges

4. **Validate install-config.yaml:**
   ```yaml
   # Ensure networking section is correct
   networking:
     networkType: OVNKubernetes
     clusterNetwork:
     - cidr: 10.128.0.0/14
       hostPrefix: 23
     machineNetwork:
     - cidr: 10.0.0.0/16  # Should match your VPC subnet
     serviceNetwork:
     - 172.30.0.0/16
   ```

5. **Re-run installation:**
   ```bash
   openshift-install create cluster --dir=YOUR_INSTALL_DIR --log-level=debug
   ```

---

## Advanced Diagnostics

### Network Flow Analysis

```bash
# Enable VPC flow logs (if not already enabled)
gcloud compute networks update YOUR_VPC_NAME \
    --enable-flow-logs \
    --flow-log-sampling=0.5 \
    --flow-log-metadata=INCLUDE_ALL_METADATA

# Query flow logs
gcloud logging read "resource.type=gce_subnetwork" \
    --limit=50 \
    --format=json
```

### Performance Monitoring

```bash
# Check instance metrics
gcloud monitoring time-series list \
    --filter='metric.type="compute.googleapis.com/instance/network/received_bytes_count"' \
    --limit=10

# Check load balancer metrics
gcloud monitoring time-series list \
    --filter='metric.type="loadbalancing.googleapis.com/https/request_count"' \
    --limit=10
```

### Collect Diagnostic Information

```bash
# Collect must-gather
oc adm must-gather --dest-dir=/tmp/must-gather

# Export cluster configuration
oc get network.config.openshift.io cluster -o yaml > network-config.yaml
oc get clusteroperators -o yaml > clusteroperators.yaml

# Export firewall rules
gcloud compute firewall-rules list \
    --filter="network:YOUR_VPC_NAME" \
    --format=json > firewall-rules.json
```

---

## Reference Commands

### Quick Health Check

```bash
# Check project and region
gcloud config list

# List all compute resources
gcloud compute instances list
gcloud compute forwarding-rules list
gcloud compute firewall-rules list

# Check cluster status
oc get nodes
oc get clusteroperators

# Monitor installation
openshift-install wait-for install-complete --dir=YOUR_INSTALL_DIR --log-level=debug
```

### Network Diagnostic Commands

```bash
# Test DNS resolution
nslookup api.YOUR_CLUSTER_DOMAIN
dig api.YOUR_CLUSTER_DOMAIN

# Test connectivity
curl -v --connect-timeout 10 https://api.YOUR_CLUSTER_DOMAIN:6443/healthz

# Check firewall rules
gcloud compute firewall-rules list --filter="network:YOUR_VPC_NAME"

# Check routes
gcloud compute routes list --filter="network:YOUR_VPC_NAME"

# Check load balancers
gcloud compute forwarding-rules list
gcloud compute backend-services list
```

### Cluster Diagnostic Commands

```bash
# Check cluster operators
oc get clusteroperators | grep -v "True.*False.*False"

# Check network operator
oc get clusteroperator network
oc describe clusteroperator network

# Check DNS
oc get pods -n openshift-dns
oc logs -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default

# Check ingress
oc get pods -n openshift-ingress
oc get routes --all-namespaces
```

---

## Additional Resources

- [GCP VPC Requirements Documentation](https://docs.openshift.com/container-platform/latest/installing/installing_gcp/installing-restricted-networks-gcp-installer-provisioned.html)
- [GCP Firewall Rules Documentation](https://cloud.google.com/vpc/docs/firewalls)
- [OpenShift Install Logs](https://docs.openshift.com/container-platform/latest/installing/installing_gcp/installing-gcp-installer-customizations.html)
- [GCP Cloud NAT Documentation](https://cloud.google.com/nat/docs/overview)

---

## Conclusion

This guide provides a comprehensive approach to troubleshooting networking issues in OpenShift Dedicated (OSD) clusters on GCP. Follow the steps systematically, starting with basic connectivity tests and progressing to more advanced diagnostics as needed. Always verify firewall rules, VPC configuration, and load balancer health before attempting cluster installation or troubleshooting connectivity issues.

