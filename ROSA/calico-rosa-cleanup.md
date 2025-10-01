# ROSA HCP Cluster - Calico CNI Clean Installation Guide

## Cluster Information
- **Cluster API**: `api.nddemo.qftf.p3.openshiftapps.com`
- **Service IP**: `172.30.0.1`
- **Pod CIDR**: `10.128.0.0/14`

---

## Phase 1: Complete Cleanup

### 1.1 Remove Failed Tigera Operator Installation

```bash
# Delete any existing subscriptions
oc delete subscription tigera-operator -n openshift-operators --ignore-not-found=true

# Delete install plans
oc get installplan -n openshift-operators -o name | xargs oc delete -n openshift-operators --ignore-not-found=true

# Delete CSV if exists
oc get csv -n openshift-operators | grep tigera | awk '{print $1}' | xargs oc delete csv -n openshift-operators --ignore-not-found=true
```

### 1.2 Remove Tigera Custom Resources

```bash
# Delete Installation CRs
oc get installation.operator.tigera.io -A -o name | xargs oc delete --ignore-not-found=true

# Delete APIServer CRs
oc get apiserver.operator.tigera.io -A -o name | xargs oc delete --ignore-not-found=true
```

### 1.3 Remove Tigera CRDs

```bash
# List and delete Tigera CRDs
oc get crd | grep tigera | awk '{print $1}' | xargs oc delete crd --ignore-not-found=true

# List and delete Calico CRDs
oc get crd | grep calico | awk '{print $1}' | xargs oc delete crd --ignore-not-found=true
```

### 1.4 Delete Tigera Namespaces

```bash
# Delete operator namespace
oc delete namespace tigera-operator --ignore-not-found=true --grace-period=0

# Delete calico namespaces
oc delete namespace calico-system --ignore-not-found=true --grace-period=0
oc delete namespace calico-apiserver --ignore-not-found=true --grace-period=0
```

### 1.5 Remove Stuck Namespaces (if needed)

If namespaces are stuck in `Terminating` state:

```bash
# Force finalize namespaces
for ns in tigera-operator calico-system calico-apiserver; do
  oc get ns $ns -o json 2>/dev/null | jq '.spec.finalizers = []' | oc replace --raw "/api/v1/namespaces/$ns/finalize" -f - 2>/dev/null || true
done
```

### 1.6 Verify Clean State

```bash
# Verify no tigera/calico resources remain
oc get all -A | grep -E "tigera|calico"
oc get crd | grep -E "tigera|calico"
oc get namespace | grep -E "tigera|calico"

# Check current node status
oc get nodes
```

---

## Phase 2: Direct Tigera Operator Installation

### 2.1 Install Tigera Operator Manifest

```bash
# Install operator directly (not via OLM)
oc create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
```

**Note**: You may see pod security warnings - these are expected and can be ignored.

### 2.2 Apply Pod Security Labels

```bash
# Apply privileged pod security labels to tigera-operator namespace
oc label namespace tigera-operator pod-security.kubernetes.io/enforce=privileged --overwrite
oc label namespace tigera-operator pod-security.kubernetes.io/audit=privileged --overwrite
oc label namespace tigera-operator pod-security.kubernetes.io/warn=privileged --overwrite
```

### 2.3 Fix Kubernetes API Connectivity

The operator needs to connect to the Kubernetes API. Set the correct service host:

```bash
# Set Kubernetes service environment variables
oc set env deployment/tigera-operator -n tigera-operator \
  KUBERNETES_SERVICE_HOST=kubernetes.default.svc.cluster.local \
  KUBERNETES_SERVICE_PORT=443

# Restart the deployment
oc rollout restart deployment/tigera-operator -n tigera-operator
```

### 2.4 Verify Operator is Running

```bash
# Wait for deployment to be ready
oc rollout status deployment/tigera-operator -n tigera-operator --timeout=300s

# Check operator pod status
oc get pods -n tigera-operator

# Check operator logs (should show no DNS errors)
oc logs -n tigera-operator deployment/tigera-operator --tail=30
```

Expected output: Pod in `Running` state with no DNS lookup errors in logs.

---

## Phase 3: Deploy Calico CNI

### 3.1 Create Installation Custom Resource

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

### 3.2 Verify Installation CR Created

```bash
# Check Installation resource
oc get installation

# View Installation status
oc get installation default -o yaml
```

### 3.3 Wait for calico-system Namespace

```bash
# Watch for namespace creation (should appear within 30 seconds)
watch -n 2 'oc get namespace | grep calico'
```

Press `Ctrl+C` once `calico-system` appears.

### 3.4 Apply Pod Security Labels to calico-system

