# PoC: Automated OpenShift Manifest Migration

## 🚀 Overview

When migrating resources between OpenShift clusters using `oc get deployment -o yaml`, the resulting manifests are cluttered with cluster-specific metadata (`uid`, `resourceVersion`, `managedFields`) and networking state (`clusterIP`, `status`).

This project demonstrates a **Hybrid Migration Pipeline** that uses **Helm** as a configuration generator for **Kustomize**. This allows us to dynamically clean "junk" data and inject target-cluster configurations (images, replicas, labels) without manually editing YAML files.

## 🛠 The Architecture

We use a two-stage transformation:

1. **Helm Stage**: Generates a `kustomization.yaml` based on your input `values.yaml`. It dynamically scans the `source-manifests/` directory using `.Files.Glob`.
2. **Kustomize Stage**: Performs the "surgical" cleanup and applies patches to the raw source files using **Strategic Merge Patches**.

## 📁 Repository Structure

```text
.
├── .gitignore              # Ignores local build and source files
├── .helmignore             # Prevents local artifacts from being packaged
├── Chart.yaml              # Helm metadata
├── values.yaml             # Migration configuration (Image, Replicas, Selectors)
├── source-manifests/       # DROP YOUR "DIRTY" EXPORTS HERE
│   └── .gitkeep            # Ensures directory exists in Git
└── templates/
    └── kustomization.yaml  # The engine: Templates the Kustomize instructions

```

---

## ⚙️ Configuration (`values.yaml`)

Users can control the migration behavior via the `values.yaml` file:

| Parameter | Description | Requirement |
| --- | --- | --- |
| `targetLabelSelector` | Only clean/patch resources matching this label (e.g., `app=web`). If omitted, applies to **all** resources. | Optional |
| `image` | The new container image for the target cluster. | Optional |
| `replicas` | Desired scale in the target cluster. | Optional |
| `commonLabels` | Labels added to **every** migrated resource. | Optional |
| `commonAnnotations` | Annotations for metadata tracking (e.g., `migrated-from`). | Optional |

---

## 📖 Usage Instructions

### 1. Export your source manifests

To demonstrate how Kustomize selectively filters resources, we first export all Deployments and Services from the source namespace into individual files:

```bash
# Export every Deployment and Service in the current namespace
for r in $(oc get deploy,svc -o name --context=source-cluster); do
    # Converts 'deployment/my-app' to 'deployment-my-app.yaml'
    oc get $r -o yaml > "source-manifests/${r/\//-}.yaml"
done

```

### 2. Run the Pipeline

You can override any setting in `values.yaml` dynamically using the `--set flag`. This is ideal for CI/CD pipelines. In this example we pass variables to change the number of replicas and the image on the updated Deployment.

In this example, we showcase selective updates by only cleaning and updating resources tagged with `tier=frontend`. Any other files in the source-manifests/ folder will remain untouched:

```bash
# 1. Prepare a clean build workspace
mkdir -p build
cp source-manifests/*.yaml build/

# 2. Generate the kustomization.yaml into the build root
helm template . --show-only templates/kustomization.yaml \
  --set image="image-registry.openshift.com/prod/my-app:v2.1" \
  --set replicas=5 \
  --set targetLabelSelector="tier=frontend" > build/kustomization.yaml

# 3. Build and Export to discrete "Clean" files
mkdir -p ./dist
oc kustomize build/ -o ./dist/

```

### 3. Apply to the Target Cluster

```bash
oc apply -f ./dist/ --context=target-cluster

```

---

## 💡 Key Features: Strategic Merge Patching

This PoC utilizes **Strategic Merge Patches (SMP)** instead of standard JSON Patching. This provides significant benefits for automated migrations:

### 1. Resilience to Missing Fields

Unlike JSON Patch (which fails if a path like `/metadata/managedFields` is missing), SMP is **schema-aware**. By setting a key to `null` in the patch, we tell Kustomize: *"Ensure this key is absent in the final output."*

* **If the key exists:** It is removed.
* **If the key is already missing:** Kustomize ignores it and continues. This prevents build crashes when migrating resources from different OpenShift versions or "cleaner" exports.

### 2. Surgical Precision (No Overwriting)

SMP only affects the specific keys defined in the patch. It **merges** with the base manifest rather than replacing blocks. For example, adding a single annotation will not delete existing annotations; it will only append or update the targeted one.

### 3. Idempotency

Because the patch describes a **desired state** (e.g., "The UID should be null"), running the pipeline multiple times always yields the same clean result, regardless of the initial state of the source manifest.

### 4. Network & Metadata Decoupling

* **Metadata:** Automatically strips `uid`, `resourceVersion`, `managedFields`, and `creationTimestamp`.
* **Networking:** Automatically removes `clusterIP` from Services, allowing the target cluster to assign fresh IP space without conflicts.

---

## 🛠 Tested Configurations

The Kustomize parser behavior changed significantly in recent versions. This pipeline is verified to work with the following stack:

| Component | Version | Note |
| --- | --- | --- |
| **OC Client** | `4.19.0+` | Required for stable Kustomize v5.5.0 features |
| **Kustomize** | `v5.5.0` | Uses the unified `patches` list with GVK-aware parsing |
| **Kubernetes** | `v1.32+` | Targets modern API schemas |
