# llm-d-sc stack

Complexity-routed local inference for the OpenShell OpenCode harness on a
mutable RHEL 9 single-server host, modeled on the
[llm-d-sc demo](https://github.com/RMalone8/llm-d-sc-demo) podman stack.

```
opencode sandbox (opencode-praxis)
  -> http://host.openshell.internal:8095/v1   (only allowed model endpoint)
     -> trace-ui :8080 front relay + dashboard  (records traces, host 8095)
        -> praxis :8080 (quay.io/rmalone/praxis-llm-d-sc, host loopback 8080)
           -> llm-d-sc gRPC :50051  (complexity classifier)
           -> SIMPLE / MEDIUM      -> trace-ui :8081 -> vllm-cpu  (Qwen/Qwen3-0.6B)
           -> COMPLEX / REASONING  -> trace-ui :8082 -> vllm-gpu  (Qwen/Qwen3.5-4B)
```

The trace-ui backends read the `X-Llm-D-Sc-Label` praxis attaches to each
request and annotate the response, which the front relay persists in sqlite:
the dashboard at `:8095` shows which model each request was routed to.

## Assumed starting point

This stack extends an existing secure-single-server deployment; it is not a
standalone installer. The deployment steps below assume you are starting
from this state:

- A mutable RHEL 9 x86_64 host with enough capacity for both inference
  tiers: 8+ vCPUs (the CPU backend's quadlet binds OMP threads to cores
  0-5 - adjust `VLLM_CPU_OMP_THREADS_BIND` and `OMP_NUM_THREADS` on
  smaller hosts), 32 GiB RAM recommended or 16 GiB minimum with the CPU
  backend's KV cache reduced, one NVIDIA GPU with ~24 GiB VRAM for the
  GPU tier, and ~100 GiB of free disk (the GPU inference image alone is
  ~30 GiB).
- OpenShell v0.1.3 installed per the upstream
  [manual single-server guide](../docs/quickstarts/openshell-single-server/manual.md)
  (`openshell/scripts/install.sh --owner openshell-svc`): locked
  `openshell-svc` account, loopback gateway on 8090/8091, harness images
  pre-pulled.
- NVIDIA driver and `nvidia-container-toolkit` installed with CDI
  generated, so `--device nvidia.com/gpu=all` resolves and `nvidia-smi`
  works.
- `podman login registry.redhat.io` completed - the rhaii vLLM images
  require authentication; the `quay.io/rmalone` images are public.
- This branch checked out at `~/secure-single-server`, readable by
  `openshell-svc` (`chmod o+x ~` when the admin home is mode 0700).

Starting from a fresh AWS instance? Run the manual guide's package step
first, add the NVIDIA CUDA repo with `kmod-nvidia` and
`nvidia-container-toolkit`, generate CDI (`nvidia-ctk cdi generate`), and
then continue from the deployment steps below.

## Contents

- `praxis.yaml` - Praxis routing config (clusters point at the trace-ui
  relays, like praxis-sr-demo's `openshift/base/praxis.yaml`)
- `quadlets/` - systemd user units (podman quadlets) for the five containers
  plus the shared `llm-d-sc` network
- `../configs/llm-d-sc/` - OpenCode harness wiring used with the repo's
  integrated mode (`harness-provider.json.in` + `profiles/dev/policy.yaml`)

## Deployment (mutable RHEL 9 host, ec2-user)

```bash
# 1. stack files where the quadlets expect them
cp -r llm-d-sc-stack ~/
mkdir -p ~/model-cache ~/llm-d-sc-stack/models ~/llm-d-sc-stack/trace-data

# 2. classifier model from the ModelCar image
cid=$(podman create quay.io/rmalone/llm-d-sc-modelcar:latest /bin/true)
podman cp "$cid":/models/. ~/llm-d-sc-stack/models/
podman rm "$cid"

# 3. quadlets
cp llm-d-sc-stack/quadlets/* ~/.config/containers/systemd/
sudo loginctl enable-linger "$USER"
systemctl --user daemon-reload
systemctl --user start llm-d-sc vllm-cpu vllm-gpu praxis trace-ui

# 4. harness sandbox (OpenShell service account, see the manual guide)
os_run env OPENSHELL_MODEL_ID=Semantic-Qwen PRAXIS_PORT=8095 \
  openshell/harnesses/opencode/create.sh --profile dev \
  --name opencode-praxis --config "$PWD/configs/llm-d-sc"
```

The bind-mount source directories (`models/`, `trace-data/`) must exist
before the services start; podman exits 125 on a missing source. Pass the
variables through `os_run env ...` rather than prefixing them: `sudo`
strips the caller's environment, and the script then fails with
"set OPENSHELL_MODEL_ID".

Dashboard: `ssh -L 8095:127.0.0.1:8095 HOST` then http://localhost:8095.

## Operational notes

- Host ports 8090/8091 are the OpenShell gateway's (management/health) -
  don't reuse them.
- Praxis does not hot-apply config changes; `systemctl --user restart praxis`.
- Both vLLM backends need `--served-model-name Semantic-Qwen`: praxis does
  not rewrite the client's model field, and the harness sends the name the
  gateway advertises.
- vLLM CPU needs explicit OMP binding (`VLLM_CPU_OMP_THREADS_BIND=0-5`,
  `OMP_NUM_THREADS=6`); auto-binding parks the worker on one core.
- CPU model: Qwen3-0.6B bf16 + `--enforce-eager` + `--reasoning-parser qwen3`
  is the working config. Quantized alternatives fail on the rhaii vllm-cpu
  images (3.5.0/3.5.1, vLLM 0.24.0+rhaiv): GPTQ-Int8 checkpoints use the
  unsupported `uint8b128` weight format, and compressed-tensors w4a16 needs
  the zentorch `cpu_gemm_wna16` kernel neither build registers. Harness-scale
  prompts (multi-k-token system prompt) remain slow on the CPU tier; routing
  MEDIUM to the GPU cluster is the practical mitigation.
- The harness provider config must NOT set `interleaved: {field: reasoning}`.
  With it, OpenCode replays earlier reasoning in the chat history, the Qwen3
  chat template writes a `</think>` into the prompt, and vLLM's reasoning
  parser then treats the whole next completion as already past reasoning:
  `<think>...</think>` shows up verbatim in `content` on every turn after a
  tool call. The `--reasoning-parser qwen3` flag itself is correct.
