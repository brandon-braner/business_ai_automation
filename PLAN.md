# Architecture Plan: Multi-Agent AI Automation Platform

**Generated:** 2026-03-23
**Status:** DRAFT
**Branch:** master
**Language:** Python
**ADK:** google-adk (PyPI)
**Web Framework:** FastAPI

---

## 1. Problem Statement

Build a multi-agent AI automation platform using Google ADK (Python) that:
- Has an orchestrator agent as the central coordinator
- Supports dynamic agent creation and delegation
- Connects to Slack and other messaging platforms
- Provides an extension framework for custom functionality
- Each agent manages its own memories and knowledge

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MESSAGING LAYER                                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │  Slack  │  │  Teams  │  │ Discord │  │  Web    │  │  Email  │        │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘        │
│       │             │             │             │             │             │
│       └─────────────┴─────────────┴─────────────┴─────────────┘             │
│                                   │                                         │
│                           ┌───────▼───────┐                                 │
│                           │  Adapter      │                                 │
│                           │  Interface     │                                 │
│                           └───────┬───────┘                                 │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────┐
│                           PLATFORM CORE                                     │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    ORCHESTRATOR AGENT                                 │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────────┐  │  │
│  │  │  Intent     │  │  Task        │  │  Extension                 │  │  │
│  │  │  Classifier │  │  Router       │  │  Manager                  │  │  │
│  │  └─────────────┘  └──────────────┘  └────────────────────────────┘  │  │
│  │                                                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │                    Memory Manager                            │    │  │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────────────────┐  │    │  │
│  │  │  │ Session   │  │  Long-term │  │  Knowledge Graph     │  │    │  │
│  │  │  │ Memory    │  │  Memory    │  │  (entity relationships)│ │    │  │
│  │  │  └───────────┘  └───────────┘  └───────────────────────┘  │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                    ┌───────────────┼───────────────┐                       │
│                    ▼               ▼               ▼                       │
│            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│            │  Agent A     │ │  Agent B    │ │  Agent N    │               │
│            │  (Specialist)│ │ (Specialist)│ │ (Dynamic)   │               │
│            └─────────────┘ └─────────────┘ └─────────────┘               │
│                 │               │               │                          │
│            ┌────────┐     ┌────────┐     ┌────────┐                      │
│            │Memory  │     │Memory  │     │Memory  │                      │
│            │Manager │     │Manager │     │Manager │                      │
│            └────────┘     └────────┘     └────────┘                      │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      EXTENSION FRAMEWORK                             │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │  Tool       │  │  Trigger    │  │  Memory     │  │  Webhook    │  │  │
│  │  │  Extensions │  │  Extensions │  │  Extensions │  │  Extensions │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      AGENT FACTORY                                    │  │
│  │  Dynamic agent creation, registration, and lifecycle management        │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Core Components

### 3.1 Messaging Adapter Layer

**Purpose:** Normalize incoming messages from diverse platforms into a unified format.

```python
# Unified message format
@dataclass
class UnifiedMessage:
    platform: str              # "slack", "teams", "discord", etc.
    channel_id: str
    user_id: str
    message: str
    thread_id: Optional[str]
    metadata: dict[str, Any]
    session_id: str           # Maps to ADK session
    agent_id: str            # Target agent
    content: str
    context: MessageContext
```

**Platform Adapters (pluggable):**
- `SlackAdapter` — handles Slack Events API, message formatting, threaded replies
- `TeamsAdapter` — Microsoft Teams webhook/connector support
- `DiscordAdapter` — Discord bot API
- `WebAdapter` — FastAPI webhook endpoint
- `EmailAdapter` — SMTP/IMAP integration (future)

### 3.2 Orchestrator Agent

**Purpose:** Central coordinator that classifies intents and delegates to specialist agents.

