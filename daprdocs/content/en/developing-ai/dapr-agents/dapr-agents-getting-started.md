---
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

In this getting started guide, you’ll work directly from the [Dapr Agents' quickstarts](https://github.com/dapr/dapr-agents/tree/main/quickstarts). We’ll focus on illustrating using the new `trigger_agent` and `call_agent` utilities.

### 1. Clone the repository and examine its content

```bash
git clone https://github.com/dapr/dapr-agents.git
cd dapr-agents/quickstarts/02-durable-agent-workflow
```

### 2. Create a virtual environment and install dependencies

From the `02-durable-agent-workflow` folder, do:

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

### 3. Understand the updated functionalities

Learn how to integrate `trigger_agent` within main application logic, demonstrating less verbose agent control by letting Dapr handle execution:

```python
from dapr_agents import trigger_agent

def main() -> None:
    result = trigger_agent(
        "WeatherAgent",
        input={"task": "What is the weather in London?"},
        app_id="weather-agent",
    )
    print(f"Result: {result}")

if __name__ == "__main__":
    main()
```

This function trigger runs without needing full workflow boilerplate code, blocking until execution completes.

### 4. Running and testing

Ensure you have the necessary components running through Dapr as per instructions, and execute the `.py` script as follows to see the `trigger_agent` in action and print results:

```bash
python 02_durable_agent_trigger.py
```

Follow similar steps for `call_agent` in orchestrating agent-run frameworks autonomously by handling more distributed, complex scenarios without excessive setup overhead.