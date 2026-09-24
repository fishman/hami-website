---
title: HAMi on Vultr
sidebar_label: HAMi on Vultr
---

A GPU worker showing `Ready` does not prove HAMi can allocate a slice of its memory. The driver, NVIDIA container runtime, HAMi device plugin, scheduler, and workload all have to agree. This example keeps the workload small: one idle CUDA Pod with a 1024 MiB GPU-memory request, not a model download.

The source configuration is in the [`environment/` directory of the demo repository](https://github.com/fishman/kubecon-demo). It has three [Atmos](https://atmos.tools/) components: Terraform `vke`, Helmfile `hami`, and an optional Helmfile `hami-demo`. The HAMi component does **not** install a driver or a GPU Operator.

## Prerequisites

- A Vultr account with VKE access and quota for an **NVIDIA GPU** plan in your chosen region. Provisioning a GPU worker costs money. Check [Vultr's VKE provisioning guide](https://docs.vultr.com/products/compute/kubernetes/provisioning) before applying anything. An AMD GPU plan will not work with this NVIDIA example.
- `atmos`, Terraform 1.6+, Helm, Helmfile, `kubectl`, and an authenticated `vultr-cli` on your own machine. Keep `VULTR_API_KEY` in your trusted terminal; never commit it or put it in an Atmos stack or Helmfile values.
- An NVIDIA driver and [NVIDIA Container Toolkit configured for HAMi's runtime strategy](./prerequisites.md#preparing-your-gpu-nodes) on the VKE GPU worker. Confirm the driver and toolkit on the actual node. This example assumes the NVIDIA runtime is the default for HAMi's `envvar` device allocation. If the node does not meet those requirements, stop and prepare it before installing HAMi.
- A private Terraform state directory. Local state includes a sensitive kubeconfig and is suitable only for this single-operator example; use a protected remote backend for team deployment. Do not apply over an existing VKE cluster managed by another state file.

## Provision VKE

The stack uses region `ewr` and one GPU worker labeled `gpu=on`. Pick an NVIDIA plan **actually available in `ewr`** and a currently supported Kubernetes version. The following runs in your local terminal from the demo repository checkout; replace neither value with a plan copied from another region.

```bash
git clone https://github.com/fishman/kubecon-demo.git
cd kubecon-demo/environment
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
umask 077
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

The official HAMi chart is pinned to `2.10.0` in `stacks/vultr-gpu.yaml`. The HAMi Helmfile selects only the `gpu=on` worker. From `kubecon-demo/environment` with the explicit `KUBECONFIG` still set:

```bash
atmos helmfile apply hami -s vultr-gpu
kubectl get pods -n kube-system
kubectl get nodes -l gpu=on -o 'custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

Wait for `hami-scheduler` and `hami-device-plugin` to become Ready. The labeled worker must report nonzero `nvidia.com/gpu` capacity now. If not, inspect the failing Pod and its events before deploying a workload. HAMi's [Helm quick start](https://project-hami.io/docs/get-started/deploy-with-helm) explains what these components register.

## Verify an allocation

`hami-demo` is a **separate**, optional Helmfile component. It runs one CUDA Pod with `schedulerName: hami-scheduler`, `nvidia.com/gpu: 1`, and `nvidia.com/gpumem: 1024`. No vLLM service or model is involved.

```bash
atmos helmfile apply hami-demo -s vultr-gpu
kubectl wait --for=condition=Ready pod/hami-demo-gpu --timeout=180s
kubectl get pod hami-demo-gpu -o jsonpath='{.spec.schedulerName}{"\n"}{.spec.containers[0].resources.limits}{"\n"}'
kubectl exec hami-demo-gpu -- nvidia-smi
```

Look for `hami-scheduler` and the requested `gpumem: 1024` in the Pod, then **1024 MiB total** GPU memory inside `nvidia-smi`. A Ready Pod only proves startup; it does not prove the memory limit was injected. If the Pod stays Pending, `kubectl describe pod hami-demo-gpu` shows scheduling events. If the container sees the whole physical card or `nvidia-smi` fails, check the device-plugin logs and node runtime configuration. These are checks to perform, not output observed from a live Vultr cluster during this write-up.

For a workload with real GPU compute requirements, start from [NVIDIA device memory allocation](../userguide/nvidia-device/examples/allocate-device-memory.md) and size the memory request to the application rather than copying 1024 MiB blindly.

## Tear down

Remove the demo Pod before HAMi. Destroying Terraform infrastructure deletes billable VKE resources: re-enter your key in your trusted terminal and review the destroy plan first.

```bash
atmos helmfile destroy hami-demo -s vultr-gpu
atmos helmfile destroy hami -s vultr-gpu
read -r -s -p 'Vultr API key: ' VULTR_API_KEY
printf '\n'
export VULTR_API_KEY
atmos terraform plan vke -s vultr-gpu -destroy
atmos terraform destroy vke -s vultr-gpu
unset VULTR_API_KEY KUBECONFIG
rm -f kubeconfigs/vultr-gpu.yaml
```

Thanks to the Vultr team for helping us put this example together. The static Terraform and Helmfile rendering checks do not establish that a particular region has GPU capacity or that the live Pod sees a 1024 MiB virtual GPU; run the checks above on your own cluster before treating it as verified.
