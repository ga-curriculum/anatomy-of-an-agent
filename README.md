# Anatomy of an Agent
**Session Time:** 120 minutes  
**Format:** Instructor-Led  

---

## Prerequisites

- Completion of Session 1: Introduction to the Agentic AI Era  
- Understanding that agents differ from chatbots by executing actions and persisting goals  
- Familiarity with the perceive–decide–act framing introduced in Session 1  

---

## Learning Objectives

By the end of this session, students will be able to:

1. **Analyze** how data, reasoning, and actions flow through a complete agent workflow.
2. **Explain** the role of reasoning engines, memory, and tools in enabling agent behavior.
3. **Analyze** how feedback from the environment improves the agent's performance over time and shapes future decisions.

> This session provides the foundational mental model for how agents are built and function. Every subsequent session builds directly on the component map introduced here.

---

## Session Breakdown

| # | Segment | Duration |
|---|---------|----------|
| 1 | The Agent as a System: Data and Action Flow | 15 min |
| 2 | Core Components: Reasoning Engine, Memory, and Tools | 15 min |
| 3 | Feedback Loops and Environmental Learning | 15 min |
| 4 | Break | 10 min |
| 5 | Lab | 60 min |
| 6 | Wrap-Up & Reflection | 5 min |
|   | **Total** | **120 min** |

---

## 1. The Agent as a System: Data and Action Flow (15 min)

<img width="496" height="269" alt="image" src="https://git.generalassemb.ly/user-attachments/assets/2076a92f-17cc-45ff-a5b4-d694839b1d90" />

### Thinking in Systems

An agent is not a feature — it is a **system**. To understand how agents work, we need to stop thinking about them as a single model that generates responses, and start thinking about them as a pipeline of interconnected components through which data, reasoning, and actions flow.

At the highest level, every agent follows the same data flow:

```
Input → Context Assembly → Reasoning → Decision → Action → Output → Feedback
```

Each of these stages has a responsible component, and understanding which component does what is the key to designing, debugging, and scaling agents in production.

### The Complete Data Flow

| Stage | What Happens | Responsible Component |
|---|---|---|
| **Input** | User message, trigger event, or API call arrives | Environment / Interface |
| **Context Assembly** | Memory is retrieved and combined with the input | Memory System |
| **Reasoning** | The LLM interprets the situation and decides what to do | Reasoning Engine (LLM) |
| **Decision** | A tool call, response, or plan is selected | Reasoning Engine (LLM) |
| **Action** | A tool is executed — API call, database query, file write | Tool Layer / Runtime |
| **Output** | Result is returned to the user or passed to the next step | Interface / Orchestrator |
| **Feedback** | The result is observed and stored for future improvement | Memory + Environment |

This flow is the anatomical map of every agent, from the simplest GPT Builder configuration to a full enterprise orchestration system.

<img width="370" height="272" alt="image" src="https://git.generalassemb.ly/user-attachments/assets/3765ed98-758a-47ba-ab17-ea91516537c1" />


### Why the Flow Matters for Production

Failures in production rarely come from a single broken component. They come from **broken handoffs between components**: memory that doesn't retrieve the right context, tools that return unexpected formats, feedback that never gets stored. Understanding the flow is what makes it possible to diagnose and fix these failures systematically.

---

## 2. Core Components: Reasoning Engine, Memory, and Tools (15 min)

### 2.1 The Reasoning Engine (LLM)

The reasoning engine is the cognitive core of the agent. In modern agentic systems, this is always an LLM — but the LLM's role is more constrained and specific than most people assume.

<img width="472" height="478" alt="image" src="https://git.generalassemb.ly/user-attachments/assets/329b2d9e-1916-4d4d-a7a9-4fdffee939e1" />


This separation of concerns is deliberate and important. Conflating the LLM with the agent leads to systems that are fragile, unauditable, and hard to debug.

**The LLM receives a structured prompt at every step:**

