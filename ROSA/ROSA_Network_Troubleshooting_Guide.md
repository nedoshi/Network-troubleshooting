# Red Hat OpenShift Service on AWS (ROSA) Network Troubleshooting Guide

Comprehensive guide for diagnosing and fixing networking issues in ROSA clusters, including classic ROSA, ROSA HCP (Hosted Control Plane), and OpenShift Virtualization scenarios.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Understanding ROSA Architecture](#understanding-rosa-architecture)
4. [ROSA HCP Specific Architecture](#rosa-hcp-specific-architecture)
5. [Cluster Health Assessment](#cluster-health-assessment)
6. [AWS Infrastructure Validation](#aws-infrastructure-validation)
7. [DNS Resolution Testing](#dns-resolution-testing)
8. [Connectivity Testing](#connectivity-testing)
9. [Service and Ingress Testing](#service-and-ingress-testing)
10. [Network Policy Testing](#network-policy-testing)
11. [Calico CNI Installation and Troubleshooting](#calico-cni-installation-and-troubleshooting)
12. [OpenShift Virtualization Networking](#openshift-virtualization-networking)
13. [Common Issues and Solutions](#common-issues-and-solutions)
14. [Advanced Diagnostics](#advanced-diagnostics)
15. [Reference Commands](#reference-commands)

---

## Overview

Red Hat OpenShift Service on AWS (ROSA) clusters can be deployed in different configurations:
- **Classic ROSA**: Full control plane and worker nodes in customer AWS account
- **ROSA HCP (Hosted Control Plane)**: Control plane managed by Red Hat, worker nodes in customer account
- **Private Clusters**: Deployed in private subnets without direct internet access

This guide covers troubleshooting networking issues across all ROSA deployment types.

---

## Prerequisites

### Required Tools

```bash
# Install OCM CLI
curl -LO https://github.com/openshift-online/ocm-cli/releases/latest/download/ocm-linux-amd64
sudo install ocm-linux-amd64 /usr/local/bin/ocm

# Install ROSA CLI
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/rosa/latest/rosa-linux.tar.gz
tar -xzf rosa-linux.tar.gz && sudo install rosa /usr/local/bin/

# Install OpenShift CLI
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
tar -xzf openshift-client-linux.tar.gz && sudo install oc /usr/local/bin/
```

### Required Permissions
- AWS IAM permissions: `EC2`, `VPC`, `Route53`, `CloudWatch`
- ROSA cluster access with appropriate RBAC
- OCM (OpenShift Cluster Manager) authentication

### Environment Setup

```bash
# Set environment variables
export CLUSTER_NAME="my-rosa-cluster"
export AWS_REGION="us-east-1"
export AWS_PROFILE="default"

# Login to OCM and ROSA
rosa login --token=<your-token>
ocm login --token=<your-token>

# Get cluster details
rosa describe cluster -c $CLUSTER_NAME
```

---

## Understanding ROSA Architecture

### Classic ROSA Network Components
- **Private Subnets**: Worker nodes deployed in private subnets
- **Public Subnets**: Load balancers and NAT gateways
- **NAT Gateway**: Provides outbound internet access for private subnets
- **VPC Endpoints**: Direct access to AWS services
- **Transit Gateway**: Hybrid connectivity (optional)
- **Private Link**: Secure connection to Red Hat services

### Traffic Flow Patterns
1. **Ingress Traffic**: Internet → ALB/NLB → Private nodes
2. **Egress Traffic**: Private nodes → NAT Gateway → Internet
3. **Internal Traffic**: Pod-to-pod via CNI network
4. **AWS Services**: Via VPC endpoints or NAT Gateway
5. **Red Hat Services**: Via AWS Private Link

---

## ROSA HCP Specific Architecture

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

### Important Notes for ROSA HCP
1. **Control Plane Access**: You cannot access control plane nodes directly as they're managed by Red Hat
2. **Limited Cluster Operators**: Some cluster operators are not accessible in HCP clusters
3. **Networking Simplification**: No customer-managed control plane networking components
4. **Debugging Limitations**: Some advanced debugging requires Red Hat support engagement

---

## Cluster Health Assessment

### Check Cluster Status

```bash
# Check overall cluster status
rosa describe cluster -c $CLUSTER_NAME --output json | jq '.state'

# List all clusters
rosa list clusters

# Get cluster details
rosa describe cluster -c $CLUSTER_NAME
```

**Expected Output:**
```
Name:                       my-rosa-cluster
ID:                         abc123def456
OpenShift Version:          4.14.8
Channel Group:              stable
DNS:                        my-rosa-cluster.abcd.p1.openshiftapps.com
AWS Account:                123456789012
API URL:                    https://api.my-rosa-cluster.abcd.p1.openshiftapps.com:443
Console URL:                https://console-openshift-console.apps.my-rosa-cluster.abcd.p1.openshiftapps.com
Region:                     us-east-1
Multi-AZ:                   false
Hosted CP:                  Yes/No
```

### Check Node Status

```bash
# First, get kubeconfig
rosa create kubeconfig --cluster=$CLUSTER_NAME

# Check nodes
oc get nodes -o wide

# Check cluster operators
oc get clusteroperators

# Check for degraded operators
oc get clusteroperators | grep -v "True.*False.*False"
```

### Check Network Operators

```bash
# Check network operator status
oc get clusteroperator network
oc describe clusteroperator network

# View network operator logs
oc logs -n openshift-network-operator deployment/network-operator

# Check CNI configuration
oc get network cluster -o yaml
```

---

## AWS Infrastructure Validation

### Get VPC Information

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

### Check Route Tables

```bash
# Get route tables for the VPC
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].[RouteTableId,Associations[0].SubnetId,Routes[*].[DestinationCidrBlock,GatewayId,NatGatewayId]]' \
  --output json | jq '.'

# Verify route tables for subnets
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].[RouteTableId,Routes[*].[DestinationCidrBlock,GatewayId]]' \
  --output table
```

### Verify NAT Gateway

```bash
# Check NAT Gateway status
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId,PublicIp,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Check NAT Gateway CloudWatch metrics
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

### Check Security Groups

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

### Check Network ACLs

```bash
# Check Network ACLs
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkAcls[*].[NetworkAclId,Entries[*].[RuleNumber,Protocol,RuleAction,CidrBlock]]' \
  --output table
```

---

## DNS Resolution Testing

### Test Internal DNS from Pod

```bash
# Create a test pod
oc run dns-test --image=registry.access.redhat.com/ubi8/ubi:latest --rm -it --restart=Never -- bash

# Inside the pod, test DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup openshift.default.svc.cluster.local
dig @172.30.0.10 kubernetes.default.svc.cluster.local
```

### Check CoreDNS Pods

```bash
# Check DNS pods status
oc get pods -n openshift-dns -o wide

# View DNS pod logs
oc logs -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default --tail=50

# Check DNS configuration
oc get configmap/dns-default -n openshift-dns -o yaml

# Check DNS operator
oc get clusteroperator dns
oc describe clusteroperator dns
```

### Test External DNS Resolution

```bash
# Test external DNS from pod
oc run dns-external-test --image=registry.access.redhat.com/ubi8/ubi:latest --rm -it --restart=Never -- nslookup google.com

# Test DNS resolution
oc exec dns-test -- nslookup google.com
```

### Check AWS Route 53 Configuration

```bash
# Check if private hosted zones exist
aws route53 list-hosted-zones --query 'HostedZones[?Config.PrivateZone==`true`]'

# Examine VPC association for private zones
aws route53 list-vpc-association-authorizations --hosted-zone-id Z1234567890ABC
```

---

## Connectivity Testing

### Test Pod-to-Pod Communication

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

### Test External Connectivity

```bash
# Test outbound internet connectivity
oc exec network-test-pod-1 -- curl -I --connect-timeout 10 https://www.redhat.com

# Test specific ports
oc exec network-test-pod-1 -- curl -I --connect-timeout 10 https://www.google.com
```

### Test Service Connectivity

```bash
# Test service resolution
oc run service-test --image=registry.access.redhat.com/ubi8/ubi:latest --rm -it --restart=Never -- nslookup my-service.my-namespace.svc.cluster.local

# Test service connectivity
oc run service-test --image=registry.access.redhat.com/ubi8/ubi:latest --rm -it --restart=Never -- curl http://my-service.my-namespace.svc.cluster.local:8080/health
```

---

## Service and Ingress Testing

### Check Core Services

```bash
# Check DNS service
oc get svc -n openshift-dns

# Check ingress controller
oc get pods -n openshift-ingress

# Check router service
oc get svc -n openshift-ingress
oc describe svc router-default -n openshift-ingress
```

### Test Route Creation and Access

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

### Check Load Balancers

```bash
# List load balancers
aws elbv2 describe-load-balancers --query 'LoadBalancers[?Scheme==`internet-facing`].[LoadBalancerName,DNSName,State.Code]' --output table

# Check target groups
aws elbv2 describe-target-groups --query 'TargetGroups[*].[TargetGroupName,Protocol,Port,HealthCheckPath]' --output table

# Examine target health
TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
```

---

## Network Policy Testing

### Check for Network Policies

```bash
# List network policies in all namespaces
oc get networkpolicy -A

# Describe specific network policy
oc describe networkpolicy <policy-name> -n <namespace>
```

### Test Network Policy Impact

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

---

## Calico CNI Installation and Troubleshooting

### Phase 1: Complete Cleanup (if needed)

#### Remove Failed Tigera Operator Installation

```bash
# Delete any existing subscriptions
oc delete subscription tigera-operator -n openshift-operators --ignore-not-found=true

# Delete install plans
oc get installplan -n openshift-operators -o name | xargs oc delete -n openshift-operators --ignore-not-found=true

# Delete CSV if exists
oc get csv -n openshift-operators | grep tigera | awk '{print $1}' | xargs oc delete csv -n openshift-operators --ignore-not-found=true
```

#### Remove Tigera Custom Resources

```bash
# Delete Installation CRs
oc get installation.operator.tigera.io -A -o name | xargs oc delete --ignore-not-found=true

# Delete APIServer CRs
oc get apiserver.operator.tigera.io -A -o name | xargs oc delete --ignore-not-found=true
```

#### Remove Tigera CRDs

```bash
# List and delete Tigera CRDs
oc get crd | grep tigera | awk '{print $1}' | xargs oc delete crd --ignore-not-found=true

# List and delete Calico CRDs
oc get crd | grep calico | awk '{print $1}' | xargs oc delete crd --ignore-not-found=true
```

#### Delete Tigera Namespaces

```bash
# Delete operator namespace
oc delete namespace tigera-operator --ignore-not-found=true --grace-period=0

# Delete calico namespaces
oc delete namespace calico-system --ignore-not-found=true --grace-period=0
oc delete namespace calico-apiserver --ignore-not-found=true --grace-period=0
```

#### Verify Clean State

```bash
# Verify no tigera/calico resources remain
oc get all -A | grep -E "tigera|calico"
oc get crd | grep -E "tigera|calico"
oc get namespace | grep -E "tigera|calico"

# Check current node status
oc get nodes
```

### Phase 2: Direct Tigera Operator Installation

#### Install Tigera Operator Manifest

```bash
# Install operator directly (not via OLM)
oc create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
```

**Note**: You may see pod security warnings - these are expected and can be ignored.

#### Apply Pod Security Labels

```bash
# Apply privileged pod security labels to tigera-operator namespace
oc label namespace tigera-operator pod-security.kubernetes.io/enforce=privileged --overwrite
oc label namespace tigera-operator pod-security.kubernetes.io/audit=privileged --overwrite
oc label namespace tigera-operator pod-security.kubernetes.io/warn=privileged --overwrite
```

#### Fix Kubernetes API Connectivity

```bash
# Set Kubernetes service environment variables
oc set env deployment/tigera-operator -n tigera-operator \
  KUBERNETES_SERVICE_HOST=kubernetes.default.svc.cluster.local \
  KUBERNETES_SERVICE_PORT=443

# Restart the deployment
oc rollout restart deployment/tigera-operator -n tigera-operator
```

#### Verify Operator is Running

```bash
# Wait for deployment to be ready
oc rollout status deployment/tigera-operator -n tigera-operator --timeout=300s

# Check operator pod status
oc get pods -n tigera-operator

# Check operator logs (should show no DNS errors)
oc logs -n tigera-operator deployment/tigera-operator --tail=30
```

### Phase 3: Deploy Calico CNI

#### Create Installation Custom Resource

```bash
cat <<EOF | oc apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  variant: Calico
  calicoNetwork:
    ipPools:
    - cidr: 10.128.0.0/14
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
  kubernetesProvider: OpenShift
EOF
```

#### Verify Installation CR Created

```bash
# Check Installation resource
oc get installation

# View Installation status
oc get installation default -o yaml
```

#### Apply Pod Security Labels to calico-system

```bash
# Apply privileged labels to calico-system namespace
oc label namespace calico-system pod-security.kubernetes.io/enforce=privileged --overwrite
oc label namespace calico-system pod-security.kubernetes.io/audit=privileged --overwrite
oc label namespace calico-system pod-security.kubernetes.io/warn=privileged --overwrite
```

#### Monitor Calico Component Deployment

```bash
# Watch Calico pods come up
watch -n 2 'oc get pods -n calico-system'
```

Expected components:
- `calico-kube-controllers` (Deployment)
- `calico-node` (DaemonSet - one pod per node)
- `calico-typha` (Deployment)

#### Verify Calico DaemonSet

```bash
# Check DaemonSet status
oc get ds -n calico-system

# Verify calico-node pods on all nodes
oc get pods -n calico-system -o wide -l k8s-app=calico-node
```

### Phase 4: Verify Node Readiness

```bash
# Watch nodes transition to Ready
watch -n 2 'oc get nodes'
```

Nodes should transition from `NotReady` to `Ready` within 3-5 minutes.

### Troubleshooting Calico Issues

#### Issue: Operator Pod Not Starting

```bash
# Check logs
oc logs -n tigera-operator deployment/tigera-operator
oc describe pod -n tigera-operator
oc get events -n tigera-operator --sort-by='.lastTimestamp'
```

**Common fixes:**
- Verify pod security labels are applied
- Check for image pull errors
- Verify KUBERNETES_SERVICE_HOST environment variable is set

#### Issue: DNS Lookup Errors

```bash
# Get Kubernetes service ClusterIP
KUBE_API_IP=$(oc get svc kubernetes -n default -o jsonpath='{.spec.clusterIP}')

# Set using ClusterIP directly
oc set env deployment/tigera-operator -n tigera-operator \
  KUBERNETES_SERVICE_HOST=$KUBE_API_IP \
  KUBERNETES_SERVICE_PORT=443

# Restart operator
oc rollout restart deployment/tigera-operator -n tigera-operator
```

#### Issue: Calico Pods in CrashLoopBackOff

```bash
# Check calico-node logs
oc logs -n calico-system daemonset/calico-node

# Check specific pod
oc logs -n calico-system <pod-name>
oc describe pod -n calico-system <pod-name>
```

**Common issues:**
- Pod security labels not applied → Apply labels
- RBAC permissions → Check operator logs
- Network conflicts → Verify IP pool CIDR doesn't conflict

---

## OpenShift Virtualization Networking

### Test 1: VM to Underlay Network Connectivity

#### Create Test VM with Bridge Networking

```bash
cat <<EOF | oc apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: test-vm-bridge
  namespace: default
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: test-vm-bridge
    spec:
      domain:
        devices:
          interfaces:
          - name: default
            pod: {}
          - name: bridge-net
            bridge: {}
          networkInterfaceMultiqueue: true
        resources:
          requests:
            memory: 1Gi
      networks:
      - name: default
        pod: {}
      - name: bridge-net
        multus:
          networkName: bridge-network
EOF
```

#### Create Bridge NetworkAttachmentDefinition

```bash
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "bridge-network",
      "type": "bridge",
      "bridge": "br0",
      "isDefaultGateway": false,
      "ipam": {
        "type": "static",
        "addresses": [
          {
            "address": "192.168.100.10/24"
          }
        ]
      }
    }
EOF
```

### Test 2: VM to Pod Network Communication

```bash
# Deploy a test pod
oc run test-pod --image=nginx --port=80

# Get pod IP
POD_IP=$(oc get pod test-pod -o jsonpath='{.status.podIP}')
echo "Pod IP: $POD_IP"

# Get VM IP
VM_IP=$(oc get vmi test-vm-bridge -o jsonpath='{.status.interfaces[0].ipAddress}')
echo "VM IP: $VM_IP"

# Test connectivity from VM to Pod (via virtctl console)
virtctl console test-vm-bridge
# Inside VM console:
ping $POD_IP
curl http://$POD_IP

# Test connectivity from Pod to VM
oc exec test-pod -- ping $VM_IP
oc exec test-pod -- curl -I http://$VM_IP:22
```

### Test 3: IP Address Management Conflicts

```bash
# Check cluster network ranges
oc get network.config cluster -o jsonpath='{.spec.clusterNetwork}'
oc get network.config cluster -o jsonpath='{.spec.serviceNetwork}'

# Create non-conflicting network
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: safe-network
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "safe-network",
      "type": "bridge",
      "bridge": "br1",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.200.0/24",
        "rangeStart": "192.168.200.10",
        "rangeEnd": "192.168.200.50",
        "gateway": "192.168.200.1"
      }
    }
EOF
```

### Test 4: Public IP Exposure and Load Balancing

```bash
# Create a service for VM
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: vm-web-service
spec:
  selector:
    kubevirt.io/vm: test-vm-bridge
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
EOF

# Create a route for external access
oc expose service vm-web-service

# Test external access
ROUTE_URL=$(oc get route vm-web-service -o jsonpath='{.spec.host}')
curl http://$ROUTE_URL
```

### Test 5: NetworkPolicy Configuration

```bash
# Create restrictive NetworkPolicy
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: vm-network-policy
spec:
  podSelector:
    matchLabels:
      kubevirt.io/vm: test-vm-bridge
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: allowed-app
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
EOF
```

---

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

### Issue 4: Inter-Pod Communication Fails

**Symptom:**
- Pods cannot communicate with each other
- Services not reachable from pods

**Troubleshooting Steps:**
```bash
# 1. Check SDN/OVN pods are healthy
oc get pods -n openshift-sdn
oc get pods -n openshift-ovn-kubernetes

# 2. Review and temporarily remove NetworkPolicies for testing
oc get networkpolicy --all-namespaces
oc delete networkpolicy <policy-name> -n <namespace>

# 3. Verify node-to-node connectivity
oc get nodes -o wide
```

---

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

### Enable Flow Logs

```bash
# Enable VPC flow logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs
```

### Monitor CloudWatch Metrics

```bash
# Check NAT Gateway metrics
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

### Packet Capture Analysis

```bash
# Enable packet capture on worker nodes (requires node debugging)
oc debug node/<node-name>

# Inside the debug pod:
chroot /host
tcpdump -i any -w /tmp/capture.pcap host <target-ip>
```

---

## Reference Commands

### Quick Health Check Script

```bash
#!/bin/bash
echo "=== ROSA Network Health Check ==="

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
# Network operator logs
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

---

## Getting Help

For complex issues:
1. Collect must-gather: `oc adm must-gather`
2. Use ROSA support: `rosa create support-case`
3. Check Red Hat Customer Portal: https://access.redhat.com
4. Engage with Red Hat support with collected diagnostic information

---

## Additional Resources

- [ROSA Documentation](https://docs.openshift.com/rosa/)
- [AWS VPC Troubleshooting Guide](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-troubleshooting.html)
- [OpenShift Network Troubleshooting](https://docs.openshift.com/container-platform/latest/networking/troubleshooting-network-issues.html)
- [Red Hat Support](https://access.redhat.com/support)
- [Calico Documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/openshift/installation)

---

## Conclusion

This guide provides a comprehensive approach to troubleshooting networking issues in ROSA clusters. Follow the steps systematically, starting with basic connectivity tests and progressing to more advanced diagnostics as needed. Always document your findings and implement monitoring to prevent future issues.

