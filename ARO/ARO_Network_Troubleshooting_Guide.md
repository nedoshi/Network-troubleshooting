# Azure Red Hat OpenShift (ARO) Network Troubleshooting Guide

Complete guide for diagnosing and fixing networking and DNS issues in private ARO clusters.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Verify Cluster Status and Basic Connectivity](#1-verify-cluster-status-and-basic-connectivity)
4. [DNS Resolution Testing](#2-dns-resolution-testing)
5. [Network Security Group (NSG) Rules Validation](#3-network-security-group-nsg-rules-validation)
6. [Virtual Network and Subnet Configuration](#4-virtual-network-and-subnet-configuration)
7. [Service Endpoints and Private Link](#5-service-endpoints-and-private-link)
8. [ARO Private Cluster API Access](#6-aro-private-cluster-api-access)
9. [CoreDNS and Cluster DNS Issues](#7-coredns-and-cluster-dns-issues)
10. [SDN/OVN Network Plugin Issues](#8-sdnovn-network-plugin-issues)
11. [Pod-to-Pod and Pod-to-Service Connectivity](#9-pod-to-pod-and-pod-to-service-connectivity)
12. [Ingress and Router Issues](#10-ingress-and-router-issues)
13. [Azure Load Balancer Configuration](#11-azure-load-balancer-configuration)
14. [Azure Private DNS Zone Configuration](#12-azure-private-dns-zone-configuration)
15. [Check Cluster Network Configuration](#13-check-cluster-network-configuration)
16. [Container Registry Access](#16-container-registry-access)
17. [Diagnostic Collection](#14-diagnostic-collection)
18. [Common Issues Quick Reference](#common-issues-quick-reference)
19. [Quick Diagnostic Script](#quick-diagnostic-script)
20. [Advanced Troubleshooting](#advanced-troubleshooting)
21. [Prevention and Best Practices](#prevention-and-best-practices)

---

## Overview

Azure Red Hat OpenShift (ARO) private clusters require careful network configuration to ensure proper connectivity between cluster components, Azure services, and external resources. This guide provides systematic troubleshooting steps for common networking issues in private ARO deployments.

### Key Network Components

- **Private Subnets**: Master and worker nodes in private subnets
- **Network Security Groups (NSGs)**: Control inbound/outbound traffic
- **Service Endpoints**: Direct connectivity to Azure services
- **Private DNS Zones**: Internal DNS resolution
- **Load Balancers**: Internal and external load balancing
- **Private Link**: Secure connection to Azure services

---

## Prerequisites

Before troubleshooting, ensure you have:
- Azure CLI installed and authenticated (`az login`)
- OpenShift CLI (`oc`) installed
- kubectl installed
- Appropriate RBAC permissions for ARO cluster and Azure resources
- Access to Azure portal or Azure CLI for network diagnostics
- Access to cluster via VPN, Bastion, or Jumpbox (for private clusters)

---

## 1. Verify Cluster Status and Basic Connectivity

### Check Cluster Status

```bash
az aro show --name <cluster-name> --resource-group <rg-name>
```

### What to Look For
- `provisioningState`: Should be "Succeeded"
- `clusterProfile.domain`: Your cluster domain
- `networkProfile`: Verify CIDR ranges don't conflict
- `apiserverProfile`: Check API server visibility and URL

### Check OpenShift Status

```bash
oc get nodes
oc get clusterversion
```

### Fix if Cluster is Not Healthy

```bash
# Check cluster operators
oc get co

# Get detailed status
oc describe clusteroperator <operator-name>

# Check for degraded operators
oc get co | grep -v "True.*False.*False"
```

### Check API Server Visibility

```bash
# Verify API server endpoint
az aro show --name <cluster-name> --resource-group <rg-name> --query apiserverProfile

# Test connectivity to API server
curl -k https://<api-server-url>:6443/healthz
```

---

## 2. DNS Resolution Testing

### Test Internal DNS from Local Machine

```bash
nslookup api.<cluster-name>.<random-string>.<region>.aroapp.io
```

### Test DNS from Within a Pod

```bash
# Create debug pod
oc run -it --rm debug --image=registry.access.redhat.com/ubi8/ubi -- bash

# Inside the pod:
nslookup kubernetes.default.svc.cluster.local
nslookup openshift.default.svc.cluster.local
dig @10.0.0.10 kubernetes.default.svc.cluster.local
```

### What to Look For
- API endpoint should resolve to internal IP (10.x.x.x range)
- Internal services should resolve to ClusterIP addresses
- No NXDOMAIN or SERVFAIL errors

### Fix DNS Issues

```bash
# Check DNS operator
oc get pods -n openshift-dns
oc logs -n openshift-dns dns-operator-<pod-id>

# Check DNS configuration
oc get dns.operator/default -o yaml

# Restart DNS pods if needed
oc delete pod -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default

# Check DNS forwarding configuration
oc get configmap/dns-default -n openshift-dns -o yaml

# Verify DNS ConfigMap
oc describe configmap dns-default -n openshift-dns
```

### Validate Private DNS Resolution

```bash
# Check if private DNS zone exists
az network private-dns zone list --resource-group <rg-name>

# Verify DNS records
az network private-dns record-set a list --zone-name <private-dns-zone> --resource-group <rg-name>
```

---

## 3. Network Security Group (NSG) Rules Validation

### Check NSG Rules

```bash
# Get NSG associated with ARO
az network nsg list --resource-group <aro-rg> -o table

# Show NSG rules
az network nsg rule list --nsg-name <nsg-name> --resource-group <aro-rg> -o table

# List NSG rules for master subnet
az network nsg rule list --nsg-name <master-nsg-name> --resource-group <rg-name> --output table
```

### Required NSG Rules

**Inbound Rules:**
- Port 6443 (API server) - from your access location
- Port 443 (Ingress) - from your access location
- Port 80 (HTTP Ingress) - optional
- Port 22 (SSH) - for management (if needed)

**Outbound Rules:**
- Port 443 to Azure services
- Port 445 (Azure Files)
- Ports 1194, 9000-9999 for Azure services
- Port 53 (DNS) for name resolution

### Fix NSG Issues

```bash
# Add missing API access rule
az network nsg rule create \
  --resource-group <aro-rg> \
  --nsg-name <nsg-name> \
  --name AllowAPIServer \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 6443 \
  --source-address-prefixes <your-ip>/32

# Add ingress access rule
az network nsg rule create \
  --resource-group <aro-rg> \
  --nsg-name <nsg-name> \
  --name AllowHTTPS \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --source-address-prefixes <your-ip>/32

# Add outbound rule for Azure services
az network nsg rule create \
  --resource-group <aro-rg> \
  --nsg-name <nsg-name> \
  --name AllowAzureServicesOut \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --destination-address-prefixes AzureCloud
```

### Verify Egress Rules

```bash
# Check worker node NSG rules
az network nsg rule list --nsg-name <worker-nsg-name> --resource-group <rg-name>

# Ensure required outbound rules exist:
# - Port 443 (HTTPS) for container registries
# - Port 80 (HTTP) for package repositories
# - DNS ports (53) for name resolution
```

---

## 4. Virtual Network and Subnet Configuration

### Verify VNet Configuration

```bash
# Check VNet
az network vnet show --name <vnet-name> --resource-group <vnet-rg>

# Check master subnet
az network vnet subnet show \
  --vnet-name <vnet-name> \
  --name <master-subnet> \
  --resource-group <vnet-rg>

# Check worker subnet
az network vnet subnet show \
  --vnet-name <vnet-name> \
  --name <worker-subnet> \
  --resource-group <vnet-rg>
```

### What to Look For
- Master subnet: Minimum /27 (32 addresses)
- Worker subnet: Minimum /24 (256 addresses)
- `privateEndpointNetworkPolicies`: Should be "Disabled"
- `privateLinkServiceNetworkPolicies`: Should be "Disabled"
- No overlapping CIDR ranges with other networks

### Fix Subnet Issues

```bash
# Disable network policies on master subnet
az network vnet subnet update \
  --name <master-subnet> \
  --vnet-name <vnet-name> \
  --resource-group <vnet-rg> \
  --disable-private-endpoint-network-policies true \
  --disable-private-link-service-network-policies true

# Disable network policies on worker subnet
az network vnet subnet update \
  --name <worker-subnet> \
  --vnet-name <vnet-name> \
  --resource-group <vnet-rg> \
  --disable-private-endpoint-network-policies true \
  --disable-private-link-service-network-policies true
```

### Check Route Tables

```bash
# Verify route tables for subnets
az network route-table list --resource-group <rg-name>

# Check routes for worker subnet
az network route-table route list --route-table-name <route-table-name> --resource-group <rg-name>
```

---

## 5. Service Endpoints and Private Link

### Check Service Endpoints

```bash
# Check master subnet service endpoints
az network vnet subnet show \
  --vnet-name <vnet-name> \
  --name <master-subnet> \
  --resource-group <vnet-rg> \
  --query serviceEndpoints

# Check worker subnet service endpoints
az network vnet subnet show \
  --vnet-name <vnet-name> \
  --name <worker-subnet> \
  --resource-group <vnet-rg> \
  --query serviceEndpoints
```

### What to Look For
Required service endpoints:
- Microsoft.ContainerRegistry
- Microsoft.Storage

### Fix Service Endpoint Issues

```bash
# Add service endpoints to master subnet
az network vnet subnet update \
  --name <master-subnet> \
  --vnet-name <vnet-name> \
  --resource-group <vnet-rg> \
  --service-endpoints Microsoft.ContainerRegistry Microsoft.Storage

# Add service endpoints to worker subnet
az network vnet subnet update \
  --name <worker-subnet> \
  --vnet-name <vnet-name> \
  --resource-group <vnet-rg> \
  --service-endpoints Microsoft.ContainerRegistry Microsoft.Storage
```

### Check Private Endpoints

```bash
# Verify private endpoints for ACR if using private link
az network private-endpoint list --resource-group <rg-name>
```

---

## 6. ARO Private Cluster API Access

### Test API Connectivity

```bash
# Test API health endpoint (from within VNet)
curl -k https://api.<cluster-name>.<random-string>.<region>.aroapp.io:6443/healthz

# Try to authenticate
oc login https://api.<cluster-name>.<random-string>.<region>.aroapp.io:6443 -u kubeadmin
```

### What to Look For
- Health endpoint should return "ok"
- Should be able to authenticate successfully
- Connection should not timeout

### Fix API Access Issues

#### Option 1: Use Azure Bastion or Jumpbox VM

```bash
# Create a jumpbox VM in the same VNet
az vm create \
  --resource-group <rg-name> \
  --name jumpbox \
  --image UbuntuLTS \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --admin-username azureuser \
  --generate-ssh-keys

# SSH into jumpbox and install oc CLI
ssh azureuser@<jumpbox-ip>
```

#### Option 2: Set Up Azure Private DNS Zone

```bash
# Create private DNS zone
az network private-dns zone create \
  --resource-group <rg-name> \
  --name <cluster-domain>

# Link to VNet
az network private-dns link vnet create \
  --resource-group <rg-name> \
  --zone-name <cluster-domain> \
  --name arolink \
  --virtual-network <vnet-name> \
  --registration-enabled false

# Add A record for API
az network private-dns record-set a add-record \
  --resource-group <rg-name> \
  --zone-name <cluster-domain> \
  --record-set-name api \
  --ipv4-address <internal-lb-ip>
```

#### Option 3: Configure VPN or ExpressRoute

Follow Azure documentation to establish VPN/ExpressRoute connectivity to your VNet.

---

## 7. CoreDNS and Cluster DNS Issues

### Check CoreDNS Pods

```bash
# List CoreDNS pods
oc get pods -n openshift-dns -o wide

# Check pod details
oc describe pod -n openshift-dns dns-default-xxxxx

# View logs
oc logs -n openshift-dns dns-default-xxxxx
```

### What to Look For
- All CoreDNS pods should be in Running state
- No CrashLoopBackOff or Error states
- Check logs for resolution failures or errors

### Fix CoreDNS Issues

```bash
# Check DNS operator status
oc get clusteroperator dns
oc describe clusteroperator dns

# View DNS operator logs
oc logs -n openshift-dns-operator deployment/dns-operator

# Restart DNS operator if needed
oc delete pod -n openshift-dns-operator -l name=dns-operator

# Check DNS cluster configuration
oc get dns.operator cluster -o yaml

# Verify DNS ConfigMap
oc get configmap dns-default -n openshift-dns -o yaml

# Restart CoreDNS pods
oc delete pod -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default
```

### Configure Custom DNS Forwarders (if needed)

```bash
# Edit DNS operator configuration
oc edit dns.operator/default

# Add upstream servers section:
# spec:
#   servers:
#   - name: custom-dns
#     zones:
#       - example.com
#     forwardPlugin:
#       upstreams:
#         - 8.8.8.8
#         - 8.8.4.4
```

### Test DNS Resolution

```bash
# Create test pod for network diagnostics
oc run network-test --image=busybox --rm -it -- sh

# Inside the pod, test DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup google.com
```

---

## 8. SDN/OVN Network Plugin Issues

### Check Network Operator

```bash
# Check network operator status
oc get clusteroperator network
oc describe clusteroperator network

# Check which network type is in use
oc get network.config.openshift.io cluster -o yaml | grep networkType
```

### Check Network Plugin Pods

```bash
# For OpenShift SDN
oc get pods -n openshift-sdn
oc logs -n openshift-sdn sdn-xxxxx

# For OVN-Kubernetes
oc get pods -n openshift-ovn-kubernetes
oc logs -n openshift-ovn-kubernetes ovnkube-node-xxxxx -c ovnkube-node
oc logs -n openshift-ovn-kubernetes ovnkube-master-xxxxx -c ovnkube-master
```

### What to Look For
- Network operator should show Available=True
- All network pods should be Running
- No errors in logs related to connectivity

### Fix Network Plugin Issues

```bash
# Check network configuration
oc get network.config.openshift.io cluster -o yaml

# For OpenShift SDN issues
oc get pods -n openshift-sdn
oc delete pod -n openshift-sdn -l app=sdn

# For OVN-Kubernetes issues
oc get pods -n openshift-ovn-kubernetes
oc delete pod -n openshift-ovn-kubernetes -l app=ovnkube-node
oc delete pod -n openshift-ovn-kubernetes -l app=ovnkube-master

# Check network operator logs
oc logs -n openshift-network-operator deployment/network-operator
```

### Examine CNI Configuration

```bash
# Check CNI configuration
oc get pods -n openshift-multus
```

---

## 9. Pod-to-Pod and Pod-to-Service Connectivity

### Create Test Pods

```bash
# Create test pod in default namespace
oc run test-pod-1 --image=registry.access.redhat.com/ubi8/ubi -- sleep infinity

# Create test pod in different namespace
oc run test-pod-2 --image=registry.access.redhat.com/ubi8/ubi -- sleep infinity -n kube-system

# Wait for pods to be ready
oc wait --for=condition=ready pod/test-pod-1 --timeout=60s
oc wait --for=condition=ready pod/test-pod-2 -n kube-system --timeout=60s

# Get pod IPs
oc get pods -o wide
oc get pods -n kube-system -o wide
```

### Test Connectivity

```bash
# Get the IP of test-pod-2
POD2_IP=$(oc get pod test-pod-2 -n kube-system -o jsonpath='{.status.podIP}')

# Test pod-to-pod connectivity
oc exec test-pod-1 -- ping -c 3 $POD2_IP

# Test service connectivity
oc exec test-pod-1 -- curl -k https://kubernetes.default.svc.cluster.local:443

# Test DNS resolution
oc exec test-pod-1 -- nslookup kubernetes.default.svc.cluster.local
oc exec test-pod-1 -- nslookup openshift.default.svc.cluster.local
```

### What to Look For
- Pods should be able to ping each other by IP
- Services should be reachable by ClusterIP
- DNS should resolve service names correctly

### Fix Pod Connectivity Issues

```bash
# Check network policies
oc get networkpolicy --all-namespaces

# Describe network policies to see rules
oc describe networkpolicy <policy-name> -n <namespace>

# Temporarily remove network policies for testing
oc delete networkpolicy <policy-name> -n <namespace>

# Check node networking using debug pod
oc debug node/<node-name>

# Inside debug pod:
chroot /host
ip addr
ip route
iptables -L -n -v

# Check OVS bridges (for SDN)
ovs-vsctl show

# Exit debug pod
exit
exit
```

### Test Inter-Node Connectivity

```bash
# Create test pods on different nodes
oc run test-pod-1 --image=busybox --command -- sleep 3600
oc run test-pod-2 --image=busybox --command -- sleep 3600

# Test connectivity between pods
oc exec test-pod-1 -- ping <test-pod-2-ip>
```

### Clean Up Test Pods

```bash
oc delete pod test-pod-1
oc delete pod test-pod-2 -n kube-system
```

---

## 10. Ingress and Router Issues

### Check Ingress Operator

```bash
# Check ingress operator status
oc get clusteroperator ingress
oc describe clusteroperator ingress

# Check router pods
oc get pods -n openshift-ingress

# View ingress controller configuration
oc get ingresscontroller default -n openshift-ingress-operator -o yaml
```

### What to Look For
- Ingress operator should show Available=True
- Router pods should be in Running state
- Load balancer should be provisioned
- Check for any error conditions

### Fix Ingress Issues

```bash
# Check router logs
oc logs -n openshift-ingress router-default-xxxxx

# Check ingress operator logs
oc logs -n openshift-ingress-operator deployment/ingress-operator

# Check router service
oc get svc -n openshift-ingress
oc describe svc router-default -n openshift-ingress

# Restart router pods
oc delete pod -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default

# Check for admission issues
oc get events -n openshift-ingress --sort-by='.lastTimestamp'

# Verify ingress controller
oc get ingresscontroller -n openshift-ingress-operator
oc describe ingresscontroller default -n openshift-ingress-operator
```

### Test Route Connectivity

```bash
# Create a test application
oc new-app --name test-app --image=registry.access.redhat.com/ubi8/ubi -- sleep infinity

# Expose the service
oc expose svc/test-app

# Get the route
oc get route test-app

# Test the route (from within VNet for private clusters)
curl -I http://$(oc get route test-app -o jsonpath='{.spec.host}')

# Clean up
oc delete all -l app=test-app
```

### Check Service Endpoints

```bash
# Check service endpoints
oc get endpoints -n <namespace>

# Verify service configuration
oc describe service <service-name> -n <namespace>
```

---

## 11. Azure Load Balancer Configuration

### Check Load Balancers

```bash
# Find infrastructure resource group
INFRA_RG=$(az aro show --name <cluster-name> --resource-group <rg-name> --query 'clusterProfile.resourceGroupId' -o tsv | cut -d'/' -f5)

# List load balancers
az network lb list --resource-group $INFRA_RG -o table

# Get load balancer details
az network lb show --name <lb-name> --resource-group $INFRA_RG

# Check backend pools
az network lb address-pool list --lb-name <lb-name> --resource-group $INFRA_RG

# Check health probes
az network lb probe list --lb-name <lb-name> --resource-group $INFRA_RG -o table

# Check load balancing rules
az network lb rule list --lb-name <lb-name> --resource-group $INFRA_RG -o table
```

### What to Look For
- Internal load balancer for API (ports 6443, 22623)
- Public or internal load balancer for ingress (ports 80, 443)
- Backend pools should contain master/worker nodes
- Health probes should be configured and healthy

### Verify Node Backend Pool Membership

```bash
# Get NICs attached to VMs
az network nic list --resource-group $INFRA_RG -o table

# Check if specific NIC is in backend pool
az network nic show --ids <nic-id> --query "ipConfigurations[].loadBalancerBackendAddressPools"
```

### Check from OpenShift Side

```bash
# Check router service (creates load balancer)
oc get svc -n openshift-ingress
oc describe svc router-default -n openshift-ingress

# Check service annotations
oc get svc router-default -n openshift-ingress -o yaml | grep -A 5 annotations
```

### Examine Load Balancer Configuration

```bash
# Check Azure Load Balancer resources
az network lb list --resource-group <node-rg-name>

# Verify load balancer rules and health probes
az network lb rule list --lb-name <lb-name> --resource-group <node-rg-name>
```

---

## 12. Azure Private DNS Zone Configuration

### Check Private DNS Zones

```bash
# List private DNS zones
az network private-dns zone list --resource-group <rg-name> -o table

# Show specific zone
az network private-dns zone show \
  --name <cluster-domain>.aroapp.io \
  --resource-group <rg-name>

# Check DNS records
az network private-dns record-set list \
  --zone-name <cluster-domain>.aroapp.io \
  --resource-group <rg-name> -o table

# Show A records
az network private-dns record-set a list \
  --zone-name <cluster-domain>.aroapp.io \
  --resource-group <rg-name>

# Check VNet links
az network private-dns link vnet list \
  --zone-name <cluster-domain>.aroapp.io \
  --resource-group <rg-name> -o table
```

### What to Look For
- Private DNS zone exists for cluster domain
- A records for "api" and "*.apps" subdomains
- VNet link exists with auto-registration disabled
- Records point to correct internal load balancer IPs

### Fix DNS Zone Issues

#### Create Missing DNS Zone

```bash
# Create private DNS zone
az network private-dns zone create \
  --resource-group <rg-name> \
  --name <cluster-domain>.aroapp.io
```

#### Link to VNet

```bash
# Create VNet link
az network private-dns link vnet create \
  --resource-group <rg-name> \
  --zone-name <cluster-domain>.aroapp.io \
  --name aro-vnet-link \
  --virtual-network <vnet-id> \
  --registration-enabled false
```

#### Create Missing A Records

```bash
# Get internal load balancer IP
INTERNAL_LB_IP=$(az network lb frontend-ip list \
  --lb-name <internal-lb-name> \
  --resource-group $INFRA_RG \
  --query "[0].privateIPAddress" -o tsv)

# Create A record for API
az network private-dns record-set a add-record \
  --resource-group <rg-name> \
  --zone-name <cluster-domain>.aroapp.io \
  --record-set-name api \
  --ipv4-address $INTERNAL_LB_IP

# Create A record for apps wildcard
az network private-dns record-set a add-record \
  --resource-group <rg-name> \
  --zone-name <cluster-domain>.aroapp.io \
  --record-set-name "*.apps" \
  --ipv4-address $INTERNAL_LB_IP
```

---

## 13. Check Cluster Network Configuration

### Review Cluster Network Settings

```bash
# Check network configuration
oc get network.config.openshift.io cluster -o yaml

# For OpenShift SDN
oc get clusternetwork -o yaml

# Check network type
oc get network.config.openshift.io cluster -o jsonpath='{.spec.networkType}'
```

### What to Look For

Network configuration should include:
- `serviceNetwork`: Typically 172.30.0.0/16
- `clusterNetwork`: Pod network CIDR (e.g., 10.128.0.0/14)
- `networkType`: Either "OpenShiftSDN" or "OVNKubernetes"
- **No CIDR overlaps** with Azure VNet CIDR

### Check for CIDR Conflicts

```bash
# Get Azure VNet CIDR
az network vnet show --name <vnet-name> --resource-group <vnet-rg> --query addressSpace.addressPrefixes

# Get OpenShift network CIDRs
oc get network.config.openshift.io cluster -o yaml | grep -A 10 "clusterNetwork\|serviceNetwork"
```

### Important Note on CIDR Conflicts

⚠️ **WARNING**: You cannot change network CIDRs after cluster creation!

If there are CIDR conflicts, you must:
1. Backup all applications and data
2. Delete the existing cluster
3. Recreate the cluster with correct, non-overlapping CIDRs
4. Restore applications and data

**Before creating a new ARO cluster**, ensure:
- Pod CIDR doesn't overlap with VNet CIDR
- Service CIDR doesn't overlap with VNet CIDR
- Pod CIDR doesn't overlap with Service CIDR

---

## 16. Container Registry Access

### Symptoms
- Image pull failures
- Authentication errors to Azure Container Registry
- Timeout errors during image pulls

### Verify Registry Configuration

```bash
# Check image pull secrets
oc get secrets --all-namespaces | grep pull-secret

# Verify registry configuration
oc get image.config.openshift.io/cluster -o yaml
```

### Test Registry Connectivity

```bash
# Create test pod to verify registry access
oc run registry-test --image=busybox --rm -it -- sh

# Test connection to Azure Container Registry
wget -q --spider https://<registry-name>.azurecr.io
```

### Check Service Endpoints for Container Registry

```bash
# Verify private endpoints for ACR if using private link
az network private-endpoint list --resource-group <rg-name>
```

---

## 14. Diagnostic Collection

### Collect Must-Gather Data

```bash
# Standard must-gather
oc adm must-gather --dest-dir=/tmp/must-gather

# Network-specific must-gather
oc adm must-gather \
  --image=quay.io/openshift/origin-must-gather \
  --dest-dir=/tmp/network-gather

# Inspect must-gather data
oc adm inspect clusteroperator/network --dest-dir=/tmp/network-inspect
```

### Check Cluster Health

```bash
# Check node status
oc get nodes
oc adm top nodes

# Check cluster operators
oc get clusteroperators

# Check recent events
oc get events --all-namespaces --sort-by='.lastTimestamp' | tail -50

# Check pod status across namespaces
oc get pods --all-namespaces | grep -v Running | grep -v Completed
```

### Check Resource Usage

```bash
# Node resource usage
oc adm top nodes

# Pod resource usage
oc adm top pods --all-namespaces

# Check for resource pressure
oc describe nodes | grep -A 5 "Conditions:"
```

### Export Cluster Information

```bash
# Export cluster version
oc get clusterversion -o yaml > clusterversion.yaml

# Export network configuration
oc get network.config.openshift.io cluster -o yaml > network-config.yaml

# Export DNS configuration
oc get dns.operator default -o yaml > dns-config.yaml

# Export all cluster operators
oc get clusteroperators -o yaml > clusteroperators.yaml
```

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

---

## Common Issues Quick Reference

### Issue 1: Cannot Access Cluster API

**Symptoms:**
- Connection timeout when trying to access API
- Unable to run `oc` commands

**Quick Diagnosis:**
```bash
# Test API endpoint
curl -k https://api.<cluster-name>.<random-string>.<region>.aroapp.io:6443/healthz
```

**Fixes:**
1. Ensure you're connected to the VNet (VPN/Bastion/Jumpbox)
2. Check NSG allows port 6443 from your source IP
3. Verify private DNS resolution is working
4. Confirm you have the correct kubeconfig

### Issue 2: Pods Can't Resolve External DNS

**Symptoms:**
- Pods cannot reach external websites
- DNS resolution failures in pod logs
- `nslookup` or `dig` commands fail in pods

**Quick Diagnosis:**
```bash
oc run dns-test --image=registry.access.redhat.com/ubi8/ubi --rm -it --restart=Never -- nslookup google.com
```

**Fixes:**
1. Check CoreDNS pods are running: `oc get pods -n openshift-dns`
2. Verify DNS forwarding configuration
3. Ensure Azure DNS (168.63.129.16) is reachable from VNet
4. Restart CoreDNS pods if needed

### Issue 3: Inter-Pod Communication Fails

**Symptoms:**
- Pods cannot communicate with each other
- Services not reachable from pods
- Application connectivity issues

**Quick Diagnosis:**
```bash
# Create two test pods and try to ping
oc run pod1 --image=registry.access.redhat.com/ubi8/ubi -- sleep infinity
oc run pod2 --image=registry.access.redhat.com/ubi8/ubi -- sleep infinity
POD2_IP=$(oc get pod pod2 -o jsonpath='{.status.podIP}')
oc exec pod1 -- ping -c 3 $POD2_IP
```

**Fixes:**
1. Check SDN/OVN pods are healthy
2. Review and temporarily remove NetworkPolicies for testing
3. Verify node-to-node connectivity
4. Check NSG rules allow inter-subnet communication

### Issue 4: Ingress/Routes Not Working

**Symptoms:**
- Cannot access applications via routes
- Router pods not starting
- 503 errors when accessing routes

**Quick Diagnosis:**
```bash
oc get pods -n openshift-ingress
oc get ingresscontroller default -n openshift-ingress-operator
oc get svc -n openshift-ingress
```

**Fixes:**
1. Check router pods status: `oc get pods -n openshift-ingress`
2. Verify load balancer is provisioned in Azure
3. Check NSG allows ports 80/443
4. Restart router pods if needed
5. Verify routes are created: `oc get routes --all-namespaces`

### Issue 5: Cluster Operators Degraded

**Symptoms:**
- `oc get co` shows degraded or unavailable operators
- Cluster functionality impaired

**Quick Diagnosis:**
```bash
oc get clusteroperators | grep -v "True.*False.*False"
```

**Fixes:**
1. Identify degraded operator: `oc get co`
2. Check operator details: `oc describe co <operator-name>`
3. Check operator pods: `oc get pods -n <operator-namespace>`
4. Review operator logs for specific errors

### Issue 6: Private DNS Not Resolving

**Symptoms:**
- Cannot resolve `api.<cluster-domain>` or `*.apps.<cluster-domain>`
- DNS queries timeout or return NXDOMAIN

**Quick Diagnosis:**
```bash
nslookup api.<cluster-name>.<random-string>.<region>.aroapp.io
```

**Fixes:**
1. Verify Private DNS zone exists in Azure
2. Check VNet is linked to the Private DNS zone
3. Verify A records exist for api and *.apps
4. Ensure DNS queries are using Azure DNS (168.63.129.16)

### Issue 7: Image Pull Failures

**Symptoms:**
- Pods stuck in ImagePullBackOff
- Authentication errors to container registry

**Fixes:**
1. Check image pull secrets: `oc get secrets --all-namespaces | grep pull-secret`
2. Verify service endpoints for ACR are configured
3. Check private endpoint connectivity if using private link
4. Test registry access from a pod

---

## Quick Diagnostic Script

Save this script as `aro-diagnostics.sh` and run it for a quick health check:

```bash
#!/bin/bash

echo "======================================"
echo "ARO Cluster Diagnostics"
echo "======================================"
echo ""

echo "=== Cluster Version ==="
oc get clusterversion
echo ""

echo "=== Cluster Operators ==="
oc get clusteroperators
echo ""

echo "=== Degraded Operators ==="
oc get clusteroperators | grep -v "True.*False.*False" | grep -v "NAME"
echo ""

echo "=== Node Status ==="
oc get nodes
echo ""

echo "=== Node Resources ==="
oc adm top nodes 2>/dev/null || echo "Metrics not available"
echo ""

echo "=== DNS Pods ==="
oc get pods -n openshift-dns
echo ""

echo "=== Network Operator ==="
oc get clusteroperator network
echo ""

echo "=== Network Plugin Pods (SDN) ==="
oc get pods -n openshift-sdn 2>/dev/null || echo "Not using OpenShift SDN"
echo ""

echo "=== Network Plugin Pods (OVN) ==="
oc get pods -n openshift-ovn-kubernetes 2>/dev/null || echo "Not using OVN-Kubernetes"
echo ""

echo "=== Ingress Operator ==="
oc get clusteroperator ingress
echo ""

echo "=== Router Pods ==="
oc get pods -n openshift-ingress
echo ""

echo "=== Router Service ==="
oc get svc -n openshift-ingress
echo ""

echo "=== Network Configuration ==="
oc get network.config.openshift.io cluster -o yaml | grep -A 10 "clusterNetwork\|serviceNetwork"
```

---

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

### Network Flow Analysis

```bash
# Enable VPC flow logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $CLUSTER_VPC \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs
```

---

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

### Common Error Codes and Solutions

| Error Code | Description | Solution |
|------------|-------------|----------|
| DNS_PROBE_FINISHED_NXDOMAIN | DNS resolution failure | Check DNS configuration and private DNS zones |
| Connection timeout | Network connectivity issues | Verify NSG rules and route tables |
| Image pull failed | Registry access issues | Check service endpoints and authentication |
| Service unavailable | Load balancer configuration | Verify Azure Load Balancer settings |

---

## Additional Resources

- [Azure Red Hat OpenShift Documentation](https://docs.microsoft.com/en-us/azure/openshift/)
- [OpenShift Network Troubleshooting](https://docs.openshift.com/container-platform/4.12/networking/troubleshooting-network-issues.html)
- [Azure Network Troubleshooting](https://docs.microsoft.com/en-us/azure/virtual-network/troubleshoot-connectivity-problem-between-vms)

---

## Conclusion

Network troubleshooting in ARO private clusters requires understanding both Azure networking components and OpenShift network architecture. Use this guide systematically, starting with basic connectivity tests and progressing to more advanced diagnostics as needed. Always document your findings and implement monitoring to prevent future issues.

