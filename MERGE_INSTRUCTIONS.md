# EC2 + OpenClaw Integration Guide

## Current State
- **EC2 Instance:** `i-0e2c7fb68a308e8e7` (g4dn.xlarge, running)
- **Public IP:** 3.236.47.203
- **Region:** us-east-1

## Architecture

### Main Agent (Clawd)
**Primary Model:** EC2 GPU → Claude Sonnet → Bedrock → Local Ollama

**Flow:**
1. Try EC2-hosted model (Llama 70B on GPU) - FAST, zero marginal cost
2. Fall back to Claude Sonnet (your current default) - reliable, high quality
3. Fall back to Bedrock Llama 3B - cheap, AWS-native
4. Fall back to local Ollama - completely free

### EC2 Manager Agent
**Role:** Instance lifecycle management
- Start/stop EC2 on demand
- Monitor costs (g4dn.xlarge ≈ $0.526/hour)
- Auto-shutdown after idle periods
- Health checks

## Implementation Steps

### Step 1: Set up model server on EC2

**Option A: vLLM (recommended for GPU)**
```bash
# SSH into your EC2 instance
ssh -i ~/.ssh/your-key.pem ubuntu@3.236.47.203

# Install vLLM
pip install vllm

# Run Llama 70B
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Meta-Llama-3.1-70B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 1
```

**Option B: Ollama (easier setup)**
```bash
# Install Ollama on EC2
curl -fsSL https://ollama.com/install.sh | sh

# Run in server mode
OLLAMA_HOST=0.0.0.0:11434 ollama serve &

# Pull a model
ollama pull llama3.1:70b
```

### Step 2: Configure Security Group
```bash
# Allow inbound on port 8000 (vLLM) or 11434 (Ollama)
aws ec2 authorize-security-group-ingress \
  --group-id $(aws ec2 describe-instances --instance-ids i-0e2c7fb68a308e8e7 \
    --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text) \
  --protocol tcp \
  --port 8000 \
  --cidr $(curl -s ifconfig.me)/32 \
  --region us-east-1
```

### Step 3: Merge config
```bash
# Apply the EC2 config patch
openclaw config patch ec2-openclaw-config.json

# Or manually merge into ~/.openclaw/openclaw.json
```

### Step 4: Activate EC2 Manager agent
```bash
# Talk to the EC2 manager
openclaw agent --id ec2-manager --message "Check EC2 instance status"

# Or spawn it as a background agent
openclaw sessions spawn --agent ec2-manager \
  --task "Monitor EC2 and shut down after 1 hour of inactivity" \
  --label "ec2-watchdog"
```

## Cost Optimization

### Automatic Shutdown Schedule
Add to the EC2 Manager agent's HEARTBEAT.md:
```markdown
## EC2 Cost Control
- Check instance runtime every 15 minutes
- If no model requests in last 30 minutes, warn user
- If no requests in 1 hour, auto-shutdown (with user consent)
- Estimated savings: ~$10/day when not in use
```

### Model Selection Strategy
- **Heavy reasoning tasks:** Use EC2 Llama 70B (fast, already paid for)
- **Quick queries:** Use Bedrock Llama 3B (cheap, serverless)
- **Critical tasks:** Use Claude Sonnet (most reliable)
- **Offline work:** Use local Ollama (zero cost)

## Testing

```bash
# Test EC2 model endpoint
curl http://3.236.47.203:8000/v1/models

# Test via OpenClaw
openclaw agent --model ec2-gpu/meta-llama/Meta-Llama-3.1-70B-Instruct \
  --message "Say hi in one sentence"

# Test EC2 manager
openclaw sessions send --label ec2-manager \
  --message "What's the current instance status and estimated cost today?"
```

## Security Notes
- EC2 endpoint is currently public (3.236.47.203)
- Consider using AWS PrivateLink or VPN
- Or restrict security group to your home IP only
