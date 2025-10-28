---
title: "LLMInferenceService Installation Guide"
description: "Install LLMInferenceService for generative AI inference workloads"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# LLMInferenceService Installation

LLMInferenceService is KServe's dedicated solution for **Generative AI inference workloads**, providing advanced features like:

- **Intelligent Routing**: KV cache-aware scheduling, prefill-decode separation
- **Multi-Node Orchestration**: Data parallelism, expert parallelism via LeaderWorkerSet
- **Gateway API Native**: Built on Kubernetes Gateway API with Inference Extension
- **Autoscaling**: Integration with KEDA for custom metrics-based scaling

:::note
LLMInferenceService is designed specifically for **Generative AI** workloads (LLMs). For **Predictive AI** workloads, use [InferenceService](kubernetes-deployment.md).
:::

## Installation Requirements

### Minimum Requirements

- **Kubernetes**: Version 1.30+
- **Cert Manager**: Version 1.16.0+
- **Gateway API**: Version 1.2.1
- **Gateway API Inference Extension (GIE)**: Version 0.3.0
- **Gateway Provider**: Envoy Gateway v1.5.0+ 
- **LeaderWorkerSet**: Version 0.7.0+ (for multi-node deployments)

:::tip
For detailed dependency information and step-by-step installation, see [LLMInferenceService Dependencies](../model-serving/generative-inference/llmisvc/llmisvc-dependencies.md).
:::

## Prerequisites

- Kubernetes cluster (v1.30+)
- `kubectl` configured to access your cluster
- Cluster admin permissions
- `helm` v3+ installed

---

## Quick Install (Recommended)

The fastest way to get started with LLMInferenceService is using the quick install script.

### 1. Clone KServe Repository

```bash
git clone https://github.com/kserve/kserve.git
cd kserve
```

### 2. Run Quick Install Script

<Tabs>
<TabItem value="full" label="Full Installation" default>

Install all dependencies and LLMInferenceService:

```bash
./hack/setup/quick-install/llmisvc-full-install-helm.sh
```

**What gets installed**:
1. ✅ Cert Manager (v1.16.1)
2. ✅ Gateway API CRDs (v1.2.1)
3. ✅ Gateway API Inference Extension (v0.3.0)
4. ✅ Envoy Gateway (v1.5.0)
5. ✅ Envoy AI Gateway (v0.3.0) - for token rate limiting
6. ✅ LeaderWorkerSet (v0.7.0) - for multi-node deployments
7. ✅ GatewayClass (envoy)
8. ✅ Gateway (kserve-ingress-gateway)
9. ✅ LLMInferenceService CRDs and Controller (v0.16.0-rc1)

**Installation time**: ~5-10 minutes

</TabItem>
<TabItem value="dependencies" label="Dependencies Only">

Install only dependencies without LLMInferenceService controller:

```bash
./hack/setup/quick-install/llmisvc-dependency-install.sh
```

This is useful when you want to:
- Install LLMInferenceService controller manually later
- Use a specific version of LLMInferenceService
- Customize LLMInferenceService installation with specific Helm values

After installing dependencies, you can install LLMInferenceService controller separately:

```bash
# Install LLMInferenceService CRDs
helm install llmisvc-crd oci://ghcr.io/kserve/charts/llmisvc-crd \
  --version v0.16.0-rc1 \
  --namespace kserve \
  --create-namespace

# Install LLMInferenceService Controller
helm install llmisvc oci://ghcr.io/kserve/charts/llmisvc-resources \
  --version v0.16.0-rc1 \
  --namespace kserve
```

</TabItem>
<TabItem value="uninstall" label="Uninstall">

Uninstall all components:

```bash
./hack/setup/quick-install/llmisvc-uninstall.sh
```

This removes:
- LLMInferenceService controller and CRDs
- All dependencies (Envoy Gateway, LWS, Cert Manager, etc.)
- Namespaces (kserve, lws-system, envoy-gateway-system, etc.)
- Gateway resources

</TabItem>
</Tabs>