```python
class OrchestratorAgent(BaseAgent):
    """
    LLM-driven coordinator using ADK's Agent primitives.
    Wraps ADK Agent with additional:
    - Intent classification
    - Task routing
    - Extension management
    - Memory management
    """
    intent_classifier: IntentClassifier
    task_router: TaskRouter
    extension_manager: ExtensionManager
    memory_manager: MemoryManager

    async def process(self, message: UnifiedMessage) -> AgentResponse:
        intent = await self.intent_classifier.classify(message)
        if intent == Intent.DELEGATE_TASK:
            agent = await self.task_router.route(intent, message)
            result = await agent.execute(message)
        elif intent == Intent.CREATE_AGENT:
            agent_def = await self._parse_agent_definition(message)
            agent = await self.agent_factory.create(agent_def)
            result = await agent.execute(message)
        # ... other intents
        return result
```

**Intent Types:**
- `CREATE_AGENT` — Spawn a new specialist agent
- `DELEGATE_TASK` — Route to existing specialist
- `QUERY_KNOWLEDGE` — Query the knowledge graph
- `MANAGE_EXTENSION` — Load/unload an extension
- `UNKNOWN` — Fallback response

### 3.3 Specialist Agents

**Purpose:** Domain-specific agents created dynamically for different tasks.

```python
class SpecialistAgent:
    """Domain-specific agent, each with isolated memory."""
    agent_id: str
    name: str
    description: str
    model: str                        # "gemini-2.0", "claude-3", etc.
    instructions: str                 # System prompt
    tools: list[Tool]
    memory_manager: MemoryManager     # ISOLATED per agent
    extensions: list[str]             # Required extension IDs

    async def execute(self, task: Task) -> TaskResult:
        context = await self.memory_manager.get_context()
        result = await self._run_with_tools(task, context)
        await self.memory_manager.store(result)
        return result
```

**Agent Types:**
- `StaticAgent` — predefined, always available (e.g., "customer_service", "scheduler")
- `DynamicAgent` — created at runtime for specific tasks, can be destroyed

### 3.4 Agent Factory

**Purpose:** Dynamic agent creation and lifecycle management.

```python
@dataclass
class AgentDefinition:
    id: str
    name: str
    description: str
    model: str
    instructions: str
    tools: list[Tool]
    memory_strategy: MemoryStrategy  # "full", "summary", "buffer"
    extensions: list[str]

class AgentFactory:
    """Creates and manages specialist agent lifecycle."""

    def __init__(self, adk_runner: ADKRunner):
        self._agents: dict[str, SpecialistAgent] = {}
        self._adk_runner = adk_runner

    async def create(self, definition: AgentDefinition) -> SpecialistAgent:
        agent = SpecialistAgent(definition)
        self._agents[agent.agent_id] = agent
        await self._adk_runner.register(agent)
        return agent

    async def get(self, agent_id: str) -> Optional[SpecialistAgent]:
        return self._agents.get(agent_id)

    async def destroy(self, agent_id: str) -> None:
        agent = self._agents.pop(agent_id)
        await agent.shutdown()
        await self._adk_runner.unregister(agent_id)
```

### 3.5 Memory System

**Per-Agent Memory Architecture:**

```
MemoryManager (per agent)
├── Short-Term (Session)
│   ├── Event history (last N messages)   — ADK Session
│   └── Working state (variables, flags)  — session.state
│
├── Medium-Term (Buffer)
│   ├── Summarized conversation history
│   └── Key facts extracted
│
└── Long-Term (Persistent)
    ├── Knowledge graph (entity → relationship)  — PostgreSQL
    ├── Embedded memories (vector search)         — pgvector
    └── User preferences

Knowledge Graph Schema
├── Entity
│   ├── id: UUID
│   ├── type: str           # "user", "task", "concept"
│   ├── properties: JSONB
│   └── embedding: vector(1536)
│
└── Relationship
    ├── from_entity: UUID
    ├── to_entity: UUID
    ├── type: str
    └── confidence: float
```

### 3.6 Extension Framework

**Purpose:** Allow custom functionality without modifying core platform.

