# AI & Automation Architecture (n8n + RAG)

This document outlines how customer inquiries are processed, qualified, and routed in real time using n8n workflows, Supabase Vector (`pgvector`), and Large Language Models.

---

## 1. Real-Time Lead Capture & AI Reply Flow

When a buyer sends a message via WhatsApp or the web widget, the automation pipeline handles the entire initial conversation without human delay.

```mermaid
sequenceDiagram
    actor Buyer
    participant Webhook as n8n Webhook
    participant DB as Supabase (PostgreSQL)
    participant Vector as Supabase pgvector
    participant AI as AI Agent (LLM)

    Buyer->>Webhook: Sends inquiry ("Looking for a 3-bed apartment")
    Webhook->>DB: Check existing lead & agency settings
    Webhook->>Vector: Semantic search for matching properties
    Vector-->>Webhook: Return relevant listings & brochure links
    Webhook->>AI: Pass chat history + property context
    AI-->>Webhook: Generate natural reply + Lead Score (Hot/Warm/Cold)
    Webhook->>DB: Save message, update score & assign agent
    Webhook-->>Buyer: Send instant reply + property brochure
```

### Key Workflow Steps
* **Agency Validation:** Before triggering AI responses, the workflow checks `can_use_ai` and `subscription_status` in the `organizations` table to ensure active plan permissions.
* **Context-Aware Answers (RAG):** Instead of guessing property prices or availability, the AI queries embeddings stored in `pgvector` so every answer is grounded in the agency's actual listings.
* **Human Handoff Control:** Each lead has an `is_ai_active` flag. If a human agent jumps into the chat from the Next.js dashboard, the AI automatically pauses for that conversation.

---

## 2. Automated Follow-Up Pipeline (Stale Lead Recovery)

Leads often go quiet after an initial inquiry. A scheduled background workflow ensures no potential buyer slips through the cracks.

```mermaid
flowchart TD
    Cron([Scheduled Trigger: Every 6 Hours]) --> Fetch[Query Stale Leads in Supabase]
    Fetch --> Filter{is_ai_active = true & follow_up_count < 3?}
    Filter -->|No| Skip[Skip Lead]
    Filter -->|Yes| Generate[AI Crafts Personalized Follow-Up]
    Generate --> Send[Send Message via WhatsApp / Email]
    Send --> Update[Increment follow_up_count & update last_contacted_at]
```

### Safeguards Built Into the Workflow
* **Spam Prevention:** The workflow tracks `follow_up_count` and `last_contacted_at` directly in the database, stopping automated nudges after a set limit or once a deal is marked `closed` or `lost`.
* **Smart Agent Routing:** When a new lead is scored as **Hot**, the workflow checks `auto_assign_leads` and assigns the lead only to team members where `is_active = true` (skipping agents on leave).
