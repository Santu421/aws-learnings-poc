# Employee Agent — AWS AgentCore + Strands + Okta JWT

A production-ready Employee Management agent running on **AWS AgentCore Runtime**,
using **Strands Agents SDK** for orchestration and a **FastMCP server** for tools.
Both user tokens and machine (M2M) tokens are accepted via Okta JWT inbound auth.

---

## Architecture

```
Caller (user or service)
  │  Authorization: Bearer <okta-token>
  ▼
AgentCore Inbound Auth          ← validates JWT signature, aud, client_id vs Okta OIDC
  │
  ▼
agent.py (Strands Agent)        ← extracts token from headers, forwards to MCP
  │  Authorization: Bearer <same-token>
  ▼
mcp_server.py (FastMCP)         ← decodes JWT, logs sub as first line on every tool call
```

- **One runtime** accepts both user tokens and machine tokens (both Okta client IDs in `allowedClients`)
- **Token is never re-issued** — the exact Okta JWT the caller sends is forwarded to MCP
- **MCP logs `sub=` as the very first line** on every tool invocation for full auditability

---

## Repo Structure

```
.
├── agent/
│   ├── agent.py
│   ├── requirements.txt
│   └── Dockerfile
├── mcp/
│   ├── mcp_server.py
│   ├── mcp_requirements.txt
│   └── Dockerfile.mcp
├── deploy.py
└── README.md
```

---

## Prerequisites

| Requirement | Notes |
|---|---|
| AWS account | `aws configure` done with admin permissions |
| Docker | For building and pushing images |
| Python 3.12+ | For running deploy script locally |
| Okta account | Need domain, audience, and two app client IDs |

### What to grab from Okta Admin Console

| Value | Where to find it |
|---|---|
| `OKTA_DOMAIN` | Settings → e.g. `dev-123456.okta.com` |
| `OKTA_AUDIENCE` | API → Authorization Servers → your server → Audience e.g. `api://default` |
| `USER_CLIENT_ID` | Applications → your user-facing app → Client ID |
| `M2M_CLIENT_ID` | Applications → your Service app → Client ID |
| `M2M_CLIENT_SECRET` | Same service app → Client Secrets (only needed to fetch M2M tokens) |

---

## Step 1 — AWS Console Setup

### 1a. Create ECR Repository
- `AWS Console → ECR → Create repository`
- Name: `emp-agent`
- Visibility: **Private**
- Copy the repository URI e.g. `123456789.dkr.ecr.us-east-1.amazonaws.com/emp-agent`

### 1b. Create IAM Role for AgentCore
- `IAM → Roles → Create role`
- Trusted entity: `bedrock-agentcore.amazonaws.com`
- Attach policies:
  - `AmazonBedrockFullAccess`
  - `AmazonEC2ContainerRegistryReadOnly`
  - `CloudWatchLogsFullAccess`
- Name: `AgentCoreEmpAgentRole`
- Copy the role ARN

---

## Step 2 — Build & Push Docker Images

### Agent image

```bash
AWS_ACCOUNT=123456789
REGION=us-east-1

aws ecr get-login-password --region $REGION \
  | docker login --username AWS \
  --password-stdin ${AWS_ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com

docker build -t emp-agent ./agent
docker tag emp-agent:latest ${AWS_ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/emp-agent:latest
docker push ${AWS_ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/emp-agent:latest
```

### MCP server image (deploy separately — ECS, EC2, or Fargate)

```bash
docker build -f mcp/Dockerfile.mcp -t emp-mcp ./mcp
# push to your preferred container host
```

---

## Step 3 — Deploy AgentCore Runtime

Edit `deploy.py` with your values and run:

```bash
pip install boto3
python deploy.py
```

This creates **one runtime** that accepts both user and machine Okta tokens.

---

## Step 4 — Get Okta Tokens (for testing)

### User token (password grant — for dev/testing only)

```bash
curl -s -X POST "https://<okta-domain>/oauth2/default/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=<USER_CLIENT_ID>" \
  -d "username=alice@yourcompany.com" \
  -d "password=YourPassword123" \
  -d "scope=openid profile" \
  | jq -r '.access_token'
```

### Machine token (client credentials)