```python
class Extension(ABC):
    """Base class for all extensions."""

    @abstractmethod
    async def on_load(self, ctx: ExtensionContext) -> None:
        """Called when extension is loaded."""

    @abstractmethod
    async def on_unload(self) -> None:
        """Called when extension is unloaded."""

    @abstractmethod
    def get_tools(self) -> list[Tool]:
        """Return tools this extension provides."""

    @abstractmethod
    def get_triggers(self) -> list[Trigger]:
        """Return event triggers this extension listens to."""

    def get_memory_schemas(self) -> list[MemorySchema]:
        """Return custom memory schemas."""
        return []

@dataclass
class ExtensionContext:
    agent_registry: AgentRegistry
    memory_api: MemoryAPI
    config_store: ConfigStore
    logger: Logger

Extension Types:
├── Tool Extension     → Adds capabilities to agents
├── Trigger Extension  → Reactive (event-driven) execution
├── Memory Extension   → Custom memory backends
└── Webhook Extension  → Outbound integrations
```

**Loading:** Extensions are discovered via entry points (`pyproject.toml`) or a configured directory. Each extension is a Python module loaded dynamically using `importlib`.

---

## 4. Data Flow

### 4.1 Incoming Message Flow

```
Slack Message
    │
    ▼
SlackAdapter.normalized_message()
    │
    ▼
PlatformRouter.route(unified_message)
    │
    ▼
OrchestratorAgent.process()
    │
    ├──► IntentClassifier.classify()
    │        │
    │        ▼
    │    Intent: DELEGATE_TASK
    │
    ▼
TaskRouter.route(intent, message)
    │
    ▼
AgentFactory.get_or_create(agent_id)
    │
    ▼
SpecialistAgent.execute(task)
    │
    ├──► MemoryManager.get_context()
    ├──► Tools.execute()
    └──► MemoryManager.store()
    │
    ▼
Result returned to orchestrator
    │
    ▼
OrchestratorAgent.format_response()
    │
    ▼
PlatformRouter.reply(session_id, response)
    │
    ▼
SlackAdapter.send_message()
```

### 4.2 Agent Delegation Flow

```
Orchestrator: "Handle customer complaint"
    │
    ▼
AgentFactory.create(AgentDefinition(
    name="complaint_handler",
    instructions="Resolve customer complaints...",
    tools=[email_tool, ticket_tool, refund_tool]
))
    │
    ▼
New Agent registered in AgentRegistry
    │
    ▼
Orchestrator.delegate(task, agent_id="complaint_handler_001")
    │
    ▼
complaint_handler Agent
    │
    ├──► Own MemoryManager.initialize()
    ├──► Execute task with its tools
    └──► Store learned context in its memory
    │
    ▼
Result + any new knowledge stored
    │
    ▼
Orchestrator.cleanup() if transient agent
```

---

## 5. Project Structure

