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

Dapr Agents provides two agent types, each designed for different use cases:

### Agent
The standard `Agent` class is a conversational agent that manages tool calls and conversations using a language model. It provides, synchronous execution with built-in conversation memory.

```python
@tool
def my_weather_func() -> str:
    """Get current weather."""
    return "It\'s 72°F and sunny"

async def main():
    weather_agent = Agent(
        name="WeatherAgent",
        role="Weather Assistant",
        goal="Provide timely weather updates across cities",
        instructions=["Help users with weather information"],
        tools=[my_weather_func],
        memory = AgentMemoryConfig(
            store=ConversationDaprStateMemory(
                store_name="historystore",
                session_id="some-id",
            )
        ),
    )

    response1 = await weather_agent.run("What\'s the weather?")
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
        memory = AgentMemoryConfig(
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
        print(itinerary)
    finally:
        runner.shutdown(travel_planner)

```
This example demonstrates creating a workflow-backed agent that runs autonomously in the background. The `AgentRunner` schedules the workflow for you, waits for completion, and ensures the agent can be triggered once yet continue execution across restarts.

**Key Characteristics:**
- Workflow-based execution using Dapr Workflows
- Persistent workflow state management across sessions and failures
- Automatic retry and recovery mechanisms
- Deterministic execution with checkpointing
- Built-in message routing and agent communication
- `AgentRunner` modes for DurableAgents: ad-hoc runs (`runner.run(...)`), pub/sub subscriptions (`runner.subscribe(...)`), and FastAPI services (`runner.serve(...)`)
- Supports complex orchestration patterns and multi-agent collaboration

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

- `DurableAgent` (Workflow-backed): Interaction is asynchronous—you trigger the agent once, and it runs autonomously in the background until completion. The conversation state and the execution are persisted  and can resume across failures or restarts.


## Core Agent Features
An agentic system is a distributed system that requires a variety of behaviors and supporting infrastructure.

### LLM Integration

Dapr Agents provides a unified interface to connect with LLM inference APIs. This abstraction allows developers to seamlessly integrate their agents with cutting-edge language models for reasoning and decision-making. The framework includes multiple LLM clients for different providers and modalities:

- `DaprChatClient`: Unified API for LLM interactions via Dapr\'s Conversation API with built-in security (scopes, secrets, PII obfuscation), resiliency (timeouts, retries, circuit breakers), and observability via OpenTelemetry & Prometheus
- `OpenAIChatClient`: Full spectrum support for OpenAI models including chat, embeddings, and audio
- `HFHubChatClient`: For Hugging Face models supporting both chat and embeddings
- `NVIDIAChatClient`: For NVIDIA AI Foundation models supporting local inference and chat
- `ElevenLabs`: Support for speech and voice capabilities

### Prompt Flexibility

Dapr Agents supports flexible prompt templates to shape agent behavior and reasoning. Users can define placeholders within prompts, enabling dynamic input of context for inference calls. By leveraging prompt formatting with [Jinja templates](https://jinja.palletsprojects.com/en/stable/templates/) and Python f-string formatting, users can include loops, conditions, and variables, providing precise control over the structure and content of prompts. This flexibility ensures that LLM responses are tailored to the task at hand, offering modularity and adaptability for diverse use cases.

### Structured Outputs

Agents in Dapr Agents leverage structured output capabilities, such as [OpenAI’s Function Calling](https://platform.openai.com/docs/guides/function-calling), to generate predictable and reliable results. These outputs follow [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/release-notes.html) and [OpenAPI Specification v3.1.0](https://github.com/OAI/OpenAPI-Specification) standards, enabling easy interoperability and tool integration.

```python
# Define our data model
class Dog(BaseModel):
    name: str
    breed: str
    reason: str

# Initialize the chat client
llm = OpenAIChatClient()

# Get structured response
response = llm.generate(
    messages=[UserMessage("One famous dog in history.")], response_format=Dog
)

print(json.dumps(response.model_dump(), indent=2))
```
This demonstrates how LLMs generate structured data according to a schema. The Pydantic model (Dog) specifies the exact structure and data types expected, while the response_format parameter instructs the LLM to return data matching the model, ensuring consistent and predictable outputs for downstream processing.


### Tool Calling

Tool Calling is an essential pattern in autonomous agent design, allowing AI agents to interact dynamically with external tools based on user input. Agents dynamically select the appropriate tool for a given task, using LLMs to analyze requirements and choose the best action.

```python
@tool(args_model=GetWeatherSchema)
def get_weather(location: str) -> str:
    """Get weather information based on location."""
    import random
    temperature = random.randint(60, 80)
    return f"{location}: {temperature}F."
```

Each tool has a descriptive docstring that helps the LLM understand when to use it. The `@tool` decorator marks a function as a tool, while the Pydantic model (`GetWeatherSchema`) defines input parameters for structured validation.

![Tool Call Flow](/images/dapr-agents/concepts_agents_toolcall_flow.png)

1. The user submits a query specifying a task and the available tools.
2. The LLM analyzes the query and selects the right tool for the task.
3. The LLM provides a structured JSON output containing the tool\'s unique ID, name, and arguments.
4. The AI agent parses the JSON, executes the tool with the provided arguments, and sends the results back as a tool message.
5. The LLM then summarizes the tool\'s execution results within the user\'s context to deliver a comprehensive final response.

This is supported directly through LLM parametric knowledge and enhanced by [Function Calling](https://platform.openai.com/docs/guides/function-calling), ensuring tools are invoked efficiently and accurately.


### MCP Support
Dapr Agents includes built-in support for the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/), enabling agents to dynamically discover and invoke external tools through a standardized interface.