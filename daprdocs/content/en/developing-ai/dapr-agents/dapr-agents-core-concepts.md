---
type: docs
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

Dapr Agents provides two primary agent types, each designed for different use cases:

### Agent
The standard `Agent` class manages tool calls and conversations using a language model, supporting synchronous execution with built-in memory.

### Durable Agent

The `DurableAgent` class extends the standard Agent for long-running, fault-tolerant, and durable execution using Dapr Workflows. It provides persistent state management, automatic retry mechanisms, and reliable execution across failures.

## AgentRunner.workflow()

The `workflow()` method in `AgentRunner` allows an agent's workflow runtime to start without setting up pub/sub or HTTP routes, making it suitable for workflows triggered by Dapr Workflow API or from within other workflows.

Example of using `workflow()`:

```python
from dapr_agents import DurableAgent, AgentRunner
from dapr_agents.workflow.utils.core import wait_for_shutdown

async def main():
    agent = DurableAgent(
        name="ExampleAgent",
        role="Example Role",
        instructions=["Perform specific tasks"],
        # No pub/sub config, uses workflows directly.
    )
    runner = AgentRunner()
    runner.workflow(agent)  # Start the agent's workflow runtime
    await wait_for_shutdown()
```

## Helper Functions: call_agent and trigger_agent

### call_agent

`call_agent` triggers a DurableAgent's workflow as a child workflow from within another Dapr workflow, facilitating interactions between workflows.

```python
from dapr.ext.workflow import DaprWorkflowContext
from dapr_agents import call_agent

async def example_workflow(ctx: DaprWorkflowContext):
    result = yield call_agent(ctx, "ExampleAgent", input={"task": "Describe task"})
    return result
```

### trigger_agent

`trigger_agent` provides a standalone way to trigger durable agents and wait for their completion without additional workflow setup.

```python
from dapr_agents import trigger_agent

def main() -> None:
    result = trigger_agent(
        "WeatherAgent",
        input={"task": "What is the weather forecast?"},
        app_id="weather-agent",
    )
    print(f"Result: {result}")

if __name__ == "__main__":
    main()
```

These helpers streamline interactions with DurableAgents, abstracting the complexity of workflow runtime and lifecycle management from the developer, allowing for cleaner and more maintainable code.