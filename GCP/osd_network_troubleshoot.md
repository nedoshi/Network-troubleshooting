# OpenShift GCP Networking Troubleshooting Guide
## Cluster: osdtest-cluster

## Error Summary
```
cluster is not reachable: Get "https://api.abc-osd-test.fyaz.p2.openshiftapps.com:6443/?timeout=5s": context deadline exceeded
```

This indicates the Kubernetes API server at port 6443 is not accessible due to networking issues.

---

## Quick Diagnostic Checklist

### 1. DNS Resolution
**Check if the API endpoint resolves:**
```bash
# Test DNS resolution
nslookup api.abc-osd-test.fyaz.p2.openshiftapps.com

# Alternative DNS test
dig api.abc-osd-test.fyaz.p2.openshiftapps.com
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
```

---

### 3. GCP Firewall Rules
**Check firewall rules allow required traffic:**

```bash
# List firewall rules for your VPC
gcloud compute firewall-rules list --filter="network:YOUR_VPC_NAME" --format="table(name,direction,sourceRanges,allowed)"

# Check specific rule details
gcloud compute firewall-rules describe RULE_NAME
```

**Required firewall rules for OpenShift:**
- **Port 6443**: API server (from installer/clients to control plane)
- **Port 22623**: Machine config server (from bootstrap/nodes to control plane)
- **Port 443/80**: Ingress (from external to worker nodes)

---

## GCP VPC Requirements

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
```

#### Internal Cluster Communication
```bash
# Allow internal cluster traffic
gcloud compute firewall-rules create osd-internal \
    --network=YOUR_VPC_NAME \
    --allow=tcp,udp,icmp \
    --source-ranges=YOUR_CLUSTER_CIDR \
    --description="Allow internal cluster communication"
```

---

### 4. VPC Network Configuration

**Verify VPC exists and is properly configured:**
```bash
# List VPCs
gcloud compute networks list

# Describe your VPC
gcloud compute networks describe YOUR_VPC_NAME

# List subnets
gcloud compute networks subnets list --network=YOUR_VPC_NAME
```

**Requirements:**
- VPC must have sufficient IP address space for cluster nodes
- Subnets for control plane and compute nodes
- Private Google Access enabled (for restricted networks)

---

### 5. Routes and Cloud NAT

**For restricted/private clusters, verify Cloud NAT:**
```bash
# List Cloud Routers
gcloud compute routers list

# Describe NAT configuration
gcloud compute routers nats list --router=YOUR_ROUTER_NAME --region=YOUR_REGION
```

**Check routes:**
```bash
# List routes in your VPC
gcloud compute routes list --filter="network:YOUR_VPC_NAME"
```

---

### 6. Load Balancer Health

**Check if load balancers are healthy:**
```bash
# List forwarding rules
gcloud compute forwarding-rules list --filter="name~osd"

# Check backend services
gcloud compute backend-services list

# Check health checks
gcloud compute health-checks list
```

---

## Common Issues and Solutions

### Issue 1: Firewall Blocking API Access
**Symptom:** Connection timeout to port 6443  
**Solution:** Add firewall rule allowing TCP 6443 from your source network

### Issue 2: DNS Not Resolving
**Symptom:** nslookup fails or returns no address  
**Solution:** 
- Verify Cloud DNS zone is created
- Check DNS records for api.* endpoint
- Ensure DNS is accessible from installer machine

### Issue 3: Private Cluster Without NAT
**Symptom:** Nodes can't reach external registries  
**Solution:** Configure Cloud NAT for outbound internet access

### Issue 4: Incorrect Source Ranges
**Symptom:** Firewall rules exist but connection still fails  
**Solution:** Verify source-ranges include the CIDR where you're installing from

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
    --format="table(name,allowed,sourceRanges)"
```

### Step 3: Test Connectivity
```bash
# From installer machine or bastion host
curl -v --max-time 10 https://api.abc-osd-test.fyaz.p2.openshiftapps.com:6443

# Check if you can reach the expected load balancer IP
LOAD_BALANCER_IP=$(dig +short api.abc-osd-test.fyaz.p2.openshiftapps.com)
ping -c 4 $LOAD_BALANCER_IP
```

### Step 4: Review Installation Logs
```bash
# Check installer logs for more details
cat .openshift_install.log | grep -i "error\|fail\|timeout"

# Look for specific network-related errors
cat .openshift_install.log | grep -i "network\|firewall\|connection"
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

## Additional Resources

- [GCP VPC Requirements Documentation](https://docs.openshift.com/container-platform/latest/installing/installing_gcp/installing-restricted-networks-gcp-installer-provisioned.html)
- GCP Firewall Rules: `gcloud compute firewall-rules --help`
- OpenShift Install Logs: `.openshift_install.log` in installation directory

---

## Quick Reference Commands

```bash
# Check project and region
gcloud config list

# List all compute resources
gcloud compute instances list
gcloud compute forwarding-rules list
gcloud compute firewall-rules list

# Monitor installation
openshift-install wait-for install-complete --dir=YOUR_INSTALL_DIR --log-level=debug
```