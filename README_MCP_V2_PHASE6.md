# MCP Orchestrator v2 (Phase 6)

A production-ready, **microservice-centric orchestration framework** for AI-driven restaurant and retail flows.

This version (Phase 6) focuses on:

- Clear separation of **Channel → Adapter → Orchestrator → Microservices**
- **Canonical JSON v2** envelopes for all interactions
- **Strongly typed I/O models** for every microservice
- **End-to-end traceability** via `trace_id` and structured logging
- **Production-safe** behaviour (no stubs, no hidden magic)

---

## 1. High-Level Architecture

```text
User / Actor
  ↓
Channel (Voice / Web / App)
  ↓
MCP Adapter (Canonical JSON v2)
  ↓
MCP Orchestrator v2
  ↓
  ┌───────────────┐
  │ Intent Service │
  └───────────────┘
          ↓
  ┌───────────────┐
  │  Flow Engine  │
  └───────────────┘
          ↓
Microservice Layer (HTTP POST)
  • Menu Service
  • Order Service
  • Recommend Service
  • Tracking Service
  • Customer Profile Service
  ↓
Orchestrator reply → Adapter → Channel → User
```

**Traceability:**  
Every request journey uses a single `trace_id` propagated across:

- Channel input
- Adapter request to orchestrator
- Intent + Flow internal calls
- Microservice calls (menu, order, recommend, tracking, profile)
- Orchestrator response
- Adapter response back to the channel

You can reconstruct the entire flow in Loki / Grafana using:

```logql
{ trace_id="<value>" }
```

---

## 2. How MCP v2 Differs from Typical Agent Frameworks (Generic Comparison)

Many modern AI solutions expose **agent-like frameworks** where a single LLM controls tools and routing logic. MCP v2 takes a different approach.

### 2.1 Typical Agent Architecture (Generic)

- LLM decides which tools to call and when  
- Tool I/O is often loosely structured JSON  
- Business logic lives inside prompts  
- Observability is limited to LLM logs  
- Scaling tools and backends is harder  
- Often built around one vendor’s transport/protocol

### 2.2 MCP Orchestrator v2 Approach

| Aspect                   | Typical Agent Framework (Generic)                     | MCP Orchestrator v2                                      |
|--------------------------|--------------------------------------------------------|----------------------------------------------------------|
| Core design              | LLM-centric (model controls everything)               | **Microservice-centric** orchestration                   |
| Orchestration logic      | Hidden in prompts/model responses                     | **Explicit Flow Engine** (code, testable, versioned)     |
| Tool integration         | Model decides tools; loosely typed responses          | **Dedicated microservices** with **strict JSON contracts** |
| Transport                | Often single-transport (WebSocket/HTTP)               | Designed to be **transport-agnostic** at core            |
| Observability            | Model logs + partial metadata                         | **Full `trace_id` across all layers**, structured logs   |
| Safety                   | JSON output parsing, best-effort validation           | **Strongly typed I/O** + error envelopes + validation    |
| Backend integration      | Tools often tied to vendor ecosystem                  | Any backend: **APIs, webhooks, n8n, Make, POS, DB**      |
| Vendor lock-in           | Often tied to one LLM vendor                          | **LLM-agnostic**, intent service can swap providers      |

### 2.3 Why This Matters

- You are building **infrastructure**, not just a chatbot.  
- Microservices can scale independently.  
- Backend integrations can evolve without touching the orchestrator or adapter.  
- You can plug in any LLM provider into the intent layer without touching flows or microservices.  

---

## 3. Canonical JSON Envelopes (End-to-End)

This section documents the **official JSON envelopes** used across the system.

### 3.1 Channel → MCP Adapter Input (`channel_input.json`)

This is what a channel (e.g. ElevenLabs, web frontend, mobile app) sends into the MCP Adapter.

```json
{
  "version": "1.1",
  "timestamp": "2025-11-21T21:28:56.146Z",
  "context": {
    "channel": "voice",
    "device": "browser",
    "locale": "en-US",
    "tenant": "blinksbuy",
    "client_app": "elevenlabs-frontend",
    "llm": {
      "model_hint": "gpt-4.1-mini",
      "temperature": 0.3,
      "extra": {
        "voice_id": "elevenlabs-voice-123"
      }
    }
  },
  "session": {
    "session_id": "sess-123:web",
    "conversation_id": "conv-abc-001",
    "user_id": "user-123",
    "turn": 1
  },
  "request": {
    "type": "text",
    "text": "Can you read me the menu?",
    "audio_url": null,
    "transcript": "can you read me the menu",
    "image_url": null,
    "alt_text": null,
    "intent_override": null,
    "metadata": {
      "source": "elevenlabs",
      "client": "web",
      "raw_transcript": "can you read me the menu"
    }
  },
  "observability": {
    "trace_id": "trace-abc-123",
    "span_id": "span-1",
    "message_id": "msg-1"
  }
}
```

