# Architecture

Derived from `SPEC.md`. Section numbers in brackets point back to the spec. The diagrams are Mermaid and render on GitHub.

## 1. System overview

```mermaid
flowchart LR
  subgraph Clients
    U[Public user<br/>search + chat UI<br/>Next.js, SSE]
    A[Admin<br/>ingest, discover, edit, takedown]
  end

  subgraph API["FastAPI (request path) [3, 6.1, 7, 8]"]
    S[Search<br/>POST /v1/search<br/>query understanding, retrieve, evaluate, rerank]
    C[Chat<br/>single-business + fan-out router<br/>capability registry, grounding check]
    AD[Admin API<br/>ingest, discover, refresh, PATCH, takedown]
  end

  subgraph TEMPORAL["Temporal workers [6.5]"]
    W1[ingest / discover<br/>refresh / reextract]
    W2[migrate / takedown<br/>lead delivery]
    BQ[Browser task queue<br/>Playwright crawl]
  end

  subgraph STORES[Data stores [9]]
    PG[(PostgreSQL 16<br/>businesses JSONB, pgvector, pg_trgm<br/>evidence, audit, sessions, leads)]
    TPG[(Temporal Postgres<br/>separate DB)]
    R[(Redis<br/>caches, rate limits)]
    OBJ[(S3 / MinIO<br/>page snapshots, PDFs<br/>pre-migration backups)]
  end

  subgraph EXT[External services]
    SITE[Business websites]
    PL[Places API]
    OD[Open data / OSM]
    GEO[Geocoder]
    SRCH[Search API<br/>Exa default, Tavily]
    LLM[LLMs<br/>Claude for extraction + chat<br/>small model for routing]
  end

  U --> S
  U --> C
  A --> AD
  AD -->|start workflows| W1
  AD --> PG
  S --> PG
  S --> R
  S --> LLM
  S --> GEO
  C --> PG
  C --> R
  C --> LLM
  W1 --> BQ
  BQ --> SITE
  W1 --> PL
  W1 --> OD
  W1 --> GEO
  W1 --> SRCH
  W1 --> LLM
  W1 --> PG
  W1 --> OBJ
  W2 --> PG
  W2 --> OBJ
  TEMPORAL --- TPG
```

## 2. Ingest pipeline and fallback ladder [5, 5.1, 6]

```mermaid
flowchart TD
  IN[POST /v1/ingest url<br/>or discovery candidate] --> CHK{First activity:<br/>suppression list<br/>+ robots.txt}
  CHK -->|host / place_id / phone_norm listed| SUP1[[suppressed<br/>no changes]]
  CHK -->|site-wide disallow| RES
  CHK -->|clear| CR[crawling<br/>llms.txt, sitemap, allowed paths<br/>httpx then Playwright]
  CR -->|noindex / nofollow on first page| RES
  CR --> CL[classifying<br/>pick profile.type]
  CL --> EX[extracting<br/>per-page extraction, merge, normalize]
  EX --> EN

  subgraph EN["enriching: fallback ladder, wanted fields only"]
    direction TB
    L1[1 Direct: own permitted pages] --> L2[2 Structured API<br/>places_api, open_data<br/>license-gated]
    L2 --> L3[3 Search, site-scoped<br/>only after a technical block<br/>cache-only, no live fetch]
    L3 --> L4[4 Search, broad<br/>press, news, listings<br/>entity-verified]
  end

  EN --> RES[resolving<br/>identity keys, allocate business_id<br/>merge into existing record]
  RES --> VAL[validating<br/>schema, provenance, restricted paths<br/>publication decision]
  VAL -->|opt-out recorded| SUP2[[suppressed<br/>stored, not indexed]]
  VAL --> IDX[[indexed<br/>embeddings + facets + keyword]]
  VAL -->|schema failure after retry| FAIL[[failed]]
```

## 3. Business JSON: three layers [4]

```mermaid
flowchart LR
  subgraph DOC[Business JSON, single source of truth]
    CORE[Layer 1: Core<br/>identity, location, contact, operations<br/>policies, guardrails, system]
    PROF[Layer 2: Profile<br/>vertical schema + manifest<br/>facets, freshness, catalog_paths]
    KNOW[Layer 3: Knowledge<br/>facts, faqs, documents, social_proof]
  end
  SCH[/schemas + manifests/] --> DOC
  DOC --> IDXS[Search index<br/>embeddings, facets, tsvector]
  DOC --> CARD[Result cards]
  DOC --> AGENT[Chat agent prompt + tools]
  PROV[field_provenance<br/>source, method, confidence, origin] --- DOC
```

## 4. Search, active set, and chat [7, 8]

```mermaid
sequenceDiagram
  participant U as User
  participant S as Search API
  participant IX as Postgres index
  participant R as Router (small model)
  participant AG as Business agents
  participant M as Match state

  U->>S: natural-language query
  S->>S: parse into requirements (hard / soft)
  S->>IX: vector + keyword retrieval, published only
  S->>S: evaluate hard requirements in code
  S-->>U: ranked results, top 5 become active set
  loop each turn
    U->>R: message
    alt refinement
      R->>S: restrict to active set, add requirements
    else question or comparison
      R->>AG: fan out, one isolated agent per business
      AG-->>R: answers from each agent's own JSON
      R-->>U: attributed answers or table
    else single agent
      U->>AG: one-to-one chat (leads allowed here)
    end
    AG->>M: extracted facts update match / partial / non_match / unknown
    M-->>U: narrower, explained results
  end
  U->>S: add / remove agents (explicit only)
```

## 5. Compliance guardrails [2, 5.1, 11]

```mermaid
flowchart LR
  OPT[Opt-out signals<br/>site-wide disallow, noindex, explicit opt-out<br/>suppression list] -->|skip crawl and ladder| SUPP[publication_status = suppressed]
  BLOCK[Technical block<br/>403, 503, challenge, timeout] -->|not an opt-out| LADDER[Ladder steps 3 and 4<br/>provider output only]
  RESTR[Restricted paths<br/>prices, offers, guarantees, policies] -->|reject third-party and search evidence| MERGE[Merge]
  LIC[Adapter license block<br/>storable_fields, cache_ttl] -->|write guard| EVID[(source_evidence)]
  TK[Takedown workflow] -->|delete record, index, snapshots, chats, leads<br/>keep keys only| SUPL[(suppression_list)]
```
