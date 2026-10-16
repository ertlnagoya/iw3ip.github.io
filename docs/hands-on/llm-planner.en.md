# Let an AI plan the steps (LLM Planner)

Replace the rule-based planner in [Run the regional safety assistant](regional-safety-assistant.en.md) with one based on a large language model (LLM), in three stages.

> **What you'll do**: Swap planner via stub provider first, check it in the frontend demo, then use a real LLM
>
> **Prerequisites**: [Run the regional safety assistant](regional-safety-assistant.en.md)
>
> **What you need**: PC (real LLM step needs an API key or a local LLM)
>
> **Time required**: approx. 60 min

This page is organized in three stages:

1. reproduce the flow locally with the `stub` provider
2. check how `planner_diagnostics` is shown in the frontend demo (`assistant-demo`)
3. switch to an actual OpenAI-compatible API if available

## Shortest path

For a first pass, these five steps are enough.

1. start the `assistant`
2. call `/assistant/plan` with the `stub` provider
3. inspect `planner_diagnostics`
4. if needed, start `assistant-demo` and inspect the same result from the UI
5. finally switch to `openai_compatible`

Branches after that:

- If you only need the API path: the `stub` provider and `curl` are enough
- If you also want the UI path: start `assistant-demo`
- If you want to connect to a real API: continue to `assistant-llm` with `.env.local.example`

## What this page covers

- what to keep fixed when replacing the rule-based planner with an LLM planner
- the roles of `planner_factory`, `validator`, and `fallback`
- how to read results and failures using `planner_diagnostics`

## Common issues

- it is easy to assume that adding an LLM also requires changing the evaluator and actuator
- the configuration difference between `stub` and `openai_compatible` is concentrated in environment variables
- on failure, the cause is hard to find without looking at `planner_diagnostics`, not just the `plan` body

## Prerequisites

