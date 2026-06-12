# FleetIQ — Sample Project Design Doc
**Purpose:** Technical interview prep for PrePass AI Full Stack Developer role  
**Demonstrates:** CQRS + MediatR, AI-as-query, async LLM patterns, NServiceBus-style messaging, React frontend, golden path thinking

---

## Project Summary

**FleetIQ** is a lightweight fleet management assistant that lets users ask natural language questions about their fleet data and receive AI-generated insights. It is intentionally scoped to be buildable in a few days while demonstrating every architectural concept the PrePass interview will probe.

The core premise: a small fleet operator can type "Which of my vehicles had the most bypass events last month?" or "Flag any vehicles overdue for inspection" and get a grounded, data-backed answer — not a hallucination.

---

## Architecture Overview

```
React Frontend (TypeScript)
        │
        ▼
.NET 8 Web API
  ├── MediatR Command Bus  ─────► Domain Handlers → SQLite (write side)
  │                                     │
  │                                  Domain Events
  │                                     │
  └── MediatR Query Bus   ─────► Query Handlers → Read Models (read side)
                                        │
                                   AI Query Handler
                                        │
                                   ┌────▼────────────────┐
                                   │  AI Gateway Service  │
                                   │  - Prompt builder    │
                                   │  - OpenAI call       │
                                   │  - Response cache    │
                                   └─────────────────────┘
                                        │
                               Background Job Queue
                               (async LLM processing)
```

---

## Domain Model

### Entities (Write Side)
- **Fleet** — a named collection of vehicles owned by an operator
- **Vehicle** — a registered commercial vehicle (VIN, license, weight class)
- **BypassEvent** — a recorded weigh station bypass (timestamp, station, outcome: pass/flag)
- **InspectionRecord** — scheduled/completed DOT inspection entries

### Read Models (Query Side, pre-computed)
- `FleetSummaryReadModel` — vehicle count, bypass totals, last activity
- `VehicleDetailReadModel` — full vehicle history, bypass rate, inspection status
- `AIQuestionResultReadModel` — stored LLM responses (prompt hash → result)

---

## Command Side

All state mutations go through commands. The LLM never writes directly.

| Command | Handler Action | Domain Event Emitted |
|---|---|---|
| `RegisterFleetCommand` | Creates fleet record | `FleetRegisteredEvent` |
| `AddVehicleCommand` | Adds vehicle to fleet | `VehicleAddedEvent` |
| `RecordBypassEventCommand` | Logs a bypass event | `BypassEventRecordedEvent` |
| `SubmitAIQuestionCommand` | Persists pending question | `AIQuestionSubmittedEvent` |
| `CompleteAIQuestionCommand` | Stores LLM result | `AIQuestionCompletedEvent` |

### Pipeline Behaviors (MediatR)
- **ValidationBehavior** — FluentValidation runs before any handler
- **LoggingBehavior** — logs command type, handler duration, success/failure
- **CachingBehavior** — for queries only: checks Redis/memory cache before hitting DB

---

## Query Side

Queries are read-only. They never mutate state. AI queries are a subset.

| Query | Returns | Cacheable? |
|---|---|---|
| `GetFleetSummaryQuery` | `FleetSummaryReadModel` | Yes — 5 min TTL |
| `GetVehicleDetailQuery` | `VehicleDetailReadModel` | Yes — 5 min TTL |
| `GetBypassHistoryQuery` | `IList<BypassEvent>` | Yes — 5 min TTL |
| `AskFleetQuestionQuery` | `AIQuestionResultReadModel` | Yes — keyed on prompt hash |
| `GetPendingAIQuestionsQuery` | Queue of unprocessed questions | No |

---

## AI Architecture

### Rule: LLMs are queries. Their output flows through a command.