```bash
curl -s -X POST "https://<okta-domain>/oauth2/default/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=<M2M_CLIENT_ID>" \
  -d "client_secret=<M2M_CLIENT_SECRET>" \
  -d "scope=emp:read emp:write" \
  | jq -r '.access_token'
```

---

## Step 5 — Invoke the Agent

Same endpoint for both token types — only the token changes:

```bash
AGENT_ENDPOINT="https://<agentcore-endpoint>/invocations"

# With user token
curl -s -X POST "$AGENT_ENDPOINT" \
  -H "Authorization: Bearer <user-token>" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "List everyone in Engineering"}' | jq .

# With machine token
curl -s -X POST "$AGENT_ENDPOINT" \
  -H "Authorization: Bearer <machine-token>" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Get employee E001"}' | jq .
```

---

## Code

### `agent/agent.py`

```python
import logging
import os
from bedrock_agentcore import BedrockAgentCoreApp
from strands import Agent
from strands.tools.mcp import MCPClient

logging.basicConfig(level=logging.INFO, format="%(levelname)-5s %(message)s")
log = logging.getLogger(__name__)

app = BedrockAgentCoreApp()
MCP_URL = os.environ.get("MCP_URL", "http://localhost:8000/mcp")

@app.entrypoint
def handle(payload: dict, context) -> dict:
    # Extract inbound Okta token from request headers
    headers = context.get("httpHeaders", {}) or context.get("headers", {})
    normalised = {k.lower(): v for k, v in headers.items()}
    auth_header = normalised.get("authorization", "")
    token = auth_header.removeprefix("Bearer ").removeprefix("bearer ").strip()

    if not token:
        return {"error": "No authorization token found in request"}

    log.info("Token received, forwarding to MCP")

    # Forward same token to MCP
    mcp_client = MCPClient(
        url=MCP_URL,
        headers={"Authorization": f"Bearer {token}"},
    )

    with mcp_client:
        agent = Agent(
            system_prompt=(
                "You are an Employee Management assistant. "
                "Help users look up employees, list teams, and update departments. "
                "Be concise and factual."
            ),
            tools=mcp_client.tools(),
        )
        response = agent(payload.get("prompt", ""))

    return {"response": str(response)}

if __name__ == "__main__":
    app.run()
```

### `agent/requirements.txt`

```
bedrock-agentcore
strands-agents
boto3
```

### `agent/Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY agent.py .
EXPOSE 8080
CMD ["python", "agent.py"]
```

---

### `mcp/mcp_server.py`

```python
import logging
import jwt
from fastmcp import FastMCP, Context

logging.basicConfig(level=logging.INFO, format="%(levelname)-5s %(message)s")
log = logging.getLogger(__name__)

mcp = FastMCP("employee-mcp")

# Swap this for your real database
EMPLOYEES = {
    "E001": {"name": "Alice Chen",  "dept": "Engineering", "role": "SWE"},
    "E002": {"name": "Bob Patel",   "dept": "HR",          "role": "Manager"},
    "E003": {"name": "Carol Diaz",  "dept": "Finance",     "role": "Analyst"},
    "E004": {"name": "Dave Kim",    "dept": "Engineering", "role": "Lead"},
}

def extract_sub(ctx: Context) -> str:
    """Decode the Bearer JWT and return the sub claim. No re-verification needed
    because AgentCore already validated the token before the agent ran."""
    try:
        auth = ctx.request_context.request.headers.get("authorization", "")
        token = auth.removeprefix("Bearer ").removeprefix("bearer ").strip()
        if not token:
            return "anonymous"
        claims = jwt.decode(
            token,
            options={"verify_signature": False},
            algorithms=["RS256"],
        )
        return claims.get("sub", "unknown")
    except Exception:
        return "decode-error"

@mcp.tool()
def get_employee(employee_id: str, ctx: Context) -> dict:
    sub = extract_sub(ctx)
    log.info("sub=%s | tool=get_employee | employee_id=%s", sub, employee_id)
    emp = EMPLOYEES.get(employee_id.upper())
    if not emp:
        return {"error": f"No employee found: {employee_id}"}
    return {"id": employee_id.upper(), **emp}

@mcp.tool()
def list_employees_by_department(department: str, ctx: Context) -> list:
    sub = extract_sub(ctx)
    log.info("sub=%s | tool=list_by_dept | department=%s", sub, department)
    return [
        {"id": eid, **emp}
        for eid, emp in EMPLOYEES.items()
        if emp["dept"].lower() == department.lower()
    ]

