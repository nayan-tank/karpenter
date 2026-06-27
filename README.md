# Cluster Autoscaler vs. Karpenter

Both solve the same problem — adding and removing nodes so pods can schedule — but they operate at different layers.

## Cluster Autoscaler (CA)

Operates on **pre-defined node groups** (GKE Managed Instance Groups, EKS ASGs). You decide the machine types up front by creating node pools; CA only adjusts the *count* within each group's min/max.

- When pods are pending, it picks an existing group that fits and scales it up.
- When nodes sit empty past a threshold, it scales them down.

## Karpenter

Is **groupless**. It watches pending pods, reads their actual requirements (CPU/mem, arch, zone, taints, topology spread), and provisions a right-sized node directly from the cloud's compute API — choosing instance type, size, and capacity type (spot/on-demand) on the fly.

- It also **consolidates**: actively replaces or bin-packs nodes to cut waste, not just removes empties.

## Side-by-side

| Dimension | Cluster Autoscaler | Karpenter |
|---|---|---|
| Unit of scaling | Node group / MIG (fixed shape) | Individual node, shape decided per-workload |
| Instance selection | You define pools in advance | Picked dynamically from a broad pool |
| Provisioning speed | Slower (MIG/ASG indirection) | Faster (direct compute API calls) |
| Bin-packing / consolidation | Scale-down of empty nodes only | Active consolidation + node replacement |
| Spot handling | Manual via separate pools | Native, mixed with on-demand in one NodePool |
| Config model | Cloud-specific autoscaling settings | NodePool + NodeClass CRDs |
| Maturity | Very mature, all clouds | GA on AWS; mature on Azure |

## The GKE reality

Karpenter's first-class support is **AWS EKS**, with **Azure** now GA as a managed AKS addon. For GKE there is **no official `kubernetes-sigs/karpenter-provider-gcp`** — GKE still runs on Cluster Autoscaler backed by Managed Instance Groups, with Node Auto-Provisioning and Compute Classes layered on top.

A community provider exists (CloudPilot AI's `karpenter-provider-gcp`), but **GCP support is alpha/preview**. Public GKE deployment attempts have hit IAM issues and a "default node template not found" bug serious enough to revert to GKE's Cluster Autoscaler.

So on GKE the practical comparison is really **Cluster Autoscaler vs. Node Auto-Provisioning (NAP) + Compute Classes**, Google's answer to Karpenter-style dynamic provisioning:

- **CA alone** — you manage node pool shapes; good when workloads are homogeneous.
- **CA + NAP** — NAP creates and deletes node pools automatically based on pending pods, approaching Karpenter's dynamic-shape behavior natively.
- **Compute Classes** — declare prioritized fallback lists of machine families (e.g., prefer spot N4, fall back to on-demand), covering much of Karpenter's flexible instance selection.

## Recommendation for a multi-region GKE footprint

For production multi-region GKE (e.g., Mumbai / Sydney / Dallas), stay on **CA and lean into NAP + Compute Classes** rather than the alpha GCP Karpenter provider. You get most of the consolidation and flexibility benefit without depending on a pre-GA third-party controller.

```
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set "settings.enableZonalShift=${ENABLE_ZONAL_SHIFT}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

```
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h # 30 * 24h = 720h
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  role: "KarpenterNodeRole-${CLUSTER_NAME}" # replace with your cluster name
  amiSelectorTerms:
    - alias: "al2023@${ALIAS_VERSION}"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
EOF
```
