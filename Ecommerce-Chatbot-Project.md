Ecommerce Chatbot Project – Unified Technical Design Document (LLD + API Mapping + Interaction Maps)
Thursday, December 18, 2025
12:19 AM
 
1. Document Overview
This is a single, end-to-end Technical Design Document for the Ecommerce-Chatbot-Project that captures:
•	LLD (module-level responsibilities and internal components)
•	API level mapping to modules (ownership, purpose, dependencies)
•	Full API catalog (contracts: request/response, params, status codes, errors)
•	Interaction map (module-to-module interactions with explicit API calls)
•	Data contracts (product, vector docs, chat session state, order)
•	AI/RAG workflow (LangChain + LangGraph, Pinecone, embeddings, LLM)
•	Ops / pipeline design (Airflow DAGs, runs, rebuilds)
•	Security, observability, resilience, versioning, deployment
Note: Where the repository does not explicitly define a route or storage engine, this document defines a recommended API surface that fits the repo’s stated behavior (RAG chatbot + e-commerce actions + ingestion pipeline). This ensures the project is implementation-ready and reviewable.
 
2. System Objectives
2.1 Functional Objectives
1.	Conversational shopping assistant: answer questions, recommend products.
2.	RAG-based accuracy: retrieval from vector DB so answers are grounded.
3.	Order flows: allow “buy now”, order confirmation, order tracking.
4.	Session continuity: maintain chat memory for follow-ups.
5.	Fresh data: daily pipeline refresh with newly scraped product data.
2.2 Non-Functional Objectives
•	Low latency chat responses (p95 target: set per environment)
•	High availability of chat even if ingestion fails
•	Secure key handling and rate limiting
•	Traceable outputs (sources, tool traces)
•	Observable pipeline health and runtime metrics
 
3. Architecture Summary
3.1 Components
•	Web App (Flask): storefront pages + chatbot widget + REST endpoints
•	Chat Orchestrator: intent routing + RAG retrieval + prompt assembly + LLM call + memory
•	Catalog/Search: product listing + semantic search
•	Order Management: create orders, invoice, status
•	Vector Store (Pinecone): semantic retrieval index
•	Embeddings (NVIDIA): embedding generation for docs and queries
•	LLM Provider (Groq): Llama family model for response generation
•	Pipeline (Airflow): scrape → clean → embed → upsert → refresh
3.2 Data Stores
•	Vector Store: Pinecone index
•	Product Store (recommended): Postgres / SQLite / files (CSV/Parquet). If not present, use dataset files + in-memory caching.
•	Order Store (recommended): DB table or JSON store (for demo). For production, DB + audit log.
•	Chat State Store (recommended): Redis (or in-memory for local). LangGraph state can be persisted.
 
4. Module LLD
4.1 Web/UI Module (Flask + Templates/Static)
Responsibilities
•	Serve website pages and assets
•	Render product pages
•	Host chatbot widget UI
Internal Components
•	Jinja templates
•	Static assets (CSS/JS)
•	Controller routes for pages
Owned Endpoints (Page / Health)
•	GET / (home)
•	GET /products (product list page)
•	GET /product/{product_id} (product detail page)
•	GET /health (basic service health)
 