```
business_ai_automation/
├── src/
│   └── business_ai_automation/
│       ├── __init__.py
│       ├── main.py                      # FastAPI app entry point
│       │
│       ├── platform/
│       │   ├── __init__.py
│       │   ├── adapter.py               # MessagingAdapter ABC
│       │   ├── slack/
│       │   │   ├── __init__.py
│       │   │   ├── adapter.py           # Slack-specific implementation
│       │   │   ├── events.py           # Slack event handling
│       │   │   └── formatting.py       # Slack message formatting
│       │   ├── teams/
│       │   │   ├── __init__.py
│       │   │   └── adapter.py
│       │   └── web/
│       │       ├── __init__.py
│       │       └── adapter.py           # FastAPI webhook adapter
│       │
│       ├── agent/
│       │   ├── __init__.py
│       │   ├── orchestrator/
│       │   │   ├── __init__.py
│       │   │   ├── agent.py            # Main orchestrator agent
│       │   │   ├── intent/
│       │   │   │   ├── __init__.py
│       │   │   │   └── classifier.py   # Intent classification
│       │   │   └── router/
│       │   │       ├── __init__.py
│       │   │       └── task_router.py  # Task routing logic
│       │   ├── specialist/
│       │   │   ├── __init__.py
│       │   │   ├── agent.py           # Base specialist agent
│       │   │   └── factory.py         # Agent factory
│       │   └── registry/
│       │       ├── __init__.py
│       │       └── registry.py         # Agent capability registry
│       │
│       ├── memory/
│       │   ├── __init__.py
│       │   ├── manager.py              # Per-agent memory manager
│       │   ├── short_term.py           # Session/working memory (ADK)
│       │   ├── long_term.py            # Persistent memory
│       │   ├── knowledge/
│       │   │   ├── __init__.py
│       │   │   ├── graph.py            # Knowledge graph
│       │   │   └── embedding.py        # Vector embeddings
│       │   └── storage/
│       │       ├── __init__.py
│       │       ├── memory_store.py      # Storage interface
│       │       └── postgres_store.py    # PostgreSQL + pgvector backend
│       │
│       ├── extension/
│       │   ├── __init__.py
│       │   ├── framework.py             # Extension base class
│       │   ├── loader.py                # Dynamic extension loader
│       │   ├── registry.py              # Extension registry
│       │   └── examples/
│       │       ├── __init__.py
│       │       ├── tool_extension.py
│       │       ├── trigger_extension.py
│       │       └── memory_extension.py
│       │
│       ├── adk/
│       │   ├── __init__.py
│       │   ├── runner.py               # ADK Runner wrapper
│       │   ├── session.py             # Session management
│       │   └── tools.py               # Tool registration
│       │
│       └── config/
│           ├── __init__.py
│           └── settings.py             # Pydantic settings
│
├── extensions/                          # User-built extensions
│   └── README.md
│
├── migrations/
│   └── 001_initial_schema.sql
│
├── tests/
│   ├── __init__.py
│   ├── unit/
│   │   ├── __init__.py
│   │   ├── test_intent_classifier.py
│   │   ├── test_task_router.py
│   │   ├── test_memory_manager.py
│   │   ├── test_agent_factory.py
│   │   └── test_extension_framework.py
│   ├── integration/
│   │   ├── __init__.py
│   │   ├── test_orchestrator.py
│   │   ├── test_slack_adapter.py
│   │   └── test_specialist_agent.py
│   └── conftest.py
│
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
└── Makefile
```

---

## 6. Key Technical Decisions

### 6.1 ADK Integration

**Package:** `google-adk` (PyPI: `google-adk`)

**Approach:** Use Google ADK as the agent runtime, wrapping it where needed.

- ADK provides: `Agent`, `Runner`, `Session`, `Tool` primitives
- We wrap ADK's `Agent` in our `SpecialistAgent` to add:
  - Per-agent memory managers
  - Extension tool injection
  - Dynamic capability registration
- Use ADK's `SequentialAgent` / `ParallelAgent` / `LoopAgent` for orchestration patterns
- Use `transfer_to_agent()` for LLM-driven delegation between agents