```bash
# Apply privileged labels to calico-system namespace
oc label namespace calico-system pod-security.kubernetes.io/enforce=privileged --overwrite
oc label namespace calico-system pod-security.kubernetes.io/audit=privileged --overwrite
oc label namespace calico-system pod-security.kubernetes.io/warn=privileged --overwrite
```

### 3.5 Monitor Calico Component Deployment

```bash
# Watch Calico pods come up
watch -n 2 'oc get pods -n calico-system'
```

Expected components:
- `calico-kube-controllers` (Deployment)
- `calico-node` (DaemonSet - one pod per node)
- `calico-typha` (Deployment)

Press `Ctrl+C` once all pods show `Running` status.

### 3.6 Verify Calico DaemonSet

```bash
# Check DaemonSet status
oc get ds -n calico-system

# Verify calico-node pods on all nodes
oc get pods -n calico-system -o wide -l k8s-app=calico-node
```

---

## Phase 4: Verify Node Readiness

### 4.1 Monitor Node Status

```bash
# Watch nodes transition to Ready
watch -n 2 'oc get nodes'
```

Nodes should transition from `NotReady` to `Ready` within 3-5 minutes.

### 4.2 Check Node Details

```bash
# View node conditions
oc get nodes -o wide

# Check specific node details
oc describe node <node-name> | grep -A 10 "Conditions:"
```

### 4.3 Verify CNI Configuration on Nodes

```bash
# Check that CNI config exists
oc debug node/<node-name> -- chroot /host ls -la /etc/cni/net.d/

# Expected: 10-calico.conflist file should exist
```

---

## Phase 5: Validate Networking

### 5.1 Create Test Pod

```bash
# Create a test pod
oc run test-networking --image=registry.access.redhat.com/ubi8/ubi-minimal --command -- sleep 3600

# Wait for pod to be running
oc wait --for=condition=Ready pod/test-networking --timeout=120s
```

### 5.2 Test DNS Resolution

```bash
# Test internal DNS
oc exec test-networking -- nslookup kubernetes.default

# Test external DNS
oc exec test-networking -- nslookup google.com
```

### 5.3 Test Service Connectivity

```bash
# Test Kubernetes API service
oc exec test-networking -- curl -k https://kubernetes.default.svc.cluster.local

# Check pod IP assignment
oc get pod test-networking -o wide
```

### 5.4 Verify Pod-to-Pod Communication

```bash
# Create second test pod
oc run test-networking-2 --image=registry.access.redhat.com/ubi8/ubi-minimal --command -- sleep 3600

# Get pod IPs
POD1_IP=$(oc get pod test-networking -o jsonpath='{.status.podIP}')
POD2_IP=$(oc get pod test-networking-2 -o jsonpath='{.status.podIP}')

# Test connectivity from pod1 to pod2
oc exec test-networking -- ping -c 3 $POD2_IP
```

### 5.5 Cleanup Test Pods

```bash
oc delete pod test-networking test-networking-2
```

---

## Phase 6: Verify Installation Status

### 6.1 Check All Cluster Operators

```bash
# View all cluster operators
oc get clusteroperators

# Check network operator specifically
oc get clusteroperator network
```

### 6.2 Check Installation CR Status

```bash
# Get Installation status
oc get installation default -o jsonpath='{.status}' | jq .

# Should show: "state": "Ready"
```

### 6.3 Verify All Calico Components

```bash
# List all resources in calico-system
oc get all -n calico-system

# Check for any errors in events
oc get events -n calico-system --sort-by='.lastTimestamp' | tail -20
```

### 6.4 Final Verification Commands

```bash
# Summary check
echo "=== Nodes ==="
oc get nodes

echo "=== Calico Pods ==="
oc get pods -n calico-system

echo "=== All Pods Cluster-Wide ==="
oc get pods -A | head -20

echo "=== Installation Status ==="
oc get installation default
```

---

## Troubleshooting Guide

### Issue: Operator Pod Not Starting

**Check logs:**
```bash
oc logs -n tigera-operator deployment/tigera-operator
oc describe pod -n tigera-operator
oc get events -n tigera-operator --sort-by='.lastTimestamp'
```

**Common fixes:**
- Verify pod security labels are applied
- Check for image pull errors
- Verify KUBERNETES_SERVICE_HOST environment variable is set

### Issue: DNS Lookup Errors

**Symptoms:** Operator logs show `no such host` errors

**Fix:**
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

### Issue: calico-system Namespace Not Created

**Check:**
```bash
# Verify Installation CR exists
oc get installation default

# Check operator logs for errors
oc logs -n tigera-operator deployment/tigera-operator --tail=50

# Describe Installation for status
oc describe installation default
```

**Fix:**
```bash
# Delete and recreate Installation CR
oc delete installation default
# Wait 30 seconds
sleep 30
# Recreate (use command from Phase 3.1)
```

