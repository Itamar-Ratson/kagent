# kagent quickstart (kind + Groq)

Combines the [official quickstart](https://kagent.dev/docs/kagent/getting-started/quickstart) with a declarative kind setup and a swap from OpenAI to Groq (free tier).

## Prerequisites

`kind`, `kubectl`, `helm`, and the `kagent` CLI:

```bash
brew install kagent
# or
curl https://raw.githubusercontent.com/kagent-dev/kagent/refs/heads/main/scripts/get-kagent | bash
```

A Groq API key from <https://console.groq.com/keys>. Save it in `.env`:

```
GROQ_API_KEY=gsk_...
```

`.env` is gitignored.

## 1. Create the kind cluster

```bash
kind create cluster --config kind-cluster.yaml
```

This creates a single-node cluster named `kagent` and sets the kubectl context to `kind-kagent`.

## 2. Install kagent

`kagent install` requires `OPENAI_API_KEY` to be set, but the value isn't validated — we replace the model in step 4, so any non-empty string works:

```bash
export OPENAI_API_KEY=placeholder
kagent install --profile demo
```

`--profile minimal` skips the preloaded agents (helm, istio, etc.).

## 3. Create the agent via the UI

```bash
kagent dashboard       # opens http://localhost:8082
```

In the wizard: pick `default-model-config` → accept the default Kubernetes settings → accept the preselected tools → **Create kagent/my-first-k8s-agent & Finish**.

At this point the agent will hit `insufficient_quota` because the OpenAI key is fake. Continue to step 4.

## 4. Switch the agent to Groq

Three files in this repo do this declaratively:

| File | Purpose |
|---|---|
| `groq-secret.yaml` | Holds `GROQ_API_KEY` in the `kagent` namespace (gitignored via `*secret*`). Copy from `groq-secret.example.yaml` and fill in your key. |
| `groq-model-config.yaml` | `ModelConfig` using `provider: OpenAI` with `baseUrl: https://api.groq.com/openai/v1` (Groq is OpenAI-compatible). Defaults to `openai/gpt-oss-20b`. |
| `agent-modelconfig-patch.yaml` | Minimal merge patch that points `spec.declarative.modelConfig` at `groq-gpt-oss-20b` |

Create your local secret file from the example, then apply:

```bash
cp groq-secret.example.yaml groq-secret.yaml
# edit groq-secret.yaml and replace <your-groq-api-key>
kubectl apply -f groq-secret.yaml -f groq-model-config.yaml
kubectl patch agent my-first-k8s-agent -n kagent \
  --type=merge --patch-file=agent-modelconfig-patch.yaml
kubectl rollout status deploy/my-first-k8s-agent -n kagent
```

## 5. Test

Refresh the dashboard. The agent should show **Ready**. Try: *"What API resources are running in my cluster?"*

To switch models later (e.g. to the stronger `openai/gpt-oss-120b`), edit `model:` in `groq-model-config.yaml` and `kubectl apply -f groq-model-config.yaml` — no agent restart needed.

## Teardown

```bash
kind delete cluster --name kagent
```
