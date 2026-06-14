# LiteLLM Proxy on AWS Elastic Beanstalk

A containerized LiteLLM proxy that exposes Anthropic Claude models (via AWS Bedrock) through a single authenticated API endpoint, deployable to AWS Elastic Beanstalk with HTTPS support.

## Overview

This project runs a [LiteLLM](https://github.com/BerriAI/litellm) proxy server backed by PostgreSQL, routing requests to Claude Sonnet 4.6 and Claude Haiku 4.5 through AWS Bedrock. It provides an OpenAI-compatible API surface with bearer token authentication and a built-in management UI.

## Available Models

| Model alias | Bedrock model ID |
|---|---|
| `claude-sonnet-4-6` | `us.anthropic.claude-sonnet-4-6` |
| `claude-haiku-4-5` | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |

Both models are served from `us-east-1`.

## Prerequisites

- Docker and Docker Compose (local development)
- AWS CLI configured with credentials that have Bedrock access
- AWS Elastic Beanstalk CLI (`eb`) for deployment
- An AWS account with Bedrock model access enabled for Claude models in `us-east-1`

## Local Development

### 1. Set up environment variables

Copy the sample env file and fill in your credentials:

```bash
cp sample.env .env
```

Edit `.env`:

```env
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
LITELLM_MASTER_KEY=sk-your-master-key
```

### 2. Start the services

```bash
docker compose up -d
```

This starts:
- LiteLLM proxy on port **4000**
- PostgreSQL 16 on port **5432**

### 3. Test the API

```bash
curl -X POST http://localhost:4000/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-master-key" \
  -d '{
    "model": "claude-sonnet-4-6",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### 4. Management UI

Open `http://localhost:4000/ui/login/` in your browser and log in with your master key.

### Useful commands

```bash
docker compose logs -f litellm     # tail LiteLLM logs
docker compose down -v             # stop and remove all volumes
```

## AWS Elastic Beanstalk Deployment

The project includes EB extensions that automatically configure HTTPS with a self-signed certificate via nginx on the EC2 instance.

### 1. Initialize the EB application

```bash
eb init ai-proxy --platform "Docker" --region us-east-1
```

### 2. Create the environment

```bash
eb create ai-proxy-env \
  --single \
  --instance-type t3.small \
  --envvars AWS_ACCESS_KEY_ID=...,AWS_SECRET_ACCESS_KEY=...,LITELLM_MASTER_KEY=...
```

### 3. Check status

```bash
eb status
```

Once deployed, the proxy is available over HTTPS on the EB environment URL (port 443). Because the certificate is self-signed you will need to pass `-k` / `--insecure` in curl or configure your client to skip certificate verification.

### EB extensions

| File | Purpose |
|---|---|
| `.ebextensions/01-https-self-signed.config` | Generates a self-signed TLS cert and configures nginx to terminate HTTPS on port 443, redirecting HTTP |
| `.ebextensions/02-open-https-port.config` | Opens port 443 in the EC2 security group |

## Architecture

```
Client
  │
  ▼ HTTPS :443 (EB) / HTTP :4000 (local)
Nginx (reverse proxy)
  │
  ▼
LiteLLM proxy (port 4000)
  │               │
  ▼               ▼
PostgreSQL     AWS Bedrock
(state/logs)   (Claude inference)
```

## Environment Variables

| Variable | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS access key with Bedrock permissions |
| `AWS_SECRET_ACCESS_KEY` | Corresponding AWS secret key |
| `LITELLM_MASTER_KEY` | Bearer token clients use to authenticate (`sk-…`) |

## Model Configuration

Edit [litellm_config.yaml](litellm_config.yaml) to add or remove models. Each entry maps a friendly alias to a Bedrock model ID:

```yaml
model_list:
  - model_name: claude-sonnet-4-6
    litellm_params:
      model: bedrock/us.anthropic.claude-sonnet-4-6
      aws_region_name: us-east-1
```

Restart the container after any config change (`docker compose restart litellm`).