```
User types question
       │
       ▼
AskFleetQuestionQuery handler
       │
       ├── Hash(prompt + fleetId + date) → check cache
       ├── Cache hit? → return immediately
       └── Cache miss?
               │
               ▼
       SubmitAIQuestionCommand
               │
               ▼
       AIQuestionSubmittedEvent → persisted to event store
               │
               ▼
       Background worker picks up event
               │
               ├── Fetch relevant fleet data (read models)
               ├── Build grounded prompt (data injected as context)
               ├── Call OpenAI API
               └── CompleteAIQuestionCommand
                           │
                           ▼
                   AIQuestionCompletedEvent
                           │
                           ▼
                   Read model updated
                           │
                           ▼
                   Frontend polls GetAIQuestionStatusQuery
```

### Why async?
- LLM calls can take 3–8 seconds. Blocking the HTTP request is poor UX and wastes thread pool.
- The event store gives a complete audit log of every question asked, what data was injected, which model responded, and what the response was.
- Background workers can be scaled independently from the web API.
- This pattern maps directly to NServiceBus sagas — the `AIQuestionSubmittedEvent` and `AIQuestionCompletedEvent` are exactly how a saga would track a long-running async workflow.

### Grounding (preventing hallucination)
The LLM is never asked to "know" things. The prompt builder injects real data:

```
System: You are a fleet analyst assistant. Answer only using the data provided below.
        If the answer is not in the data, say "I don't have enough data to answer that."

Data:
Fleet: Acme Logistics | Vehicles: 12 | Active: 10
Vehicle VIN-001 | Bypasses: 47 last 30 days | Last bypass: 2026-06-09
Vehicle VIN-002 | Bypasses: 12 last 30 days | Last bypass: 2026-06-01
...

Question: Which vehicle had the most bypass events last month?
```

---

## NServiceBus Mapping

The in-process MediatR events in this project map 1:1 to NServiceBus across service boundaries:

| FleetIQ (MediatR) | NServiceBus equivalent |
|---|---|
| `INotificationHandler<BypassEventRecordedEvent>` | Event subscriber on a queue |
| `AIQuestionSubmittedEvent` → background worker | Saga step 1 |
| `AIQuestionCompletedEvent` → read model update | Saga step 2 / saga completion |
| Background job queue | NServiceBus transport (Azure Service Bus, RabbitMQ) |
| `SubmitAIQuestionCommand` → `CompleteAIQuestionCommand` | Saga tracking state across messages |

**Interview talking point:** *"In FleetIQ I modeled async AI calls as a two-event workflow — question submitted, question completed. That's a saga. MediatR handles it in-process; NServiceBus handles the same pattern across service boundaries over a queue."*

---

## React Frontend

Keep it minimal — the goal is to show you can wire a React app to a CQRS backend cleanly.

### Pages
1. **Fleet Dashboard** — vehicle list, bypass summary stats, last activity
2. **Vehicle Detail** — bypass history chart, inspection status
3. **FleetIQ Chat** — natural language question input, async result display

### CQRS frontend patterns to demonstrate
- **Commands via POST** — `POST /api/vehicles` → maps to `AddVehicleCommand`
- **Queries via GET** — `GET /api/fleet/summary` → maps to `GetFleetSummaryQuery`
- **Async polling** — submit AI question → poll `GET /api/questions/{id}/status` → display when complete
- **Optimistic UI** — record bypass event optimistically, revert on failure

---

## Project Structure

