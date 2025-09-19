# ROSA HCP Private Cluster Network Troubleshooting Guide

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Understanding ROSA HCP Architecture](#understanding-rosa-hcp-architecture)
4. [Environment Setup](#environment-setup)
5. [Step-by-Step Troubleshooting](#step-by-step-troubleshooting)
6. [Common Issues and Solutions](#common-issues-and-solutions)
7. [Advanced Diagnostics](#advanced-diagnostics)
8. [Reference Commands](#reference-commands)

## Overview

ROSA Hosted Control Plane (HCP) private clusters have a different architecture than classic ROSA clusters. The control plane runs in a Red Hat-managed AWS account, while worker nodes run in your AWS account within private subnets. This guide provides accurate troubleshooting steps with correct CLI commands and expected outputs.

## Prerequisites

### Required Tools
```bash
# Install required tools
curl -LO https://github.com/openshift-online/ocm-cli/releases/latest/download/ocm-linux-amd64
sudo install ocm-linux-amd64 /usr/local/bin/ocm

# ROSA CLI
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/rosa/latest/rosa-linux.tar.gz
tar -xzf rosa-linux.tar.gz && sudo install rosa /usr/local/bin/

# OpenShift CLI
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
tar -xzf openshift-client-linux.tar.gz && sudo install oc /usr/local/bin/
```

### Required Permissions
- AWS IAM permissions: `EC2`, `VPC`, `Route53`, `CloudWatch`
- ROSA cluster access with appropriate RBAC
- OCM (OpenShift Cluster Manager) authentication

## Understanding ROSA HCP Architecture

### Key Differences from Classic ROSA
- **Control Plane**: Managed by Red Hat in Red Hat's AWS account
- **Worker Nodes**: Run in customer's private subnets
- **API Access**: Through Red Hat's managed load balancer or private endpoint
- **Networking**: Simplified - no customer-managed control plane components

### Network Components
- **Private Subnets**: Worker nodes only
- **NAT Gateway**: Required for outbound internet access
- **VPC Endpoints**: Optional for AWS service access
- **Route Tables**: Customer-managed routing
- **Security Groups**: Customer-managed node security

## Environment Setup

```bash
# Set environment variables
export CLUSTER_NAME="my-hcp-cluster"
export AWS_REGION="us-east-1"
export AWS_PROFILE="default"

# Login to OCM and ROSA
rosa login --token=<your-token>
ocm login --token=<your-token>

# Get cluster details
rosa describe cluster -c $CLUSTER_NAME
```

**Expected Output:**
```
Name:                       my-hcp-cluster
ID:                         abc123def456
External ID:                
OpenShift Version:          4.14.8
Channel Group:              stable
DNS:                        my-hcp-cluster.abcd.p1.openshiftapps.com
AWS Account:                123456789012
API URL:                    https://api.my-hcp-cluster.abcd.p1.openshiftapps.com:443
Console URL:                https://console-openshift-console.apps.my-hcp-cluster.abcd.p1.openshiftapps.com
Region:                     us-east-1
Multi-AZ:                   false
Hosted CP:                  Yes
```

## Step-by-Step Troubleshooting

### Step 1: Cluster Status Verification

#### Check Cluster Status
```bash
# Check overall cluster status
rosa describe cluster -c $CLUSTER_NAME --output json | jq '.state'

# List all clusters
rosa list clusters
```

**Expected Output:**
```json
"ready"
```

#### Check Node Status
```bash
# First, get kubeconfig
rosa create kubeconfig --cluster=$CLUSTER_NAME

# Check nodes
oc get nodes -o wide
```

**Expected Output:**
```
NAME                                         STATUS   ROLES    AGE   VERSION           INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                                                       KERNEL-VERSION                 CONTAINER-RUNTIME
ip-10-0-1-100.us-east-1.compute.internal    Ready    worker   2h    v1.27.8+4fab27b   10.0.1.100    <none>        Red Hat Enterprise Linux CoreOS 414.92.202312080947-0 (Ootpa)   5.14.0-362.18.1.el9_3.x86_64   cri-o://1.27.1-6.rhaos4.14.git38e9d75.el9
ip-10-0-2-200.us-east-1.compute.internal    Ready    worker   2h    v1.27.8+4fab27b   10.0.2.200    <none>        Red Hat Enterprise Linux CoreOS 414.92.202312080947-0 (Ootpa)   5.14.0-362.18.1.el9_3.x86_64   cri-o://1.27.1-6.rhaos4.14.git38e9d75.el9
```

### Step 2: AWS Infrastructure Validation

#### Get VPC Information
```bash
# Get cluster subnet information
SUBNET_IDS=$(rosa describe cluster -c $CLUSTER_NAME --output json | jq -r '.aws.subnet_ids[]' | tr '\n' ' ')
echo "Subnet IDs: $SUBNET_IDS"

# Get VPC ID from subnets
VPC_ID=$(aws ec2 describe-subnets --subnet-ids $(echo $SUBNET_IDS | cut -d' ' -f1) --query 'Subnets[0].VpcId' --output text)
echo "VPC ID: $VPC_ID"

# List all subnets in the cluster VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock,MapPublicIpOnLaunch,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

**Expected Output:**
```
----------------------------------------------------------------------
|                         DescribeSubnets                          |
+---------------------------+----------------+----------------+------+-----------------+
|  subnet-abc123           |  us-east-1a    |  10.0.1.0/24   |False | private-subnet-1|
|  subnet-def456           |  us-east-1b    |  10.0.2.0/24   |False | private-subnet-2|
|  subnet-ghi789           |  us-east-1a    |  10.0.0.0/24   |True  | public-subnet-1 |
|  subnet-jkl012           |  us-east-1b    |  10.0.3.0/24   |True  | public-subnet-2 |
----------------------------------------------------------------------
```

#### Check Route Tables
```bash
# Get route tables for the VPC
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].[RouteTableId,Associations[0].SubnetId,Routes[*].[DestinationCidrBlock,GatewayId,NatGatewayId]]' \
  --output json | jq '.'
```

**Expected Output (condensed):**
```json
[
  [
    "rtb-abc123",
    "subnet-abc123",
    [
      ["10.0.0.0/16", "local", null],
      ["0.0.0.0/0", null, "nat-xyz789"]
    ]
  ]
]
```

#### Verify NAT Gateway
```bash
# Check NAT Gateway status
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId,PublicIp,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

**Expected Output:**
```
--------------------------------------------------------------------
|                        DescribeNatGateways                       |
+----------------+----------+----------------+---------------+------+
|  nat-xyz789    |available |  subnet-ghi789 |  3.208.123.45 | None |
+----------------+----------+----------------+---------------+------+
```

### Step 3: Connectivity Testing

#### Test Pod-to-Pod Communication
```bash
# Create test pods
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: network-test-pod-1
  labels:
    app: network-test
spec:
  containers:
  - name: test-container
    image: registry.access.redhat.com/ubi8/ubi:latest
    command: ['sleep', '3600']
  restartPolicy: Never
---
apiVersion: v1
kind: Pod
metadata:
  name: network-test-pod-2
  labels:
    app: network-test
spec:
  containers:
  - name: test-container
    image: registry.access.redhat.com/ubi8/ubi:latest
    command: ['sleep', '3600']
  restartPolicy: Never
EOF

# Wait for pods to be ready
oc wait --for=condition=Ready pod/network-test-pod-1 --timeout=60s
oc wait --for=condition=Ready pod/network-test-pod-2 --timeout=60s

# Get pod IPs
POD1_IP=$(oc get pod network-test-pod-1 -o jsonpath='{.status.podIP}')
POD2_IP=$(oc get pod network-test-pod-2 -o jsonpath='{.status.podIP}')
echo "Pod 1 IP: $POD1_IP"
echo "Pod 2 IP: $POD2_IP"

# Test connectivity between pods
oc exec network-test-pod-1 -- ping -c 3 $POD2_IP
```

**Expected Output:**
```
PING 10.128.0.45 (10.128.0.45) 56(84) bytes of data.
64 bytes from 10.128.0.45: icmp_seq=1 ttl=64 time=0.156 ms
64 bytes from 10.128.0.45: icmp_seq=2 ttl=64 time=0.091 ms
64 bytes from 10.128.0.45: icmp_seq=3 ttl=64 time=0.075 ms

--- 10.128.0.45 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2046ms
rtt min/avg/max/mdev = 0.075/0.107/0.156/0.034 ms
```

#### Test External Connectivity
```bash
# Test outbound internet connectivity
oc exec network-test-pod-1 -- curl -I --connect-timeout 10 https://www.redhat.com
```

**Expected Output:**
```
HTTP/2 200 
server: Apache
content-type: text/html; charset=utf-8
...
```

#### Test DNS Resolution
```bash
# Test internal DNS
oc exec network-test-pod-1 -- nslookup kubernetes.default.svc.cluster.local
```

**Expected Output:**
```
Server:		172.30.0.10
Address:	172.30.0.10:53

Name:	kubernetes.default.svc.cluster.local
Address: 172.30.0.1
```

```bash
# Test external DNS
oc exec network-test-pod-1 -- nslookup google.com
```

**Expected Output:**
```
Server:		172.30.0.10
Address:	172.30.0.10:53

Non-authoritative answer:
Name:	google.com
Address: 142.250.191.14
```

### Step 4: Service and Ingress Testing

#### Check Core Services
```bash
# Check DNS service
oc get svc -n openshift-dns
```

**Expected Output:**
```
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
dns-default   ClusterIP   172.30.0.10    <none>        53/UDP,53/TCP,9154/TCP   3h
```

```bash
# Check ingress controller
oc get pods -n openshift-ingress
```

**Expected Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE
router-default-7d9c8c8f4b-abc12   1/1     Running   0          3h
router-default-7d9c8c8f4b-def34   1/1     Running   0          3h
```

#### Test Route Creation and Access
```bash
# Create a test application
oc new-app --name=test-app --image=registry.access.redhat.com/ubi8/httpd-24:latest

# Wait for deployment
oc wait --for=condition=Available deployment/test-app --timeout=300s

# Create a route
oc expose service/test-app

# Get route URL
ROUTE_URL=$(oc get route test-app -o jsonpath='{.spec.host}')
echo "Route URL: https://$ROUTE_URL"

# Test route access from outside (if you have external access)
curl -I https://$ROUTE_URL
```

### Step 5: Network Policy Testing

#### Check for Network Policies
```bash
# List network policies in all namespaces
oc get networkpolicy -A
```

**Expected Output:**
```
NAMESPACE   NAME                    POD-SELECTOR   AGE
default     allow-from-openshift-ingress   <none>    3h
```

#### Test Network Policy Impact
```bash
# Create a test network policy
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-deny-all
spec:
  podSelector:
    matchLabels:
      app: network-test
  policyTypes:
  - Ingress
  - Egress
EOF

# Test connectivity with policy in place
oc exec network-test-pod-1 -- ping -c 2 $POD2_IP

# Remove the policy
oc delete networkpolicy test-deny-all

# Test connectivity again
oc exec network-test-pod-1 -- ping -c 2 $POD2_IP
```

### Step 6: Advanced AWS Diagnostics

#### Check Security Groups
```bash
# Get worker node security groups
WORKER_INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=*worker*" "Name=vpc-id,Values=$VPC_ID" \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)

SECURITY_GROUPS=$(aws ec2 describe-instances --instance-ids $WORKER_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].SecurityGroups[*].GroupId' --output text)

echo "Security Groups: $SECURITY_GROUPS"

# Check security group rules
for sg in $SECURITY_GROUPS; do
  echo "=== Security Group: $sg ==="
  aws ec2 describe-security-groups --group-ids $sg \
    --query 'SecurityGroups[0].IpPermissions[*].[IpProtocol,FromPort,ToPort,IpRanges[0].CidrIp]' \
    --output table
done
```

#### Monitor CloudWatch Metrics
```bash
# Check NAT Gateway metrics (if you have NAT Gateway)
NAT_GATEWAY_ID=$(aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" \
  --query 'NatGateways[0].NatGatewayId' --output text)

if [ "$NAT_GATEWAY_ID" != "None" ] && [ "$NAT_GATEWAY_ID" != "null" ]; then
  aws cloudwatch get-metric-statistics \
    --namespace AWS/NATGateway \
    --metric-name BytesOutToDestination \
    --dimensions Name=NatGatewayId,Value=$NAT_GATEWAY_ID \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Sum Average \
    --output table
fi
```

## Common Issues and Solutions

### Issue 1: Pods Cannot Access Internet

**Symptom:**
```bash
oc exec network-test-pod-1 -- curl --connect-timeout 5 https://www.google.com
# Output: curl: (28) Connection timed out after 5000 milliseconds
```

**Troubleshooting Steps:**
```bash
# 1. Check NAT Gateway status
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" --query 'NatGateways[*].State'

# 2. Verify route table has NAT Gateway route
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[?Associations[?SubnetId!=null]][*].Routes[?DestinationCidrBlock==`0.0.0.0/0`]'

# 3. Check security group allows outbound traffic
aws ec2 describe-security-groups --group-ids $SECURITY_GROUPS \
  --query 'SecurityGroups[*].IpPermissionsEgress[?IpProtocol==`-1`]'
```

### Issue 2: DNS Resolution Failures

**Symptom:**
```bash
oc exec network-test-pod-1 -- nslookup google.com
# Output: server can't find google.com: NXDOMAIN
```

**Troubleshooting Steps:**
```bash
# 1. Check CoreDNS pods
oc get pods -n openshift-dns -o wide

# 2. Check CoreDNS service
oc get svc -n openshift-dns

# 3. Test DNS server directly
oc exec network-test-pod-1 -- nslookup google.com 8.8.8.8

# 4. Check CoreDNS configuration
oc get configmap/dns-default -n openshift-dns -o yaml
```

### Issue 3: Application Routes Not Accessible

**Symptom:**
```bash
curl -I https://my-app.apps.cluster.example.com
# Output: curl: (6) Could not resolve host
```

**Troubleshooting Steps:**
```bash
# 1. Check if route exists
oc get route my-app

# 2. Check ingress controller status
oc get pods -n openshift-ingress

# 3. Check service endpoints
oc get endpoints my-app

# 4. Test from within cluster
oc exec network-test-pod-1 -- curl -I http://my-app.default.svc.cluster.local:8080
```

## Advanced Diagnostics

### Collect Must-Gather
```bash
# Collect comprehensive diagnostic information
oc adm must-gather --dest-dir=/tmp/must-gather

# For network-specific issues
oc adm must-gather \
  --image=quay.io/openshift/origin-must-gather:latest \
  --dest-dir=/tmp/network-must-gather \
  -- /usr/bin/gather_network_logs
```

### Enable Debug Mode
```bash
# Create debug node (if needed for advanced troubleshooting)
oc debug node/$(oc get nodes -o jsonpath='{.items[0].metadata.name}')
```

## Reference Commands

### Quick Health Check Script
```bash
#!/bin/bash
echo "=== ROSA HCP Network Health Check ==="

# Cluster status
echo "1. Cluster Status:"
rosa describe cluster $CLUSTER_NAME --output json | jq -r '.state'

# Node status
echo "2. Node Status:"
oc get nodes --no-headers | awk '{print $2}' | sort | uniq -c

# DNS test
echo "3. DNS Test:"
oc run dns-test --image=registry.access.redhat.com/ubi8/ubi:latest --rm -i --restart=Never \
  --command -- nslookup google.com > /dev/null 2>&1 && echo "✓ DNS working" || echo "✗ DNS failed"

# Connectivity test
echo "4. Connectivity Test:"
oc run conn-test --image=registry.access.redhat.com/ubi8/ubi:latest --rm -i --restart=Never \
  --command -- curl -I --connect-timeout 5 https://www.redhat.com > /dev/null 2>&1 && echo "✓ External connectivity working" || echo "✗ External connectivity failed"

# Ingress test
echo "5. Ingress Status:"
oc get pods -n openshift-ingress --no-headers | grep Running | wc -l | xargs echo "Running ingress pods:"

echo "=== Health check complete ==="
```

### Log Collection Commands
```bash
# Network operator logs (for HCP, limited access)
oc logs -n openshift-network-operator deployment/network-operator --tail=50

# DNS logs
oc logs -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default --tail=50

# Ingress logs
oc logs -n openshift-ingress deployment/router-default --tail=50

# Node logs (using oc debug)
oc debug node/<node-name> -- journalctl -u crio -u kubelet --since "1 hour ago"
```

### Cleanup Test Resources
```bash
# Clean up test pods and applications
oc delete pod network-test-pod-1 network-test-pod-2 --ignore-not-found=true
oc delete all -l app=test-app --ignore-not-found=true
oc delete route test-app --ignore-not-found=true
oc delete networkpolicy test-deny-all --ignore-not-found=true
```

## Important Notes for ROSA HCP

1. **Control Plane Access**: You cannot access control plane nodes directly as they're managed by Red Hat
2. **Limited Cluster Operators**: Some cluster operators are not accessible in HCP clusters
3. **Networking Simplification**: No customer-managed control plane networking components
4. **Debugging Limitations**: Some advanced debugging requires Red Hat support engagement

## Getting Help

For complex issues:
1. Collect must-gather: `oc adm must-gather`
2. Use ROSA support: `rosa create support-case`
3. Check Red Hat Customer Portal: https://access.redhat.com
4. Engage with Red Hat support with collected diagnostic information

This guide provides working commands specific to ROSA HCP private clusters. Always test in a development environment first and follow your organization's change management processes.
