# ROSA Virtualization Networking Testing Guide

This comprehensive guide provides step-by-step instructions for testing networking challenges in Red Hat OpenShift Service on AWS (ROSA) with OpenShift Virtualization.

## 🌐 Testing Dual-Stack Networking Complexity

### Test 1: VM to Underlay Network Connectivity

#### Step-by-Step Testing:

**Prerequisites:**
- ROSA cluster with OpenShift Virtualization installed
- CLI tools: `oc`, `virtctl`, `kubectl`
- VM running in the cluster

```bash
# 1. Check cluster network configuration
oc get network.config cluster -o yaml

# 2. List available NetworkAttachmentDefinitions
oc get net-attach-def -A

# 3. Create a test VM with bridge networking
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

# 4. Test connectivity from VM to AWS VPC
oc console # Access VM console
# Inside VM:
ip route show
ping <AWS-VPC-gateway-IP>
curl -I http://169.254.169.254/latest/meta-data/ # Test AWS metadata
```

**Expected Issues:**
- Direct VPC connectivity will fail
- AWS metadata endpoint unreachable from VM

**Workaround:**
```bash
# Create a bridge NetworkAttachmentDefinition
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

#### Step-by-Step Testing:

```bash
# 1. Deploy a test pod
oc run test-pod --image=nginx --port=80

# 2. Get pod IP
POD_IP=$(oc get pod test-pod -o jsonpath='{.status.podIP}')
echo "Pod IP: $POD_IP"

# 3. Get VM IP
VM_IP=$(oc get vmi test-vm-bridge -o jsonpath='{.status.interfaces[0].ipAddress}')
echo "VM IP: $VM_IP"

# 4. Test connectivity from VM to Pod
virtctl console test-vm-bridge
# Inside VM console:
ping $POD_IP
curl http://$POD_IP

# 5. Test connectivity from Pod to VM
oc exec test-pod -- ping $VM_IP
oc exec test-pod -- curl -I http://$VM_IP:22
```

**Debugging Steps:**
```bash
# Check CNI configuration
oc get nodes -o wide
oc describe node <node-name> | grep -A 10 "PodCIDR"

# Verify network policies
oc get networkpolicy -A
oc describe networkpolicy <policy-name>

# Check routing tables
oc debug node/<node-name>
# In debug pod:
ip route show
iptables -L -n
```

### Test 3: IP Address Management Conflicts

#### Step-by-Step Testing:

```bash
# 1. Check cluster network ranges
oc get network.config cluster -o jsonpath='{.spec.clusterNetwork}'
oc get network.config cluster -o jsonpath='{.spec.serviceNetwork}'

# 2. Create VMs with potentially conflicting IPs
cat <<EOF | oc apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: test-vm-conflict
spec:
  running: true
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: conflict-net
            bridge: {}
        resources:
          requests:
            memory: 1Gi
      networks:
      - name: conflict-net
        multus:
          networkName: conflict-network
EOF

# 3. Check for IP conflicts
oc get pods -o wide | grep -E "10\.128\.|172\.30\."
oc get vmi -o wide
```

**Workaround - Proper IPAM Configuration:**
```bash
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

## ⚙️ Testing External Connectivity & Load Balancing

### Test 4: Public IP Exposure

#### Step-by-Step Testing:

```bash
# 1. Create a VM with a service
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

# 2. Test internal connectivity
oc port-forward service/vm-web-service 8080:80 &
curl http://localhost:8080

# 3. Create a route for external access
oc expose service vm-web-service

# 4. Test external access
ROUTE_URL=$(oc get route vm-web-service -o jsonpath='{.spec.host}')
curl http://$ROUTE_URL
```

**Load Balancer Configuration Test:**
```bash
# 1. Create LoadBalancer service
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: vm-lb-service
spec:
  selector:
    kubevirt.io/vm: test-vm-bridge
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
EOF

# 2. Wait for external IP
oc get service vm-lb-service -w

# 3. Test external load balancer
EXTERNAL_IP=$(oc get service vm-lb-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$EXTERNAL_IP

# 4. Check AWS Load Balancer configuration
aws elbv2 describe-load-balancers --region <your-region>
aws elbv2 describe-target-groups --region <your-region>
```

### Test 5: NetworkPolicy Configuration

#### Step-by-Step Testing:

```bash
# 1. Create restrictive NetworkPolicy
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

# 2. Test blocked traffic
oc run blocked-pod --image=nginx
oc exec blocked-pod -- curl -I http://$VM_IP --timeout=5

# 3. Test allowed traffic
oc run allowed-pod --image=nginx --labels="app=allowed-app"
oc exec allowed-pod -- curl -I http://$VM_IP

# 4. Check egress restrictions
virtctl console test-vm-bridge
# Inside VM:
curl http://google.com --timeout=5  # Should fail
```

## 🚨 Testing Security and Compliance

### Test 6: Security Group Configuration

#### Step-by-Step Testing:

```bash
# 1. Get cluster security groups
CLUSTER_ID=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
aws ec2 describe-security-groups --filters "Name=tag:Name,Values=*$CLUSTER_ID*" --region <your-region>

# 2. Test port accessibility
WORKER_NODE_IP=$(oc get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')

# Test SSH access (should be blocked)
nc -zv $WORKER_NODE_IP 22

# Test allowed OpenShift ports
nc -zv $WORKER_NODE_IP 80
nc -zv $WORKER_NODE_IP 443

# 3. Create custom security group rule test
aws ec2 authorize-security-group-ingress \
  --group-id <worker-sg-id> \
  --protocol tcp \
  --port 8080 \
  --source-group <master-sg-id> \
  --region <your-region>
```