4.2 API Gateway Layer (Flask REST controllers)
Responsibilities
•	Request validation
•	Response formatting
•	Auth checks (if enabled)
•	Rate limits (especially chat)
Owned Endpoint Groups
•	/api/chat/*
•	/api/products/*
•	/api/search/*
•	/api/orders/*
•	/internal/* (ops)
 
4.3 Chat Orchestrator Module (RAG + Memory)
Responsibilities
•	Session state management (LangGraph)
•	Intent detection
•	Query rewriting (optional)
•	Retrieval from Pinecone
•	Prompt assembly + response generation (Groq)
•	Tool routing for order/tracking
•	Citation + trace output
Internal Components
•	IntentRouter
•	RetrieverClient (Pinecone)
•	EmbeddingClient (NVIDIA)
•	LLMClient (Groq)
•	PromptTemplates
•	MemoryStore (LangGraph state)
•	ToolExecutor (calls Catalog/Order APIs)
Owned Endpoints
•	POST /api/chat/message
•	POST /api/chat/reset
•	GET /api/chat/session/{session_id} (optional)
 
4.4 Catalog & Search Module
Responsibilities
•	Store product metadata
•	Filtered browsing and lookup
•	Semantic search entrypoint
Internal Components
•	ProductRepository (DB/file)
•	CatalogFilters (category, price, rating, brand)
•	SemanticSearchAdapter (Pinecone query wrapper)
Owned Endpoints
•	GET /api/products
•	GET /api/products/{product_id}
•	GET /api/categories
•	POST /api/search/semantic
 
4.5 Order Management Module
Responsibilities
•	Create orders from cart/chat intents
•	Validate products and compute totals
•	Persist order state and timeline
•	Provide invoice and status
Order State Machine
•	CREATED → CONFIRMED → SHIPPED → DELIVERED → CLOSED
•	Optional: CANCELLED, RETURN_REQUESTED, RETURNED, REFUNDED
Internal Components
•	OrderRepository (DB)
•	PricingEngine
•	OrderStatusService
•	InvoiceGenerator
Owned Endpoints
•	POST /api/orders
•	GET /api/orders/{order_id}
•	GET /api/orders/{order_id}/status
•	GET /api/orders/{order_id}/invoice
•	POST /api/orders/{order_id}/cancel (recommended)
 
4.6 Ingestion & MLOps Module (Airflow)
Responsibilities
•	Daily scraping and dataset refresh
•	Data cleaning and schema normalization
•	Embedding generation
•	Pinecone index management
•	Post-run smoke tests
Internal Components
•	Airflow DAGs: scrape_dag, clean_dag, embed_dag, upsert_dag, smoke_test_dag (can be single DAG with tasks)
Owned Endpoints (Internal/Ops)
•	POST /internal/ingestion/run (manual trigger)
•	GET /internal/ingestion/status (last run info)
•	POST /internal/vector/rebuild (force re-embed/upsert)
•	GET /internal/vector/stats (index health)
 
5. API Level Mapping to Modules (Ownership Matrix)
5.1 Public Page Routes
API	Method	Owner Module	Purpose	Downstream Calls
/	GET	Web/UI	Home page	none
/products	GET	Web/UI	Product list page	GET /api/products (optional SSR)
/product/{product_id}	GET	Web/UI	Product detail page	GET /api/products/{id}
/health	GET	Web/UI	Liveness/health	checks: Pinecone/Groq/NVIDIA (optional)
5.2 Chat APIs
API	Method	Owner Module	Purpose	Downstream Calls
/api/chat/message	POST	Chat Orchestrator	Chat entrypoint (RAG + memory + tools)	Pinecone query, Groq completion, optional Catalog/Order APIs
/api/chat/reset	POST	Chat Orchestrator	Clear session memory	Memory store
/api/chat/session/{session_id}	GET	Chat Orchestrator	Fetch session state (debug/admin)	Memory store
5.3 Catalog/Search APIs
API	Method	Owner Module	Purpose	Downstream Calls
/api/products	GET	Catalog/Search	List products with filters	Product store
/api/products/{product_id}	GET	Catalog/Search	Product detail	Product store
/api/categories	GET	Catalog/Search	Category list	Product store
/api/search/semantic	POST	Catalog/Search	Semantic product search	Pinecone + (optional) product store
5.4 Order APIs
API	Method	Owner Module	Purpose	Downstream Calls
/api/orders	POST	Order Mgmt	Create order	Catalog lookup + pricing engine
/api/orders/{order_id}	GET	Order Mgmt	Order detail	Order store
/api/orders/{order_id}/status	GET	Order Mgmt	Current status + ETA	Order store
/api/orders/{order_id}/invoice	GET	Order Mgmt	Invoice view/export	Order store
/api/orders/{order_id}/cancel	POST	Order Mgmt	Cancel order	Order store
5.5 Internal/Ops APIs
API	Method	Owner Module	Purpose	Downstream Calls
/internal/ingestion/run	POST	Ingestion/MLOps	Trigger ingestion	Airflow REST/CLI trigger
/internal/ingestion/status	GET	Ingestion/MLOps	Last run status	Airflow metadata
/internal/vector/rebuild	POST	Ingestion/MLOps	Force re-embed/upsert	NVIDIA embed + Pinecone upsert
/internal/vector/stats	GET	Ingestion/MLOps	Index stats	Pinecone describe
/internal/metrics	GET	Observability	Metrics endpoint	app metrics store
 
6. API Catalog (Detailed Contracts)
6.1 Common Conventions
•	Base URL: http://host:port
•	Content-Type: application/json
•	Authentication (recommended):
o	Public pages: none
o	Public APIs: optional API key for abuse prevention
o	Internal APIs: API key or network-restricted
•	Correlation IDs (recommended):
o	Request header: X-Request-Id (optional)
o	Response header: X-Request-Id
6.2 Standard Error Shape
{
  "error": {
    "code": "VECTOR_DB_UNAVAILABLE",
    "message": "Pinecone service unreachable",
    "request_id": "...",
    "details": {"retry_after_s": 5}
  }
}
6.3 Chat APIs
6.3.1 POST /api/chat/message
Owner: Chat Orchestrator
Purpose
•	Accept user message
•	Determine intent (product Q&A, recommendation, order, tracking)
•	Run retrieval from vector store
•	Generate grounded response
•	Return citations and traces
Request
{
  "session_id": "uuid-or-cookie-id",
  "message": "Recommend a saree under 2000",
  "channel": "web",
  "locale": "en-IN",
  "context": {
    "budget": 2000,
    "category": "sarees",
    "min_rating": 4.0
  },
  "debug": false
}
Response
{
  "session_id": "uuid-or-cookie-id",
  "reply": "Here are 3 sarees under ₹2000 with 4.0+ rating...",
  "citations": [
    {
      "product_id": "AMZ-SR-001",
      "name": "Brand A Saree",
      "price_sale": 1899,
      "rating": 4.2,
      "source": "pinecone"
    }
  ],
  "detected_intent": "recommendation",
  "tool_trace": {
    "retrieval_top_k": 6,
    "vector_namespace": "sarees",
    "llm_provider": "groq",
    "llm_model": "llama-3.3-70b-versatile",
    "latency_ms": {"retrieve": 120, "llm": 820}
  }
}
Status Codes
•	200 OK
•	400 INVALID_REQUEST (missing message/session_id)
•	401 UNAUTHORIZED (if auth enabled)
•	429 RATE_LIMITED
•	502 LLM_UPSTREAM_ERROR
•	503 VECTOR_DB_UNAVAILABLE
Notes
•	If retrieval returns low confidence / no results, respond with clarifying questions and avoid hallucinations.
 
6.3.2 POST /api/chat/reset
Owner: Chat Orchestrator
Request
{ "session_id": "uuid" }
Response
{ "session_id": "uuid", "status": "cleared" }
Status Codes: 200, 400, 401
 
6.3.3 GET /api/chat/session/{session_id} (Optional)
Owner: Chat Orchestrator
Response
{
  "session_id": "uuid",
  "state": {
    "chat_history_count": 12,
    "last_intent": "product_query",
    "last_retrieved_products": ["AMZ123", "AMZ456"]
  }
}
Status Codes: 200, 404, 401
 
6.4 Catalog & Search APIs
6.4.1 GET /api/products
Owner: Catalog/Search
Query Parameters
•	category (string)
•	brand (string)
•	min_price, max_price (number)
•	min_rating (number)
•	sort (e.g., price_asc, rating_desc)
•	limit (default 20)
•	offset (default 0)
Response
{
  "items": [
    {
      "product_id": "AMZ123",
      "category": "shirts",
      "brand": "XYZ",
      "name": "Formal Shirt",
      "selling_price": 1399,
      "mrp": 2499,
      "discount_pct": 44,
      "rating": 4.2,
      "rating_count": 1823
    }
  ],
  "page": {"limit": 20, "offset": 0, "total": 2500}
}
 
6.4.2 GET /api/products/{product_id}
Owner: Catalog/Search
Response
{
  "product_id": "AMZ123",
  "category": "shirts",
  "brand": "XYZ",
  "name": "Formal Shirt",
  "selling_price": 1399,
  "mrp": 2499,
  "discount_pct": 44,
  "rating": 4.2,
  "rating_count": 1823,
  "highlights": ["cotton", "regular fit"],
  "source_url": "..."
}
 
6.4.3 GET /api/categories
Owner: Catalog/Search
Response
{ "categories": ["shirts", "sarees", "watches"] }
 
6.4.4 POST /api/search/semantic
Owner: Catalog/Search
Purpose
•	Semantic retrieval for products
•	Direct endpoint for UI/search page and for chat debugging
Request
{
  "query": "lightweight cotton shirt for office",
  "top_k": 8,
  "filters": {
    "category": "shirts",
    "max_price": 1500,
    "min_rating": 4.0
  }
}
Response
{
  "matches": [
    {
      "product_id": "AMZ123",
      "score": 0.86,
      "metadata": {"selling_price": 1399, "rating": 4.2, "brand": "XYZ"}
    }
  ]
}
Status Codes
•	200, 400, 503 (Pinecone down)
 
6.5 Order APIs
6.5.1 POST /api/orders
Owner: Order Mgmt
Request
{
  "session_id": "uuid",
  "customer": {"name": "User", "phone": "+91xxxxxxxxxx"},
  "items": [
    {"product_id": "AMZ123", "qty": 1},
    {"product_id": "AMZ999", "qty": 2}
  ],
  "shipping_address": {"line1": "...", "city": "...", "state": "...", "pincode": "..."}
}
Response
{
  "order_id": "ORD-2025-00021",
  "status": "CONFIRMED",
  "pricing": {"subtotal": 4397, "shipping": 0, "tax": 0, "total": 4397},
  "created_at": "2025-12-18T23:10:00+05:30"
}
Status Codes
•	201 CREATED
•	400 INVALID_PRODUCT / 400 INVALID_QTY
•	409 OUT_OF_STOCK (if supported)
 
6.5.2 GET /api/orders/{order_id}
Response
{
  "order_id": "ORD-2025-00021",
  "status": "CONFIRMED",
  "items": [{"product_id": "AMZ123", "qty": 1, "unit_price": 1399}],
  "pricing": {"total": 1399},
  "timeline": [{"status": "CONFIRMED", "at": "..."}]
}
 
6.5.3 GET /api/orders/{order_id}/status
Response
{ "order_id": "ORD-2025-00021", "status": "SHIPPED", "eta": "2025-12-22" }
 
6.5.4 GET /api/orders/{order_id}/invoice
Response
{
  "order_id": "ORD-2025-00021",
  "invoice_no": "INV-2025-01001",
  "line_items": [{"name": "Formal Shirt", "qty": 1, "amount": 1399}],
  "total": 1399,
  "currency": "INR",
  "format": "json"
}
 
6.5.5 POST /api/orders/{order_id}/cancel (Recommended)
Request
{ "reason": "Ordered by mistake" }
Response
{ "order_id": "ORD-2025-00021", "status": "CANCELLED" }
 
6.6 Internal/Ops APIs
6.6.1 POST /internal/ingestion/run
Owner: Ingestion/MLOps
Purpose
•	Manually trigger pipeline (in addition to daily schedule)
Request
{ "pipeline": "daily_refresh", "categories": ["shirts", "sarees"], "force": false }
Response
{ "run_id": "airflow-2025-12-18T18:00Z", "status": "TRIGGERED" }
 
6.6.2 GET /internal/ingestion/status
Response
{
  "last_run_id": "airflow-...",
  "last_run_status": "SUCCESS",
  "stats": {"products_scraped": 1234, "vectors_upserted": 1234},
  "completed_at": "..."
}
 
6.6.3 POST /internal/vector/rebuild
Purpose
•	Re-embed and re-upsert (useful after embedding model change)
Request
{ "namespace": "shirts", "force": true }
Response
{ "status": "STARTED", "namespace": "shirts" }
 
6.6.4 GET /internal/vector/stats
Response
{
  "index": "ecom-products",
  "namespaces": [{"name": "shirts", "vector_count": 50000}],
  "dimension": 1024
}
 
7. Interaction Map (Module-to-Module with API Calls)
7.1 Runtime Chat Flow (RAG + Memory + Citations)
Scenario: User asks for product recommendation.
1.	UI → Chat API
•	POST /api/chat/message
2.	Chat Orchestrator → Embeddings (NVIDIA)
•	internal call: embed(query)
3.	Chat Orchestrator → Vector Store (Pinecone)
•	internal call: query(top_k, filters)
4.	Chat Orchestrator → LLM (Groq)
•	internal call: completion(prompt + retrieved_docs + history)
5.	Chat Orchestrator → UI
•	returns response with citations
Sequence diagram (Mermaid)
sequenceDiagram
  participant UI as Web UI
  participant ChatAPI as POST /api/chat/message
  participant Chat as Chat Orchestrator
  participant Emb as NVIDIA Embeddings
  participant Vec as Pinecone
  participant LLM as Groq LLM
UI->>ChatAPI: message(session_id, text, context)
  ChatAPI->>Chat: validate + route
  Chat->>Emb: embed(query)
  Emb-->>Chat: query_vector
  Chat->>Vec: query(top_k, filters)
  Vec-->>Chat: docs + metadata
  Chat->>LLM: generate(prompt + docs + history)
  LLM-->>Chat: response
  Chat-->>ChatAPI: reply + citations + trace
  ChatAPI-->>UI: JSON reply
 
7.2 Chat → Order Creation Flow (Tool Call)
Scenario: User says “Buy AMZ123 qty 2”.
1.	UI → POST /api/chat/message
2.	Chat detects intent: order_create
3.	Chat calls Order API:
o	POST /api/orders
4.	Order service validates products:
o	GET /api/products/{product_id} (or internal repo call)
5.	Order returns confirmation; chat replies to user.
sequenceDiagram
  participant UI as Web UI
  participant ChatAPI as POST /api/chat/message
  participant OrderAPI as POST /api/orders
  participant CatalogAPI as GET /api/products/{id}
UI->>ChatAPI: "Buy AMZ123 x2"
  ChatAPI->>OrderAPI: create_order(items)
  OrderAPI->>CatalogAPI: validate product + fetch price
  CatalogAPI-->>OrderAPI: product details
  OrderAPI-->>ChatAPI: order_id + totals
  ChatAPI-->>UI: confirmation message
 
7.3 Chat → Order Tracking Flow
Scenario: “Where is my order ORD-2025-00021?”
•	Chat API: POST /api/chat/message
•	Tool call: GET /api/orders/{order_id}/status
•	Chat composes user-friendly response with ETA.
 
7.4 Daily Ingestion Pipeline Flow (Airflow)
Scenario: Scheduled daily run.
•	Airflow runs tasks in order:
1.	Scrape
2.	Clean
3.	Embed
4.	Upsert
5.	Smoke test
flowchart LR
  A[Airflow Scheduler] --> B[Scrape: Selenium]
  B --> C[Clean: Normalize + Impute]
  C --> D[Embed: NVIDIA]
  D --> E[Upsert: Pinecone]
  E --> F[Smoke test: /api/chat/message]
 
8. Data Contracts (Complete)
8.1 Product Record (Clean)
{
  "product_id": "AMZ123",
  "category": "shirts",
  "brand": "XYZ",
  "name": "Formal Shirt",
  "rating": 4.2,
  "rating_count": 1823,
  "selling_price": 1399,
  "mrp": 2499,
  "discount_pct": 44,
  "source_url": "...",
  "scraped_at": "2025-12-18T00:00:00Z"
}
8.2 Vector Document
{
  "doc_id": "AMZ123",
  "text": "XYZ Formal Shirt category shirts price 1399 rating 4.2 discount 44% ...",
  "metadata": {
    "category": "shirts",
    "brand": "XYZ",
    "selling_price": 1399,
    "rating": 4.2,
    "scraped_at": "2025-12-18"
  }
}
8.3 Chat Session State (LangGraph)
{
  "session_id": "uuid",
  "chat_history": [
    {"role": "user", "text": "Suggest a shirt under 1500"},
    {"role": "assistant", "text": "Here are options..."}
  ],
  "last_intent": "recommendation",
  "last_retrieved": [
    {"product_id": "AMZ123", "score": 0.86}
  ],
  "user_prefs": {"budget": 1500, "category": "shirts"}
}
8.4 Order Record
{
  "order_id": "ORD-2025-00021",
  "customer": {"name": "User", "phone": "+91..."},
  "items": [{"product_id": "AMZ123", "qty": 1, "unit_price": 1399}],
  "status": "CONFIRMED",
  "pricing": {"subtotal": 1399, "shipping": 0, "tax": 0, "total": 1399},
  "timeline": [{"status": "CONFIRMED", "at": "..."}]
}
 
9. RAG/LLM Design Details
9.1 Retrieval Strategy
•	Default top_k = 5–10
•	Apply structured filters where possible: category, max_price, min_rating
•	Add guardrail: if max similarity score < threshold → ask clarifying question
9.2 Prompting Contract (Recommended)
•	System: role, tone, grounding rule (“use only retrieved products/policies”)
•	Developer: formatting rules (bullets, show price/rating, ask follow-up)
•	User: query and constraints
•	Context: retrieved docs with metadata
9.3 Output Format Contract
•	Always return:
o	reply
o	citations (product IDs)
o	intent
o	tool_trace (optional)
 
10. Resilience, Error Handling, and Fallbacks
•	Pinecone failure → respond with “search temporarily unavailable” and offer filter-based browsing (GET /api/products)
•	Groq failure → respond with apology + retry suggestion; log upstream error
•	Embedding failure → skip semantic retrieval and use keyword filters where possible
•	Ingestion failures → keep serving last good index
 
11. Security & Compliance
•	Secrets only via environment variables
•	Rate limiting:
o	/api/chat/message (strict)
o	/internal/* (very strict)
•	Prompt injection mitigations:
o	Ignore user attempts to override system
o	Only answer from retrieved docs
 
12. Observability (What to Measure)
12.1 Metrics
•	Chat latency: p50, p95
•	Retrieval hit rate: % queries with results above threshold
•	Token usage / cost (if available)
•	Error rate per dependency
12.2 Logging (Structured)
•	request_id, session_id
•	endpoint, status_code
•	intent, top_k, namespace
•	upstream latencies
 
13. Versioning (Reproducibility)
•	prompt_version
•	embedding_model_version
•	llm_model_version
•	vector_index_version / namespace
•	ingestion_run_id
 
14. Deployment & Environment Configuration
14.1 Environments
•	Local dev: Flask + local dataset
•	Pipeline: Docker Compose Airflow
•	Production: Render/VM + managed Pinecone
14.2 Required Environment Variables
•	GROQ_API_KEY
•	NVIDIA_API_KEY
•	PINECONE_API_KEY
•	Optional:
o	PINECONE_INDEX_NAME
o	PINECONE_NAMESPACE
o	CHAT_RATE_LIMIT_RPS
o	SESSION_STORE_URL (Redis)
 
15. What Else Can Be Included (to capture entire technical detail)
If you want the doc to be “complete enterprise-grade,” add these appendices:
Appendix A — OpenAPI Specification
•	Provide a Swagger/OpenAPI YAML for all endpoints.
Appendix B — State Machines
•	Order status transitions and allowed operations.
Appendix C — Threat Model
•	Abuse cases, prompt injection cases, rate-limit policies.
Appendix D — Data Quality Rules
•	Cleaning rules, missing-value strategy, schema constraints.
Appendix E — Pipeline SLAs
•	Expected run time, retries, alerting thresholds.
Appendix F — Test Plan
•	Unit tests per module
•	Contract tests per API
•	Smoke test queries for chatbot after each ingestion
 
End of Document
