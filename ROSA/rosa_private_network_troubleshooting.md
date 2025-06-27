# ROSA Private Cluster Network Troubleshooting Tutorial

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Understanding ROSA Private Cluster Architecture](#understanding-rosa-private-cluster-architecture)
4. [Common Network Issues](#common-network-issues)
5. [Troubleshooting Methodology](#troubleshooting-methodology)
6. [Step-by-Step Troubleshooting Guide](#step-by-step-troubleshooting-guide)
7. [Advanced Diagnostics](#advanced-diagnostics)
8. [Preventive Measures](#preventive-measures)
9. [Reference Commands](#reference-commands)

## Overview

Red Hat OpenShift Service on AWS (ROSA) private clusters are deployed in private subnets without direct internet access. This tutorial provides a comprehensive guide to troubleshoot networking issues specific to ROSA private clusters, including connectivity problems, DNS resolution issues, and service access challenges.

## Prerequisites

### Required Tools
- AWS CLI configured with appropriate permissions
- ROSA CLI (`rosa`)
- OpenShift CLI (`oc`)
- Network troubleshooting tools (`ping`, `dig`, `curl`, `telnet`)
- SSH access to bastion host (if configured)

### Required Permissions
- AWS IAM permissions for VPC, subnet, and route table inspection
- OpenShift cluster-admin or monitoring permissions
- Access to AWS CloudWatch logs

### Environment Setup
```bash
# Verify tool installations
aws --version
rosa version
oc version

# Set environment variables
export CLUSTER_NAME="your-rosa-cluster"
export AWS_REGION="us-east-1"
export KUBECONFIG="/path/to/kubeconfig"
```

## Understanding ROSA Private Cluster Architecture

### Network Components
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

## Common Network Issues

### Connectivity Issues
- Pods cannot reach external services
- Applications inaccessible from outside
- Inter-pod communication failures
- DNS resolution problems
- Certificate validation errors

### Performance Issues
- High latency for external calls
- NAT Gateway bandwidth limitations
- VPC endpoint throttling
- Cross-AZ traffic costs

### Security Issues
- Network policy blocking traffic
- Security group misconfigurations
- NACL restrictions
- Private Link connection failures

## Troubleshooting Methodology

### 1. Identify the Scope
- **Application-specific**: Single pod/service affected
- **Node-specific**: All pods on specific nodes affected
- **Cluster-wide**: All applications experiencing issues
- **External**: Internet or AWS service connectivity

### 2. Layer-by-Layer Analysis
1. **Physical/Data Link**: AWS infrastructure
2. **Network**: IP routing and VPC configuration
3. **Transport**: Port accessibility and load balancing
4. **Application**: Service discovery and application logic

### 3. Systematic Approach
1. Gather information and reproduce the issue
2. Check recent changes (deployments, configuration updates)
3. Examine logs and metrics
4. Test connectivity at different layers
5. Implement fixes and verify resolution

## Step-by-Step Troubleshooting Guide

### Step 1: Cluster Health Assessment

#### Check Cluster Status
```bash
# Verify cluster is accessible
rosa describe cluster $CLUSTER_NAME

# Check cluster operators
oc get clusteroperators

# Verify node status
oc get nodes -o wide

# Check for cluster-wide alerts
oc get events --all-namespaces --sort-by='.lastTimestamp'
```

#### Examine Network Operators
```bash
# Check network operator status
oc get clusteroperator network

# View network operator logs
oc logs -n openshift-network-operator deployment/network-operator

# Check CNI configuration
oc get network cluster -o yaml
```

### Step 2: AWS Infrastructure Validation

#### VPC and Subnet Configuration
```bash
# Get cluster VPC information
CLUSTER_VPC=$(rosa describe cluster $CLUSTER_NAME --output json | jq -r '.aws.subnet_ids[0]' | xargs aws ec2 describe-subnets --subnet-ids --query 'Subnets[0].VpcId' --output text)

# List all subnets in the VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$CLUSTER_VPC" --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock,MapPublicIpOnLaunch]' --output table

# Check route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$CLUSTER_VPC" --query 'RouteTables[*].[RouteTableId,Routes[*].[DestinationCidrBlock,GatewayId]]' --output table
```

#### NAT Gateway Status
```bash
# Find NAT Gateways in the VPC
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$CLUSTER_VPC" --query 'NatGateways[*].[NatGatewayId,State,SubnetId,PublicIp]' --output table

# Check NAT Gateway CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-xxxxxxxx \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

#### Security Groups and NACLs
```bash
# Get security groups for worker nodes
WORKER_SG=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=*worker*" "Name=vpc-id,Values=$CLUSTER_VPC" --query 'Reservations[*].Instances[*].SecurityGroups[*].GroupId' --output text | head -1)

# Examine security group rules
aws ec2 describe-security-groups --group-ids $WORKER_SG --query 'SecurityGroups[*].[GroupId,IpPermissions[*].[IpProtocol,FromPort,ToPort,IpRanges[*].CidrIp]]' --output table

# Check Network ACLs
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$CLUSTER_VPC" --query 'NetworkAcls[*].[NetworkAclId,Entries[*].[RuleNumber,Protocol,RuleAction,CidrBlock]]' --output table
```

### Step 3: DNS Resolution Testing

#### Internal DNS
```bash
# Test DNS resolution from a pod
oc run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS pods
oc get pods -n openshift-dns

# Examine CoreDNS configuration
oc get configmap/dns-default -n openshift-dns -o yaml

# Test external DNS resolution
oc run dns-test --image=busybox --rm -it --restart=Never -- nslookup google.com
```

#### AWS Route 53 Configuration
```bash
# Check if private hosted zones exist
aws route53 list-hosted-zones --query 'HostedZones[?Config.PrivateZone==`true`]'

# Examine VPC association for private zones
aws route53 list-vpc-association-authorizations --hosted-zone-id Z1234567890ABC
```

### Step 4: Connectivity Testing

#### Pod-to-Pod Communication
```bash
# Deploy test pods in different nodes
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: network-test-1
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
  nodeName: $(oc get nodes -o name | head -1 | cut -d'/' -f2)
---
apiVersion: v1
kind: Pod
metadata:
  name: network-test-2
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
  nodeName: $(oc get nodes -o name | tail -1 | cut -d'/' -f2)
EOF

# Test connectivity between pods
POD1_IP=$(oc get pod network-test-1 -o jsonpath='{.status.podIP}')
oc exec network-test-2 -- ping -c 3 $POD1_IP
```

#### External Connectivity
```bash
# Test outbound internet connectivity
oc run connectivity-test --image=curlimages/curl --rm -it --restart=Never -- curl -v https://www.google.com

# Test AWS service connectivity
oc run aws-test --image=amazon/aws-cli --rm -it --restart=Never -- aws s3 ls

# Test specific ports
oc run port-test --image=busybox --rm -it --restart=Never -- telnet google.com 443
```

#### Service Connectivity
```bash
# Test service resolution
oc run service-test --image=busybox --rm -it --restart=Never -- nslookup my-service.my-namespace.svc.cluster.local

# Test service connectivity
oc run service-test --image=curlimages/curl --rm -it --restart=Never -- curl http://my-service.my-namespace.svc.cluster.local:8080/health
```

### Step 5: Load Balancer and Ingress Testing

#### Application Load Balancer
```bash
# List load balancers
aws elbv2 describe-load-balancers --query 'LoadBalancers[?Scheme==`internet-facing`].[LoadBalancerName,DNSName,State.Code]' --output table

# Check target groups
aws elbv2 describe-target-groups --query 'TargetGroups[*].[TargetGroupName,Protocol,Port,HealthCheckPath]' --output table

# Examine target health
TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
```

#### Ingress Controller
```bash
# Check ingress controller pods
oc get pods -n openshift-ingress

# Examine ingress controller logs
oc logs -n openshift-ingress deployment/router-default -c router

# List ingress/routes
oc get routes --all-namespaces
oc get ingress --all-namespaces
```

### Step 6: Network Policy Analysis

#### Check Network Policies
```bash
# List all network policies
oc get networkpolicies --all-namespaces

# Examine specific network policy
oc describe networkpolicy my-policy -n my-namespace

# Check for default deny policies
oc get networkpolicy -A -o yaml | grep -A 10 -B 5 "podSelector: {}"
```

#### Test Network Policy Impact
```bash
# Create test pods to verify policy enforcement
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: policy-test-allowed
  labels:
    app: allowed
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
---
apiVersion: v1
kind: Pod
metadata:
  name: policy-test-denied
  labels:
    app: denied
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
EOF

# Test connectivity with different labels
TARGET_IP=$(oc get pod target-pod -o jsonpath='{.status.podIP}')
oc exec policy-test-allowed -- ping -c 3 $TARGET_IP
oc exec policy-test-denied -- ping -c 3 $TARGET_IP
```

## Advanced Diagnostics

### Network Flow Analysis

#### Enable Flow Logs
```bash
# Enable VPC flow logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $CLUSTER_VPC \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs
```

#### Analyze Traffic Patterns
```bash
# Query flow logs for specific patterns
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --filter-pattern "[version, account, eni, source, destination=\"10.0.*\", srcport, destport=\"443\", protocol=\"6\", packets, bytes, windowstart, windowend, action=\"REJECT\", flowlogstatus]"
```

### Performance Monitoring

#### Monitor NAT Gateway Metrics
```bash
# Create CloudWatch dashboard for NAT Gateway metrics
aws cloudwatch put-dashboard \
  --dashboard-name "ROSA-NAT-Gateway-Metrics" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/NATGateway", "BytesOutToDestination", "NatGatewayId", "nat-xxxxxxxx"],
            ["AWS/NATGateway", "BytesInFromDestination", "NatGatewayId", "nat-xxxxxxxx"]
          ],
          "period": 300,
          "stat": "Sum",
          "region": "us-east-1",
          "title": "NAT Gateway Data Transfer"
        }
      }
    ]
  }'
```

#### Application Performance Monitoring
```bash
# Check application response times
oc run perf-test --image=curlimages/curl --rm -it --restart=Never -- curl -w "@curl-format.txt" http://my-app.apps.cluster.example.com

# Monitor pod resource usage
oc top pods --all-namespaces --sort-by=memory
```

### Certificate and TLS Issues

#### Verify Certificates
```bash
# Check certificate chain
oc run cert-test --image=curlimages/curl --rm -it --restart=Never -- curl -vvI https://my-app.apps.cluster.example.com

# Examine certificate details
echo | openssl s_client -servername my-app.apps.cluster.example.com -connect my-app.apps.cluster.example.com:443 2>/dev/null | openssl x509 -noout -dates -subject -issuer
```

#### CA Bundle Issues
```bash
# Check cluster CA bundle
oc get configmap kube-root-ca.crt -o yaml

# Verify custom CA certificates
oc get configmap user-ca-bundle -n openshift-config -o yaml
```

## Preventive Measures

### Monitoring Setup
```bash
# Set up alerts for network issues
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: network-monitoring
  namespace: openshift-monitoring
spec:
  groups:
  - name: network.rules
    rules:
    - alert: HighNATGatewayUsage
      expr: aws_nat_gateway_bytes_out_to_destination > 1000000000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High NAT Gateway usage detected"
    - alert: DNSResolutionFailure
      expr: increase(coredns_dns_request_duration_seconds_total[5m]) == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "DNS resolution failures detected"
EOF
```

### Network Validation Scripts
```bash
# Create network validation script
cat > network-health-check.sh << 'EOF'
#!/bin/bash

echo "=== ROSA Private Cluster Network Health Check ==="

# Check cluster connectivity
echo "1. Checking cluster connectivity..."
oc get nodes > /dev/null 2>&1 && echo "✓ Cluster accessible" || echo "✗ Cluster not accessible"

# Check DNS resolution
echo "2. Testing DNS resolution..."
oc run dns-check --image=busybox --rm -i --restart=Never -- nslookup kubernetes.default.svc.cluster.local > /dev/null 2>&1 && echo "✓ Internal DNS working" || echo "✗ Internal DNS failed"

# Check external connectivity
echo "3. Testing external connectivity..."
oc run connectivity-check --image=curlimages/curl --rm -i --restart=Never -- curl -s https://www.google.com > /dev/null 2>&1 && echo "✓ External connectivity working" || echo "✗ External connectivity failed"

# Check ingress
echo "4. Checking ingress controller..."
oc get pods -n openshift-ingress | grep -q "Running" && echo "✓ Ingress controller running" || echo "✗ Ingress controller issues"

echo "=== Health check complete ==="
EOF

chmod +x network-health-check.sh
```

## Reference Commands

### Quick Diagnostics
```bash
# One-liner cluster health check
oc get nodes,pods -A | grep -E "(NotReady|Error|CrashLoop)"

# Network operator status
oc get co network -o jsonpath='{.status.conditions[?(@.type=="Available")].status}'

# DNS test
oc run dns-test --image=busybox --rm -i --restart=Never -- nslookup google.com

# Connectivity test
oc run conn-test --image=curlimages/curl --rm -i --restart=Never -- curl -I https://httpbin.org/ip
```

### Log Collection
```bash
# Collect network operator logs
oc adm must-gather --image=quay.io/openshift/origin-must-gather:latest -- /usr/bin/gather_network_logs

# Collect specific pod logs
oc logs -n openshift-ingress -l app=router --tail=100

# Export cluster network configuration
oc get network,networkpolicy,routes,services -A -o yaml > cluster-network-config.yaml
```

### Emergency Procedures
```bash
# Restart network operator (if needed)
oc delete pod -n openshift-network-operator -l app=network-operator

# Restart CoreDNS (if needed)
oc delete pod -n openshift-dns -l app=dns

# Force route refresh
oc delete pod -n openshift-ingress -l app=router
```

## Conclusion

This tutorial provides a comprehensive approach to troubleshooting ROSA private cluster networking issues. Remember to:

1. **Start with the basics**: Verify cluster health and AWS infrastructure
2. **Follow a systematic approach**: Work through layers methodically
3. **Document findings**: Keep track of what you've checked and results
4. **Test thoroughly**: Validate fixes before considering issues resolved
5. **Monitor proactively**: Set up monitoring to catch issues early

For complex issues or when in doubt, engage Red Hat support with the diagnostic information gathered using these procedures.

## Additional Resources

- [ROSA Documentation](https://docs.openshift.com/rosa/)
- [AWS VPC Troubleshooting Guide](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-troubleshooting.html)
- [OpenShift Network Troubleshooting](https://docs.openshift.com/container-platform/latest/networking/troubleshooting-network-issues.html)
- [Red Hat Support](https://access.redhat.com/support)