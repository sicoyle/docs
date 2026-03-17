# New Features in Dapr-Agents

This documentation covers the new features introduced in the recent update of **dapr-agents**, referenced in PR [#526](https://github.com/dapr/dapr-agents/pull/526).

## New Features

### 1. `runner.workflow(agent)`
This method allows starting an agent's workflow runtime without exposing it on pub/sub or HTTP routes. It is designed for agents triggered by external Dapr workflows or the Dapr Workflow API, rather than pub/sub messages or HTTP requests.

Usage:
```python
runner = AgentRunner()
runner.workflow(agent)
await wait_for_shutdown()
```

### 2. `trigger_agent`
This standalone blocking helper method triggers agents without workflow boilerplate code, handling the WorkflowRuntime lifecycle internally.

Usage:
```python
result = trigger_agent("WeatherAgent", input={"task": "What's the weather?"}, app_id="weather-agent")
```

### 3. `call_agent`
This yieldable helper allows calling a DurableAgent's workflow as a child workflow from within another Dapr workflow.

Usage:
```python
result = yield call_agent(ctx, "WeatherAgent", input={...}, app_id="weather-agent")
```

These methods utilize the Dapr registry to resolve fields and handle timeouts effectively, improving the automation and resilience of your Dapr workflows with agents.