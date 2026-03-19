---
title: "Agentic Patterns"
linkTitle: "Agentic Patterns"
weight: 50
description: "Common design patterns and use cases for building agentic systems"
aliases:
  - /developing-applications/dapr-agents/dapr-agents-patterns
---

Dapr Agents simplify the implementation of agentic systems, from simple augmented LLMs to fully autonomous agents in enterprise environments. The following sections describe several application patterns that can benefit from Dapr Agents.

## Overview

Agentic systems use design patterns such as reflection, tool use, planning, and multi-agent collaboration to achieve better results than simple single-prompt interactions. Rather than thinking of "agent" as a binary classification, it's more useful to think of systems as being agentic to different degrees.

This ranges from simple workflows that prompt a model once, to sophisticated systems that can carry out multiple iterative steps with greater autonomy. 
There are two fundamental architectural approaches:

* **Workflows**: Systems where LLMs and tools are orchestrated through predefined code paths (more prescriptive)
* **Agents**: Systems where LLMs dynamically direct their own processes and tool usage (more autonomous)

On one end, we have predictable workflows with well-defined decision paths and deterministic outcomes. On the other end, we have AI agents that can dynamically direct their own strategies. While fully autonomous agents might seem appealing, workflows often provide better predictability and consistency for well-defined tasks. This aligns with enterprise requirements where reliability and maintainability are crucial.

![Agents Patterns Overview](/images/dapr-agents/agents-patterns-overview.png)

The patterns in this documentation start with the Augmented LLM, then progress through workflow-based approaches that offer predictability and control, before moving toward more autonomous patterns. Each addresses specific use cases and offers different trade-offs between deterministic outcomes and autonomy.

## Durable Agent

Moving to the far end of the agentic spectrum, the Durable Agent pattern represents a shift from workflow-based approaches. Instead of predefined steps, we have an autonomous agent that can plan its own steps and execute them based on its understanding of the goal.

Enterprise applications often need durable execution and reliability that go beyond in-memory capabilities. Dapr's `DurableAgent` class helps you implement autonomous agents with the reliability of workflows, as these agents are backed by Dapr workflows behind the scenes. The `DurableAgent` extends the basic `Agent` class by adding durability to agent execution.

<img src="/images/dapr-agents/agents-stateful-llm.png" width=600 alt="Diagram showing how the durable agent pattern works">

This pattern doesn't just persist message history – it dynamically creates workflows with durable activities for each interaction, where LLM calls and tool executions are stored reliably in Dapr's state stores. This makes it ideal for environments where reliability and durability is critical.

**New Workflow Execution Details**
A recent update enhanced the `DurableAgent` capabilities by including new workflow management methods `runner.workflow(agent)`, allowing for pub/sub and HTTP integration freedom, hence broadening how workflows can be triggered or controlled.

Implementations using `trigger_agent` demonstrated in workflows allow streamlined execution without the need for excessive boilerplate code, enabling agents to be initiated within other app frameworks or self-contained within a single workflow management.

The Durable Agent also enables the "headless agents" approach where autonomous systems operate without direct user interaction. Dapr's Durable Agent exposes REST and Pub/Sub APIs, making it ideal for long-running operations that are triggered by other applications or external events.

**Use Cases:**
- Long-running tasks that may take minutes or days to complete
- Distributed systems running across multiple services
- Customer support handling complex multi-session tickets
- Business processes with LLM intelligence at each step
- Personal assistants handling scheduling and information lookup
- Autonomous background processes triggered by external systems

**Implementation with Dapr Agents:**

```python
import asyncio

from dapr_agents import DurableAgent
from dapr_agents.agents.configs import (
    AgentExecutionConfig,
    AgentMemoryConfig,
    AgentPubSubConfig,
    AgentRegistryConfig,
    AgentStateConfig,
)
from dapr_agents.memory import ConversationDaprStateMemory
from dapr_agents.storage.daprstores.stateservice import StateStoreService
from dapr_agents.workflow.runners import AgentRunner

travel_planner = DurableAgent(
    name="TravelBuddy",
    role="Travel Planner",
    goal="Help users find flights and remember preferences",
    instructions=[
        "Find flights to destinations",
        "Remember user preferences",
        "Provide clear flight info",
    ],
    tools=[search_flights],
    pubsub=AgentPubSubConfig(
        pubsub_name="messagepubsub",
        agent_topic="travel.requests",
        broadcast_topic="travel.broadcast",
    ),
    state=AgentStateConfig(
        store=StateStoreService(store_name="workflowstatestore"),
    ),
    registry=AgentRegistryConfig(
        store=StateStoreService(store_name="registrystatestore"),
        team_name="travel-team",
    ),
    execution=AgentExecutionConfig(max_iterations=3),
    memory=AgentMemoryConfig(
        store=ConversationDaprStateMemory(
            store_name="conversationstore",
            session_id="travel-session",
        )
    ),
)

async def main():
    runner = AgentRunner()
    try:
        result = await runner.run(
            travel_planner,
            payload={"task": "Find weekend flights to Paris"},
        )
        print(result)
    finally:
        runner.shutdown(travel_planner)

asyncio.run(main())
```

The implementation follows Dapr's sidecar architecture model, where all infrastructure concerns are handled by the Dapr runtime:
- **Persistent Memory** - Agent state is stored in Dapr's state store, surviving process crashes
- **Workflow Orchestration** - All agent interactions managed through Dapr's workflow system
- **Service Exposure** - `AgentRunner.serve()` exposes REST endpoints (e.g., `POST /agent/run`) that schedule the agent's `@workflow_entry`
- **Pub/Sub Input/Output** - `AgentRunner.subscribe()` scans the agent for `@message_router` methods and wires the configured topics with schema validation

These options make it easy to process requests asynchronously and integrate seamlessly into larger distributed systems.