:::note[Local Development]
The quick install script automatically configures **MetalLB** if detected (e.g., in minikube), providing LoadBalancer support for local testing. For kind clusters, see [Local Development Setup](#local-development-setup) below.
:::

---

## Verification

After installation, verify all components are working:

```bash
# Check all pods are running
kubectl get pods -n cert-manager
kubectl get pods -n envoy-gateway-system
kubectl get pods -n envoy-ai-gateway-system
kubectl get pods -n lws-system
kubectl get pods -n kserve

# Check LLMInferenceService CRD
kubectl get crd llminferenceservices.serving.kserve.io

# Check Gateway status
kubectl get gateway kserve-ingress-gateway -n kserve

# Check Gateway has external IP (may take a few minutes)
kubectl get gateway kserve-ingress-gateway -n kserve -o jsonpath='{.status.addresses[0].value}'
```

**Expected output**:
- ✅ All pods in `Running` state
- ✅ Gateway shows `READY: True`
- ✅ Gateway has `EXTERNAL-IP` or `ADDRESS` assigned

---

## Manual Installation

For production environments or custom configurations, follow the detailed step-by-step installation guide:

**📖 [LLMInferenceService Dependencies - Installation Order](../model-serving/generative-inference/llmisvc/llmisvc-dependencies.md#installation-order-critical-sequence)**

The manual installation covers:
1. Installing each component individually with version pinning
2. Understanding why installation order matters (GIE before Gateway Provider)
3. Choosing between Gateway Providers (Envoy Gateway vs Istio)
4. Configuring for production environments
5. OpenShift-specific installation steps

---

## Next Steps

Now that LLMInferenceService is installed, you can:

1. **Deploy Your First LLM**: Follow the [Quick Start Guide](../getting-started/genai-first-llmisvc.md)
2. **Understand the Architecture**: Read [LLMInferenceService Overview](../model-serving/generative-inference/llmisvc/llmisvc-overview.md)
3. **Explore Configuration Options**: Check [LLMInferenceService Configuration](../model-serving/generative-inference/llmisvc/llmisvc-configuration.md)
4. **Learn Advanced Features**:
   - [Multi-Node Deployments](https://github.com/kserve/kserve/tree/main/docs/samples/llmisvc/dp-ep) - Data/Expert parallelism
   - [Prefill-Decode Separation](../model-serving/generative-inference/llmisvc/llmisvc-architecture.md#prefill-decode-separation) - Performance optimization
   - [Autoscaling](../model-serving/generative-inference/autoscaling/autoscaling.md) - Scale based on metrics

---

## Troubleshooting

### Gateway Not Getting External IP

**Symptom**: Gateway service stuck in `Pending` state

```bash
kubectl get gateway kserve-ingress-gateway -n kserve
# READY: False, ADDRESS: <pending>
```

**Solutions**:
- **Cloud Platform**: Check if your cluster supports LoadBalancer services (AWS ELB, GCP Load Balancer, etc.)
- **Local (kind)**: Install [cloud-provider-kind](#local-development-setup)
- **Local (minikube)**: Enable [MetalLB addon](#local-development-setup)
- **Testing**: Use [port-forwarding](#local-development-setup) as a workaround

### GIE CRDs Not Recognized

**Symptom**: HTTPRoute cannot reference InferencePool as backend

```bash
kubectl get crd | grep inference.networking
# No output
```

**Solutions**:
1. Install GIE CRDs: `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v0.3.0/install.yaml`
2. Restart Gateway Provider: `kubectl rollout restart -n envoy-gateway-system deployment/envoy-gateway`
3. Verify GIE is installed: `kubectl get crd inferencepools.inference.networking.x-k8s.io`

**Root Cause**: GIE must be installed **before** Gateway Provider. See [Dependencies - Installation Order](../model-serving/generative-inference/llmisvc/llmisvc-dependencies.md#installation-order-critical-sequence).

### LWS Pods Not Starting

**Symptom**: LeaderWorkerSet pods stuck in `Pending` or `ContainerCreating`

**Solutions**:
1. Check cert-manager is running: `kubectl get pods -n cert-manager`
2. Check LWS webhooks: `kubectl get validatingwebhookconfigurations | grep lws`
3. Check LWS operator logs: `kubectl logs -n lws-system deployment/lws-controller-manager`
4. Verify Kubernetes version: `kubectl version --short` (must be 1.30+)

### LLMInferenceService Controller Not Starting

**Symptom**: `llmisvc-controller-manager` pod in CrashLoopBackOff

**Solutions**:
1. Check controller logs: `kubectl logs -n kserve deployment/llmisvc-controller-manager`
2. Verify all CRDs installed: `kubectl get crd | grep llminferenceservice`
3. Check RBAC permissions: `kubectl get clusterrole llmisvc-controller-manager-role`
4. Ensure Gateway and GatewayClass exist: `kubectl get gateway,gatewayclass`

---

## Advanced Configuration

### Installing Specific Versions

```bash
# Edit version variables in the script
export KSERVE_VERSION=v0.16.0-rc1
export LLMISVC_VERSION=v0.16.0-rc1
export LWS_VERSION=0.7.0
export ENVOY_GATEWAY_VERSION=v1.5.0

# Run installation
./hack/setup/quick-install/llmisvc-full-install-helm.sh
```

### Using Local Charts (Development)

```bash
# Build local charts first
cd charts/
helm package llmisvc-crd
helm package llmisvc-resources

# Install with local charts
export USE_LOCAL_CHARTS=true
./hack/setup/quick-install/llmisvc-full-install-helm.sh
```

---

## Uninstallation

To completely remove LLMInferenceService and all dependencies:

```bash
# Using quick install script (recommended)
./hack/setup/quick-install/llmisvc-uninstall.sh
```

**Or manually**:

```bash
# Uninstall LLMInferenceService
helm uninstall llmisvc -n kserve
helm uninstall llmisvc-crd -n kserve

# Uninstall dependencies
helm uninstall lws -n lws-system
helm uninstall eg -n envoy-gateway-system
helm uninstall aieg aieg-crd -n envoy-ai-gateway-system
helm uninstall cert-manager -n cert-manager

# Delete Gateway resources
kubectl delete gateway kserve-ingress-gateway -n kserve
kubectl delete gatewayclass envoy

# Delete namespaces
kubectl delete namespace kserve lws-system envoy-gateway-system envoy-ai-gateway-system cert-manager
```

:::warning
Uninstallation will delete all LLMInferenceService resources and their data. Make sure to backup any important configurations before proceeding.
:::