---

### 3.2 MCP Adapter → Orchestrator Input (`mcp_adapter_request.json`)

The adapter converts the canonical envelope into a **thin orchestrator request**:

```json
{
  "text": "Can you read me the menu?",
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123",

  "order_items": [],
  "order_payment_method": null,
  "order_delivery_mode": null,
  "order_special_instructions": null,
  "order_table_number": null,

  "tracking_order_id": null,

  "profile_mode": null,
  "profile_preferences": null
}
```

Only the relevant fields are used depending on the intent (menu, order, tracking, profile, etc.).

---

### 3.3 Orchestrator → Adapter Output (`mcp_orchestrator_response.json`)

The orchestrator returns a **simple decision + reply** envelope:

```json
{
  "decision": "reply",
  "reply_text": "Here is the menu from Uno Bistro:
1. Garlic Chicken – 500
2. Sisig ni Mayor – 400",
  "session_id": "sess-123:web",
  "route": "menu",
  "intent": "menu",
  "intent_confidence": 0.92,
  "trace_id": "trace-abc-123",
  "structured": {
    "menu": {
      "categories": [
        {
          "name": "Best Sellers",
          "items": [
            {
              "id": "menu-001",
              "name": "Garlic Chicken",
              "price": 500,
              "currency": "PHP"
            }
          ]
        }
      ]
    }
  },
  "meta": {
    "latency_ms": 160.2,
    "turn": 1
  }
}
```

The adapter then wraps this into a canonical response envelope for the channel (not shown here).

---

## 4. Internal Service I/O Envelopes

### 4.1 Intent Service

**Request** (`intent_service_input.json`):

```json
{
  "text": "Can you read me the menu?",
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123",
  "history": [
    {
      "role": "user",
      "content": "Hi",
      "timestamp": "2025-11-21T21:28:00Z"
    }
  ]
}
```

**Response** (`intent_service_output.json`):

```json
{
  "intent": "menu",
  "confidence": 0.92,
  "raw_reasoning": "User asked to read the menu; classify as 'menu'."
}
```

---

### 4.2 Flow Service

**Request** (`flow_service_input.json`):

```json
{
  "intent": "menu",
  "text": "Can you read me the menu?",
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123",
  "order_items": [],
  "tracking_order_id": null,
  "profile_preferences": null
}
```

**Response** (`flow_service_output.json`):

```json
{
  "reply_text": "Here is the menu from Uno Bistro:
1. Garlic Chicken – 500
2. Sisig ni Mayor – 400",
  "route": "menu",
  "next_intent_hint": null,
  "context_flags": {
    "menu_read": true
  }
}
```

The Flow Service is the orchestrator’s “domain logic” brain: it decides which microservice(s) to call.

---

## 5. Microservice Layer I/O (HTTP POST)

All microservices use:

- **POST**
- **JSON body**
- A common base:

```json
{
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123"
}
```

Then each service adds its own fields.

---

### 5.1 Menu Service

**Request** (`menu_service_input.json`):

```json
{
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123",
  "locale": "en-US",
  "tenant": "blinksbuy",
  "category": null,
  "dietary_preferences": ["no_pork"]
}
```

**Response** (`menu_service_output.json`):

```json
{
  "output": "Here is the menu from Uno Bistro:
1. Garlic Chicken – 500
2. Sisig ni Mayor – 400",
  "categories": [
    {
      "name": "Best Sellers",
      "items": [
        {
          "id": "menu-001",
          "name": "Garlic Chicken",
          "description": "Crispy chicken with garlic sauce",
          "price": 500,
          "currency": "PHP",
          "tags": ["chicken", "best_seller"]
        }
      ]
    }
  ]
}
```

---

### 5.2 Order Service

**Request** (`order_service_input.json`):

```json
{
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123",
  "items": [
    {
      "id": "menu-001",
      "name": "Garlic Chicken",
      "quantity": 1,
      "price": 500,
      "currency": "PHP"
    }
  ],
  "payment_method": "card",
  "delivery_mode": "dine_in",
  "special_instructions": "no onions",
  "table_number": "A1",
  "customer_note": null
}
```

**Response** (`order_service_output.json`):

```json
{
  "order_id": "ORD-2025-0001",
  "status": "confirmed",
  "eta_minutes": 15,
  "total_amount": 500,
  "currency": "PHP"
}
```

---

### 5.3 Recommend Service

**Request** (`recommend_service_input.json`):

