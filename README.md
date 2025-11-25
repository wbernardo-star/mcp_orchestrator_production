# MCP Orchestrator V2 – Production-Safe (Phase 1–6)

Ultra-modular, transport-agnostic, microservice-driven orchestration layer for AI-powered food-ordering, menu exploration, recommendations, and customer-profile automation.

This version represents the **first fully production-safe milestone** with:

- No stubs  
- Real HTTP POST microservice contracts  
- Strict Pydantic request/response validation (Phase 6)  
- Unified error handling  
- Full traceable envelope (trace_id, session_id, service_span)  
- Structured logging via Grafana Loki  
- Clean flow engine + tool layer  
- Adapter-ready, multi-channel compatible  

## Table of Contents
1. Overview  
2. Architecture  
3. Transport & Adapter Model  
4. Microservice Layer  
5. Typed I/O Models (Phase 6)  
6. Observability & Traceability  
7. Endpoints  
8. Environment Variables  
9. Directory Structure  
10. Example End-to-End Flow  
11. How to Add New Microservices  
12. Compared to Typical “LLM-Only” Agents  
13. Roadmap  

## 1. Overview
The MCP Orchestrator is a thin, deterministic AI automation engine responsible for:
- Accepting `/orchestrate` input  
- Classifying intent using an LLM  
- Invoking microservices via POST  
- Strictly validating responses  
- Building the final user message  
- Logging every step with `trace_id`

## 2. Architecture
[Channel] → [Orchestrator] → [Flow Engine] → [Tools] → [Microservices via POST]

## 3. Transport & Adapter Model
Supports HTTP JSON today. Backend-agnostic—works with APIs, Make, n8n, or internal services.

## 4. Microservice Layer
Active services:
- Menu  
- Order  
- Recommendations  
- Tracking  
- Customer Profile  

All accept POST, validate inputs, and return typed responses.

## 5. Typed I/O Models
Includes:
- MenuResponse  
- OrderResponse  
- RecommendResponse  
- TrackingResponse  
- UserProfileResponse  

## 6. Observability & Traceability
Structured Loki logs include:
- trace_id  
- session_id  
- event_type  
- latency_ms  
- request/response payloads  

Queryable with:  
`{ trace_id="trace-abc-123" }`

## 7. Endpoints
### GET /health  
Returns service status.

### POST /orchestrate  
Main entry point.

## 8. Environment Variables
Core, Observability, and Microservice URLs.

## 9. Directory Structure
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

## 10. Example Flow
User → Orchestrator → Intent → Flow → MenuService → Flow → Output → Channel

## 11. Add New Microservices
Create service + models + tool + flow.

## 12. Comparison vs LLM-Only Agents
Typed, deterministic, traceable, multi-service, production-safe.

## 13. Roadmap
- Phase 7 error taxonomy  
- Phase 8 transport-agnostic  
- Phase 9 multi-tenant  
- Phase 10 automated testing  