- Docker / Docker Compose
- `curl`
- Python 3.11 or later (to run `uvicorn` in Stage 1 and 5.2 and `pytest` in 5.5 on the host). Create a virtual environment (venv) at the top of the `Blockchain_IoT_Marketplace` repository and install the packages into it. With Homebrew Python and similar setups, `pip install` outside a venv fails with an `externally-managed-environment` error. If `python3 --version` is older than 3.11, run the `python3 -m venv` line below with a specific version, such as `python3.11 -m venv`

    ```bash
    cd ~/program/Blockchain_IoT_Marketplace
    python3 -m venv .venv && source .venv/bin/activate
    pip install fastapi uvicorn pydantic pydantic-settings httpx pytest
    ```

    `pip install -e . pytest` also installs the required packages (along with the publisher's). In every newly opened terminal ("another terminal"), run `source ~/program/Blockchain_IoT_Marketplace/.venv/bin/activate` before using `uvicorn`, `python`, or `pytest`. With the venv active, `python` points to the venv's Python 3
- the `main` branch of the `Blockchain_IoT_Marketplace` repository

References:

- [Docker official site](https://docs.docker.com/get-docker/)
- [httpx documentation](https://www.python-httpx.org/)

## Matching source files

- [assistant/app/planner_interface.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/assistant/app/planner_interface.py)
- [assistant/app/planner_factory.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/assistant/app/planner_factory.py)
- [assistant/app/llm_planner.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/assistant/app/llm_planner.py)
- [assistant/app/llm_prompt.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/assistant/app/llm_prompt.py)
- [assistant/app/llm_provider.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/assistant/app/llm_provider.py)
- [assistant/app/plan_validator.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/assistant/app/plan_validator.py)
- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase3_llm_planner/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase3_llm_planner/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase3_llm_planner/README.md)
- [React frontend demo](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/tree/main/assistant-ui)
- [.env.local.example](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/.env.local.example)
- [examples/phase3_llm.env.example](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/phase3_llm.env.example)
- [examples/phase3_llm_mock.env.example](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/phase3_llm_mock.env.example)
- [examples/phase3_llm_expected_plan.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/phase3_llm_expected_plan.json)
- [examples/phase3_llm_mock_server.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/phase3_llm_mock_server.py)
- [examples/phase3_request_station_warning.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/phase3_request_station_warning.json)

The exercise programs focus on the minimum request body for `/assistant/plan` and on how to summarize the returned plan.

## Architecture Diagram

```mermaid
flowchart LR
  A["request_text"] --> B["planner_factory"]
  B --> C["RuleBasedPlanner or LLMPlanner"]
  C --> D["llm_prompt / llm_provider"]
  C --> E["plan_validator"]
  E --> F["ExecutionPlan"]
  F --> G["Evaluator / Actuator"]
```

Rather than embedding the LLM directly in `main.py`, the replacement is isolated within `planner_factory`.

## Stage 1: Understand the planner shape with the stub provider

### 1. Start with the `stub` provider

Run the following in the `Blockchain_IoT_Marketplace` repository.

```bash
cd ~/program/Blockchain_IoT_Marketplace
source .venv/bin/activate
ASSISTANT_PLANNER_MODE=llm \
ASSISTANT_PLANNER_NAME=llm-planner-stub-v1 \
ASSISTANT_LLM_PROVIDER=stub \
uvicorn assistant.app.main:app --host 0.0.0.0 --port 8090
```

`uvicorn` runs on the host Python (the venv created in the prerequisites). The Docker Compose `assistant` service starts with the rule-based planner and returns `planner_name` `rule-based-planner-v1`, so use the host for this stage.

In another terminal:

```bash
cd ~/program/Blockchain_IoT_Marketplace
source .venv/bin/activate
curl http://localhost:8090/health
```

Expected:

```json
{"status":"ok","service":"assistant"}
```

### 2. Inspect a Japanese request

```bash
curl -X POST http://localhost:8090/assistant/plan \
  -H 'Content-Type: application/json' \
  -d @examples/phase3_request_park_safety.json
```

Example expected output:

```json
{
  "status": "planned",
  "plan": {
    "planner_name": "llm-planner-stub-v1",
    "target_area": "park-north",
    "watch_events": ["possible_littering", "suspicious_activity"]
  }
}
```

Checkpoints:

- `planner_name` is `llm-planner-stub-v1`
- `target_area` is `park-north`
- `watch_events` is returned as JSON array

Sending the same request from the exercise program (answer version) prints only the key parts of `plan` and `planner_diagnostics`:

```bash
python examples/hands_on/phase3_llm_planner/answer_program.py --base-url http://localhost:8090
```

### 3. Inspect an English request

```bash
curl -X POST http://localhost:8090/assistant/plan \
  -H 'Content-Type: application/json' \
  -d @examples/phase3_request_station_warning.json
```

Example expected output:

```json
{
  "status": "planned",
  "plan": {
    "planner_name": "llm-planner-stub-v1",
    "target_area": "station-front",
    "watch_events": ["suspicious_activity"],
    "actions": [
      {"action_type": "send_notification"},
      {"action_type": "show_warning"}
    ]
  }
}
```

This confirms that the planner can also map an English request to `station-front`.

At this point, the minimum path from natural-language request to `plan` is working. The next step is to inspect how the same planner output should be read from the frontend side.

## Stage 2: Inspect the frontend demo and diagnostics

### 3.5 Start the React frontend demo

There is also a minimal React screen for calling the Phase 3 assistant from a browser.

First stop the `uvicorn` from Stage 1 with Ctrl+C (it uses the same port 8090).

```bash
docker compose -f infra/docker-compose.yml --profile assistant up --build -d assistant
docker compose -f infra/docker-compose.yml --profile assistant-ui up --build -d assistant-ui
```

Open:

- `http://localhost:4173`

This screen lets you check:

- editing the natural-language request
- `POST /assistant/plan`
- `POST /assistant/execute`
- `GET /assistant/executions`
- badge / alert rendering for `planner_diagnostics`

Screen image:

![assistant-demo overview](../assets/assistant-demo-overview.svg)

If you want a one-command path including the local mock LLM, stop the `assistant` above first (`assistant-demo` also uses port 8090):

```bash
docker compose -f infra/docker-compose.yml stop assistant
docker compose -f infra/docker-compose.yml --profile assistant-demo up --build -d
```

This profile starts:

- `assistant-demo`
- `llm-mock`
- `assistant-ui`

### 4. Inspect the real-API environment template

To use an actual OpenAI-compatible API, inspect the example environment file:

```bash
cat .env.local.example
```

Main fields:

- `ASSISTANT_PLANNER_MODE=llm`
- `ASSISTANT_LLM_PROVIDER=openai_compatible`
- `ASSISTANT_LLM_API_BASE_URL`
- `ASSISTANT_LLM_API_KEY`
- `ASSISTANT_LLM_MODEL`

How to read `planner_diagnostics`:

- `status`
  - `ok` or `fallback`
- `severity`
  - `info`, `warning`, or `error`
- `label`
  - short text that can be shown directly in a badge
- `color_hint`
  - color hint such as `green`, `amber`, or `red`
- `code`
  - stable code that the frontend can branch on
- `category`
  - broad group such as `success`, `provider`, `validation`, or `planner`
- `user_message`
  - short message that can be shown directly to the end user
- `summary`
  - short explanation of what happened
- `suggestion`
  - next troubleshooting step

Example for frontend use:

```ts
type PlannerDiagnostics = {
  status: "ok" | "fallback";
  severity: "info" | "warning" | "error";
  label: string;
  color_hint: "green" | "amber" | "red";
  code: string;
  category: "success" | "provider" | "validation" | "planner";
  user_message: string;
  summary?: string | null;
  suggestion?: string | null;
};

type BadgeViewModel = {
  label: string;
  tone: "success" | "warning" | "danger" | "neutral";
  detail: string;
  code: string;
  category: string;
};

export function diagnosticsToBadge(diagnostics: PlannerDiagnostics): BadgeViewModel {
  const toneMap: Record<PlannerDiagnostics["color_hint"], BadgeViewModel["tone"]> = {
    green: "success",
    amber: "warning",
    red: "danger",
  };

  return {
    label: diagnostics.label,
    tone: toneMap[diagnostics.color_hint] ?? "neutral",
    detail: diagnostics.user_message,
    code: diagnostics.code,
    category: diagnostics.category,
  };
}
```

For example, a fallback case can be rendered like this:

```ts
const badge = diagnosticsToBadge({
  status: "fallback",
  severity: "warning",
  label: "Fallback",
  color_hint: "amber",
  code: "llm_provider_error",
  category: "provider",
  user_message: "The LLM API could not be used, so the rule-based planner was used instead.",
});
```

Minimal React example:

```tsx
type PlannerDiagnostics = {
  label: string;
  color_hint: "green" | "amber" | "red";
  code: string;
  category: string;
  user_message: string;
  summary?: string | null;
  suggestion?: string | null;
};

function DiagnosticsBadge({ diagnostics }: { diagnostics: PlannerDiagnostics }) {
  return (
    <span className={`badge badge-${diagnostics.color_hint}`}>
      {diagnostics.label}
    </span>
  );
}

function DiagnosticsAlertPanel({ diagnostics }: { diagnostics: PlannerDiagnostics }) {
  return (
    <div className={`alert alert-${diagnostics.color_hint}`}>
      <strong>{diagnostics.label}</strong>
      <p>{diagnostics.user_message}</p>
      <small>
        code={diagnostics.code} / category={diagnostics.category}
      </small>
      {diagnostics.suggestion ? <p>Next: {diagnostics.suggestion}</p> : null}
    </div>
  );
}
```

Minimal Vue example:

```vue
<script setup lang="ts">
type PlannerDiagnostics = {
  label: string;
  color_hint: "green" | "amber" | "red";
  code: string;
  category: string;
  user_message: string;
  summary?: string | null;
  suggestion?: string | null;
};

defineProps<{
  diagnostics: PlannerDiagnostics;
}>();
</script>

<template>
  <span :class="`badge badge-${diagnostics.color_hint}`">
    {{ diagnostics.label }}
  </span>

  <div :class="`alert alert-${diagnostics.color_hint}`">
    <strong>{{ diagnostics.label }}</strong>
    <p>{{ diagnostics.user_message }}</p>
    <small>code={{ diagnostics.code }} / category={{ diagnostics.category }}</small>
    <p v-if="diagnostics.suggestion">Next: {{ diagnostics.suggestion }}</p>
  </div>
</template>
```

Minimal CSS example:

```css
.badge {
  display: inline-flex;
  align-items: center;
  border-radius: 999px;
  padding: 0.2rem 0.65rem;
  font-size: 0.85rem;
  font-weight: 700;
}

.badge-green {
  background: #e8f7ee;
  color: #146c43;
}

.badge-amber {
  background: #fff3cd;
  color: #9a6700;
}

.badge-red {
  background: #fbe9e7;
  color: #b42318;
}

.alert {
  margin-top: 0.75rem;
  border-left: 4px solid currentColor;
  border-radius: 0.5rem;
  padding: 0.75rem 1rem;
  background: #f8f9fa;
}

.alert-green {
  color: #146c43;
  background: #eefbf3;
}

.alert-amber {
  color: #9a6700;
  background: #fff8e1;
}

.alert-red {
  color: #b42318;
  background: #fef3f2;
}
```

At this point, `planner_diagnostics` can be read from both the API and the UI. The last step is to replace the provider while keeping the same structure.

## Stage 3: Switch to an OpenAI-compatible API

### 5. Switch to an OpenAI-compatible API

Copy `.env.local.example`, set `ASSISTANT_LLM_API_KEY` (and, if needed, `ASSISTANT_LLM_MODEL` and `ASSISTANT_LLM_API_BASE_URL`) in `.env.local` to your own values, then start the server. Stop any other assistant on port 8090 first (`assistant`, `assistant-demo`, or the host `uvicorn`).

```bash
cp .env.local.example .env.local
# edit ASSISTANT_LLM_API_KEY etc. in .env.local
docker compose -f infra/docker-compose.yml --profile assistant-llm up --build -d assistant-llm
```

Reference screenshot:

![assistant-llm startup](../assets/llm-planner-compose.svg)

Then call the same endpoint:

```bash
curl -X POST http://localhost:8090/assistant/plan \
  -H 'Content-Type: application/json' \
  -d @examples/phase3_request_park_safety.json
```

The expected JSON shape is documented here:

- [examples/phase3_llm_expected_plan.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/phase3_llm_expected_plan.json)

Reference screenshot:

![LLM planner response](../assets/llm-planner-response.svg)

### 5.2 Try the HTTP path with the local mock server

If you do not want to call a real API yet, you can check just the HTTP path with an OpenAI-compatible mock server.

If `assistant-demo` from §3.5 is still running, its `llm-mock` already holds port 18000. Stop it first (`assistant-demo` also uses port 8090, so stop it either way):

```bash
docker compose -f infra/docker-compose.yml --profile assistant-demo down
```

Start the mock server:

```bash
cd ~/program/Blockchain_IoT_Marketplace
source .venv/bin/activate
uvicorn examples.phase3_llm_mock_server:app --host 127.0.0.1 --port 18000
```

Docker Compose example:

```bash
docker compose -f infra/docker-compose.yml --profile llm-mock up --build -d llm-mock
```

In another terminal, start the assistant (the commands below move to the repository top and activate the venv). Stop any other assistant on port 8090 first (the Docker `assistant`, `assistant-demo`, or `assistant-llm`):

```bash
cd ~/program/Blockchain_IoT_Marketplace
source .venv/bin/activate
source examples/phase3_llm_mock.env.example
uvicorn assistant.app.main:app --host 0.0.0.0 --port 8090
```

Then call the same endpoint:

```bash
curl -X POST http://localhost:8090/assistant/plan \
  -H 'Content-Type: application/json' \
  -d @examples/phase3_request_park_safety.json
```

Checkpoints:

- `planner_name` is `llm-planner-mock-http-v1`
- `planner_diagnostics.provider_name` is `openai_compatible`
- `planner_diagnostics.severity` is `info`
- `planner_diagnostics.label` is `OK`
- `planner_diagnostics.color_hint` is `green`
- `planner_diagnostics.code` is `llm_plan_generated`
- `planner_diagnostics.category` is `success`
- `planner_diagnostics.user_message` is readable enough to show in the UI
- `planner_diagnostics.used_fallback` is `false`

### 5.5 Use the exercise pytest

This hands-on also includes a reusable pytest file for both the problem and answer programs. In the problem program (`problem_program.py`) you implement `build_plan_request()` (TODO 1), which builds the JSON to send, and `summarize_plan()` (TODO 2), which summarizes the response.

First, validate the answer program:

```bash
pytest -q tests/test_phase3_llm_hands_on_program.py
```

After the TODOs in the problem program are implemented, switch the target module like this:

```bash
PHASE3_LLM_HANDS_ON_MODULE=examples.hands_on.phase3_llm_planner.problem_program \
pytest -q tests/test_phase3_llm_hands_on_program.py
```

### 6. Check validator and fallback behavior

The LLM output is not adopted as-is; the following are always validated:

- whether the events are allowed
- whether the actions are allowed
- whether the area is allowed
- whether the response can be parsed as JSON

If invalid, it falls back to the rule-based planner.

Points to check:

- if `planner_name` differs from the expected value, a fallback may have occurred
- actions outside the allow list are not executed as-is
- `/assistant/executions` retains `planner_diagnostics.error_type` and `planner_diagnostics.error_message`

## Success criteria

This hands-on is successful if you can confirm:

- `llm` mode works with the `stub` provider
- both Japanese and English requests return plans
- you can explain the environment variables for an OpenAI-compatible API
- you can explain the role of validator and fallback

## Common issues

### `.env.local` still has placeholder values

- a real API call will not work while `REPLACE_WITH_YOUR_API_KEY` is still present

### `401 Unauthorized`

- check `ASSISTANT_LLM_API_KEY`

### `404` or `405`

- check whether `ASSISTANT_LLM_API_BASE_URL` points to an OpenAI-compatible `/chat/completions` endpoint

### `docker compose --profile assistant-llm up` fails

- make sure `.env.local` exists
- make sure `ASSISTANT_LLM_API_KEY` is not empty in `.env.local`

### A real API always falls back instead of returning the LLM plan

Check:

- API key
- base URL
- model name
- whether the returned content is valid JSON

### `station-front` does not appear

- make sure your `main` branch is up to date (`git pull`)

## Next steps

- For the design rationale: [LLM Planner replacement spec](llm-planner-spec.md)
- To go back to the original Phase 3 sample: [Run the regional safety assistant (Phase 3)](regional-safety-assistant.md)
