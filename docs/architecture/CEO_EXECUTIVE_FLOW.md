# Cache Purge & Warm-Up — Executive Architecture Overview

**For:** Leadership Approval  
**Version:** 2.0 — Simplified Real-Time Architecture  
**Date:** 2026-05-23  
**Status:** Pending Approval  
**Approach:** Real-Time Webhook | 1 Attempt | Manual Fallback | ~10s Freshness

---

## The Problem

When a product's price, name, or image changes in Adobe Commerce, the website continues showing **stale/outdated content** to customers because cached pages aren't automatically refreshed.

**Impact:** Customers see wrong prices, old product names, outdated images → revenue risk, trust erosion.

**Why it's hard:** Product pages are dynamically generated — they don't exist as traditional pages in AEM, so standard "publish" buttons don't work.

---

## The Solution — One Slide View

```mermaid
graph LR
    A[Product Updated<br/>in Commerce] -->|"instant event"| B[Adobe I/O Events<br/>Real-Time Webhook]
    B -->|"push ~2s"| C[App Builder<br/>1 Action, 1 Attempt]
    C -->|"authenticated call"| D[AEM Publish<br/>Purge Broker]
    D -->|"Sling Distribution"| E[ALL Cache Pods<br/>Invalidated ✓]

    style A fill:#ff6b35,color:#fff
    style B fill:#1473e6,color:#fff
    style C fill:#7c3aed,color:#fff
    style D fill:#059669,color:#fff
    style E fill:#059669,color:#fff
```

**+ Manual Fallback:** AEM UI Extension for admin-triggered purge when needed.

---

## How It Works — Executive Flow

```mermaid
flowchart TB
    subgraph TRIGGER ["① TRIGGER — Commerce Side"]
        direction LR
        T1[Admin changes price] --> T2[API updates stock]
        T2 --> T3[Bulk import runs]
    end

    subgraph EVENT ["② EVENT — Real-Time Push"]
        direction LR
        E1[Change detected] --> E2[Event emitted<br/>Adobe I/O Events]
        E2 --> E3[Webhook pushed<br/>instantly to App Builder]
    end

    subgraph PROCESS ["③ PROCESS — App Builder (1 Attempt)"]
        direction LR
        O1[Receive webhook] --> O2[Expand SKU<br/>to URLs per store]
        O2 --> O3[Call AEM Publish<br/>single attempt]
    end

    subgraph PURGE ["④ PURGE — All Pods Instantly"]
        direction LR
        P1[Validate &<br/>authenticate] --> P2[Sling Distribution<br/>INVALIDATE]
        P2 --> P3[Pod 1 ✓]
        P2 --> P4[Pod 2 ✓]
        P2 --> P5[Pod N ✓]
    end

    subgraph FALLBACK ["⑤ FALLBACK — If Anything Fails"]
        direction LR
        F1[Admin opens<br/>AEM UI Extension] --> F2[Enters URLs<br/>or SKU]
        F2 --> F3[Manual purge<br/>all pods ✓]
    end

    TRIGGER --> EVENT
    EVENT --> PROCESS
    PROCESS --> PURGE
    PROCESS -.->|"on failure"| FALLBACK
```

---

## End-to-End Timeline

```mermaid
flowchart LR
    subgraph Commerce ["Commerce ~2s"]
        A[Product Saved]
    end
    subgraph IO ["Adobe I/O ~3s"]
        B[Webhook Push]
    end
    subgraph AppBuilder ["App Builder ~4s"]
        C[Expand URLs + Call AEM]
    end
    subgraph AEM ["AEM Publish ~3s"]
        D[Sling Distribution Invalidate]
    end
    subgraph Result ["Done ~10s total"]
        E[All Dispatchers Flushed]
    end

    A --> B --> C --> D --> E
```

**Total time: ~10 seconds** from product save to fresh content available.

No polling delays. No batching waits. Real-time webhook push.

---

## Business Value

| Benefit | Impact |
|---------|--------|
| **Correct pricing displayed** | Eliminates revenue leakage from stale prices |
| **Automated, no manual effort** | Operations team freed from manual cache clearing |
| **Multi-store support** | All markets/languages updated simultaneously |
| **Resilient** | Adobe platform retries webhook delivery; manual fallback for edge cases |
| **Observable** | Dashboard shows exactly what was purged, when, and status |
| **Secure** | Adobe IMS authentication end-to-end, no exposed endpoints |
| **Scalable** | Handles bulk imports (1000s of products) without degradation |
| **Minimal complexity** | 3 components total. No databases, no queues, no retry logic |
| **Phased rollout** | Phase 1 low-risk, Phase 2 adds CDN caching for performance |

---

## Architecture — Full View