### Test 7: VLAN Support Testing

#### Step-by-Step Testing:

```bash
# 1. Check available network interfaces on nodes
oc debug node/<node-name>
# In debug container:
ip link show
lspci | grep -i ethernet

# 2. Create VLAN NetworkAttachmentDefinition
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-network
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vlan-network",
      "type": "macvlan",
      "master": "eth0.100",
      "mode": "bridge",
      "ipam": {
        "type": "static",
        "addresses": [
          {
            "address": "192.168.100.10/24",
            "gateway": "192.168.100.1"
          }
        ]
      }
    }
EOF

# 3. Test VLAN connectivity (will likely fail in ROSA)
oc apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vlan-test-vm
spec:
  running: true
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: vlan-interface
            bridge: {}
        resources:
          requests:
            memory: 1Gi
      networks:
      - name: vlan-interface
        multus:
          networkName: vlan-network
EOF
```

## 🎛️ Testing Troubleshooting and Observability

### Test 8: Layered Network Troubleshooting

#### Step-by-Step Debugging Process:

```bash
# Layer 1: VM Guest OS
virtctl console test-vm-bridge
# Inside VM:
ip addr show
ip route show
ping -c 3 8.8.8.8
netstat -tuln

# Layer 2: OpenShift CNI
oc get pods -n openshift-sdn
oc logs -n openshift-sdn <sdn-pod-name>

# Check OVS flows (if using OVN-Kubernetes)
oc debug node/<node-name>
# In debug container:
ovs-vsctl show
ovs-ofctl dump-flows br-int

# Layer 3: AWS Security Groups
aws ec2 describe-security-groups --group-ids <sg-id> --region <region>

# Layer 4: Load Balancer
aws elbv2 describe-target-health --target-group-arn <arn> --region <region>
```

### Test 9: Network Flow Visibility

#### Step-by-Step Monitoring:

```bash
# 1. Install network monitoring tools
oc apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

# 2. Capture network traffic
oc debug node/<node-name>
# In debug container:
tcpdump -i any -nn host $VM_IP

# 3. Monitor CNI logs
oc logs -f -n openshift-sdn -l app=sdn

# 4. Check cluster network operator logs
oc logs -f -n openshift-cluster-network-operator deployment/network-operator

# 5. Use network policy debugging
oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: netshoot
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
EOF

oc exec -it netshoot -- nslookup kubernetes.default
oc exec -it netshoot -- traceroute $VM_IP
```

### Test 10: Live Migration Network Testing

#### Step-by-Step Testing:

```bash
# 1. Enable live migration
cat <<EOF | oc apply -f -
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: openshift-cnv
spec:
  configuration:
    migrations:
      allowAutoConverge: true
      bandwidthPerMigration: 64Mi
      completionTimeoutPerGiB: 800
      progressTimeout: 150
EOF

# 2. Start continuous connectivity test
virtctl console test-vm-bridge
# Inside VM:
while true; do
  ping -c 1 8.8.8.8
  sleep 1
done

# 3. Trigger live migration in another terminal
virtctl migrate test-vm-bridge

# 4. Monitor migration status
oc get vmi test-vm-bridge -o jsonpath='{.status.migrationState}'
oc describe vmi test-vm-bridge

# 5. Check for network interruptions
oc logs -f -n openshift-cnv -l app=virt-handler

# 6. Verify VM accessibility during migration
# Keep pinging from external source:
while true; do
  curl -I http://$ROUTE_URL --timeout=2
  sleep 1
done
```

## 🔧 Common Workarounds and Best Practices

### Workaround 1: VM-to-VPC Connectivity
```bash
# Use NodePort services as proxy
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: vm-nodeport-proxy
spec:
  type: NodePort
  selector:
    kubevirt.io/vm: test-vm-bridge
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
EOF

# Access VM through node IP
NODE_IP=$(oc get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
curl http://$NODE_IP:30080
```

### Workaround 2: Advanced Networking with Multus
```bash
# Create advanced network configuration
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: advanced-network
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "advanced-network",
      "plugins": [
        {
          "type": "bridge",
          "bridge": "adv-br0",
          "ipam": {
            "type": "host-local",
            "subnet": "10.200.0.0/16"
          }
        },
        {
          "type": "tuning",
          "sysctl": {
            "net.core.somaxconn": "500"
          }
        }
      ]
    }
EOF
```

### Troubleshooting Commands Cheat Sheet
```bash
# Quick network diagnostics
oc get nodes -o wide
oc get pods -o wide -A | grep -E "(Running|Pending|Error)"
oc get vmi -A
oc get svc -A
oc get routes -A

# Network policy debugging
oc get networkpolicy -A
oc describe networkpolicy <name> -n <namespace>

# CNI troubleshooting
oc get network.operator cluster -o yaml
oc logs -n openshift-sdn -l app=sdn
oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node

# VM-specific networking
virtctl guestosinfo <vm-name>
virtctl interface-list <vm-name>
```

## 📊 Testing Results Documentation Template

Create a spreadsheet or document to track your testing results:

| Test Case | Expected Result | Actual Result | Issues Found | Workaround Applied | Status |
|-----------|----------------|---------------|--------------|-------------------|---------|
| VM to Pod connectivity | Success | Success | None | N/A | ✅ Pass |
| VM to VPC direct | Fail | Fail | Expected limitation | NodePort proxy | ⚠️ Known Issue |
| External LB access | Success | Timeout | SG misconfiguration | Fixed SG rules | ✅ Pass |

This comprehensive testing guide will help you identify and work around the networking challenges in ROSA with OpenShift Virtualization.