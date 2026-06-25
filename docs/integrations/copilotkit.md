---
catalog_title: CopilotKit
catalog_description: Build generative experiences for web, mobile, Slack, Teams, and any messaging platform
catalog_icon: /integrations/assets/copilotkit.png
catalog_tags: ["frontend"]
---

# CopilotKit frontend for ADK

<div class="language-support-tag">
  <span class="lst-supported">Supported in ADK</span><span class="lst-python">Python</span><span class="lst-typescript">TypeScript</span>
</div>

[CopilotKit](https://docs.copilotkit.ai/) is the frontend stack for building
[ADK (Agent Development Kit)](/get-started/about/) agent experiences for
[web](https://docs.copilotkit.ai/google-adk/quickstart),
[mobile](https://docs.copilotkit.ai/google-adk/react-native),
[Slack](https://docs.copilotkit.ai/google-adk/slack),
[Microsoft Teams](https://docs.copilotkit.ai/google-adk/microsoft-teams), and
other messaging platforms. It connects your application to the agent through
[AG-UI](/integrations/ag-ui/), then adds chat, browser-side actions,
synchronized application state, and generative UI inside the product
experience.

The fastest path is to let CopilotKit create the full-stack starter. The
scaffold sets up the frontend app, CopilotKit runtime route, local ADK agent
server, environment files, dependency scripts, and a starter UI so you can focus
on the agent behavior and application experience.

## Use cases

- **Agentic chat**: Add a chat surface backed by an ADK agent.
- **[Generative UI](https://docs.copilotkit.ai/google-adk/generative-ui)**:
  Render rich, user-interactive components with
  [controlled generative UI](https://docs.copilotkit.ai/google-adk/generative-ui/components-as-tools),
  [A2UI](https://docs.copilotkit.ai/google-adk/generative-ui/a2ui),
  [MCP Apps](https://docs.copilotkit.ai/google-adk/generative-ui/mcp-apps),
  your own components, and more.
- **[Frontend tools](https://docs.copilotkit.ai/google-adk/frontend-tools)**:
  Let the agent call browser-side actions and update local UI state.
- **[Shared state](https://docs.copilotkit.ai/google-adk/shared-state)**:
  Keep the agent and application working from the same state.

## Prerequisites

- Node.js 20+
- Python 3.9+
- A [Google API key](https://aistudio.google.com/apikey) for the ADK agent

## Create a new full-stack ADK agent

Use the CopilotKit scaffold to create the frontend app, ADK agent, local
development scripts, and runtime route in one project:

```bash
npx copilotkit create -f adk
```

Follow the CLI prompts. When the project is created, install dependencies and
set your Google API key:

```bash
npm install
export GOOGLE_API_KEY="your-google-api-key"
npm run dev
```

The starter runs the web app and the ADK agent server together. Open the local
URL printed by the frontend server and chat with the generated agent.

The scaffold includes a minimal CopilotKit generated UI example. Ask:

```text
Get the weather in San Francisco.
```

The app renders a weather card instead of returning only text. This confirms the
starter can render agent-driven UI in the frontend.

## Add CopilotKit to an existing project

If you already have an ADK project, give your coding agent the current
CopilotKit setup instructions and let it make the small wiring changes for you.
From your project root, install the
[CopilotKit skills](https://docs.copilotkit.ai/google-adk/build-with-agents):

```bash
npx skills add CopilotKit/CopilotKit/skills -y
```

Then paste this prompt into your coding agent:

```text
Help me add CopilotKit to this ADK project. Use the copilotkit-setup skill.
Connect my existing ADK agent to CopilotKit through AG-UI, add the runtime route
and chat UI, and keep the implementation minimal.
```

## Add generative UI with A2UI

Use [A2UI](https://docs.copilotkit.ai/google-adk/generative-ui/a2ui) when the
agent should generate structured UI that CopilotKit renders for the user.
Register the allowed UI pieces as an A2UI
[catalog](https://a2ui.org/concepts/catalogs/) with React
[renderers](https://a2ui.org/reference/renderers/), then enable A2UI in the
CopilotKit runtime.

First, enable A2UI in the runtime route the scaffold created:

```ts title="src/app/api/copilotkit/[[...slug]]/route.ts"
const agentUrl = process.env.AGENT_URL || "http://localhost:8000";

const runtime = new CopilotRuntime({
  agents: {
    "a2ui-agent": new HttpAgent({
      url: `${agentUrl}/a2ui_agent`,
    }),
  },
  a2ui: {},
});
```

Then define the components the agent is allowed to use and the React renderers
that paint them:

```tsx title="src/app/a2ui/catalog.tsx"
import { createCatalog } from "@copilotkit/a2ui-renderer";
import type {
  CatalogDefinitions,
  CatalogRenderers,
} from "@copilotkit/a2ui-renderer";
import { z } from "zod";

const definitions = {
  Metric: {
    description: "A key/value KPI tile.",
    props: z.object({
      label: z.string(),
      value: z.string(),
    }),
  },
} satisfies CatalogDefinitions;

const renderers: CatalogRenderers<typeof definitions> = {
  Metric: ({ props }) => (
    <div className="rounded-lg border p-4">
      <div className="text-sm text-gray-500">{props.label}</div>
      <div className="text-2xl font-semibold">{props.value}</div>
    </div>
  ),
};

export const catalog = createCatalog(definitions, renderers, {
  catalogId: "adk-a2ui-catalog",
  includeBasicCatalog: true,
});
```

Register the catalog where the scaffold mounts `CopilotKit`:

```tsx title="src/app/page.tsx"
<CopilotKit
  runtimeUrl="/api/copilotkit"
  agent="a2ui-agent"
  a2ui={{ catalog }}
>
  <CopilotChat agentId="a2ui-agent" />
</CopilotKit>
```

With `a2ui: {}`, the runtime applies A2UI middleware to the registered agent.
The agent can call `generate_a2ui`, and the frontend renders the returned
`a2ui_operations`:

```python title="agent.py"
from google.adk.agents import LlmAgent

root_agent = LlmAgent(
    model="gemini-flash-latest",
    name="a2ui_agent",
    instruction=(
        "When a visual answer would help, call generate_a2ui. "
        "Use the available catalog components, then reply "
        "with one short sentence."
    ),
)
```

For a longer version with A2UI patterns, catalog definitions, and renderer
setup, see the
[CopilotKit A2UI docs](https://docs.copilotkit.ai/google-adk/generative-ui/a2ui).

## Use with agent

If you already have an ADK agent, keep it. CopilotKit only needs an AG-UI
endpoint and a runtime route that points to it.

### Expose your ADK agent over AG-UI

Wrap the existing `LlmAgent` with the `ag-ui-adk` middleware:

```python title="agent/main.py"
from ag_ui_adk import ADKAgent, add_adk_fastapi_endpoint
from fastapi import FastAPI
from google.adk.agents import LlmAgent

root_agent = LlmAgent(
    model="gemini-flash-latest",
    name="support_agent",
    instruction="Help the user with their support request.",
    tools=[],
)

ag_ui_agent = ADKAgent(
    adk_agent=root_agent,
    user_id="demo_user",
    session_timeout_seconds=3600,
    use_in_memory_services=True,
)

app = FastAPI(title="ADK Agent")
add_adk_fastapi_endpoint(app, ag_ui_agent, path="/")
```

### Point CopilotKit at the endpoint

In the scaffolded runtime route, keep the generated route handlers and set the
agent URL:

```ts title="src/app/api/copilotkit/[[...slug]]/route.ts"
import { HttpAgent } from "@ag-ui/client";
import { CopilotRuntime, createCopilotEndpoint } from "@copilotkit/runtime/v2";

const runtime = new CopilotRuntime({
  agents: {
    default: new HttpAgent({
      url: process.env.AGENT_URL || "http://localhost:8000/",
    }),
  },
});

const app = createCopilotEndpoint({
  runtime,
  basePath: "/api/copilotkit",
});
```

Start the ADK agent server and the frontend app. Set `AGENT_URL` if the agent
server is not running at `http://localhost:8000/`.

The starter uses Next.js for the quickest runnable path, but the pattern is not
limited to one UI surface. Use the generated project as a reference for any
frontend that needs to connect users to the same ADK agent.

## Additional resources

- [CopilotKit documentation](https://docs.copilotkit.ai/)
- [Web quickstart](https://docs.copilotkit.ai/google-adk/quickstart)
- [React Native](https://docs.copilotkit.ai/google-adk/react-native)
- [Slack](https://docs.copilotkit.ai/google-adk/slack)
- [Microsoft Teams](https://docs.copilotkit.ai/google-adk/microsoft-teams)
- [AG-UI for ADK](/integrations/ag-ui/)
- [A2UI for ADK](/integrations/a2ui/)
- [Generative UI for Google ADK](https://docs.copilotkit.ai/google-adk/generative-ui)
- [Frontend tools for Google ADK](https://docs.copilotkit.ai/google-adk/frontend-tools)
- [Shared state for Google ADK](https://docs.copilotkit.ai/google-adk/shared-state)
