# Blinksbuy MCP Orchestrator v2 – Production Safe (Phase 6)

This repo contains the Blinksbuy MCP Orchestrator v2 with:

- Thin orchestrator (`/orchestrate`)
- Intent service (OpenAI + stub)
- Flow service with tools for menu, order, recommend, tracking, profile
- Microservices for each domain (all HTTP POST)
- Strongly typed service I/O models (Phase 6)
- Loki-based structured logging and tracing

Run locally:

```bash
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Then call:

```bash
curl -X POST http://localhost:8000/orchestrate \\
  -H "Content-Type: application/json" \\
  -d '{
    "text": "Can you read me the menu?",
    "user_id": "user-123",
    "channel": "web"
  }'
```
