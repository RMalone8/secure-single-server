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
  tiers: 16+ vCPUs, 32 GiB RAM recommended or 16 GiB minimum with the CPU
  backend's KV cache reduced, one NVIDIA GPU with ~24 GiB VRAM for the
  GPU tier, and ~100 GiB of free disk (the GPU inference image alone is
  ~30 GiB).
- OpenShell installed per
  [manual single-server guide](../docs/quickstarts/openshell-single-server/manual.md);
  see also [model access](../docs/quickstarts/openshell-single-server/model-access.md).
- NVIDIA driver and `nvidia-container-toolkit` installed with CDI
  generated.
- `podman login registry.redhat.io` completed - the rhaii vLLM images
  require authentication; the `quay.io/rmalone` images are public.

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
