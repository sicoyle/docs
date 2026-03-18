---
type: docs
title: "Getting Started"
linkTitle: "Getting Started"
weight: 20
description: "How to install Dapr Agents and run your first agent"
aliases:
  - /developing-applications/dapr-agents/dapr-agents-getting-started
---

{{% alert title="Dapr Agents Concepts" color="primary" %}}
If you are looking for an introductory overview of Dapr Agents and want to learn more about basic Dapr Agents terminology, we recommend starting with the [introduction](dapr-agents-introduction.md) and [concepts](dapr-agents-core-concepts.md) sections.
{{% /alert %}}

## Install Dapr CLI

While simple examples in Dapr Agents can be used without the sidecar, the recommended mode is with the Dapr sidecar. To benefit from the full power of Dapr Agents, install the Dapr CLI for running Dapr locally or on Kubernetes for development purposes. For a complete step-by-step guide, follow the  [Dapr CLI installation page]({{% ref install-dapr-cli.md %}}).


Verify the CLI is installed by restarting your terminal/command prompt and running the following:

```bash
dapr -h
```

## Initialize Dapr in Local Mode

{{% alert title="Note" color="info" %}}
Make sure you have [Docker](https://docs.docker.com/get-started/get-docker/) already installed.
{{% /alert %}}

Initialize Dapr locally to set up a self-hosted environment for development. This process fetches and installs the Dapr sidecar binaries, runs essential services as Docker containers, and prepares a default components folder for your application. For detailed steps, see the official [guide on initializing Dapr locally]({{% ref install-dapr-selfhost.md %}}).

![Dapr Initialization](/images/dapr-agents/home_installation_init.png)

To initialize the Dapr control plane containers and create a default configuration file, run:

```bash
dapr init
```

Verify you have container instances with `daprio/dapr`, `openzipkin/zipkin`, and `redis` images running:

```bash
docker ps
```

## Install Python

{{% alert title="Note" color="info" %}}
Make sure you have Python already installed. `Python >=3.10`. For installation instructions, visit the official [Python installation guide](https://www.python.org/downloads/).
{{% /alert %}}

## Prepare your environment

In this getting started guide, you’ll work directly from the [Dapr Agents' quickstarts](https://github.com/dapr/dapr-agents/tree/main/quickstarts). We’ll focus on the **`02_durable_agent_workflow.py`** example, which utilizes the new `workflow` method to start agents purely for workflow orchestration without needing pub/sub or HTTP.

### 1. Clone the repository and examine its content

```bash
git clone https://github.com/dapr/dapr-agents.git
cd dapr-agents/quickstarts/02-durable-agent
```

### 2. Create a virtual environment and install dependencies

From the `02-durable-agent` folder, do:

```bash
python3.10 -m venv .venv

# Activate the virtual environment
# On Windows:
.venv\\Scripts\\activate
# On macOS/Linux:
source .venv/bin/activate

# Install dependencies from the quickstart
pip install -r requirements.txt
```

This installs `dapr-agents` and any additional libraries needed by the examples.

## Understand the application

This example creates an agent that can be triggered directly within workflow orchestration to handle tasks such as weather information. It utilizes Dapr not only for LLM interactions but also to sustain the workflow execution state.

For this updated quickstart you’ll primarily work with:

* `02_durable_agent_workflow.py` – the main durable agent application that starts with the workflow-oriented approach.

Open `02_durable_agent_workflow.py`:

```python
import asyncio

from dapr_agents.llm import DaprChatClient

from dapr_agents import DurableAgent
from dapr_agents.agents.configs import (
    AgentMemoryConfig,
    AgentStateConfig,
)
from dapr_agents.memory import ConversationDaprStateMemory
from dapr_agents.storage.daprstores.stateservice import StateStoreService
from dapr_agents import AgentRunner
from dapr_agents.workflow.utils.core import wait_for_shutdown
from function_tools import slow_weather_func


async def main() -> None:
    weather_agent = DurableAgent(
        name="WeatherAgent",
        role="Weather Assistant",
        instructions=["Help users with weather information"],
        tools=[slow_weather_func],
        # Configure this agent to use Dapr Conversation API.
        llm=DaprChatClient(component_name="llm-provider"),
        # Configure the agent to use Dapr State Store for conversation history.
        memory=AgentMemoryConfig(
            store=ConversationDaprStateMemory(
                store_name="agent-memory",
            )
        ),
        # This is where the execution state is stored.
        state=AgentStateConfig(
            store=StateStoreService(store_name="agent-workflow"),
        ),
    )

    runner = AgentRunner()
    try:
        # workflow() starts the workflow runtime without wiring pub/sub or HTTP routes.
        # The agent is available to be triggered by external Dapr workflows.
        runner.workflow(weather_agent)
        await wait_for_shutdown()
    finally:
        runner.shutdown()


if __name__ == "__main__":
    asyncio.run(main())
```

Highlights the use of `runner.workflow()` to initiate the agent for workflow exclusive operations without external exposure via HTTP or PubSub.

## Run the durable agent with Dapr

From the `02-durable-agent` folder, with your virtual environment activated:

```bash
dapr run --app-id durable-agent-workflow --resources-path resources -- python 02_durable_agent_workflow.py
```

Ensures that the agent operates solely within Dapr's workflow scope:

* The Dapr sidecar is initialized using the components in `resources/`.
* `02_durable_agent_workflow.py` runs dedicating all operations to workflows rather than HTTP or PubSub.

This exemplifies the updated capability of handling internal operations magnifying workflows as a primary interface.