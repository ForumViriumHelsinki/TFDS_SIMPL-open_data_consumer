# TFDS SIMPL-open Data Consumer Agent

> **Notice: TFDS Project Fork**
> This repository is a GitOps-optimized fork maintained by the TFDS Project. 
> 
> **Modifications in this repository:**
> - Substituted the upstream `authentication-provider` dependency with a TFDS-maintained GitOps fork to eliminate declarative synchronization drift in ArgoCD caused by dynamically generated Job names.
> - Overridden `tier2_gateway` service types to `ClusterIP` and added `singleNode` SSL passthrough routing to accommodate local multi-agent deployments on single-IP k3s clusters.
> - Decoupled the Governance Authority `domainSuffix` variable from the agent's base domain to enable true cross-cluster federation.
> - Disabled the default `crossplane` provisioning engine to prevent severe resource exhaustion on local development clusters.
>
> *The core SIMPL applications are unmodified and dynamically pulled from official registries under EUPL 1.2.*

This repository contains the deployment manifests and configurations for the **SIMPL-Open Data Consumer** agent, adapted for both distributed environments and local, single-node Kubernetes (k3s) deployments.

## 🚀 Quick Start Deployment Guide

This guide is designed for users who want to quickly deploy the Data Consumer agent using **ArgoCD**. While these instructions are optimized for a local, single-node k3s cluster, the agent is fully capable of distributed deployments.

### Prerequisites

Before deploying the Data Consumer, ensure your environment meets the following requirements:
1. **Running Kubernetes Cluster:** A standard multi-node cluster or a single-node setup (like k3s with MetalLB) with an `nginx` Ingress Controller.
2. **Common Components Deployed:** The core SIMPL infrastructure (Kafka, PostgreSQL, Elastic, OpenBao) must already be running in the `common` namespace. You can find the deployment guide for this layer in the [Common Components Repository](https://github.com/ForumViriumHelsinki/TFDS_SIMPL-open_common_components).
3. **DNS Routing:** A wildcard DNS record (e.g., `*.consumer.yourdomain.com`) pointing to your MetalLB public IP.
4. **Governance Authority:** You need the domain name of the external Governance Authority cluster you will federate with (e.g., `ds.helsinki.tfds.io`).

---

### Step 1: Configure the ArgoCD Manifest

All configuration for the deployment is managed through a single file: `ArgoCD/data-consumer_manifest.yaml`.

Open this file and verify or update the `values` block to match your environment:

1. **Namespace Tags:** Update the identifiers for your namespaces:
   ```yaml
   namespaceTag:
     consumer: consumer             # Your data consumer namespace
     authority: authority           # The governance authority namespace
     common: common                 # Your common components namespace
   ```
2. **Domain Federation:** Update the domain suffixes to ensure secure cross-cluster communication:
   ```yaml
   domainSuffix: org-x.helsinki.tfds.io          # Your local cluster's base domain
   authorityDomainSuffix: ds.helsinki.tfds.io    # The external Governance Authority's base domain
   ```
3. **Single-Node Mode (Optional):** If deploying to a single-node environment with a single IP, ensure `singleNode: true` is set in the manifest. For standard distributed clusters, remove this or set it to `false`.
4. **Cloud Provisioning (Crossplane):**  By default, this is set to `enabled: false`. **Leave this as false** for local k3s deployments. Setting this to true will attempt to provision heavy cloud-native components (IONOS/OVH) that will crash a vanilla k3s node.
5. **Monitoring (Elastic/Filebeat):** Configure the logging and metrics agents:
   ```yaml
   monitoring:
     enabled: false                     # should monitoring be disabled
   ```
   *   **Impact if `false` (Recommended for Local):** Skips deploying Filebeat and Metricbeat log shippers to this namespace. This saves significant CPU and Memory on your local node. For troubleshooting, you will rely on standard `kubectl logs <pod-name>` commands instead of a centralized dashboard.
   *   **Impact if `true`:** Deploys log shippers. **Warning:** This requires the heavy Elastic stack (`eck-operator`, Elasticsearch, Kibana) to be running and healthy in your `common` namespace. Enabling this on a constrained single-node cluster will likely lead to severe resource exhaustion and pod evictions.

---

### Step 2: Pre-Deployment Secrets (OpenBao)

Before you deploy the agent, you **must** manually inject your MinIO (S3) credentials into the OpenBao (Vault) server. The Eclipse Dataspace Connector (EDC) requires these to boot properly and handle data transfers.

1. Log in to your OpenBao UI (e.g., `https://secrets.common.yourdomain.com`) using your Root Token.
2. Navigate to the Key/Value secrets engine.
3. Locate or create the secret named `<your-consumer-namespace>-simpl-edc` (e.g., `consumer-simpl-edc`).
4. Add the following keys with your specific MinIO details:
   * `fr_gxfs_s3_access_key`: Your MinIO Access Key (e.g., `admin`)
   * `fr_gxfs_s3_secret_key`: Your MinIO Secret Key
   * `fr_gxfs_s3_endpoint`: Your MinIO API URL (e.g., `https://s3.yourdomain.com`)

---

### Step 3: Deploy the Agent

Once your configuration is set and your secrets are in OpenBao, you can trigger the deployment using ArgoCD:

```bash
kubectl apply -f ArgoCD/data-consumer_manifest.yaml
```

ArgoCD will automatically read the configuration and begin spinning up the Data Consumer agent in the specified namespace.

---

### Step 4: Expected Behavior & Participant Onboarding

**⚠️ IMPORTANT: Please read this before troubleshooting!**

When you monitor the deployment in ArgoCD or via `kubectl get pods`, you will notice that three specific components appear to be "stuck" or "crashing":

*   🔴 **`tier2-gateway`:** Stuck in `Init:0/1`
*   🔴 **`tier2-proxy`:** Stuck in `Init:0/1`
*   🔴 **`simpl-edc`:** Crashing (`CrashLoopBackOff` / `NullPointerException`)

**This is 100% expected behavior.**

These components act as the secure trust gateways for the data space. They are intentionally stalling because they are waiting for business-level certificates from the Governance Authority.

**How to fix them:**
You must complete the manual **Participant Onboarding** process through the newly deployed Data Consumer UI. Once the agent is officially onboarded and the certificates are generated and synced, these pods will automatically recover, initialize, and turn green.

*Reference the official [SIMPL Open Onboarding Manual](https://code.europa.eu/simpl/simpl-open/development/iaa/documentation/-/blob/main/versioned_docs/2.5.x/user-manual/ONBOARD.md) to complete this final step.*