### Issue: Calico Pods in CrashLoopBackOff

**Check pod logs:**
```bash
# Check calico-node logs
oc logs -n calico-system daemonset/calico-node

# Check specific pod
oc logs -n calico-system <pod-name>
oc describe pod -n calico-system <pod-name>
```

**Common issues:**
- Pod security labels not applied → Apply labels from Phase 3.4
- RBAC permissions → Check operator logs
- Network conflicts → Verify IP pool CIDR doesn't conflict

### Issue: Nodes Still NotReady

**Check node conditions:**
```bash
# Describe node
oc describe node <node-name>

# Check kubelet logs on node
oc adm node-logs <node-name> -u kubelet | tail -100

# Verify calico-node pod on that node
oc get pods -n calico-system -o wide | grep <node-name>
```

**Fix:**
```bash
# Check if CNI plugin is present on node
oc debug node/<node-name> -- chroot /host ls -la /etc/cni/net.d/
oc debug node/<node-name> -- chroot /host ls -la /opt/cni/bin/

# Restart calico-node pod on affected node
oc delete pod -n calico-system <calico-node-pod-name>
```

### Issue: Stuck Resources During Cleanup

**Remove finalizers from CRDs:**
```bash
# List CRDs with finalizers
oc get crd | grep tigera | awk '{print $1}' | while read crd; do
  echo "Checking $crd"
  oc get crd $crd -o jsonpath='{.metadata.finalizers}'
done

# Remove finalizers
oc get crd | grep tigera | awk '{print $1}' | while read crd; do
  oc patch crd $crd -p '{"metadata":{"finalizers":[]}}' --type=merge
done
```

### Issue: Installation CR Shows Degraded

**Check status:**
```bash
oc get installation default -o yaml | grep -A 20 status
```

**Get detailed conditions:**
```bash
oc describe installation default
```

---

## Expected Timeline

| Phase | Expected Duration |
|-------|-------------------|
| Cleanup (Phase 1) | 2-3 minutes |
| Operator Installation (Phase 2) | 2-3 minutes |
| Calico Deployment (Phase 3) | 3-5 minutes |
| Node Ready (Phase 4) | 3-5 minutes after Calico pods running |
| **Total Time** | **10-15 minutes** |

---

## Success Criteria

Your installation is successful when all of the following are true:

✅ Tigera operator pod is `Running` in `tigera-operator` namespace  
✅ All Calico pods are `Running` in `calico-system` namespace  
✅ `calico-node` DaemonSet has pods on all worker nodes  
✅ All nodes show status `Ready`  
✅ Installation CR shows `state: Ready`  
✅ Test pods can resolve DNS and communicate  
✅ New pods get IP addresses from the configured IP pool

---

## Important Notes for ROSA HCP

1. **ROSA HCP uses hosted control plane** - some operations differ from standard OpenShift
2. **OVN-Kubernetes is the default CNI** - Calico is replacing it
3. **Pod security is enforced** - privileged labels are required for network operators
4. **Direct manifest installation is more reliable** than OLM for CNI operators
5. **DNS resolution can be tricky** - setting KUBERNETES_SERVICE_HOST explicitly helps

---

## Quick Reference Commands

```bash
# Check overall status
oc get nodes && oc get pods -n calico-system && oc get installation

# View operator logs
oc logs -n tigera-operator deployment/tigera-operator -f

# View calico-node logs
oc logs -n calico-system daemonset/calico-node -f

# Restart operator
oc rollout restart deployment/tigera-operator -n tigera-operator

# Restart all calico-node pods
oc delete pods -n calico-system -l k8s-app=calico-node

# Full cleanup and restart (emergency)
oc delete installation default
oc rollout restart deployment/tigera-operator -n tigera-operator
# Wait 60 seconds then recreate Installation CR
```

---

## Support Resources

- **Calico Documentation**: https://docs.tigera.io/calico/latest/getting-started/kubernetes/openshift/installation
- **ROSA Documentation**: https://docs.openshift.com/rosa/welcome/index.html
- **Red Hat Support**: https://access.redhat.com/support/
- **Tigera Support**: https://www.tigera.io/support/

---

## Completion Checklist

After completing all phases, verify:

- [ ] All cleanup commands executed successfully
- [ ] Tigera operator pod is running without errors
- [ ] calico-system namespace exists
- [ ] All Calico pods are running (calico-node, calico-kube-controllers, calico-typha)
- [ ] calico-node DaemonSet has correct number of pods (matches worker node count)
- [ ] All worker nodes show `Ready` status
- [ ] Test pod can be created and gets an IP address
- [ ] DNS resolution works from test pod
- [ ] Pod-to-pod communication works
- [ ] Installation CR shows `state: Ready`

**Your ROSA HCP cluster networking is now fully operational with Calico CNI!**