```
FleetIQ/
├── FleetIQ.Api/                    # .NET 8 Web API
│   ├── Controllers/
│   │   ├── FleetsController.cs
│   │   ├── VehiclesController.cs
│   │   └── QuestionsController.cs
│   └── Program.cs
│
├── FleetIQ.Application/            # MediatR handlers — vertical slices
│   ├── Fleet/
│   │   ├── RegisterFleet.cs        # Command + Handler in one file
│   │   └── GetFleetSummary.cs      # Query + Handler in one file
│   ├── Vehicles/
│   │   ├── AddVehicle.cs
│   │   └── GetVehicleDetail.cs
│   ├── Bypass/
│   │   └── RecordBypassEvent.cs
│   ├── AI/
│   │   ├── SubmitAIQuestion.cs     # Command
│   │   ├── CompleteAIQuestion.cs   # Command (called by background worker)
│   │   └── AskFleetQuestion.cs     # Query (checks cache, submits if miss)
│   └── Behaviors/
│       ├── ValidationBehavior.cs
│       ├── LoggingBehavior.cs
│       └── CachingBehavior.cs
│
├── FleetIQ.Domain/                 # Entities, domain events, no dependencies
│   ├── Fleet.cs
│   ├── Vehicle.cs
│   ├── BypassEvent.cs
│   └── Events/
│       ├── FleetRegisteredEvent.cs
│       ├── BypassEventRecordedEvent.cs
│       ├── AIQuestionSubmittedEvent.cs
│       └── AIQuestionCompletedEvent.cs
│
├── FleetIQ.Infrastructure/         # DB, AI gateway, cache, background worker
│   ├── Persistence/
│   │   └── FleetIQDbContext.cs     # EF Core + SQLite
│   ├── AI/
│   │   ├── AIGateway.cs            # OpenAI wrapper
│   │   └── PromptBuilder.cs        # Data injection logic
│   ├── Cache/
│   │   └── QueryCache.cs
│   └── Workers/
│       └── AIQuestionWorker.cs     # BackgroundService polling event store
│
└── fleetiq-ui/                     # React + TypeScript
    ├── src/
    │   ├── pages/
    │   │   ├── Dashboard.tsx
    │   │   ├── VehicleDetail.tsx
    │   │   └── FleetChat.tsx
    │   ├── api/
    │   │   ├── commands.ts         # POST wrappers
    │   │   └── queries.ts          # GET wrappers
    │   └── hooks/
    │       └── useAIQuestion.ts    # Polling hook for async AI results
    └── package.json
```

---

## Build Order (Interview Prep Sequence)

Build in this order to always have something demonstrable:

1. **Domain + Application layer skeleton** — entities, MediatR wired, no DB yet (in-memory)
2. **Fleet + Vehicle CRUD** — commands and queries working end to end via API
3. **Bypass event recording** — domain event emitted, notification handler logs it
4. **Read model projections** — `FleetSummaryReadModel` built from events
5. **AI synchronous path** — `AskFleetQuestionQuery` calls OpenAI directly, returns result
6. **AI async path** — refactor to submit/complete pattern with background worker
7. **Cache layer** — prompt hash caching on AI queries
8. **React frontend** — dashboard → vehicle detail → chat UI
9. **Golden path story** — document it as a template others could clone for any domain

---

## Interview Talking Points This Project Enables

**For Ryan (Solutions Architect):**
- "I separated the read and write models from the start. The fleet summary is a pre-computed projection — the UI just fetches it, no joins at query time."
- "Pipeline behaviors handle cross-cutting concerns — validation, logging, caching — without polluting handlers."
- "Domain events decouple side effects. Recording a bypass event just emits `BypassEventRecordedEvent`. What reacts to that is the infrastructure's concern, not the domain's."

**For Wyatt (AI Enablement):**
- "LLM calls live on the query side. Same input = cacheable result. No state mutation from the LLM itself."
- "The async path — submit event, background worker, complete event — is a saga. In NServiceBus that's a first-class pattern. I built the same thing with MediatR in-process."
- "The prompt builder injects real data from read models. The LLM is told explicitly: answer only from this data. That's grounding, not hallucination prevention theater."

**For Thomas (Hiring Manager):**
- "I scoped this to be shippable, not perfect. The domain is simple so the architecture is the point."
- "The folder structure is intentionally a golden path — any developer could clone this and have messaging, validation, AI, and CI pre-wired."
- "I built this to learn the patterns I'd be working with at PrePass, not just to prep for the interview."

---

## Target Job Description — PrePass AI Full Stack Developer

**Platform Engineering Team · Fully Remote**

PrePass® is North America's most utilized and technologically advanced weigh station bypass and toll payment platform. Our technologies enable safe, qualified motor carriers to bypass inspection facilities at highway speeds, saving time, fuel, and money while reducing emissions. More than 105,000 fleets subscribe over 750,000 commercial vehicles to PrePass services.