```
[System Prompt]       ← defines the agent's role, goals, and constraints
[Memory / Context]    ← relevant past interactions and retrieved knowledge
[Current Input]       ← the user's message or the result of the last tool call
[Available Tools]     ← schema descriptions of tools the agent can call
```

The LLM's output is either a **tool call** (structured JSON) or a **final response** (natural language). Nothing else.

### 2.2 Memory

Memory is what separates a stateful agent from a stateless chatbot. Without memory, every interaction starts from zero — the agent has no continuity, no context, and no ability to learn from past results.

There are two fundamentally different types of memory in agentic systems:

#### Short-Term Memory (Contextual)

Short-term memory is the **current conversation window** — everything the agent can "see" in its active context at a given moment.

| Property | Value |
|---|---|
| Scope | Current session only |
| Storage | In the LLM's context window (tokens) |
| Retrieval | Automatic — everything is present |
| Persistence | Lost when the session ends |
| Limit | Bounded by the model's context window size |

Short-term memory is the simplest form of memory, but it has a critical limitation: context windows are finite. Long conversations, large documents, and multi-session workflows quickly exceed what can fit in a single prompt.

#### Long-Term Memory (Persistent)

Long-term memory stores information outside the LLM, in external systems that can be queried selectively.

| Property | Value |
|---|---|
| Scope | Cross-session, persistent |
| Storage | Vector databases, key-value stores, SQL databases |
| Retrieval | Query-based (semantic search, exact lookup) |
| Persistence | Survives session end |
| Limit | Effectively unbounded |

The most common implementation uses **vector databases** (e.g., Pinecone, Weaviate, ChromaDB): information is stored as embeddings and retrieved by semantic similarity to the current query. This allows the agent to retrieve relevant memories even when the exact keywords don't match.

#### Memory in the Data Flow

Memory plays two roles in the data flow:

1. **Before reasoning:** The memory system retrieves relevant context and injects it into the prompt assembled for the LLM.
2. **After action:** The result of the agent's action is written back to memory for future retrieval.

```
[Input arrives]
      ↓
[Memory retrieval] → relevant past context injected into prompt
      ↓
[LLM reasons and acts]
      ↓
[Memory write] → result stored for future sessions
```

#### Memory and Privacy

Long-term memory introduces data governance responsibilities that short-term memory does not. Stored user data must be treated with the same rigor as any other personal data: access controls, retention policies, and auditability of what was stored and when. This will be addressed in depth in Session 10 (Integration with Business Data) and Session 13 (Risks and Security).

### 2.3 Tools

Tools are how agents act on the world. Without tools, an agent can only produce text — it cannot query a database, send an email, update a record, or trigger a workflow. Tools are what transform an advanced chatbot into a system that creates real outcomes.

#### What a Tool Is

A tool is any **external function or API that the agent runtime can call on the LLM's instruction**. From the agent's perspective, a tool is defined by three things:

- **Name:** how the LLM refers to it
- **Description:** what it does (written in natural language, for the LLM to understand)
- **Parameters:** the inputs it requires (typed schema, for structured generation)

```json
{
  "name": "get_order_status",
  "description": "Retrieves the current status of a customer order from the warehouse system.",
  "parameters": {
    "order_id": { "type": "string", "description": "The unique order identifier" }
  }
}
```

The LLM reads this schema and decides whether and how to call the tool based on the current situation and goal.


<img width="710" height="420" alt="image" src="https://git.generalassemb.ly/user-attachments/assets/812c4df4-52e6-45de-9e4b-9c4a4d9197b7" />


#### Tool Safety and Governance

Not all tools carry the same risk. A tool that reads data is far safer than a tool that writes or deletes it. A tool that sends a message to one person is far safer than one that triggers a mass communication. Designing the tool layer requires explicit decisions about:

- **Scope:** what the tool is allowed to access or modify
- **Permissions:** what credentials are used and who authorizes them
- **Reversibility:** can the action be undone if the agent made an error?
- **Human-in-the-loop:** should high-risk tool calls require human approval before execution?

