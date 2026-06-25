---
catalog_title: AG-UI
catalog_description: Build interactive chat UIs with streaming, state sync, and agentic actions
catalog_icon: /integrations/assets/ag-ui.png
---

# AG-UI user interface for ADK

<div class="language-support-tag">
  <span class="lst-supported">Supported in ADK</span><span class="lst-python">Python</span><span class="lst-typescript">TypeScript</span><span class="lst-go">Go</span><span class="lst-java">Java</span>
</div>

[AG-UI](https://docs.ag-ui.com/introduction) is the event protocol for
connecting [Agent Development Kit (ADK)](/get-started/about/) agents to
user-facing applications. The `ag-ui-adk` middleware wraps an ADK agent,
translates the agent run into AG-UI events, and exposes those events from a
FastAPI endpoint that any AG-UI-compatible client can consume.

Use AG-UI when your ADK agent needs a live interface. Instead of treating an
agent run as one request and one response, AG-UI models the run as an ordered
stream of messages, tool calls, state updates, lifecycle events, and user
interaction.

## Use cases

AG-UI defines how an agent backend and client exchange interaction events. The
client can be a chat surface, a workflow UI, a dashboard, a mobile app, or a
custom interface that consumes the same event stream.

- **Streaming messages**: Show assistant output as it is generated, including
  message chunks, reasoning summaries, and completion events.
- **Tool calls and actions**: Display backend tool activity, run
  frontend-defined actions, and capture human-in-the-loop decisions.
- **Generative UI**: Render rich, interactive components from tool calls,
  agent state, or structured UI payloads such as A2UI.
- **State synchronization**: Keep the agent and UI working from the same
  application context using snapshots and deltas.
- **Run lifecycle visibility**: Show start, progress, finish, error, interrupt,
  and resume signals for long-running agent work.

AG-UI does not force one UI framework or visual style. It standardizes the event
contract so your frontend can render the interaction in the way your application
needs.

## How AG-UI and A2UI fit

AG-UI and [A2UI](https://a2ui.org/) solve different parts of the
user-interface stack. They are complementary and can be used together:

- **AG-UI:** event protocol for streaming messages, tool calls/actions, state,
  lifecycle, and user interaction between an agent backend and client.
- **A2UI:** declarative UI payload/schema format for structured UI payloads
  that describe components such as cards, forms, tables, and charts.

A common stack is: ADK runs the agent, AG-UI carries the interaction events,
A2UI describes any declarative UI payloads the agent emits, and your client
renders the experience.

## How it works

### Event stream

An AG-UI run is a stream of JSON events. Each event has a `type` discriminator,
and related events share stable identifiers such as `runId`, `messageId`, and
`toolCallId`.

??? example "See an example stream"

    ```json
    [
      {
        "type": "RUN_STARTED",
        "threadId": "thread_1",
        "runId": "run_1"
      },
      {
        "type": "TEXT_MESSAGE_START",
        "messageId": "msg_1",
        "role": "assistant"
      },
      {
        "type": "TEXT_MESSAGE_CONTENT",
        "messageId": "msg_1",
        "delta": "I can check that"
      },
      {
        "type": "TEXT_MESSAGE_CONTENT",
        "messageId": "msg_1",
        "delta": " for you."
      },
      {
        "type": "TEXT_MESSAGE_END",
        "messageId": "msg_1"
      },
      {
        "type": "RUN_FINISHED",
        "threadId": "thread_1",
        "runId": "run_1"
      }
    ]
    ```

The frontend renders incrementally. In this example, the client starts a run
indicator when it receives `RUN_STARTED`, creates an assistant message for
`TEXT_MESSAGE_START`, appends each `TEXT_MESSAGE_CONTENT.delta`, and finalizes
the message when `TEXT_MESSAGE_END` arrives.

### Tool calls and actions

AG-UI uses the same start, stream, end pattern for tool calls. The agent can
announce a tool call, stream arguments, finish the call, and emit the result.

??? example "See an example stream"

    ```json
    [
      {
        "type": "TOOL_CALL_START",
        "toolCallId": "call_1",
        "toolCallName": "get_weather",
        "parentMessageId": "msg_1"
      },
      {
        "type": "TOOL_CALL_ARGS",
        "toolCallId": "call_1",
        "delta": "{\"city\":\"Mountain View\""
      },
      {
        "type": "TOOL_CALL_ARGS",
        "toolCallId": "call_1",
        "delta": ",\"units\":\"fahrenheit\"}"
      },
      {
        "type": "TOOL_CALL_END",
        "toolCallId": "call_1"
      },
      {
        "type": "TOOL_CALL_RESULT",
        "messageId": "tool_msg_1",
        "toolCallId": "call_1",
        "content": "{\"temperature\":72,\"condition\":\"clear\"}",
        "role": "tool"
      }
    ]
    ```

Backend tools still belong in the ADK agent or toolset. Frontend actions belong
in the client, where they can use browser state, user permissions, and local UI
context safely. For example, a frontend-defined action might open a confirmation
dialog, navigate the application, or edit local state after the agent requests
it.

### State sync

State events let the UI and agent share application context. A snapshot gives
the client a complete baseline. A delta communicates a smaller change using
[JSON Patch](https://datatracker.ietf.org/doc/html/rfc6902), such as the
selected item, current form values, generated artifacts, or progress for a
multi-step task.

??? example "See an example stream"

    ```json
    [
      {
        "type": "STATE_SNAPSHOT",
        "snapshot": {
          "selectedCity": "Mountain View",
          "units": "fahrenheit",
          "weatherCardVisible": false
        }
      },
      {
        "type": "STATE_DELTA",
        "delta": [
          {
            "op": "replace",
            "path": "/weatherCardVisible",
            "value": true
          },
          {
            "op": "add",
            "path": "/lastUpdatedBy",
            "value": "agent"
          }
        ]
      }
    ]
    ```

Use state sync for product state that should influence the agent or be updated
by it. Continue to validate user-originated state changes in your application
before applying side effects.

## Get started

### Prerequisites

- Python 3.10+
- A [Gemini API key](https://aistudio.google.com/apikey) for the ADK agent

### Install the middleware

Install ADK, FastAPI, and the AG-UI middleware package:

```bash
pip install google-adk ag-ui-adk fastapi "uvicorn[standard]" python-dotenv
```

### Wrap your ADK agent

An ADK `LlmAgent` does not speak AG-UI by itself. Keep your existing agent code,
then wrap the agent with `ADKAgent` and expose it with
`add_adk_fastapi_endpoint`.

```python title="agent/main.py"
from ag_ui_adk import ADKAgent, add_adk_fastapi_endpoint
from dotenv import load_dotenv
from fastapi import FastAPI
from google.adk.agents import LlmAgent
from google.adk.tools import ToolContext

load_dotenv()


def get_weather(tool_context: ToolContext, location: str) -> dict:
    """Get the weather for a location."""
    return {
        "status": "success",
        "message": f"The weather in {location} is sunny.",
    }


root_agent = LlmAgent(
    model="gemini-flash-latest",
    name="support_agent",
    instruction="Help the user. Use tools when they answer the request.",
    tools=[get_weather],
)

ag_ui_agent = ADKAgent(
    adk_agent=root_agent,
    user_id="demo_user",
    session_timeout_seconds=3600,
    use_in_memory_services=True,
)

app = FastAPI(title="ADK AG-UI Agent")
add_adk_fastapi_endpoint(app, ag_ui_agent, path="/")
```

`ADKAgent` is the middleware boundary. It manages the ADK run, sessions, and
event conversion so the endpoint can stream AG-UI events to a client.

### Run the server

Add your API key, then start the FastAPI app:

```bash
export GOOGLE_API_KEY="your-google-api-key"
uvicorn agent.main:app --host 0.0.0.0 --port 8000
```

The AG-UI endpoint is now available at `http://localhost:8000/`.

### Connect a client

Any AG-UI-compatible client can connect to the endpoint. The client sends user
input, available client-side actions, state, and resume data. The middleware
streams AG-UI events back in order.

[CopilotKit](https://docs.copilotkit.ai/) is one option when you want
ready-made chat components, frontend tools, shared state, and A2UI rendering.
Use `npx copilotkit create -f adk` if you want a full-stack starter that
already points a frontend at an ADK AG-UI server.

## Implementation shape

The middleware handles the ADK-to-AG-UI translation, but the boundary is still
straightforward:

1. Accept client input for the conversation, available frontend actions, current
   state, and resume data.
2. Run the ADK agent.
3. Convert the agent's messages, tool activity, state changes, and run
   lifecycle into AG-UI events.
4. Stream those events to the client in order.
5. Accept action results or user decisions from the client and continue the run
   when needed.

This keeps the boundary clear: ADK owns agent execution, AG-UI owns the event
contract, and your client owns rendering and user-side side effects.

## Additional resources

- [AG-UI overview](https://docs.ag-ui.com/introduction)
- [AG-UI events](https://docs.ag-ui.com/concepts/events)
- [AG-UI state management](https://docs.ag-ui.com/concepts/state)
- [AG-UI tools](https://docs.ag-ui.com/concepts/tools)
- [A2UI documentation](https://a2ui.org/)
- [CopilotKit documentation](https://docs.copilotkit.ai/)
- [CopilotKit frontend tools](https://docs.copilotkit.ai/google-adk/frontend-tools)
