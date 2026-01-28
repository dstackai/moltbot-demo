# Deploying Moltbot on GPU clouds with `dstack`

## Prerequisites

1. Install [`dstack`](https://github.com/dstackai/dstack/)
2. Set up a `dstack` [gateway](https://dstack.ai/docs/concepts/gateways/) (already set up for you if you sign up with [dstack Sky](https://sky.dstack.ai))

## Deploy a model

```yaml
type: service
name: qwen32

image: lmsysorg/sglang:latest
env:
  - HF_TOKEN
  - MODEL_ID=deepseek-ai/DeepSeek-R1-Distill-Qwen-32B
commands:
  # Launch SGLang server
  - |
    python3 -m sglang.launch_server \
      --model-path $MODEL_ID \
      --host 0.0.0.0 \
      --port 8000

port: 8000
model: deepseek-ai/DeepSeek-R1-Distill-Qwen-32B

resources:
  gpu: H100
```

```shell
export HF_TOKEN=...
dstack apply -f qwen32.dstack.yml
```

Once the model is deployed, the OpenAI-compatible base URL will be avaiable at `https://qwen32.<your gateway domain>/v1`.

## Deploy Moltbot

```yaml
type: service
name: moltbot

env:
  - MODEL_BASE_URL
  - DSTACK_TOKEN

commands:
  # System dependencies
  - sudo apt update
  - sudo apt install -y ca-certificates curl gnupg

  # Node.js & moltbot installation
  - curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
  - sudo apt install -y nodejs
  - curl -fsSL https://molt.bot/install.sh | bash -s -- --no-onboard

  # Model configuration
  - |
    clawdbot config set models.providers.custom '{
      "baseUrl": "'"$MODEL_BASE_URL"'",
      "apiKey": "'"$DSTACK_TOKEN"'",
      "api": "openai-completions",
      "models": [
        {
          "id": "deepseek-ai/DeepSeek-R1-Distill-Qwen-32B",
          "name": "DeepSeek-R1-Distill-Qwen-32B",
          "reasoning": true,
          "input": ["text"],
          "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
          "contextWindow": 128000,
          "maxTokens": 512
        }
      ]
    }' --json

  # Set active model & gateway settings
  - clawdbot models set custom/deepseek-ai/DeepSeek-R1-Distill-Qwen-32B
  - clawdbot config set gateway.mode local
  - clawdbot config set gateway.auth.mode token
  - clawdbot config set gateway.auth.token "$DSTACK_TOKEN"
  - clawdbot config set gateway.controlUi.allowInsecureAuth true
  - clawdbot config set gateway.trustedProxies '["127.0.0.1"]'

  # Start service
  - clawdbot gateway

port: 18789
auth: false

resources:
  cpu: 2
```

```shell
export MODEL_BASE_URL=<your model endpoint>
DSTACK_TOKEN=<your dstack token>
dstack apply -f moltbot.dstack.yml
```

The Moltbot UI will be available at `https://moltbot.<your gateway domain>/chat`.

<img src="https://dstack.ai/static-assets/static-assets/images/dstack-moltbot-ui.png" width="750" />