```mermaid
graph TB
    subgraph COMMERCE ["Adobe Commerce PaaS"]
        CM_Save[Product Save/Import/API]
        CM_Observer[Change Detection Module]
        CM_Admin[Admin Config Panel]
    end

    subgraph ADOBE_IO ["Adobe I/O Platform"]
        IO_Events[Adobe I/O Events<br/>Real-Time Webhook Delivery]
    end

    subgraph APP_BUILDER ["Adobe App Builder — Single Action"]
        AB_Action[Purge Action<br/>1 attempt, no state, no DB]
    end

    subgraph AEM_PUBLISH ["AEMaaCS Publish Tier — Multi-Pod"]
        AEM_Servlet[Purge Broker Servlet<br/>IMS-authenticated]
        AEM_SCD[Sling Content Distribution<br/>INVALIDATE]
        AEM_P1["Pod 1 Dispatcher ✓"]
        AEM_P2["Pod 2 Dispatcher ✓"]
        AEM_PN["Pod N Dispatcher ✓"]
    end

    subgraph AEM_EXT ["AEM UI Extension — Manual Fallback"]
        UIExt[Purge URLs / SKUs<br/>Dry-run + Execute]
    end

    subgraph CDN_LAYER ["CDN Layer"]
        CDN[Adobe CDN<br/>Phase 1: pass-through for PDPs]
    end

    subgraph USERS ["End Users"]
        Browser[Customer Browser]
    end

    %% Automated Flow
    CM_Save --> CM_Observer
    CM_Observer -->|"async event"| IO_Events
    IO_Events -->|"webhook push (~2s)"| AB_Action
    AB_Action -->|"HTTPS + IMS Token (1 attempt)"| AEM_Servlet
    AEM_Servlet --> AEM_SCD
    AEM_SCD --> AEM_P1
    AEM_SCD --> AEM_P2
    AEM_SCD --> AEM_PN

    %% Manual Fallback
    UIExt -->|"direct purge"| AEM_Servlet

    %% User Flow
    Browser --> CDN
    CDN --> AEM_P1
    CDN --> AEM_P2

    %% Phase 2
    AB_Action -.->|"Phase 2: CDN Purge"| CDN

    CM_Admin -.->|configure| CM_Observer

    %% Styling
    style COMMERCE fill:#ff6b3520,stroke:#ff6b35
    style ADOBE_IO fill:#1473e620,stroke:#1473e6
    style APP_BUILDER fill:#7c3aed20,stroke:#7c3aed
    style AEM_PUBLISH fill:#05966920,stroke:#059669
    style AEM_EXT fill:#f59e0b20,stroke:#f59e0b
    style CDN_LAYER fill:#0891b220,stroke:#0891b2
```

---

## Phase Roadmap

```mermaid
timeline
    title Implementation Phases
    section Phase 1 — Real-Time Dispatcher Purge
        Commerce event module       : Observer + Adobe I/O Events
        App Builder action          : 1 webhook action (simple)
        AEM Publish broker          : Sling Distribution INVALIDATE
        Dispatcher config           : PDP cache rules + CDN pass-through
        AEM UI Extension            : Manual purge fallback
    section Phase 2 — Add CDN Purge
        CDN Purge API call          : Add to App Builder action
        Surrogate key headers       : Add to AEM Publish responses
        Re-enable CDN caching       : PDPs cached at CDN with purge
        UI Extension CDN toggle     : CDN purge option in manual UI
```

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| AEM Publish temporarily unavailable | Admin uses AEM UI Extension to manually purge once recovered |
| Bulk import (1000+ products) | App Builder processes concurrently (queues beyond limit); all process within ~60s |
| Stale content | CDN disabled for PDPs in Phase 1 — Dispatcher purge = instant freshness |
| Security breach via purge endpoint | IMS authentication mandatory, URL pattern allowlist, audit logging |
| Webhook delivery fails | Adobe platform retries automatically; if still fails, admin purges manually |

---

## Cost & Investment

| Component | Adobe License | Custom Build |
|-----------|:---:|:---:|
| Adobe Commerce PaaS | Existing ✓ | Module development |
| Adobe I/O Events | Included ✓ | Event configuration |
| Adobe App Builder | Included with license ✓ | Actions + UI development |
| AEMaaCS | Existing ✓ | OSGi bundle development |
| Adobe CDN | Existing ✓ | Configuration (Phase 2) |

**All infrastructure is included in existing Adobe licenses.** Investment is development effort only.

---

## Decision Required

| # | Decision | Recommendation |
|---|----------|----------------|
| 1 | Approve Phase 1 implementation | ✅ Low risk, high value, uses existing Adobe platform |
| 2 | Approve real-time webhook approach | ✅ ~10s freshness. No polling delays. |
| 3 | Accept 1-attempt + manual fallback | ✅ Simple, reliable. Manual UI Extension for rare failures. |
| 4 | Disable CDN caching for PDPs (Phase 1) | ✅ Ensures Dispatcher purge = instant customer freshness |
| 5 | Plan Phase 2 CDN purge | ✅ Re-enable CDN caching for performance. Builds on Phase 1. |

---

## Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| CTO / VP Engineering | | | |
| Enterprise Architect | | | |
| Product Owner | | | |
| Security Lead | | | |