### Position Summary

We are adding to our Web Platform Engineering team — a small, nimble group supporting a larger engineering organization, whose mission is developer enablement for the web tier: building the shared UI foundations, standardized frontend scaffolds (golden path templates), and the composable building blocks that product teams assemble into customer-facing experiences. This team operates with an open-source maintainer mindset: other teams can contribute, but Web Platform Engineering drives quality, direction, and standards.

The platform is built on a modern, service-oriented architecture with clear separation between customer-facing experiences, product feature services, and core business capabilities. On the web side, that means React static site frontends backed by a unified design system, a shared component library, and well-defined patterns for consuming .NET CQRS backends, where physical separation of write and read paths shapes how the UI fetches, caches, and renders data. Integration with the broader system flows through messaging-first patterns via NServiceBus and domain-driven service boundaries, and your job is to make those boundaries feel seamless from the browser.

This is a high-autonomy, high-impact role. You will help shape the web foundation other engineers build on, the unified web/platform layer that becomes the default way teams ship customer-facing experiences. You will deliver quickly, iterate openly, and use AI-powered development tools as a core part of how you work.

### Key Responsibilities

**Platform Engineering & Shared Services**
- Build and maintain shared components, internal libraries, and platform services that the broader engineering org depends on.
- Maintain and evolve standardized service scaffolds (golden path templates) — ensuring new services start with messaging, CI/CD, and observability pre-wired.
- Operate with an open-source maintainer mindset: clear contribution standards, good documentation, internal consumers treated as first-class users.
- Help define and uphold platform standards: messaging-first integration, service data ownership, and proper layering across the architecture.

**Full Stack Development**
- Build React static site frontends — fast, decoupled, and independently deployable.
- Build and extend .NET CQRS services following established service patterns and architectural standards.
- Design and implement NServiceBus message contracts, handlers, and sagas for async business workflows.
- Build and maintain read model projectors and query services — keeping the read path fast and purpose-built.
- Contribute to CI/CD pipelines, observability tooling, and DevOps practices across the platform.

**AI-Accelerated Development**
- Use AI development tools — coding agents, agentic pipelines, AI-assisted scaffolding — as the default mode of working, not a novelty.
- Drive adoption of AI-powered development workflows across the team; raise the floor for how quickly good software gets shipped.
- Stay current with AI development tooling and bring new approaches to the team proactively.

**Collaboration & Communication**
- Partner with product managers, architects, and feature teams to align on platform direction and contribution patterns.
- Communicate architecture decisions clearly — to engineers building on the platform and to non-technical stakeholders.
- Document service contracts, architectural decisions, and contribution guides to enable the broader engineering community.

### What We're Looking For

**Mindset (Most Important)**
- Startup energy in a professional context: you default to action, ship working software early, and refine from there.
- Genuinely agile — you adapt process to context rather than following a framework for its own sake.
- More interested in solving the problem than mastering any specific technology.
- Energized by AI tools as a force multiplier — you're already using them seriously and always looking for what's next.

**Technical Skills**
- JavaScript/TypeScript proficiency is required. React experience is a strong advantage.
- .NET / C# experience is a strong advantage. Object-oriented design fundamentals are required.
- Understanding of or genuine interest in CQRS, event-driven architecture, and async messaging patterns.
- Experience with RESTful API design and consumption.
- Familiarity with CI/CD, cloud platforms (Azure preferred), and modern DevOps practices.
- Hands-on experience using AI tools in a real development workflow — coding assistants, agentic pipelines, or prompt engineering for code generation.

**Experience**
- Mid-career to senior level: enough experience to operate independently, make sound architectural judgments, and know when to ask.
- Demonstrated track record of shipping software — not just designing it.
- Bachelor's degree in Computer Science, Software Engineering, or equivalent practical experience.
- Bonus: experience maintaining or contributing to open source projects.
