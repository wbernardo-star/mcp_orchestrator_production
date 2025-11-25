# MCP Orchestrator V2 – Production-Safe (Phase 1–6)

Ultra-modular, transport-agnostic, microservice-driven orchestration layer for AI-powered food-ordering, menu exploration, recommendations, and customer-profile automation.

This version represents the first fully production-safe milestone with:

- No stubs  
- Real HTTP POST microservice contracts  
- Strict Pydantic request/response validation (Phase 6)  
- Unified error handling  
- Full traceable envelope (trace_id, session_id, service_span)  
- Structured logging via Grafana Loki  
- Clean flow engine + tool layer  
- Adapter-ready, multi-channel compatible  

---

# Environment Variables (Complete Summary Table)

| Category | Variable | Required | Example | Description |
|---------|----------|----------|----------|-------------|
| **Core / Intent Classification** | `OPENAI_API_KEY` | Yes | `sk-xxxxx` | API key for LLM-based intent classification. If missing, system falls back to rules. |
|  | `OPENAI_MODEL` | Optional | `gpt-4o-mini` | LLM model used for intent classification. |
| **Microservices (All POST endpoints)** | `MENU_SERVICE_URL` | Yes | `https://hook.make.com/menu123` | Menu backend (POST). Must return JSON matching `MenuResponse`. |
|  | `ORDER_SERVICE_URL` | Yes | `https://hook.make.com/order456` | Order backend (POST). Validated using `OrderResponse`. |
|  | `RECOMMEND_SERVICE_URL` | Yes | `https://hook.make.com/reco789` | Recommendation backend (POST). Validated using `RecommendResponse`. |
|  | `TRACKING_SERVICE_URL` | Yes | `https://hook.make.com/track321` | Order tracking backend (POST). Validated using `TrackingResponse`. |
|  | `PROFILE_SERVICE_URL` | Yes | `https://hook.make.com/profile654` | User profile backend (POST). Supports get/save profile modes. |
| **Observability / Logging** | `GRAFANA_LOKI_URL` | Yes | `https://logs-prod.grafana.net/loki/api/v1/push` | Loki HTTP push endpoint. |
|  | `GRAFANA_LOKI_USERNAME` | Yes | `123456` | Grafana Cloud user / tenant ID. |
|  | `GRAFANA_LOKI_API_TOKEN` | Yes | `glc_xxx` | Token with “logs:write” permissions. |
|  | `MCP_APP_LABEL` | Optional | `mcp_orchestrator_v2` | Label for Loki log grouping. |
| **System / Runtime** | `PORT` | No (Railway sets) | `8080` | Uvicorn port when deployed. |
|  | `LOG_LEVEL` | Optional | `info` | Logging granularity. |
|  | `MCP_ENV` | Optional | `production` | Marks orchestrator environment (future extension). |

---

# Architecture

```
[Channel / Adapter] → [Orchestrator] → [Flow Engine]
                                   ↘
                                     [Tools Layer]
                                      ↘
                                        [Microservices via POST]
                                           → Menu
                                           → Order
                                           → Recommendations
                                           → Tracking
                                           → Customer Profile

---

```
# MCP JSON Envelope Travel Path

![Demo]()

# Endpoints

### GET /health  
Basic readiness check.

### POST /orchestrate  
Main endpoint for all channel traffic.

---

# Directory Structure

```
app/
 ├── main.py
 ├── flows/
 ├── tools/
 ├── services/
 ├── models/
 ├── logging_loki.py
 └── session_manager.py
README.md
requirements.txt
Procfile
```

---

# Observability & Traceability

All logs include:

- trace_id  
- session_id  
- event_type  
- service_type  
- io (in/out)  
- latency_ms  
- serialized request/response payloads  

Search in Grafana:

```
{ trace_id="trace-abc-123" }
```

---

# Typed Microservice Models

Includes:

- MenuResponse  
- OrderResponse  
- RecommendResponse  
- TrackingResponse  
- UserProfileResponse  

Fully validated using strict Pydantic models.

---

# Roadmap

- Phase 7 – Error taxonomy  
- Phase 8 – Transport-agnostic orchestration  
- Phase 9 – Multi-tenant routing  
- Phase 10 – Automated tests & mocks  