These questions will be central in Sessions 8 (Tools and Actions) and 13 (Risks and Security).

---

## 3. Feedback Loops and Environmental Learning (15 min)

### The Environment as Teacher

Every action an agent takes produces a result — and that result is a signal. The agent's environment is not just a passive backdrop; it is the source of the feedback that makes agents adaptive and improvable over time.

This feedback operates at two levels:

**1. Immediate feedback (within the session):**  
The result of a tool call is returned to the agent and added to its context. The agent observes the outcome and adjusts its next action accordingly. If `get_order_status` returns "not found," the agent doesn't repeat the call — it reasons about what to do instead.

**2. Accumulated feedback (across sessions):**  
Patterns from many interactions — which prompts led to successful outcomes, which tool calls failed frequently, which types of requests were escalated to humans — are stored and used to improve the agent's behavior over time.

### Feedback in the Agent Loop

<img width="484" height="474" alt="image" src="https://git.generalassemb.ly/user-attachments/assets/2bd4880c-e2dc-4e9f-bcdc-e54d3bb0ecb9" />

### Three Mechanisms of Environmental Improvement

#### Mechanism 1: In-Context Adaptation

The most immediate form of learning. When a tool returns an error or unexpected result, the agent's reasoning trace incorporates that information and selects a different action. No external system is required — this happens entirely within the current session's context window.

**Example:**
```
ACT:     → Tool call: flight_search(destination="GRU", date="2024-03-15")
OBSERVE: "No flights available on this date."
THINK:   "The direct date failed. I'll try adjacent dates."
ACT:     → Tool call: flight_search(destination="GRU", date_range="2024-03-14 to 2024-03-16")
```

#### Mechanism 2: Memory-Based Improvement

When long-term memory is implemented, the agent can retrieve knowledge from past interactions to inform current decisions. A support agent that handled a similar complaint last month can retrieve the resolution strategy that worked, without the user having to re-explain the context.

This is not learning in the machine learning sense — the model's weights don't change. But the agent's effective behavior improves because it has access to a growing body of relevant prior experience.

#### Mechanism 3: Prompt and Configuration Adjustment

At the system level, operators monitor agent performance through logs and analytics, identify failure patterns, and update the agent's system prompt, tool configurations, or memory retrieval strategies. This is the human-in-the-loop equivalent of fine-tuning — targeted, interpretable, and reversible.

**Example failure pattern → improvement cycle:**

```
Observed pattern:   Agent frequently fails to escalate urgent requests
Root cause:         System prompt lacks explicit escalation criteria
Adjustment:         Add rule to system prompt: "If the customer mentions a deadline 
                    within 24 hours, always call the escalation_trigger tool"
Result:             Escalation rate on urgent tickets improves
```

### Why Feedback Loops Matter for Production Reliability

An agent deployed without a feedback loop is a static system. It will make the same errors indefinitely, with no mechanism for improvement. Production-grade agents require:

- **Logging** of every reasoning step and tool call
- **Evaluation** of outcomes against defined success metrics
- **Iteration** on prompts, tools, and memory based on observed failures

This is the foundation of Sessions 14 (Building Your First Full Agent) and 15 (Monitoring and Orchestration).

---

## 4. Break (10 min)

---

## 5. Lab (60 min)

*Lab details in the Lab Guide document.*

---

## 6. Wrap-Up & Reflection (5 min)

### Key Takeaways

- An Agent Is a System, Not a Model
- The LLM Thinks. The Runtime Acts.
- Feedback Is What Makes Agents Adaptive

> Next session: **How Agents Work** 

---

## Resources

- Russell & Norvig — *Artificial Intelligence: A Modern Approach*, Chapter 2 (Intelligent Agents)
- [LangChain Memory Documentation](https://docs.langchain.com/docs/components/memory)
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
- [Pinecone — What is a Vector Database?](https://www.pinecone.io/learn/vector-database/)
- Chase, H. — *LangChain Blog: Agents and Tools* (2023)