```json
{
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123",
  "context": "User likes spicy food, budget 300–800, no pork.",
  "max_items": 5
}
```

**Response** (`recommend_service_output.json`):

```json
{
  "recommendations": [
    {
      "id": "menu-010",
      "name": "Spicy Sisig",
      "description": "Classic sisig with extra chili",
      "price": 450,
      "currency": "PHP",
      "reason": "Spicy and fits your budget"
    },
    {
      "id": "menu-011",
      "name": "Spicy Garlic Chicken",
      "price": 520,
      "currency": "PHP",
      "reason": "You ordered Garlic Chicken before and like spicy."
    }
  ]
}
```

---

### 5.4 Tracking Service

**Request** (`tracking_service_input.json`):

```json
{
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123",
  "order_id": "ORD-2025-0001"
}
```

**Response** (`tracking_service_output.json`):

```json
{
  "order_id": "ORD-2025-0001",
  "status": "on_the_way",
  "eta_minutes": 7
}
```

---

### 5.5 Profile Service

**Request (get_profile)** (`profile_service_input_get.json`):

```json
{
  "mode": "get_profile",
  "user_id": "user-123",
  "channel": "web",
  "session_id": "sess-123:web",
  "trace_id": "trace-abc-123"
}
```

**Response (get_profile)** (`profile_service_output_get.json`):

```json
{
  "preferences": {
    "dietary": ["no_pork"],
    "spice_level": "medium",
    "allergies": ["peanuts"]
  },
  "order_history_summary": {
    "total_orders": 12,
    "favorite_items": ["Garlic Chicken", "Airport Express"],
    "avg_spend": 520
  }
}
```

---

## 6. Mermaid Data Flow Diagram

You can render this diagram in GitHub, docs tools, or Mermaid Live Editor:

```mermaid
flowchart TD
    classDef actor fill:#fdf6e3,stroke:#555,stroke-width:1px;
    classDef layer fill:#eef3ff,stroke:#555,stroke-width:1px;
    classDef svc fill:#f9f9ff,stroke:#777,stroke-width:1px,font-size:12px;
    classDef json fill:#fff,stroke:#aaa,stroke-width:1px,font-size:11px,font-style:italic;

    U[User / Actor]:::actor
    C[Channel Layer<br/>(Voice / Web / App)]:::layer
    CI[[channel_input.json]]:::json
    A[MCP Adapter]:::layer
    AR[[mcp_adapter_request.json]]:::json
    O[MCP Orchestrator v2]:::layer

    I[Intent Service]:::svc
    II[[intent_service_input.json]]:::json
    IO[[intent_service_output.json]]:::json

    F[Flow Engine]:::svc
    FI[[flow_service_input.json]]:::json
    FO[[flow_service_output.json]]:::json

    subgraph MS[Microservice Layer]
        direction LR
        MMenu[Menu Service]:::svc
        MOrder[Order Service]:::svc
        MRec[Recommend Service]:::svc
        MTrack[Tracking Service]:::svc
        MProf[Profile Service]:::svc
    end

    MMenuIn[[menu_service_input.json]]:::json
    MMenuOut[[menu_service_output.json]]:::json
    MOrderIn[[order_service_input.json]]:::json
    MOrderOut[[order_service_output.json]]:::json
    MRecIn[[recommend_service_input.json]]:::json
    MRecOut[[recommend_service_output.json]]:::json
    MTrackIn[[tracking_service_input.json]]:::json
    MTrackOut[[tracking_service_output.json]]:::json
    MProfIn[[profile_service_input_get.json]]:::json
    MProfOut[[profile_service_output_get.json]]:::json

    OR[[mcp_orchestrator_response.json]]:::json

    U --> C --> CI --> A --> AR --> O
    O --> II --> I --> IO --> O
    O --> FI --> F --> FO --> O

    F --> MMenuIn --> MMenu --> MMenuOut --> F
    F --> MOrderIn --> MOrder --> MOrderOut --> F
    F --> MRecIn --> MRec --> MRecOut --> F
    F --> MTrackIn --> MTrack --> MTrackOut --> F
    F --> MProfIn --> MProf --> MProfOut --> F

    O --> OR --> A --> C --> U
```

---

## 7. Why This Is Ready for Production

- No stub logic left in the microservices  
- Strongly-typed request/response contracts  
- Traceable with a single `trace_id`  
- Each microservice can be independently wired to real systems (Make, n8n, APIs, DBs)  
- Orchestrator is thin and focused on coordination, not business rules  
- Flow engine and microservices are easy to extend and maintain  

This README should be sufficient for:

- Internal architecture reviews  
- Demo to stakeholders  
- Developer onboarding  
- Future extension to multi-transport MCP v2.5 / v3.0