**Reference:** [Google ADK Python](https://google.github.io/adk-docs/get-started/python/)

### 6.2 Memory Storage

**Choice:** PostgreSQL with pgvector for knowledge graph + embeddings

- Short-term: ADK `InMemorySessionService` (Phase 1), swap to `DatabaseSessionService` later
- Long-term: PostgreSQL with `pgvector` for semantic search
- Knowledge graph: PostgreSQL with JSONB + recursive CTEs
- Async: Use `asyncpg` for non-blocking DB access

**Why:** Single database simplifies ops, pgvector enables embedding search.

### 6.3 Extension Loading

**Choice:** Python dynamic import via `importlib` + entry points

- Extensions are Python packages/modules
- Loaded via `importlib.import_module()` at runtime
- Registered via `entry_points` in `pyproject.toml` or a config directory
- Sandboxed execution for untrusted extensions (future)

**Why:** Native Python, no compiled code, works with any Python extension.

### 6.4 Messaging Normalization

**Choice:** Platform adapters normalize to `UnifiedMessage` before hitting core

- Each adapter handles platform-specific auth, formatting, rate limits
- Core platform is platform-agnostic
- Easy to add new platforms (just implement `MessagingAdapter`)

### 6.5 Web Framework

**Choice:** FastAPI for webhook adapters

- Handles Slack/Teams webhook endpoints
- `AsyncSession` for async request handling
- Built-in OpenAPI/Swagger docs
- Pydantic for request validation

---

## 7. NOT in Scope

- **Email integration** — future phase
- **Native mobile SDK** — future phase
- **Multi-tenancy** — single-tenant for v1
- **Extension sandboxing** — v1 extensions are trusted
- **Agent migration/sharing** — v1 agents are single-instance
- **High availability** — single orchestrator instance in v1
- **Multiple orchestrator instances** — v2+

---

## 8. What Already Exists

- Design doc (`brandon-master-design-20260322-125322.md`) — defines product direction
- Empty git repository (no code yet)

This plan provides the technical implementation architecture for the "Niche-Focused Platform Approach" in the design doc.

---

## 9. Implementation Phases

### Phase 1: Foundation (Week 1-2)
- Project setup, ADK integration (`google-adk`)
- Basic orchestrator agent with intent classification
- In-memory session management (ADK default)
- Slack adapter (MVP)
- FastAPI webhook endpoints
- Basic extension skeleton (interface + in-process registry)

### Phase 2: Agent System (Week 3-4)
- Specialist agent framework
- Agent factory (dynamic creation/destruction)
- Basic memory manager (short-term + medium-term summarization)
- Static specialist agents (customer_service, scheduler)
- Agent capability registry

### Phase 3: Persistence (Week 5-6)
- PostgreSQL integration with `asyncpg`
- Long-term memory (knowledge graph + embeddings via pgvector)
- Session persistence across restarts
- Tool registration from extensions

### Phase 4: Platform Polish (Week 7-8)
- Additional messaging adapters (Teams, Webhook)
- Extension examples (tool, trigger, memory)
- Error handling and retry logic
- Logging and observability (OpenTelemetry)
- Orchestrator health check + restart

---

## 10. Dependencies

```toml
# pyproject.toml (key dependencies)
[project]
dependencies = [
    "google-adk>=1.0.0",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "asyncpg>=0.30.0",
    "pgvector>=0.3.0",
    "slack-sdk>=3.30.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "structlog>=24.0.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "ruff>=0.8.0",
    "mypy>=1.0.0",
]
```

---

## 11. Open Questions

1. **Database:** PostgreSQL only, or SQLite for local dev?
2. **Agent isolation:** Should specialist agents run in separate processes?
3. **Scaling:** Single-node first, or design for multi-node from start?
4. **Extension security:** Allow untrusted extensions in v1?
5. **Agent persistence:** Save/restore agent state across restarts?

---

## 12. Failure Modes

| Component | Failure Mode | Impact | Mitigation |
|-----------|-------------|--------|------------|
| Orchestrator | Crash loop / hang | System offline | Health check + restart |
| SlackAdapter | Token expiry | Messages not received | Token refresh + alerting |
| SpecialistAgent | Infinite loop in LLM | Resource exhaustion | Per-agent timeout + circuit breaker |
| MemoryManager | DB connection lost | Lost context | Reconnect + local buffer |
| Extension | Extension crashes | Tool unavailable | Isolation + restart |
| AgentFactory | Create bad agent | Resource leak | Agent limits + cleanup |

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 1 | CLEAR | 2 proposals, 2 accepted, 0 deferred |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | no |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | issues_open | 14 issues, 0 critical gaps |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | no |

CODEX:
CROSS-MODEL:
UNRESOLVED: 0
VERDICT: CEO + ENG REVIEW CLEARED — eng review required.
