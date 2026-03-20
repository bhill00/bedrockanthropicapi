---
layout: default
title: Using Claude with Protected Data via AWS Bedrock
---

# Using Claude / Anthropic-Compatible AI Tools with Protected Data via AWS Bedrock

## Background

UC IS-3 requires that any vendor handling UC Protected Data (P3) have a Data Processing Agreement (DPA) in place with UC that meets UC's data handling standards. For AI tools, this means the model provider must be a covered vendor — not just technically secure, but contractually accountable.

**AWS Bedrock is covered under the UC/AWS agreement**, which satisfies the IS-3 DPA burden for protected data. UCSB department and sub-accounts under [aws.cloud.ucsb.edu](https://aws.cloud.ucsb.edu) inherit this coverage.

Critically, when Claude models are accessed through Bedrock, **Anthropic never sees your data** — prompts and outputs remain within the AWS boundary. Anthropic is not a data processor in this architecture, so no separate UC-Anthropic agreement is required.

---

## What This Enables

By generating Bedrock API credentials from your UCSB department AWS account and routing them through a lightweight proxy ([LiteLLM](https://docs.litellm.ai/)), you can use Anthropic-compatible tools — including **Claude Code**, the **Anthropic SDK**, and **OpenAI-compatible clients** — against protected data, while remaining within the UC/AWS DPA boundary.

---

## The Stack

```
Claude Code / Anthropic SDK / OpenAI-compatible client
                        |
            LiteLLM proxy (local or hosted)
       presents Anthropic + OpenAI-compatible API surface
                        |
            AWS Bedrock (UCSB department account)
                        |
           Claude models -- data never leaves AWS
```

---

## Requirements

- A UCSB AWS sub-account via [aws.cloud.ucsb.edu](https://aws.cloud.ucsb.edu)
- Bedrock model access enabled in your account (see [Step 2](#2-enable-anthropic-model-access))
- Python 3.8+ with pip, or Docker
- LiteLLM installed: `pip install 'litellm[proxy]' boto3`

---

## Setup

### 1. Obtain a UCSB AWS Account

If you don't already have one, request a UCSB department AWS sub-account through [docs.cloud.ucsb.edu](https://docs.cloud.ucsb.edu). Your account will be set up under the UCSB AWS Organization and inherits the UC/AWS DPA coverage.

### 2. Enable Anthropic Model Access

The first time you use Anthropic models in a Bedrock account, AWS requires a **one-time use case submission** that is shared with Anthropic. Serverless foundation models are now automatically enabled on first invocation — the old Model Access page has been retired — but the Anthropic use case form is still required before first use.

To trigger the form, open a Claude model in the **Bedrock Playground** (AWS Console > Bedrock > Playgrounds) or visit the **Model Catalog** and select an Anthropic model. The form should appear on first use.

**Example values for UCSB use:**

| Field | Example Value |
|-------|---------------|
| Company name | UCSB Bren School (or your department) |
| Company website | `https://bren.ucsb.edu` (full URL required) |
| Industry | Education |
| Intended users | Internal users |
| Use case description | Testing, Teaching, Internal Tools, PII Data Under Parent (UCSB Agreement) |

> **Note:** Aggregated usage metrics (token counts, usage patterns) may be shared with Anthropic for security purposes per this submission. The content of your prompts and outputs is not shared.

After submission, it may take up to 15 minutes for access to propagate.

---

### 3. Generate a Bedrock API Key

In the AWS Console, navigate to **Bedrock > API keys**.

There are two types of Bedrock API keys:

| Type | Prefix | Lifetime | Use Case |
|------|--------|----------|----------|
| **Long-term** | `ABSK...` | Configurable (1 day to no expiration) | Recommended for ongoing use |
| **Short-term** | `bedrock-api-key-...` | Up to 12 hours (tied to console session) | Quick testing only |

**Recommended: Generate a long-term key** under the **Long-term API keys** tab:

- **Set an expiration date** appropriate to your use case (e.g., 30-90 days) for best security practices.
- Under **Advanced permissions**, attach **AmazonBedrockMarketplaceAccess** to enable access to Marketplace models.

Copy the key — you'll need it in the next step.

> **Security note:** Bedrock API keys are not IAM access keys, but treat them as credentials. Do not commit them to code repositories. Long-term keys create an IAM user behind the scenes with `AmazonBedrockLimitedAccess` policy. Set an expiration date and rotate keys periodically.

---

### 4. Create a LiteLLM Config

Create a file called `config.yaml`:

```yaml
model_list:
  # Claude Sonnet 4.6
  - model_name: claude-sonnet-4-6
    litellm_params:
      model: bedrock/converse/us.anthropic.claude-sonnet-4-6
      aws_region_name: us-west-2
      api_key: os.environ/BEDROCK_API_KEY

  - model_name: claude-sonnet-4-6-20260220
    litellm_params:
      model: bedrock/converse/us.anthropic.claude-sonnet-4-6
      aws_region_name: us-west-2
      api_key: os.environ/BEDROCK_API_KEY

  # Claude Opus 4.6
  - model_name: claude-opus-4-6
    litellm_params:
      model: bedrock/converse/us.anthropic.claude-opus-4-6-v1
      aws_region_name: us-west-2
      api_key: os.environ/BEDROCK_API_KEY

  - model_name: claude-opus-4-6-20260220
    litellm_params:
      model: bedrock/converse/us.anthropic.claude-opus-4-6-v1
      aws_region_name: us-west-2
      api_key: os.environ/BEDROCK_API_KEY

  # Claude Haiku 4.5
  - model_name: claude-haiku-4-5-20251001
    litellm_params:
      model: bedrock/converse/us.anthropic.claude-haiku-4-5-20251001-v1:0
      aws_region_name: us-west-2
      api_key: os.environ/BEDROCK_API_KEY

  - model_name: claude-haiku-4-5
    litellm_params:
      model: bedrock/converse/us.anthropic.claude-haiku-4-5-20251001-v1:0
      aws_region_name: us-west-2
      api_key: os.environ/BEDROCK_API_KEY

general_settings:
  master_key: sk-1234  # Change this to a secure key for non-local deployments
```

> **Why duplicate model names?** Tools like Claude Code may request models by short name (`claude-sonnet-4-6`) or full date-stamped name (`claude-sonnet-4-6-20260220`). Both aliases point to the same Bedrock model.

> **Note on Bedrock model IDs:** Bedrock model IDs don't follow a single naming pattern. Sonnet 4.6 uses `us.anthropic.claude-sonnet-4-6` (no version suffix), while other models use suffixes like `-v1:0`. The `us.` prefix enables cross-region inference within US regions. Check the [Bedrock model documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) if you need to add other models.

---

### 5. Start the Proxy

```bash
BEDROCK_API_KEY="ABSK..." \
litellm --config config.yaml --port 4000
```

Verify it's working:

```bash
curl http://localhost:4000/v1/messages \
  -H "x-api-key: sk-1234" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

You should get a JSON response with Claude's reply.

---

### 6. Point Your Tools at the Proxy

#### Claude Code

```bash
export ANTHROPIC_BASE_URL="http://localhost:4000"
export ANTHROPIC_API_KEY="sk-1234"
export DISABLE_TELEMETRY=1
export DISABLE_ERROR_REPORTING=1
claude
```

> **Why disable telemetry?** Even though your prompts and model traffic are routed through Bedrock via LiteLLM, Claude Code may still send telemetry and error reports directly to Anthropic's servers. Setting `DISABLE_TELEMETRY=1` and `DISABLE_ERROR_REPORTING=1` prevents this metadata from leaving your environment. This is recommended when working with protected data to ensure no information — even operational metadata — is sent outside the AWS boundary. See [claude-code#7031](https://github.com/anthropics/claude-code/issues/7031) for details.

#### Anthropic Python SDK

```python
from anthropic import Anthropic

client = Anthropic(
    base_url="http://localhost:4000",
    api_key="sk-1234",
)
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello!"}],
)
```

#### OpenAI-compatible clients

LiteLLM also exposes an OpenAI-compatible endpoint at `/v1/chat/completions`:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:4000/v1",
    api_key="sk-1234",
)
response = client.chat.completions.create(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

---

### 7. Clean Up (When Done)

Revert your shell to use the direct Anthropic API:

```bash
unset ANTHROPIC_BASE_URL
unset ANTHROPIC_API_KEY
```

Stop the proxy:

```bash
kill $(pgrep -f litellm)
```

---

## Caveats

- **Your AWS account, your responsibility** — standard IS-3 data handling practices still apply. The DPA coverage doesn't replace good data hygiene.
- **Local proxy = local exposure** — if you run LiteLLM on a shared or networked machine, secure it appropriately (change the `master_key`, bind to localhost only). For personal use on localhost, the defaults are fine.
- **Bedrock API keys are not IAM keys** — they are scoped to Bedrock only, but treat them as credentials. Don't commit them to code repositories. Set expiration dates and rotate periodically.
- **Model availability varies by region** — confirm your target models are available in your configured AWS region. The `us.` prefix in model IDs enables cross-region inference within US regions.
- **This does not cover direct Anthropic API use** — pointing tools at `api.anthropic.com` directly routes data outside the AWS boundary and is **not covered** under the UC/AWS DPA.
- **New models require config updates** — when Anthropic releases new models, you may need to add entries to `config.yaml` with the correct Bedrock model IDs.

---

## References

- [UCSB Cloud Services](https://aws.cloud.ucsb.edu)
- [UC IS-3 Electronic Information Security Policy](https://security.ucop.edu/policies/institutional-information-and-it-resource-classification.html)
- [LiteLLM Documentation](https://docs.litellm.ai/)
- [AWS Bedrock — Supported Models](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html)
- [AWS Bedrock API Keys](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys.html)
- [Securing Amazon Bedrock API Keys — Best Practices](https://aws.amazon.com/blogs/security/securing-amazon-bedrock-api-keys-best-practices-for-implementation-and-management/)
