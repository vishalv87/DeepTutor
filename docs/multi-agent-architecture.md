# DeepTutor Multi-Agent Architecture

> A comprehensive guide to how DeepTutor orchestrates 15+ AI agents across 6 capabilities to power an intelligent tutoring system.

---

## Table of Contents

1. [Introduction & High-Level Overview](#1-introduction--high-level-overview)
2. [The Request Journey (Input to Output)](#2-the-request-journey-input-to-output)
3. [Core Building Blocks](#3-core-building-blocks)
4. [The Orchestration Layer](#4-the-orchestration-layer)
5. [Deep Dive: Each Capability & Its Agent Pipeline](#5-deep-dive-each-capability--its-agent-pipeline)
6. [Coordination Patterns](#6-coordination-patterns)
7. [The Tool System](#7-the-tool-system)
8. [Data & State Management](#8-data--state-management)
9. [Streaming Architecture](#9-streaming-architecture)
10. [Key Files Reference](#10-key-files-reference)

---

## 1. Introduction & High-Level Overview

### What is DeepTutor?

DeepTutor is an AI-powered tutoring platform that helps users learn from documents, solve problems, generate quiz questions, and conduct deep research. Under the hood, it coordinates **multiple specialized AI agents** — each with a specific role — to produce high-quality, multi-step responses.

### What is Multi-Agent Orchestration?

Think of it like a hospital. When a patient arrives:
- A **receptionist** (entry point) takes in the request
- A **triage nurse** (orchestrator) decides which department handles it
- The **department** (capability) has a team of **specialists** (agents) who collaborate

In DeepTutor:
- The **WebSocket endpoint** receives the user's question
- The **ChatOrchestrator** routes it to the right capability
- The **capability** runs a pipeline of specialized agents to produce the answer

### The 4-Layer Architecture

```
                          USER (Browser / CLI)
                               |
                    ===========================
                    |    LAYER 1: ENTRY       |
                    |  WebSocket / REST API   |
                    ===========================
                               |
                    ===========================
                    |  LAYER 2: ORCHESTRATOR  |
                    |    ChatOrchestrator     |
                    |  (routes to capability) |
                    ===========================
                               |
              +----------------+----------------+
              |                |                |
     ==================  ==================  ==================
     | LAYER 3:       |  | LAYER 3:       |  | LAYER 3:       |
     | CAPABILITY     |  | CAPABILITY     |  | CAPABILITY     |
     | Chat           |  | Deep Solve     |  | Deep Research  |  ...
     ==================  ==================  ==================
              |                |                |
     ==================  ==================  ==================
     | LAYER 4:       |  | LAYER 4:       |  | LAYER 4:       |
     | AGENT(S)       |  | AGENTS         |  | AGENTS         |
     | Agentic Chat   |  | Planner        |  | Rephrase       |
     | Pipeline       |  | Solver         |  | Decompose      |
     |                |  | Writer         |  | Manager        |
     |                |  |                |  | Research        |
     |                |  |                |  | Note            |
     |                |  |                |  | Reporting       |
     ==================  ==================  ==================
```

### Key Design Decisions

DeepTutor uses a **custom pipeline-based orchestration** model rather than frameworks like LangGraph or LangChain's StateGraph. This gives the system:

- **Fine-grained control** over how agents communicate
- **Real-time streaming** of each agent's progress to the user
- **Flexible coordination patterns** — different capabilities use different patterns (sequential, queue-based, batch)
- **Simpler debugging** — no hidden framework magic

---

## 2. The Request Journey (Input to Output)

Let's trace a real user question — *"Explain the Pythagorean theorem"* sent via the **Deep Solve** mode — through every layer of the system.

### Step-by-Step Walkthrough

```
 BROWSER                    BACKEND                           AGENTS
    |                          |                                 |
    |  1. WebSocket message    |                                 |
    |  {capability: "deep_solve", content: "Explain..."}         |
    |------------------------->|                                 |
    |                          |                                 |
    |                     2. TurnRuntimeManager                  |
    |                        creates turn in SQLite              |
    |                        assembles UnifiedContext             |
    |                          |                                 |
    |                     3. ChatOrchestrator.handle()            |
    |                        looks up "deep_solve" capability     |
    |                        creates StreamBus                   |
    |                          |                                 |
    |                     4. DeepSolveCapability.run()            |
    |                          |                                 |
    |                          |  5. PlannerAgent.process()      |
    |  <-- STAGE_START --------|     breaks problem into steps   |
    |  <-- THINKING ---------- |                                 |
    |  <-- TOOL_CALL (rag) --- |     (retrieves from KB)         |
    |  <-- TOOL_RESULT --------|                                 |
    |  <-- STAGE_END ----------|                                 |
    |                          |                                 |
    |                          |  6. SolverAgent.process()       |
    |  <-- STAGE_START --------|     ReAct loop: think-act-observe|
    |  <-- THINKING ---------- |     uses tools (rag, code, web) |
    |  <-- TOOL_CALL ----------|                                 |
    |  <-- TOOL_RESULT --------|                                 |
    |  <-- STAGE_END ----------|                                 |
    |                          |                                 |
    |                          |  7. WriterAgent.process()       |
    |  <-- STAGE_START --------|     formats final solution      |
    |  <-- CONTENT (streamed)--|                                 |
    |  <-- STAGE_END ----------|                                 |
    |                          |                                 |
    |  <-- DONE ---------------|  8. Turn complete               |
    |                          |     message saved to SQLite     |
    |                          |     memory updated              |
```

### Detailed Breakdown

#### Step 1: WebSocket Entry

**File:** `deeptutor/api/routers/unified_ws.py`

The browser opens a WebSocket connection to `/api/v1/ws` and sends a JSON message:

```json
{
  "type": "message",
  "session_id": "abc-123",
  "capability": "deep_solve",
  "content": "Explain the Pythagorean theorem",
  "tools": ["rag", "web_search", "code_execution"],
  "knowledge_bases": ["my_textbook"],
  "language": "en"
}
```

The WebSocket handler parses this and passes it to the `TurnRuntimeManager`.

#### Step 2: Turn Creation & Context Assembly

**File:** `deeptutor/services/session/turn_runtime.py`

The `TurnRuntimeManager` does several things:

1. **Creates a turn record** in SQLite (status: "running")
2. **Assembles the context** by gathering:
   - **Conversation history** — previous messages from this session (via `ContextBuilder`)
   - **Memory context** — what the system remembers about this user (episodic, facts, understanding)
   - **Notebook context** — any notes the user has referenced
   - **History context** — referenced past sessions
3. **Builds a `UnifiedContext`** object — a single data structure that carries everything downstream:

```python
context = UnifiedContext(
    session_id="abc-123",
    user_message="Explain the Pythagorean theorem",
    conversation_history=[...],        # Previous messages
    enabled_tools=["rag", "web_search", "code_execution"],
    active_capability="deep_solve",
    knowledge_bases=["my_textbook"],
    language="en",
    memory_context="User is a high school student...",
    metadata={"turn_id": "turn-456"}
)
```

#### Step 3: Orchestrator Routing

**File:** `deeptutor/runtime/orchestrator.py`

The `ChatOrchestrator` receives the `UnifiedContext` and:

1. Reads `context.active_capability` — here it's `"deep_solve"`
2. Looks up the capability in the `CapabilityRegistry`
3. Creates a `StreamBus` — the real-time event channel
4. Spawns an async task running `capability.run(context, bus)`
5. Starts yielding events from the bus back to the WebSocket

```python
cap_name = context.active_capability or "chat"   # "deep_solve"
capability = self._cap_registry.get(cap_name)     # DeepSolveCapability

bus = StreamBus()
task = asyncio.create_task(capability.run(context, bus))

async for event in bus.subscribe():
    yield event  # Events flow back to the WebSocket handler
```

#### Step 4-7: Capability Runs Its Agent Pipeline

The `DeepSolveCapability` creates a `MainSolver` and runs three agents in sequence: Planner, Solver, Writer. Each agent emits events into the StreamBus as it works.

(We'll explore each capability's pipeline in detail in [Section 5](#5-deep-dive-each-capability--its-agent-pipeline).)

#### Step 8: Turn Completion

After the capability finishes:

1. The `StreamBus` emits a `DONE` event and closes
2. The assistant's response is saved to SQLite as a message
3. The `MemoryService` updates user memory from the turn
4. A `CAPABILITY_COMPLETE` event is published to the global `EventBus`

---

## 3. Core Building Blocks

These are the fundamental abstractions that every part of the system uses.

### 3.1 UnifiedContext — The Request Envelope

**File:** `deeptutor/core/context.py`

Every user turn is wrapped in a `UnifiedContext`. Think of it as a **letter** that gets passed from the entry point all the way down to individual agents:

```python
@dataclass
class UnifiedContext:
    session_id: str                    # Persistent conversation ID
    user_message: str                  # The current question
    conversation_history: list[dict]   # Previous messages (OpenAI format)
    enabled_tools: list[str] | None    # Which tools are available
    active_capability: str | None      # "chat", "deep_solve", etc.
    knowledge_bases: list[str]         # KB names for RAG
    attachments: list[Attachment]      # Images or files
    config_overrides: dict             # Per-request settings (e.g., temperature)
    language: str                      # "en" or "zh"
    notebook_context: str              # Referenced notes
    history_context: str               # Referenced past sessions
    memory_context: str                # What the system remembers about the user
    metadata: dict                     # Extra data (turn_id, etc.)
```

**Why it matters:** By putting everything into one object, any component in the system can access what it needs without tight coupling. A capability doesn't need to know _how_ the context was assembled — it just reads what it needs.

### 3.2 StreamBus — The Real-Time Event Channel

**File:** `deeptutor/core/stream_bus.py`

The `StreamBus` is a **fan-out async event channel**. Agents and capabilities _emit_ events into it; consumers (WebSocket, CLI, etc.) _subscribe_ to it.

```
  Producer (Agent)              StreamBus                 Consumers
  ================         ==================         ================
  emit(THINKING)   ------> |  _history: []  | ------> WebSocket Client
  emit(TOOL_CALL)  ------> |  _subscribers: | ------> CLI Renderer
  emit(CONTENT)    ------> |    [Queue, ...]| ------> JSON Writer
  ...              ------> |                |
                           ==================
```

Key features:
- **Multiple subscribers** can listen simultaneously
- **Late subscribers** get the full history (replay)
- **Convenience methods** like `bus.thinking("...")`, `bus.tool_call("rag", {...})`, `bus.content("...")`
- **Stage management** via `async with bus.stage("planning"):` context manager

### 3.3 StreamEvent — The Message Format

**File:** `deeptutor/core/stream.py`

Every event flowing through the StreamBus is a `StreamEvent`:

```python
@dataclass
class StreamEvent:
    type: StreamEventType    # What kind of event
    source: str              # Who produced it ("deep_solve", "rag", etc.)
    stage: str               # Current phase ("planning", "reasoning", etc.)
    content: str             # Human-readable payload
    metadata: dict           # Structured data (tool args, sources, metrics)
    session_id: str
    turn_id: str
    seq: int                 # Auto-incremented sequence number
    timestamp: float         # When it was created
```

The **13 event types** are:

| Event Type | Purpose | Example |
|---|---|---|
| `SESSION` | Marks the start of a new turn | Session ID, turn ID |
| `STAGE_START` | A named stage begins | "planning", "reasoning" |
| `STAGE_END` | A named stage ends | "planning" complete |
| `THINKING` | Agent's internal reasoning | "Let me break this into steps..." |
| `OBSERVATION` | Agent observes tool results | "The RAG search returned 3 chunks..." |
| `CONTENT` | Final response text (streamed) | "The Pythagorean theorem states..." |
| `TOOL_CALL` | Agent invokes a tool | tool="rag", args={query: "..."} |
| `TOOL_RESULT` | Tool returns its result | Retrieved document chunks |
| `PROGRESS` | Progress update | "Researching subtopic 3 of 5" |
| `SOURCES` | Citation/source data | URLs, document references |
| `RESULT` | Structured final result | JSON data payload |
| `ERROR` | Something went wrong | Error message |
| `DONE` | Turn is complete | End marker |

### 3.4 BaseCapability — The Capability Interface

**File:** `deeptutor/core/capability_protocol.py`

Every "mode" in DeepTutor (Chat, Deep Solve, Deep Research, etc.) is a **Capability**. All capabilities implement this interface:

```python
class BaseCapability(ABC):
    manifest: CapabilityManifest  # Static metadata

    @abstractmethod
    async def run(self, context: UnifiedContext, stream: StreamBus) -> None:
        """Execute the full pipeline, emitting events to stream."""
        ...
```

The `CapabilityManifest` describes the capability:

```python
@dataclass
class CapabilityManifest:
    name: str                  # "deep_solve"
    description: str           # "Multi-agent problem solving."
    stages: list[str]          # ["planning", "reasoning", "writing"]
    tools_used: list[str]      # ["rag", "web_search", "code_execution"]
    cli_aliases: list[str]     # ["solve", "s"]
    request_schema: dict       # Expected request fields
    config_defaults: dict      # Default configuration values
```

### 3.5 BaseAgent — The Agent Foundation

**File:** `deeptutor/agents/base_agent.py`

Every individual agent (PlannerAgent, ResearchAgent, etc.) extends `BaseAgent`. It provides:

- **LLM configuration** — API keys, model selection, temperature, max tokens
- **Prompt loading** — prompts loaded from YAML files via `PromptManager`
- **LLM call interface** — `call_llm()` (non-streaming) and `stream_llm()` (streaming)
- **Token tracking** — counts tokens across all calls for cost monitoring
- **Trace callbacks** — structured logging of every LLM call

```python
class BaseAgent(ABC):
    @abstractmethod
    async def process(self, *args, **kwargs) -> Any:
        """Main processing logic — must be implemented by subclasses."""
        ...

    async def call_llm(self, user_prompt, system_prompt, ...) -> str:
        """Call the LLM and get a complete response."""
        ...

    async def stream_llm(self, user_prompt, system_prompt, ...) -> AsyncGenerator[str]:
        """Stream LLM responses chunk by chunk."""
        ...
```

The config resolution chain for the model is:
1. Agent-specific config (from `agents.yaml`)
2. General LLM config
3. Instance model
4. Environment variable (`LLM_MODEL`)

---

## 4. The Orchestration Layer

### 4.1 ChatOrchestrator — The Traffic Cop

**File:** `deeptutor/runtime/orchestrator.py`

The `ChatOrchestrator` is the **single unified entry point** for all user interactions. Whether the request comes from the WebSocket, CLI, or SDK — it all flows through here.

```python
class ChatOrchestrator:
    async def handle(self, context: UnifiedContext) -> AsyncIterator[StreamEvent]:
        # 1. Which capability should handle this?
        cap_name = context.active_capability or "chat"
        capability = self._cap_registry.get(cap_name)

        # 2. Create the event channel
        bus = StreamBus()

        # 3. Run the capability in a background task
        task = asyncio.create_task(capability.run(context, bus))

        # 4. Yield events as they arrive
        async for event in bus.subscribe():
            yield event
```

This design means:
- The orchestrator **doesn't know or care** what happens inside each capability
- It just connects the capability to the event channel and streams events back
- Error handling is centralized — if any capability throws, the orchestrator catches it and emits an ERROR event

### 4.2 CapabilityRegistry — Discovery & Loading

**File:** `deeptutor/runtime/registry/capability_registry.py`

At startup, the registry loads all available capabilities:

```
Built-in Capabilities (from builtin_capabilities.py):
  - "chat"           → ChatCapability
  - "deep_solve"     → DeepSolveCapability
  - "deep_question"  → DeepQuestionCapability
  - "deep_research"  → DeepResearchCapability
  - "math_animator"  → MathAnimatorCapability
  - "visualize"      → VisualizeCapability
```

**File:** `deeptutor/runtime/bootstrap/builtin_capabilities.py`

The registry also supports **plugin capabilities** — third-party extensions that register themselves at startup.

### 4.3 ToolRegistry — Tool Management

**File:** `deeptutor/runtime/registry/tool_registry.py`

The `ToolRegistry` manages all tools available to agents:

- **Registration**: Tools are discovered and registered at startup
- **Schema generation**: Generates OpenAI-compatible tool schemas for LLM function calling
- **Execution**: Routes `tool_name + args` to the correct tool implementation

```python
registry = get_tool_registry()
registry.list_tools()          # ["rag", "web_search", "code_execution", ...]
registry.execute("rag", query="Pythagorean theorem", kb_name="textbook")
registry.build_openai_schemas(["rag", "web_search"])  # For LLM tool calling
```

---

## 5. Deep Dive: Each Capability & Its Agent Pipeline

### 5.1 Chat Capability — The Conversational Agent

**Files:**
- `deeptutor/capabilities/chat.py`
- `deeptutor/agents/chat/agentic_pipeline.py`

The Chat capability is the **default mode**. It uses a single `AgenticChatPipeline` that runs a **4-stage agentic loop**:

```
    User Question
         |
         v
  +------------------+
  |  1. THINKING     |  LLM analyzes the question
  |  "What do I need |  Decides if tools are needed
  |   to answer this?"|
  +------------------+
         |
         v
  +------------------+
  |  2. ACTING       |  LLM selects tools & arguments
  |  Calls: rag,     |  Executes up to 8 tools in parallel
  |  web_search, etc.|
  +------------------+
         |
         v
  +------------------+
  |  3. OBSERVING    |  LLM reviews tool results
  |  "The RAG found  |  Synthesizes information
  |   3 relevant..." |
  +------------------+
         |
         v
  +------------------+
  |  4. RESPONDING   |  LLM generates final answer
  |  Streams the     |  (streamed token by token)
  |  response back   |
  +------------------+
```

**Available tools:** All built-in tools except `geogebra_analysis` — this includes RAG, web search, code execution, paper search, reasoning, and brainstorming.

**Key details:**
- Tool calls in the Acting stage run **in parallel** (up to `MAX_PARALLEL_TOOL_CALLS = 8`)
- Tool results are truncated to `MAX_TOOL_RESULT_CHARS = 4000` characters
- The final response is **streamed** token-by-token for real-time display
- If the LLM decides no tools are needed, stages 2 and 3 are skipped

### 5.2 Deep Solve Capability — The Problem Solver

**Files:**
- `deeptutor/capabilities/deep_solve.py`
- `deeptutor/agents/solve/main_solver.py`
- `deeptutor/agents/solve/agents/planner_agent.py`
- `deeptutor/agents/solve/agents/solver_agent.py`
- `deeptutor/agents/solve/agents/writer_agent.py`

Deep Solve uses **3 agents** in a **sequential pipeline**: Plan, then Solve, then Write.

```
    User Problem
         |
         v
  +-------------------+
  |  PLANNER AGENT    |  Breaks the problem into steps
  |                   |  Optionally retrieves from KB
  |  Input:  problem  |
  |  Output: plan     |  "Step 1: ... Step 2: ... Step 3: ..."
  +-------------------+
         |
         | plan
         v
  +-------------------+
  |  SOLVER AGENT     |  Solves each step using ReAct loop
  |                   |
  |  Input:  plan +   |  Think -> Act (use tools) -> Observe
  |          problem  |  Think -> Act -> Observe
  |  Output: solution |  ... (repeats until solved)
  |          steps    |
  +-------------------+
         |
         | plan + solution steps
         v
  +-------------------+
  |  WRITER AGENT     |  Formats into a clear explanation
  |                   |
  |  Input:  plan +   |  Structures the solution
  |   solver output   |  Adds explanations
  |  Output: final    |  Formats LaTeX, code, etc.
  |          solution |
  +-------------------+
         |
         v
    Formatted Solution (streamed to user)
```

**Coordination pattern:** Sequential — each agent must complete before the next starts. The output of one becomes the input to the next.

**Tools:** RAG, web search, code execution, reason

**Key details:**
- The Planner can optionally retrieve from the knowledge base to inform its plan
- The Solver uses a **ReAct (Reasoning + Acting) loop** — it thinks about what to do, takes an action (calls a tool), observes the result, and repeats
- A `Scratchpad` object accumulates reasoning steps and tool outputs across the loop
- A `TokenTracker` monitors cost across all three agents

### 5.3 Deep Research Capability — The Research Team

**Files:**
- `deeptutor/capabilities/deep_research.py`
- `deeptutor/agents/research/research_pipeline.py`
- `deeptutor/agents/research/agents/` (6 agent files)

Deep Research is the **most complex capability**, using **6 agents** with a **managed queue** coordination pattern:

```
    User Topic: "Quantum Computing Applications"
         |
         v
  +-------------------+
  |  REPHRASE AGENT   |  Clarifies and optimizes the topic
  |  Input:  topic    |  "Quantum computing applications in
  |  Output: refined  |   cryptography, optimization, and ML"
  +-------------------+
         |
         v
  +-------------------+
  |  DECOMPOSE AGENT  |  Breaks into subtopics
  |  Input:  topic    |  1. "Quantum cryptography (Shor's, BB84)"
  |  Output: subtopics|  2. "Quantum optimization (QAOA, VQE)"
  |          + queue  |  3. "Quantum ML (QNN, kernel methods)"
  +-------------------+
         |
         v (subtopics go into DynamicTopicQueue)
  +=====================================================+
  |                  RESEARCH LOOP                       |
  |                                                     |
  |  +-------------------+                              |
  |  |  MANAGER AGENT    |  Decides what to research    |
  |  |  Manages queue    |  next from the queue         |
  |  +-------------------+                              |
  |         |                                           |
  |         v                                           |
  |  +-------------------+                              |
  |  |  RESEARCH AGENT   |  Deep-dives into one topic   |
  |  |  Uses: RAG, web,  |  using tools                 |
  |  |  paper search     |                              |
  |  +-------------------+                              |
  |         |                                           |
  |         v                                           |
  |  +-------------------+                              |
  |  |  NOTE AGENT       |  Summarizes findings into    |
  |  |  Takes structured |  organized notes             |
  |  |  notes            |                              |
  |  +-------------------+                              |
  |         |                                           |
  |         v                                           |
  |  (Loop back to Manager if queue not empty)          |
  +=====================================================+
         |
         v (all notes collected)
  +-------------------+
  |  REPORTING AGENT   |  Synthesizes everything into
  |  Input: all notes  |  a final research report
  |  Output: report    |  with citations and structure
  +-------------------+
         |
         v
    Research Report (streamed to user)
```

**Coordination pattern:** Managed Queue — the `DynamicTopicQueue` holds subtopics. The Manager Agent decides the order. New subtopics can be added dynamically during research.

**Tools:** RAG, web search, paper search (arXiv), code execution

**Key details:**
- The `DynamicTopicQueue` can grow during research — if the Research Agent discovers a new important subtopic, it can be added to the queue
- A `CitationManager` tracks all sources across all research agents
- Progress is persisted to JSON files so research can be resumed if interrupted
- The Reporting Agent produces a structured report with citations from all gathered notes

### 5.4 Deep Question Capability — The Quiz Generator

**Files:**
- `deeptutor/capabilities/deep_question.py`
- `deeptutor/agents/question/coordinator.py`
- `deeptutor/agents/question/agents/idea_agent.py`
- `deeptutor/agents/question/agents/generator.py`
- `deeptutor/agents/question/agents/followup_agent.py`

Deep Question generates quiz questions using **3 agents** in a **batch + fan-out** pattern:

```
    Topic: "Linear Algebra" + preferences (difficulty, type, count)
         |
         v
  +==============================================+
  |           BATCH GENERATION LOOP              |
  |                                              |
  |  +-------------------+                       |
  |  |  IDEA AGENT       |  Generates question   |
  |  |  (max 5 per batch)|  templates/ideas      |
  |  +-------------------+                       |
  |         |                                    |
  |         v  (for each template)               |
  |  +-------------------+                       |
  |  |  GENERATOR        |  Turns template into  |
  |  |  (one per template)|  full Q&A pair with  |
  |  |                   |  answer + explanation  |
  |  +-------------------+                       |
  |         |                                    |
  |  (repeat batches until enough questions)     |
  +==============================================+
         |
         v
    List of QAPair objects (question + answer + explanation)

    Later, during quiz:
  +-------------------+
  |  FOLLOWUP AGENT   |  Answers follow-up
  |  "Why is option B |  questions about a
  |   wrong?"         |  specific quiz item
  +-------------------+
```

**Coordination pattern:** Batch + Fan-out — templates are generated in batches of 5, then each template fans out to its own Generator instance.

**Tools:** RAG, web search, code execution

**Key details:**
- Supports two modes: **"custom"** (topic-driven) and **"mimic"** (paper-driven, extracts patterns from uploaded papers)
- The IdeaAgent tracks `existing_concentrations` to avoid generating duplicate question themes
- Each `QAPair` is a structured object with question text, answer, explanation, difficulty level, and question type
- The FollowupAgent is invoked later when a user asks about a specific quiz question

### 5.5 Math Animator Capability

**File:** `deeptutor/capabilities/math_animator.py`

Generates animated step-by-step math solutions using the Manim library. Stages: **planning** (breaks the math into animation scenes) and **animating** (generates Manim Python code).

### 5.6 Visualize Capability

**File:** `deeptutor/capabilities/visualize.py`

Generates visual representations (charts, diagrams) of concepts or solutions.

---

## 6. Coordination Patterns

DeepTutor uses five distinct patterns for coordinating agents. Understanding these patterns helps you predict how any new capability would work.

### Pattern 1: Sequential Pipeline

```
  Agent A  --->  Agent B  --->  Agent C
  (output)       (output)       (output)
```

**Used by:** Deep Solve (Planner → Solver → Writer)

Each agent must complete before the next starts. The output of one agent becomes input to the next. Simple, predictable, and easy to debug.

**Trade-off:** No parallelism — if Agent B could start before Agent A finishes, this pattern wastes that opportunity. But it guarantees each agent has complete context from the previous one.

### Pattern 2: Managed Queue

```
                    DynamicTopicQueue
                   +---+---+---+---+
                   | T1| T2| T3| T4|
                   +---+---+---+---+
                          |
                  Manager decides order
                          |
              +-----------+-----------+
              |                       |
        Research Agent          (new topics added)
              |
         Note Agent
```

**Used by:** Deep Research

A queue holds work items. A Manager Agent decides what to process next. New items can be added to the queue during processing. This allows dynamic, adaptive research.

**Trade-off:** More complex to implement and debug, but handles open-ended tasks where the scope isn't known upfront.

### Pattern 3: Batch + Fan-out

```
  IdeaAgent ──> [Template 1] ──> Generator 1 ──> QAPair 1
            ──> [Template 2] ──> Generator 2 ──> QAPair 2
            ──> [Template 3] ──> Generator 3 ──> QAPair 3
            ──> [Template 4] ──> Generator 4 ──> QAPair 4
            ──> [Template 5] ──> Generator 5 ──> QAPair 5
```

**Used by:** Deep Question

One agent generates a batch of items, then each item is processed independently. Batches repeat until the target count is reached.

**Trade-off:** Good parallelism potential within each batch, but requires tracking to avoid duplicates across batches.

### Pattern 4: Agentic Loop with Tool Calling

```
  +--> THINK ──> ACT (call tools) ──> OBSERVE ──+
  |                                               |
  +──────────── (repeat if needed) ──────────────+
                      |
                      v
                   RESPOND
```

**Used by:** Chat, and internally by the Solver Agent

The agent enters a think-act-observe loop. It decides which tools to call, executes them, observes the results, and either loops again or produces a final response.

**Trade-off:** Very flexible — the agent adapts in real-time based on tool results. But harder to predict how many iterations it will take (and thus how much it will cost).

### Pattern 5: TutorBot Multi-Agent Teams

**Files:**
- `deeptutor/tutorbot/agent/team/state.py`
- `deeptutor/tutorbot/agent/loop.py`

The TutorBot framework supports a more advanced **team-based** coordination model:

```
  Team Lead Agent
       |
       +-- Teammate Agent A (role: researcher)
       +-- Teammate Agent B (role: writer)
       +-- Teammate Agent C (role: reviewer)
```

Agents communicate via a **Mail system** (structured messages between agents) and coordinate through **Task dependencies** (Task B can't start until Task A completes).

```python
@dataclass
class Mail:
    id: str
    from_agent: str
    to_agent: str
    content: str
    timestamp: str
    read_by: list[str]

@dataclass
class Task:
    id: str
    title: str
    owner: str
    status: str           # "pending" | "in_progress" | "done"
    depends_on: list[str] # Task IDs this depends on
    plan: str
    result: str
    requires_approval: bool
```

---

## 7. The Tool System

Agents don't just think — they can **take actions** by calling tools. Tools are the agents' hands and eyes.

### 7.1 Built-in Tools

| Tool | Purpose | File |
|---|---|---|
| **rag** | Retrieve relevant chunks from knowledge bases | `deeptutor/tools/rag_tool.py` |
| **web_search** | Search the web for current information | `deeptutor/tools/web_search.py` |
| **code_execution** | Execute Python code in a sandbox | `deeptutor/tools/code_executor.py` |
| **paper_search** | Search academic papers on arXiv | `deeptutor/tools/paper_search_tool.py` |
| **reason** | Extended deep reasoning (dedicated LLM call) | `deeptutor/tools/reason.py` |
| **brainstorm** | Generate diverse ideas breadth-first | `deeptutor/tools/brainstorm.py` |
| **geogebra_analysis** | Mathematical analysis with GeoGebra | `deeptutor/tools/vision/` |

### 7.2 How Tool Calling Works

When an agent decides to use a tool, here's what happens:

```
  Agent (LLM)                    Tool Registry                Tool Implementation
  ===========                    =============                ===================
  1. LLM returns a              2. Registry looks
     tool_call:                    up the tool:
     {                             registry.execute(
       name: "rag",                 "rag",
       args: {                      query="...",
         query: "...",              kb_name="..."
         kb_name: "..."           )
       }
     }
                                 3. Tool executes:
                                    RAGService.search(
                                      query, kb_name
                                    )
                                                              4. Returns results:
                                                                 [Chunk(...), ...]
  5. Agent receives
     tool result and
     continues reasoning
```

### 7.3 Tool Schemas

Tools are described using **OpenAI-compatible function schemas** so LLMs know how to call them:

```json
{
  "type": "function",
  "function": {
    "name": "rag",
    "description": "Search the knowledge base for relevant information",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {
          "type": "string",
          "description": "The search query"
        },
        "kb_name": {
          "type": "string",
          "description": "Knowledge base to search"
        }
      },
      "required": ["query"]
    }
  }
}
```

The LLM sees these schemas and generates structured tool calls as part of its response.

### 7.4 Tool Selection

Not all tools are available to all capabilities:
- **Chat**: All tools except `geogebra_analysis`
- **Deep Solve**: RAG, web search, code execution, reason
- **Deep Research**: RAG, web search, paper search, code execution
- **Deep Question**: RAG, web search, code execution

Users can also **toggle tools on/off** per request via the `enabled_tools` field in `UnifiedContext`.

---

## 8. Data & State Management

### 8.1 Session Persistence (SQLite)

**File:** `deeptutor/services/session/sqlite_store.py`

All conversations are stored in a SQLite database:

```
  sessions table          turns table             turn_events table
  +---------+------+      +--------+---------+    +--------+-----+---------+
  | id      | title|      | id     | session |    | turn_id| seq | type    |
  | created | ...  |      | status | cap_name|    | content| ... | metadata|
  +---------+------+      +--------+---------+    +--------+-----+---------+
                                                         |
                          messages table                  | (event replay)
                          +--------+------+------+
                          | session| role | content|
                          +--------+------+------+
```

**Key tables:**
- **sessions** — one row per conversation
- **turns** — one row per user message + assistant response
- **turn_events** — every `StreamEvent` from that turn (for replay)
- **messages** — conversation messages in OpenAI format

The event replay system means clients can **resume a turn** mid-stream. If the browser disconnects and reconnects, it can request `subscribe_turn(turn_id, after_seq=42)` to resume from event 42.

### 8.2 Conversation History & Token Budgeting

**File:** `deeptutor/services/session/context_builder.py`

Before each turn, the `ContextBuilder` assembles the conversation history:

1. Fetches all messages from the session
2. Applies a **token budget** — keeps the most recent messages that fit within the limit
3. If the history is too long, generates a **summary** via LLM to compress older messages
4. Returns a `ContextBuildResult` with the history, summary, and token count

This ensures the LLM always has relevant context without exceeding its context window.

### 8.3 Memory Service

**File:** `deeptutor/services/memory/service.py`

DeepTutor maintains **5 types of long-term memory** about each user:

| Memory Type | What it Stores | Example |
|---|---|---|
| **Episodic** | Notable events and interactions | "User struggled with integration by parts" |
| **Facts** | Known facts about the user | "User is a high school junior" |
| **Understanding** | User's knowledge level per topic | "Strong in algebra, weak in calculus" |
| **Strategies** | Effective teaching strategies | "Visual explanations work well" |
| **Mistakes** | Common errors the user makes | "Often forgets the chain rule" |

After each turn, `MemoryService.refresh_from_turn()` updates these memory files. The accumulated memory is injected into the `UnifiedContext.memory_context` field for the next turn.

### 8.4 RAG Pipeline (Retrieval-Augmented Generation)

**Directory:** `deeptutor/services/rag/`

When a user uploads a document, it goes through:

```
  PDF / Markdown / Text
         |
         v
  +------------------+
  |  PARSER          |  Extracts text from the document
  |  (PDF: PyMuPDF)  |  Handles tables, figures, equations
  +------------------+
         |
         v
  +------------------+
  |  CHUNKER         |  Splits text into manageable pieces
  |  (Fixed /        |  - Fixed: token-based chunks
  |   Semantic /     |  - Semantic: sentence-aware
  |   Numbered Item) |  - Numbered Item: list/heading-aware
  +------------------+
         |
         v
  +------------------+
  |  EMBEDDER        |  Converts chunks to vectors
  |  (OpenAI /       |  Using embedding models
  |   Cohere /       |  (text-embedding-3-large default)
  |   Jina / Ollama) |
  +------------------+
         |
         v
  +------------------+
  |  INDEXER          |  Stores vectors for retrieval
  |  (LlamaIndex)    |  In local vector store
  +------------------+
```

When an agent calls the `rag` tool:

```
  Query: "Pythagorean theorem"
         |
         v
  +------------------+
  |  RETRIEVER       |  Finds most similar chunks
  |  (Dense search)  |  via vector similarity
  +------------------+
         |
         v
  Relevant document chunks returned to agent
```

**Data types:**

```python
@dataclass
class Chunk:
    content: str              # The text
    chunk_type: str           # "text", "definition", "theorem", "equation", "figure", "table"
    metadata: dict            # Source info, page number, etc.
    embedding: list[float]    # Vector representation

@dataclass
class SearchResult:
    query: str
    answer: str               # LLM-synthesized answer
    content: str              # Raw content
    chunks: list[Chunk]       # Retrieved chunks
    metadata: dict
```

---

## 9. Streaming Architecture

One of DeepTutor's key features is **real-time streaming** — users see agent thinking, tool calls, and responses as they happen, not after the entire pipeline finishes.

### 9.1 End-to-End Event Flow

```
  Agent                StreamBus              TurnRuntime           WebSocket         Browser
  =====                =========              ===========           =========         =======
  emit(THINKING) ----> queue.put() ---------> persist to SQLite --> send_json() ----> render
  emit(TOOL_CALL) ---> queue.put() ---------> persist to SQLite --> send_json() ----> render
  emit(CONTENT) -----> queue.put() ---------> persist to SQLite --> send_json() ----> render
  ...
  emit(DONE) --------> queue.put(None) -----> mark turn done ----> close ----------> done
```

### 9.2 The StreamBus Fan-Out Pattern

The `StreamBus` supports **multiple concurrent subscribers**:

```python
bus = StreamBus()

# The capability emits events
await bus.thinking("Analyzing the problem...")
await bus.tool_call("rag", {"query": "..."})
await bus.content("The answer is...")

# Multiple consumers receive every event
async for event in bus.subscribe():  # Consumer 1 (WebSocket)
    await ws.send_json(event.to_dict())

async for event in bus.subscribe():  # Consumer 2 (JSON logger)
    log_file.write(event.to_dict())
```

Late subscribers automatically receive the full history (events emitted before they subscribed) via the `_history` buffer.

### 9.3 Event Persistence & Replay

Every event is persisted to SQLite with an auto-incremented sequence number:

```sql
INSERT INTO turn_events (turn_id, seq, type, source, content, metadata, timestamp)
VALUES ('turn-456', 1, 'thinking', 'deep_solve', 'Analyzing...', '{}', 1713190800.0);
```

This enables **client resume**: if a client disconnects, it can reconnect and request:

```json
{"type": "subscribe_turn", "turn_id": "turn-456", "after_seq": 42}
```

The server replays events from seq 43 onwards, then streams live events. The user never misses anything.

### 9.4 Stage Context Manager

Capabilities use the `bus.stage()` context manager to group events into logical phases:

```python
async with stream.stage("planning", source="deep_solve"):
    # Everything emitted here has stage="planning"
    await stream.thinking("Breaking the problem into steps...")
    plan = await self.planner_agent.process(question)
    await stream.progress("Plan created with 3 steps")

# STAGE_START and STAGE_END are emitted automatically
```

The frontend uses these stage boundaries to render progress indicators (e.g., "Planning... Reasoning... Writing...").

---

## 10. Key Files Reference

### Core Abstractions

| Component | File | Purpose |
|---|---|---|
| UnifiedContext | `deeptutor/core/context.py` | Request data envelope |
| StreamEvent | `deeptutor/core/stream.py` | Event format (13 types) |
| StreamBus | `deeptutor/core/stream_bus.py` | Async event fan-out channel |
| BaseCapability | `deeptutor/core/capability_protocol.py` | Capability interface + manifest |
| Tool Protocol | `deeptutor/core/tool_protocol.py` | Tool interface |

### Orchestration

| Component | File | Purpose |
|---|---|---|
| ChatOrchestrator | `deeptutor/runtime/orchestrator.py` | Routes requests to capabilities |
| Capability Registry | `deeptutor/runtime/registry/capability_registry.py` | Discovers and loads capabilities |
| Tool Registry | `deeptutor/runtime/registry/tool_registry.py` | Manages tool lifecycle |
| Built-in Capabilities | `deeptutor/runtime/bootstrap/builtin_capabilities.py` | Registers default capabilities |

### Entry Points

| Component | File | Purpose |
|---|---|---|
| API Server | `deeptutor/api/main.py` | FastAPI app + lifespan |
| WebSocket | `deeptutor/api/routers/unified_ws.py` | Unified WS endpoint |
| Turn Runtime | `deeptutor/services/session/turn_runtime.py` | Turn execution + persistence |

### Agent Pipelines

| Pipeline | File | Agents |
|---|---|---|
| Chat | `deeptutor/agents/chat/agentic_pipeline.py` | Single 4-stage loop |
| Deep Solve | `deeptutor/agents/solve/main_solver.py` | Planner, Solver, Writer |
| Deep Research | `deeptutor/agents/research/research_pipeline.py` | Rephrase, Decompose, Manager, Research, Note, Reporting |
| Deep Question | `deeptutor/agents/question/coordinator.py` | Idea, Generator, Followup |
| Base Agent | `deeptutor/agents/base_agent.py` | Shared agent foundation |

### Individual Agents

| Agent | File | Role |
|---|---|---|
| PlannerAgent | `deeptutor/agents/solve/agents/planner_agent.py` | Breaks problems into steps |
| SolverAgent | `deeptutor/agents/solve/agents/solver_agent.py` | ReAct-based problem solving |
| WriterAgent | `deeptutor/agents/solve/agents/writer_agent.py` | Formats solutions clearly |
| RephraseAgent | `deeptutor/agents/research/agents/rephrase_agent.py` | Optimizes research topics |
| DecomposeAgent | `deeptutor/agents/research/agents/decompose_agent.py` | Breaks topics into subtopics |
| ManagerAgent | `deeptutor/agents/research/agents/manager_agent.py` | Manages research queue |
| ResearchAgent | `deeptutor/agents/research/agents/research_agent.py` | Conducts deep research |
| NoteAgent | `deeptutor/agents/research/agents/note_agent.py` | Summarizes findings |
| ReportingAgent | `deeptutor/agents/research/agents/reporting_agent.py` | Synthesizes final report |
| IdeaAgent | `deeptutor/agents/question/agents/idea_agent.py` | Generates question ideas |
| Generator | `deeptutor/agents/question/agents/generator.py` | Creates full Q&A pairs |
| FollowupAgent | `deeptutor/agents/question/agents/followup_agent.py` | Answers quiz follow-ups |

### Data & Services

| Component | File | Purpose |
|---|---|---|
| SQLite Store | `deeptutor/services/session/sqlite_store.py` | Session/turn persistence |
| Context Builder | `deeptutor/services/session/context_builder.py` | Assembles conversation history |
| Memory Service | `deeptutor/services/memory/service.py` | Long-term user memory |
| RAG Service | `deeptutor/services/rag/service.py` | Document retrieval pipeline |
| LLM Config | `deeptutor/services/llm/config.py` | LLM provider configuration |
| Prompt Manager | `deeptutor/services/prompt/` | Template-based prompt loading |

### Tools

| Tool | File | Purpose |
|---|---|---|
| RAG | `deeptutor/tools/rag_tool.py` | Knowledge base search |
| Web Search | `deeptutor/tools/web_search.py` | Web search integration |
| Code Executor | `deeptutor/tools/code_executor.py` | Sandboxed Python execution |
| Paper Search | `deeptutor/tools/paper_search_tool.py` | arXiv paper search |
| Reason | `deeptutor/tools/reason.py` | Deep reasoning tool |
| Brainstorm | `deeptutor/tools/brainstorm.py` | Idea generation tool |

---

## Glossary

| Term | Definition |
|---|---|
| **Agent** | A specialized AI component with a specific role (e.g., Planner, Researcher). Each agent wraps an LLM with custom prompts and tools. |
| **Capability** | A mode of operation (e.g., Chat, Deep Solve). Each capability orchestrates one or more agents in a specific pipeline pattern. |
| **Orchestrator** | The central router that decides which capability handles a user request. |
| **StreamBus** | The async event channel that carries real-time events from agents to the user. |
| **StreamEvent** | A single event in the stream (thinking, tool call, content, etc.). |
| **UnifiedContext** | The data object carrying everything about a user's request. |
| **Tool** | An action an agent can take — searching the web, running code, retrieving documents, etc. |
| **ReAct Loop** | A "Reasoning + Acting" pattern where the agent thinks, takes an action, observes the result, and repeats. |
| **RAG** | Retrieval-Augmented Generation — enhancing LLM responses with retrieved document content. |
| **Turn** | One user message + the assistant's complete response. |
| **Session** | A conversation containing multiple turns. |
| **Knowledge Base (KB)** | A collection of uploaded documents indexed for retrieval. |
