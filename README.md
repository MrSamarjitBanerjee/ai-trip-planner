<div align="center">
  <img src="docs/banner.svg" alt="AI Trip Planner banner" width="840"/>
</div>

# AI Trip Planner

Ask for a trip and seven specialist agents get to work — flights, hotels, weather,
budget, trains, buses — each pulling live data through MCP servers. A supervisor
agent decides who actually needs to run, an input guardrail keeps the system on-topic,
and the draft itinerary pauses for your approval before anything is finalized.
Approve it, or send revision feedback and watch the graph loop back through.

Built with LangGraph's state machine, FastAPI on the Server Side , and PostgreSQL
checkpoints underneath so an interrupted plan survives until you come back to judge it.

<div align="center">

![Python](https://img.shields.io/badge/python-3.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-async-009688?logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-state%20machine-1C3C3C)
![MCP](https://img.shields.io/badge/MCP-tool%20servers-D97757)
![License](https://img.shields.io/badge/license-Apache--2.0-green)

</div>

<!-- Drop a screenshot below when you have one:
<img width="840" alt="app screenshot" src="screenshots/plan.png"/>
-->

## How a request flows

```text
                 ┌────────────────┐
   user prompt ──▶|   supervisor   |── blocks non-travel asks (input guardrail)
                  └───────┬────────┘
                          | picks whichever specialists this trip needs
        ┌────────┬────────┼────────┬─────────┬─────────┐
        ▼        ▼        ▼        ▼         ▼         ▼
      flight   hotel   weather   budget    train      bus
        └────────┴────────┴───┬────┴─────────┴─────────┘
                              ▼
                     itinerary agent  (draft plan)
                              │
                interrupt() — graph pauses here
                              │
                approve ──────┴────── revise (back through supervisor)
                              ▼
                  final agent → polished plan
```

## The crew

| Agent | Job | Data source |
|---|---|---|
| **supervisor** | validates the request, extracts constraints (destination, budget, dates), routes to the right specialists | Groq LLM |
| **flight** | likely routes, airlines, fare ranges, booking advice | AviationStack MCP (`uvx`) |
| **hotel** | areas to stay, accommodation guidance | Tavily MCP (live search) |
| **weather** | current conditions + short forecast | OpenWeather via a custom MCP server |
| **budget** | cost breakdown, feasibility, where you'll overspend | LLM over upstream results |
| **train** | Indian Railways routes, classes, tatkal tips | Tavily search |
| **bus** | operators, bus types, fare ranges | Tavily search |
| **itinerary** | stitches everything into a day-by-day draft | LLM |
| **human approval** | `interrupt()` freezes the run until you respond | *you* |
| **final** | polished markdown plan, folds in revision feedback | LLM |

## ⚙️ Under the Hood

 # Shared State
-All agents work with a common TravelState. Each agent reads the state and updates only what it needs, while LangGraph handles state merging.

# Dynamic Routing
The supervisor uses the LLM to return structured JSON containing the required agents and travel constraints. LangGraph then routes the request dynamically instead of following a fixed pipeline.

Human-in-the-Loop
After generating the itinerary, interrupt() pauses the graph for approval. The state is checkpointed in PostgreSQL, allowing the workflow to resume later with Command(resume=...) — even after restarting the server.

MCP Integration
The project connects to multiple MCP servers using different transports: Tavily via HTTP, AviationStack via uvx/stdio, and a custom weather MCP server. Tools are loaded only when required.

Async API + Sync Graph
FastAPI endpoints are async, while graph execution runs in a threadpool so long-running LLM and MCP calls don't block other API requests.

Context & Cost Control
Large tool outputs are trimmed before being passed between agents, and an llm_calls counter tracks model usage for each workflow.

## Design notes

Things I cared about beyond the happy path:

- **The guardrail fails open.** If the validation response comes back malformed,
  the request passes instead of erroring out. Blocking a legit trip over a JSON
  glitch is worse than a rare false negative.
  
- **Rejections loop back through supervision.** Revision feedback isn't glued onto
  the old draft — the graph routes back to the supervisor, which re-selects agents
  based on what you actually asked to change.
  
- **Simple questions stay cheap.** Because routing picks only the specialists a
  request needs, "when should I visit Kashmir?" costs two LLM calls instead of
  ten. The `llm_calls` counter on every response makes that visible.
  
- **Every agent degrades gracefully.** When live data dies mid-run, the agent
  returns clearly-labelled general advice instead of crashing the request.

## Run it locally

Python 3.11 recommended (anything 3.10+ should work).

```powershell
git clone https://github.com/<your-name>/ai-trip-planner.git
cd ai-trip-planner

python -m venv .venv
.venv\Scripts\Activate.ps1          # windows powershell
pip install -r requirements.txt

Copy-Item .env.example .env         # then fill in your keys
python app.py
```

Open http://127.0.0.1:8000, type something like *"plan a 7 day Japan trip under
2 lakhs"* and watch the workflow panel light up.

You need a Postgres instance somewhere — any will do:

```powershell
docker run -d --name trip-pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 postgres:16
# then in .env:
# DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres
```

Checkpoint tables are created automatically on first boot, no migrations to run.

One heads-up: the flight agent shells out to `uvx aviationstack-mcp`.
Install [uv](https://docs.astral.sh/uv/) (`pip install uv`) if you want live flight
data — every other agent works without it.

### Environment variables

| Variable | Required | What it does |
|---|---|---|
| `GROQ_API_KEY` | yes | powers every LLM agent ([console.groq.com](https://console.groq.com)) |
| `DATABASE_URL` | yes | Postgres connection string for graph checkpoints |
| `TAVILY_API_KEY` | yes | live web search for hotels/train/bus research |
| `AVIATION_STACK_API_KEY` | yes | flight data |
| `OPENWEATHER_API_KEY` | yes | weather + forecast |
| `GROQ_MODEL` | no | model override (defaults to `openai/gpt-oss-120b`) |
| `LANGSMITH_*` | no | turn on LangSmith tracing while debugging |

## Docker

```powershell
docker build -t ai-trip-planner .
docker run -p 8000:8000 --env-file .env ai-trip-planner
```

The image installs `uv` itself, so the flight MCP server works out of the box.

## Deploying to Render

This repo ships a `render.yaml` blueprint. Push to GitHub, then in Render:
**New → Blueprint** → pick the repo. It creates the web service and a Postgres
instance together and asks you once for the five API keys. Done.

Two things to know about Render's free tier: the service sleeps after 15 minutes
idle (~a minute to wake), and free Postgres expires after 30 days. Fine for a
demo, upgrade if it becomes more than that.

## API

| Endpoint | Body | Returns |
|---|---|---|
| `POST /api/travel` | `{"message": "...", "thread_id": "optional"}` | agent results + draft itinerary, or the final answer when no approval needed |
| `POST /api/travel/approve` | `{"thread_id": "...", "approved": true/false, "feedback": "..."}` | resumed result; rejecting with feedback re-runs supervision |
| `GET /health` | – | liveness + feature flags |

## Project layout

```text
app.py                        FastAPI routes, templates, static wiring
backend.py                    LangGraph state, agents, graph assembly
mcp_client.py                 MCP client + per-server tool loading
custom_weather_mcp_server.py  standalone weather MCP server (stdio)
templates/ static/            boarding-pass themed UI
docs/                         banner + design file
render.yaml                   one-click Render deployment
```

## License

[Apache-2.0](LICENSE)

