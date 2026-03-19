---
title: "Core Concepts"
linkTitle: "Core Concepts"
weight: 40
description: "Learn about the core concepts of Dapr Agents"
aliases:
  - /developing-applications/dapr-agents/dapr-agents-core-concepts
---

Dapr Agents provides a structured way to build and orchestrate applications that use LLMs without getting bogged down in infrastructure details. The primary goal is to enable AI development by abstracting away the complexities of working with LLMs, tools, memory management, and distributed systems, allowing developers to focus on the business logic of their AI applications. Agents in this framework are the fundamental building blocks.

## Agents

Agents are autonomous units powered by Large Language Models (LLMs), designed to execute tasks, reason through problems, and collaborate within workflows. Acting as intelligent building blocks, agents combine reasoning with tool integration, memory, and collaboration features to get to the desired outcome.

![Concepts Agents](/images/dapr-agents/concepts-agents.png)

Dapr Agents provides two agent types, each designed for different use cases:

### Agent
The standard `Agent` class is a conversational agent that manages tool calls and conversations using a language model. It provides, synchronous execution with built-in conversation memory.

```python
@tool
def my_weather_func() -> str:
    """Get current weather."""
    return "It's 72°F and sunny"

async def main():
    weather_agent = Agent(
        name="WeatherAgent",
        role="Weather Assistant",
        goal="Provide timely weather updates across cities",
        instructions=["Help users with weather information"],
        tools=[my_weather_func],
        memory=AgentMemoryConfig(
            store=ConversationDaprStateMemory(
                store_name="historystore",
                session_id="some-id",
            )
        ),
    )

    response1 = await weather_agent.run("What's the weather?")
    response2 = await weather_agent.run("How about now?")
```

This example shows how to create a simple agent with tool integration. The agent processes queries synchronously and maintains conversation context across multiple interactions using Dapr State Store API.

### Durable Agent

The `DurableAgent` class is a workflow-based agent that extends the standard Agent with Dapr Workflows for long-running, fault-tolerant, and durable execution. It provides persistent state management, automatic retry mechanisms, and deterministic execution across failures.

```python
from dapr_agents.workflow.runners import AgentRunner

async def main():
    travel_planner = DurableAgent(
        name="TravelBuddy",
        role="Travel Planner",
        goal="Help users find flights and remember preferences",
        instructions=["Help users find flights and remember preferences"],
        tools=[search_flights],
        memory=AgentMemoryConfig(
            store=ConversationDaprStateMemory(
                store_name="conversationstore",
                session_id="travel-session",
            )
        )
    )

    runner = AgentRunner()

    try:
        itinerary = await runner.run(
            travel_planner,
            payload={"task": "Plan a 3-day trip to Paris"},
        )
    finally:
        runner.shutdown(travel_planner)

```
This example demonstrates creating a workflow-backed agent that runs autonomously in the background. The `AgentRunner` schedules the workflow for you, waits for completion, and ensures the agent can be triggered once to continue execution across restarts.

**Key Characteristics:**
- Workflow-based execution using Dapr Workflows
- Persistent workflow state management across sessions and failures
- Automatic retry and recovery mechanisms
- Deterministic execution with checkpointing
- Built-in message routing and agent communication

### New Workflow Execution Method
The recent enhancement introduces the ability to start the workflow runtime without wiring pub/sub or HTTP routes using the `runner.workflow(agent)` method. This allows agents to be triggered by workflows or the Dapr Workflow API directly. Additionally, `trigger_agent` and `call_agent` methods enable triggering agents without full workflow integration, improving modularity and reuse.

**When to use:**
- Multi-step workflows that span time or systems
- Tasks requiring guaranteed progress tracking and state persistence
- Scenarios where operations may pause, fail, or need recovery without data loss
- Complex agent orchestration and multi-agent collaboration
- Production systems requiring fault tolerance and scalability

In Summary:

| Agent Type      | Memory Type             | Execution | Interaction Mode         |
|-----------------|-------------------------|-----------|--------------------------|
| `Agent`         | In-memory or Persistent | Ephemeral | Embedded                 |
| `Durable Agent` | Persistent | Durable   | PubSub / HTTP / Embedded |


- Regular `Agent`: Interaction is synchronous—you send conversational prompts and receive responses immediately. The conversation can be stored in memory or persisted, but the execution is ephemeral and does not survive restarts.

- `DurableAgent` (Workflow-backed): Interaction is asynchronous—you trigger the agent once, and it runs autonomously in the background until completion. The conversation state and the execution is persisted  and can resume across failures or restarts.