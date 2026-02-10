---
name: agent-deployment_ai_toolkit
description: Deploy AI apps, agents, and workflows in Microsoft Foundry.
---

### Deployment

**When to use**: User asks to deploy, publish, host, or go production with AI apps, agents, or workflows to Microsoft Foundry. Common trigger phrases include: "deploy to Foundry", "publish my agent", "host on Azure", "deploy to Azure AI Foundry", "go live", "production deployment", or "deploy my workflow".

#### Prerequisites

- Azure CLI installed and authenticated
- Azure AI Foundry Resource and Project created
- Agent must be wrapped as HTTP server using Agent-as-Server pattern. See [agent-as-server.md](reference/agent-as-server.md).


#### Deployment Workflow

**Step 1: Verify locally**

Verify agent runs locally as HTTP server.

**Step 2: Gather Context**

Call tools to collect deployment information:

- `aitk-list_foundry_models` - get user's Foundry project, subscription, and resource group
- Check if `Dockerfile` and `requirements.txt` exist in project directory

**Step 3: Collect Missing Information**

Ask user for information not available from tools:
- `AGENT_NAME` - name for the hosted agent
- `ENTRYPOINT` - main Python file to run, e.g., `app.py` (if Dockerfile needs to be generated)

**Step 4: Prepare Dockerfile**

If no `Dockerfile` exists, generate from template:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY ./ user_agent/

WORKDIR /app/user_agent

RUN if [ -f requirements.txt ]; then \
        pip install -r requirements.txt; \
    else \
        echo "No requirements.txt found"; \
    fi

EXPOSE 8088

ENV ASPNETCORE_URLS=http://+:8088
ENV ASPNETCORE_ENVIRONMENT=Production

CMD ["python", "<ENTRYPOINT_FILE>"]
```

Replace `<ENTRYPOINT_FILE>` with user's specified entry point.

**Step 5: Configure ACR**

**If ACR exists**: Use existing ACR directly, skip role assignment.

**If no ACR exists**: Create new ACR with ABAC repository permissions mode, then assign:
- **Container Registry Repository Reader** to Foundry project managed identity
- **Container Registry Repository Writer** to current user

**Step 6: Build and Push Container Image**

Use ACR remote build (no local Docker required):

```bash
IMAGE_TAG=$(cat /dev/urandom | tr -dc 'a-z0-9' | head -c 12)
IMAGE_NAME="$ACR_NAME.azurecr.io/$AGENT_NAME:$IMAGE_TAG"
az acr build --registry $ACR_NAME --image $IMAGE_NAME --subscription $SUB_ID --source-acr-auth-id "[caller]" <SOURCE_DIR>
```

**Important**: For ABAC-mode ACR, `--source-acr-auth-id "[caller]"` is required.

**Step 7: Deploy Agent Version**

Get access token and create agent version:

```bash
az account get-access-token --resource https://ai.azure.com --query accessToken --output tsv
```

```
POST https://$FOUNDRY_RESOURCE.services.ai.azure.com/api/projects/$PROJECT_NAME/agents/$AGENT_NAME/versions?api-version=2025-05-15-preview
```

**Body**:
```json
{
    "definition": {
        "kind": "hosted",
        "container_protocol_versions": [
            { "protocol": "RESPONSES", "version": "v1" }
        ],
        "cpu": "0.5",
        "memory": "1Gi",
        "image": "$IMAGE_NAME",
        "environment_variables": { "LOG_LEVEL": "debug" }
    }
}
```

Record `$AGENT_VERSION` from response.

**Step 8: Start Agent Container**

```
POST https://$FOUNDRY_RESOURCE.services.ai.azure.com/api/projects/$PROJECT_NAME/agents/$AGENT_NAME/versions/$AGENT_VERSION/containers/default:start?api-version=2025-05-15-preview
```

**Body**:
```json
{
    "min_replicas": 1,
    "max_replicas": 1
}
```

**Step 9: Validate Deployment**

Test the agent:

```
POST https://$FOUNDRY_RESOURCE.services.ai.azure.com/api/projects/$PROJECT_NAME/openai/responses?api-version=2025-05-15-preview
```

**Body**:
```json
{
    "agent": {
        "type": "agent_reference",
        "name": "$AGENT_NAME",
        "version": "$AGENT_VERSION"
    },
    "input": [
        {
            "role": "user",
            "content": [
                {
                    "type": "input_text",
                    "text": "Hi, what can you do"
                }
            ]
        }
    ],
    "stream": false
}
```

**Step 10: Post-Deployment**

- Create reusable deployment script in project root
- Save deployment summary to `.azure/foundry-deployment-summary.md`