@mcp.tool()
def update_department(employee_id: str, new_department: str, ctx: Context) -> dict:
    sub = extract_sub(ctx)
    log.info("sub=%s | tool=update_department | employee_id=%s new_dept=%s",
             sub, employee_id, new_department)
    emp_id = employee_id.upper()
    if emp_id not in EMPLOYEES:
        return {"error": f"Employee {employee_id} not found"}
    EMPLOYEES[emp_id]["dept"] = new_department
    return {"success": True, "id": emp_id, "new_dept": new_department}

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8000)
```

### `mcp/mcp_requirements.txt`

```
fastmcp>=0.5.0
PyJWT>=2.8.0
cryptography>=41.0.0
```

### `mcp/Dockerfile.mcp`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY mcp_requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY mcp_server.py .
EXPOSE 8000
CMD ["python", "mcp_server.py"]
```

---

### `deploy.py`

```python
import boto3

client = boto3.client("bedrock-agentcore-control", region_name="us-east-1")

# ── Replace these values ──────────────────────────────────────────
ECR_IMAGE       = "123456789.dkr.ecr.us-east-1.amazonaws.com/emp-agent:latest"
EXECUTION_ROLE  = "arn:aws:iam::123456789:role/AgentCoreEmpAgentRole"
MCP_URL         = "http://your-mcp-server:8000/mcp"
OKTA_DOMAIN     = "dev-123456.okta.com"
OKTA_AUDIENCE   = "api://default"
USER_CLIENT_ID  = "0oaXXXXXXXXXXXXXX"   # Okta user-facing app
M2M_CLIENT_ID   = "0oaYYYYYYYYYYYYYY"   # Okta service / M2M app
# ─────────────────────────────────────────────────────────────────

response = client.create_agent_runtime(
    agentRuntimeName="emp-agent",
    description="Employee agent — accepts both user and machine Okta JWTs",

    agentRuntimeArtifact={
        "containerConfiguration": {
            "containerUri": ECR_IMAGE,
        }
    },

    executionRoleArn=EXECUTION_ROLE,

    environmentVariables={
        "MCP_URL": MCP_URL,
    },

    # One runtime, both client IDs accepted
    authorizerConfiguration={
        "customJWTAuthorizer": {
            "discoveryUrl": f"https://{OKTA_DOMAIN}/oauth2/default/.well-known/openid-configuration",
            "allowedAudience": [OKTA_AUDIENCE],
            "allowedClients": [USER_CLIENT_ID, M2M_CLIENT_ID],
        }
    },

    networkConfiguration={"networkMode": "PUBLIC"},
)

print("ARN:     ", response["agentRuntimeArn"])
print("Endpoint:", response["agentRuntimeEndpoint"])
```

---

## How AgentCore validates your Okta token

1. Fetches Okta's OIDC discovery document from `discoveryUrl`
2. Downloads Okta's public keys from `jwks_uri`
3. Verifies JWT signature
4. Checks `aud` claim matches `allowedAudience`
5. Checks `client_id` claim matches one of `allowedClients`
6. Pass → request reaches agent. Fail → `401 Unauthorized`

You write none of this validation code. It is all handled by `authorizerConfiguration`.

---

## MCP Audit Log Format

Every tool call logs `sub` as the first field:

```
INFO  sub=alice@corp.com | tool=get_employee        | employee_id=E001
INFO  sub=alice@corp.com | tool=list_by_dept        | department=Engineering
INFO  sub=svc-emp-sync   | tool=update_department   | employee_id=E003 new_dept=Engineering
```

`sub` is `alice@corp.com` for a human user, `svc-emp-sync` (or similar) for a machine client.

---

## Key Concepts

| Question | Answer |
|---|---|
| Does the agent die after each call? | No. Container stays alive. Each call gets an isolated session. |
| Bedrock Agents vs Strands Agents? | Strands = open source Python SDK, you own the code. Bedrock Agents = separate console-driven service, not used here. |
| Do we need two runtimes? | No. One runtime with both client IDs in `allowedClients`. |
| Is the token re-issued before MCP? | No. Same Okta token is forwarded as-is. |
| Does MCP re-verify the JWT signature? | No. AgentCore already validated it. MCP just decodes to read `sub`. |
