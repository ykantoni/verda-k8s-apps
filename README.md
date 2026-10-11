# verda-k8s-apps

Second-wave Argo CD apps for the [`verda-cloud`](https://github.com/ykantoni/verda-cloud)
lab cluster. `verda-k8s-infra`'s root app-of-apps points at this repo's
[`argocd-apps/`](argocd-apps) directory (`argo-apps/verda-k8s-apps.yaml`
over there, tagged `argocd.argoproj.io/sync-wave: "1"`) — so everything
here gets applied only after `verda-k8s-infra`'s own first-wave apps
(OpenBao, External Secrets Operator, the NVIDIA GPU Operator, etc.) are
already in place. Nothing in this repo needs its own `terraform apply` or
Justfile recipe — Argo CD picks it up purely by polling the pushed git
state, same as `verda-k8s-infra`'s own `argo-apps/`.

## Structure

- [`argocd-apps/`](argocd-apps) — Argo CD `Application` manifests, the
  things this repo's own entry in `verda-k8s-infra` recurses into.
- [`vllm-secrets/`](vllm-secrets) — plain Kubernetes manifests (no Helm
  chart), referenced by `argocd-apps/vllm-secrets.yaml`'s directory
  source — not reachable from `argocd-apps/` itself.

## Apps

- **[`argocd-apps/vllm-secrets.yaml`](argocd-apps/vllm-secrets.yaml)** —
  a plain-manifest directory Application pointing at
  [`vllm-secrets/`](vllm-secrets): a `ServiceAccount` (`vllm`), a
  `SecretStore` pointing External Secrets Operator at OpenBao's
  in-cluster Service (`openbao.openbao.svc.cluster.local:8200`, in the
  `verda-k8s-infra`-managed cluster), and an `ExternalSecret` that turns
  OpenBao's `secret/vllm/hf-token` into a real `vllm-hf-token` Secret. No
  secret *values* live in any of these three files — the Hugging Face
  token itself only ever exists inside OpenBao and inside the Secret ESO
  materializes from it.

  The OpenBao-side half of this wiring (enabling its kv engine, enabling
  and configuring Kubernetes auth, writing the policy/role these files'
  `auth.kubernetes.role: vllm` depends on) lives in `verda-k8s-infra`, not
  here — it's that repo's `scripts/configure-vllm-secret.sh`, which runs
  automatically on every `just k8s-apply` from the `verda-cloud` root.
  The one piece that script can't automate is the actual Hugging Face
  token value: set `$HF_TOKEN` or save it to `~/.hf-token` first (after
  accepting Google's Gemma license and generating a READ-scoped token on
  Hugging Face) for it to get written; otherwise that step is skipped
  with a message and this `ExternalSecret` stays unresolved until you
  supply one and rerun `just configure-vllm-secret`.

- **[`argocd-apps/vllm.yaml`](argocd-apps/vllm.yaml)** — self-hosted
  [vLLM](https://docs.vllm.ai/), serving
  [`google/gemma-4-E4B-it`](https://huggingface.co/google/gemma-4-E4B-it)
  (~16GB checkpoint) via the official
  [`vllm-project/production-stack`](https://github.com/vllm-project/production-stack)
  chart, namespace `vllm`. Requires a GPU node — see `verda-vm-infra`'s
  README for `gpu1` (`create_gpu_node`, off by default); the pod stays
  `Pending` until one joins, same as `nvidia-gpu-operator` has been all
  along. Uses a 20Gi Longhorn-backed PVC for the model cache (per-model
  `pvcStorage`, chart-provisioned) and the `vllm-hf-token` Secret from
  `vllm-secrets.yaml` above. The chart's own router component is exposed
  as `NodePort` `30095` (pinned directly in the chart's values, unlike
  the Gateway API's dynamically-assigned port in `verda-vm-infra` — this
  chart supports a fixed `routerSpec.nodePort`):

  ```bash
  curl http://<any-node-ip>:30095/v1/models
  curl http://<any-node-ip>:30095/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "gemma-4-e4b", "messages": [{"role": "user", "content": "Hello!"}]}'
  ```

  A few deliberate, documented-in-`vllm.yaml` deviations from the chart's
  own defaults, worth knowing if you ever touch this file: `repository`
  set to the upstream `vllm/vllm-openai` image rather than the chart's
  default `lmcache/vllm-openai` — kept from an earlier debugging session
  (the previous model, Gemma 3n, needed `timm`, which that fork's image
  was missing) and left in place for Gemma 4 too, since
  `vllm/vllm-openai:latest` is a confirmed-recent, confirmed-working
  vLLM build (`0.29.0`) for a model released only months ago
  (2026-04), where the LMCache fork's bundled vLLM version is unverified
  — Gemma 4 itself doesn't actually need `timm` (its vision tower's
  `vision_config.model_type` is `gemma4_vision`, a dedicated native
  architecture, confirmed against the real `config.json` on Hugging
  Face, not the generic `timm_wrapper` class Gemma 3n's was); still,
  `lmcacheConfig.enabled` stays `false` below, since nothing
  LMCache-specific is actually in use regardless of which image serves
  it. Plus: `tensorParallelSize` forced to `1` (the chart defaults to
  `2`, assuming multiple GPUs), `requestGPUType` forced to
  `nvidia.com/gpu` (the chart defaults to a MIG-sliced GPU type, which
  doesn't apply to a whole, unpartitioned A100), and `keda.enabled`
  forced to `false` (KEDA isn't installed in this cluster — its CRDs
  don't exist).

## GPU Operator compatibility note

Not an app in this repo, but directly relevant to `vllm.yaml` actually
scheduling: `verda-k8s-infra`'s `nvidia-gpu-operator.yaml` needs a
`toolkit.env` override (`CONTAINERD_SOCKET`/`CONTAINERD_CONFIG` pointed
at RKE2's actual paths, not vanilla Kubernetes' defaults) to work at all
on this cluster — without it, the toolkit container crash-loops and
`nvidia.com/gpu` never becomes allocatable. See that repo's README.

## Why a separate repo at all

`verda-k8s-infra` already has its own `argo-apps/` directory using this
exact pattern (plain Application manifests + a couple of
plain-manifest-directory apps like `gateway-routes/`). Splitting the
vLLM app out into its own repo, synced as an explicit second wave via
`argocd.argoproj.io/sync-wave`, keeps app-specific experimentation
(swapping models, chart versions, GPU-dependent workloads) separate from
the core cluster bootstrap apps (storage, secrets, monitoring, ingress)
that everything else — including this repo's own apps — depends on.
