---
title: HAMi on Vultr
sidebar_label: HAMi on Vultr
---

A GPU worker showing `Ready` does not prove HAMi can allocate a slice of its memory. The driver, NVIDIA container runtime, HAMi device plugin, scheduler, and workload all have to agree. This example keeps the workload small: one idle CUDA Pod with a 1024 MiB GPU-memory request, not a model download.

The implementation belongs in [`Project-HAMi/hami-demo`](https://github.com/Project-HAMi/hami-demo) under `environment/`: Terraform `vke`, Helmfile `hami`, and optional Helmfile `hami-demo` are separate [Atmos](https://atmos.tools/) components. The snippets below let you reproduce that layout without relying on the repository's current public default branch, which does not yet contain `environment/`. HAMi does **not** install a driver or GPU Operator here.

## Prerequisites

- A Vultr account with VKE access and quota for an **NVIDIA GPU** plan in your chosen region. Provisioning a GPU worker costs money. Check [Vultr's VKE provisioning guide](https://docs.vultr.com/products/compute/kubernetes/provisioning) before applying anything. An AMD GPU plan will not work with this NVIDIA example.
- `atmos`, Terraform 1.6+, Helm, Helmfile, `kubectl`, and an authenticated `vultr-cli` on your own machine. Keep `VULTR_API_KEY` in your trusted terminal; never commit it or put it in an Atmos stack or Helmfile values.
- An NVIDIA driver and [NVIDIA Container Toolkit configured for HAMi's runtime strategy](./prerequisites.md#preparing-your-gpu-nodes) on the VKE GPU worker. Confirm the driver and toolkit on the actual node. This example assumes the NVIDIA runtime is the default for HAMi's `envvar` device allocation. If the node does not meet those requirements, stop and prepare it before installing HAMi.
- A private Terraform state directory. Local state includes a sensitive kubeconfig and is suitable only for this single-operator example; use a protected remote backend for team deployment. Do not apply over an existing VKE cluster managed by another state file.

Set `umask 077` **before** the first Terraform command, not only before writing a kubeconfig. Terraform state and backup files inherit the shell's permissions; if you already created them under a weaker umask, restrict their permissions and treat prior access as a possible credential exposure.

## Provision VKE

The stack uses region `ewr` and one GPU worker labeled `gpu=on`. Pick an NVIDIA plan **actually available in `ewr`** and a currently supported Kubernetes version; do not copy a plan from another region. Create a private local workspace first, then save each snippet below at its named path relative to `hami-vultr/environment`:

```bash
mkdir -p hami-vultr/environment/stacks hami-vultr/environment/components/terraform/vke \
  hami-vultr/environment/components/helmfile/hami \
  hami-vultr/environment/components/helmfile/hami-demo/chart/templates
cd hami-vultr/environment
umask 077
```

Save this Atmos configuration as `atmos.yaml`:

```yaml
base_path: .
components:
  terraform:
    base_path: components/terraform
  helmfile:
    use_eks: false
    base_path: components/helmfile
stacks:
  base_path: stacks
  included_paths:
    - "**/*.yaml"
  name_pattern: "{stage}"
```

If this workspace is under Git, save `.gitignore` beside `atmos.yaml`:

```text
.terraform/
*.tfstate*
*.tfplan
*.planfile
*.terraform.tfvars.json
*.helmfile.vars.yaml
kubeconfigs/
```

Git ignore does not protect local file permissions; keep the `umask 077` setting for Terraform state and kubeconfigs.

Save the stack as `stacks/vultr-gpu.yaml`. It keeps provider inputs out of source control and separates HAMi from the optional demo Pod:

```yaml
name: vultr-gpu
vars:
  stage: vultr-gpu
components:
  terraform:
    vke:
      vars:
        cluster_name: hami-demo
        region: ewr
        kubernetes_version: !env VULTR_K8S_VERSION
        gpu_plan: !env VULTR_GPU_PLAN
        gpu_node_count: 1
  helmfile:
    hami:
      dependencies:
        components:
          - name: vke
            kind: terraform
      vars:
        gpu_node_label: gpu
        gpu_node_value: "on"
        hami_chart_version: "2.10.0"
    hami-demo:
      dependencies:
        components:
          - name: hami
            kind: helmfile
      vars:
        gpu_node_label: gpu
        gpu_node_value: "on"
        gpu_memory_mib: 1024
```

Save the VKE resource and decoded kubeconfig output as `components/terraform/vke/main.tf`:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    vultr = {
      source  = "vultr/vultr"
      version = "~> 2.32"
    }
  }
}

resource "vultr_kubernetes" "this" {
  label   = var.cluster_name
  region  = var.region
  version = var.kubernetes_version

  node_pools {
    label         = "gpu"
    node_quantity = var.gpu_node_count
    plan          = var.gpu_plan

    labels {
      key   = "gpu"
      value = "on"
    }
  }
}

output "cluster_id" {
  value = vultr_kubernetes.this.id
}

output "kube_config" {
  value     = base64decode(vultr_kubernetes.this.kube_config)
  sensitive = true
}
```

Save the following as `components/terraform/vke/variables.tf`:

```hcl
variable "cluster_name" {
  type        = string
  description = "Vultr Kubernetes Engine cluster label."
}

variable "region" {
  type        = string
  description = "Vultr region with the selected NVIDIA GPU VKE plan."
}

variable "kubernetes_version" {
  type        = string
  description = "Supported version returned by vultr-cli kubernetes versions."

  validation {
    condition     = length(trimspace(var.kubernetes_version)) > 0
    error_message = "VULTR_K8S_VERSION must be a supported VKE version."
  }
}

variable "gpu_plan" {
  type        = string
  description = "NVIDIA GPU VKE plan available in the selected region."

  validation {
    condition     = length(trimspace(var.gpu_plan)) > 0
    error_message = "VULTR_GPU_PLAN must be an NVIDIA GPU plan available in the selected region."
  }
}

variable "gpu_node_count" {
  type        = number
  description = "Number of GPU workers."
  default     = 1

  validation {
    condition     = var.gpu_node_count > 0
    error_message = "gpu_node_count must be positive."
  }
}
```

Vultr's provider returns a base64-encoded kubeconfig; the output above decodes it before writing a YAML file. Terraform still stores that credential in state.

```bash
vultr-cli regions availability ewr
vultr-cli kubernetes versions
read -r -p 'NVIDIA VKE plan available in ewr: ' VULTR_GPU_PLAN
read -r -p 'Supported VKE version: ' VULTR_K8S_VERSION
export VULTR_GPU_PLAN VULTR_K8S_VERSION
read -r -s -p 'Vultr API key: ' VULTR_API_KEY
printf '\n'
export VULTR_API_KEY
atmos terraform plan vke -s vultr-gpu
```

Read the plan: cluster name, region, node count, GPU plan, and cost must match what you intend. Only then create the cluster:

```bash
atmos terraform apply vke -s vultr-gpu
unset VULTR_API_KEY
mkdir -p kubeconfigs
atmos terraform output vke -s vultr-gpu -raw kube_config > kubeconfigs/vultr-gpu.yaml
export KUBECONFIG="$PWD/kubeconfigs/vultr-gpu.yaml"
kubectl config current-context
kubectl get nodes -l gpu=on
```

The kubeconfig is saved under a Git-ignored path with restrictive permissions. Verify `kubectl` reaches the **new VKE cluster** before running Helmfile; an implicit context from `~/.kube/config` could install HAMi on a different cluster. If either required environment input was omitted, `atmos describe` may show it as an empty string; Terraform's variable validation rejects it during plan.

## Check the NVIDIA node

Before installing HAMi, check the host driver and Toolkit. If you have permission to create a **privileged node debug Pod**, run this in your local terminal:

```bash
NODE=$(kubectl get nodes -l gpu=on -o jsonpath='{.items[0].metadata.name}')
kubectl debug "node/$NODE" -it --profile=sysadmin --image=ubuntu:22.04 -- chroot /host sh -c 'nvidia-smi && nvidia-ctk --version'
```

Delete the named debug Pod afterwards. If privileged node debugging is forbidden, use approved host access instead. Seeing `nvidia-ctk` installed is not enough: confirm the NVIDIA runtime is configured as the default for this HAMi chart's `envvar` strategy, following the [NVIDIA node preparation guide](./prerequisites.md#preparing-your-gpu-nodes). If the driver or runtime is absent, stop; the Helmfile below does not fix it. `nvidia.com/gpu` may not appear as allocatable **until after** HAMi's device plugin starts.

Check whether VKE has already installed an NVIDIA Device Plugin on the same worker:

```bash
kubectl get daemonsets -A
```

If a different plugin already owns `nvidia.com/gpu`, disable it through the provider-supported mechanism before installing HAMi. Two device plugins registering the same resource on one node can conflict; do not delete a provider-managed add-on blindly. HAMi's [NVIDIA prerequisites](./prerequisites.md#preparing-your-gpu-nodes) call out this constraint.

## Install with Helmfile

The official HAMi chart is pinned to `2.10.0` in `stacks/vultr-gpu.yaml`. Save the HAMi-only release as `components/helmfile/hami/helmfile.yaml.gotmpl`:

```yaml
{{- $_ := requiredEnv "KUBECONFIG" -}}
repositories:
  - name: hami-charts
    url: https://project-hami.github.io/HAMi/

releases:
  - name: hami
    namespace: kube-system
    chart: hami-charts/hami
    version: {{ .Values.hami_chart_version | quote }}
    wait: true
    atomic: true
    values:
      - global:
          managedNodeSelectorEnable: true
          managedNodeSelector:
            {{ .Values.gpu_node_label }}: {{ .Values.gpu_node_value | quote }}
        devicePlugin:
          nvidiaNodeSelector:
            {{ .Values.gpu_node_label }}: {{ .Values.gpu_node_value | quote }}
```

From `hami-vultr/environment` with explicit `KUBECONFIG` set, apply HAMi:

```bash
atmos helmfile apply hami -s vultr-gpu
kubectl get pods -n kube-system
kubectl get nodes -l gpu=on -o 'custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

Wait for `hami-scheduler` and `hami-device-plugin` to become Ready. The labeled worker must report nonzero `nvidia.com/gpu` capacity now. If not, inspect the failing Pod and its events before deploying a workload. HAMi's [Helm quick start](https://project-hami.io/docs/get-started/deploy-with-helm) explains what these components register.

## Verify an allocation

`hami-demo` is a **separate**, optional Helmfile component. It runs one CUDA Pod with `schedulerName: hami-scheduler`, `nvidia.com/gpu: 1`, and `nvidia.com/gpumem: 1024`. No vLLM service or model is involved.

Save `components/helmfile/hami-demo/helmfile.yaml.gotmpl`:

```yaml
{{- $_ := requiredEnv "KUBECONFIG" -}}
releases:
  - name: hami-demo
    namespace: default
    chart: ./chart
    wait: true
    values:
      - gpu:
          nodeSelector:
            {{ .Values.gpu_node_label }}: {{ .Values.gpu_node_value | quote }}
          memoryMiB: {{ .Values.gpu_memory_mib }}
```

Save `components/helmfile/hami-demo/chart/Chart.yaml`:

```yaml
apiVersion: v2
name: hami-demo
version: 0.1.0
description: One small NVIDIA GPU Pod for checking HAMi memory allocation.
```

Save `components/helmfile/hami-demo/chart/values.yaml`:

```yaml
image: nvidia/cuda:12.8.1-base-ubuntu24.04
gpu:
  nodeSelector:
    gpu: "on"
  memoryMiB: 1024
```

Save `components/helmfile/hami-demo/chart/templates/pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hami-demo-gpu
spec:
  schedulerName: hami-scheduler
  nodeSelector:{{ toYaml .Values.gpu.nodeSelector | nindent 4 }}
  containers:
    - name: cuda
      image: {{ .Values.image | quote }}
      command: ["sleep", "infinity"]
      resources:
        limits:
          nvidia.com/gpu: "1"
          nvidia.com/gpumem: {{ .Values.gpu.memoryMiB | quote }}
```

This is the Pod YAML rendered from `environment/components/helmfile/hami-demo/chart/templates/pod.yaml` with the stack values above:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hami-demo-gpu
spec:
  schedulerName: hami-scheduler
  nodeSelector:
    gpu: "on"
  containers:
    - name: cuda
      image: "nvidia/cuda:12.8.1-base-ubuntu24.04"
      command: ["sleep", "infinity"]
      resources:
        limits:
          nvidia.com/gpu: "1"
          nvidia.com/gpumem: "1024"
```

The Pod requests no model weights, ingress, or Service. The `hami-demo` Helmfile installs it separately from the HAMi chart.

```bash
atmos helmfile apply hami-demo -s vultr-gpu
kubectl wait --for=condition=Ready pod/hami-demo-gpu --timeout=180s
kubectl get pod hami-demo-gpu -o jsonpath='{.spec.schedulerName}{"\n"}{.spec.containers[0].resources.limits}{"\n"}'
kubectl exec hami-demo-gpu -- nvidia-smi
```

Look for `hami-scheduler` and the requested `gpumem: 1024` in the Pod, then **1024 MiB total** GPU memory inside `nvidia-smi`. A Ready Pod only proves startup; it does not prove the memory limit was injected. The demo Helmfile waits but does not atomically roll back a failed install: if it times out with a Pending Pod, run `kubectl describe pod hami-demo-gpu` to read scheduling events, then remove the failed release with `atmos helmfile destroy hami-demo -s vultr-gpu` after diagnosis. If the container sees the whole physical card or `nvidia-smi` fails, check the device-plugin logs and node runtime configuration. These are checks to perform, not output observed from a live Vultr cluster during this write-up.

For a workload with real GPU compute requirements, start from [NVIDIA device memory allocation](../userguide/nvidia-device/examples/allocate-device-memory.md) and size the memory request to the application rather than copying 1024 MiB blindly.

## Tear down

From `hami-vultr/environment`, explicitly re-select the saved VKE kubeconfig even in a new shell and confirm the cluster before removing releases. Re-enter the original GPU plan/version: Atmos needs them when destroying VKE. Terraform destroy deletes billable resources, so review the destroy plan first.

```bash
umask 077
export KUBECONFIG="$PWD/kubeconfigs/vultr-gpu.yaml"
kubectl config current-context
kubectl get nodes -l gpu=on
read -r -p 'Original NVIDIA VKE plan: ' VULTR_GPU_PLAN
read -r -p 'Original VKE version: ' VULTR_K8S_VERSION
export VULTR_GPU_PLAN VULTR_K8S_VERSION
atmos helmfile destroy hami-demo -s vultr-gpu
atmos helmfile destroy hami -s vultr-gpu
read -r -s -p 'Vultr API key: ' VULTR_API_KEY
printf '\n'
export VULTR_API_KEY
atmos terraform plan vke -s vultr-gpu -destroy
atmos terraform destroy vke -s vultr-gpu
unset VULTR_API_KEY VULTR_GPU_PLAN VULTR_K8S_VERSION KUBECONFIG
rm -f kubeconfigs/vultr-gpu.yaml
```

Thanks to the Vultr team for helping us put this example together. The static Terraform and Helmfile rendering checks do not establish that a particular region has GPU capacity or that the live Pod sees a 1024 MiB virtual GPU; run the checks above on your own cluster before treating it as verified.
