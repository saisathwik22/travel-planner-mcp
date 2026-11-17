<div align="center">

# ✈️ TripMate AI

### A multi-agent travel planner powered by LangGraph and the Model Context Protocol (MCP)

Describe your trip in plain English and get flight options, hotel ideas, weather insights and a day-by-day itinerary, all in one place. TripMate AI extends [TripPilot AI](https://travel-planner-gww0.onrender.com) by moving its tool integrations onto **MCP**, so each data source is a swappable, standards-based tool server instead of a hard-wired API call.

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-multi--agent-1C3C3C)
![MCP](https://img.shields.io/badge/MCP-tool%20servers-6E56CF)
![Groq](https://img.shields.io/badge/LLM-Groq-F55036)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)

</div>

---

## Overview

Planning a trip usually means juggling flight sites, hotel listings, weather apps and spreadsheets. **TripMate AI** replaces that with one prompt. Five specialised agents, coordinated by a LangGraph workflow, research each part of the trip through MCP tool servers and merge the results into a single, readable plan.

> **Example prompt:** *Plan a 5-day Bali trip from Hyderabad including flights, hotels, weather and sightseeing under 1.5 lakhs.*

## Why MCP?

TripPilot AI called each API directly from agent code. TripMate AI wraps every external capability behind the Model Context Protocol, which gives you:

- **Decoupled tools:** agents talk to a uniform tool interface, not vendor SDKs.
- **Mixed transports:** remote HTTP and local stdio servers live side by side.
- **Easy extension:** adding a new data source (trains, visas, currency) means adding an MCP server, not rewriting the workflow.
- **Custom tools as first-class citizens:** the weather tool is our own MCP server, built the same way as the third-party ones.

## Highlights

- **Multi-agent workflow:** flight, hotel, weather, itinerary and final-response agents orchestrated with LangGraph.
- **Flight research:** AviationStack via a local stdio MCP server.
- **Hotel research:** Tavily web search via a remote MCP endpoint.
- **Weather and forecasts:** a custom local MCP server.
- **Structured itineraries:** budget-aware, day-by-day plans with timings and practical tips.
- **Conversation memory:** threads persisted in PostgreSQL, so follow-ups like *"make it cheaper"* keep context.
- **Fast inference:** LLM responses served through Groq.
- **Web UI:** FastAPI backend with a Jinja2 + vanilla JS frontend.

## Architecture

```text
                 ┌─────────────────────┐
  User request ──►  FastAPI /api/travel │
                 └──────────┬──────────┘
                            ▼
                 ┌─────────────────────┐
                 │  LangGraph workflow │◄──── PostgreSQL (thread state)
                 └──────────┬──────────┘
      ┌──────────────┬──────┴───────┬───────────────┐
      ▼              ▼              ▼               ▼
 ✈️ Flight      🏨 Hotel       🌤 Weather      🗺️ Itinerary
   agent          agent          agent           agent
      │              │              │            (Groq LLM)
      ▼              ▼              ▼
 ┌─────────────────────────────────────┐
 │         mcp_client.py (MCP)         │
 └───┬─────────────┬─────────────┬─────┘
     ▼             ▼             ▼
 AviationStack   Tavily      Custom Weather
 MCP (stdio)   MCP (remote)  MCP (stdio)
 uvx aviationstack-mcp   custom_weather_mcp_server.py
                            │
                            ▼
                  ✍️ Final response agent
                            ▼
                Formatted Markdown travel plan
```

1. The user submits a travel request from the web UI.
2. `extract_destination` parses the destination from the request.
3. The **flight**, **hotel** and **weather** agents call their MCP tools in the workflow.
4. The **itinerary agent** builds a day-by-day plan around those results.
5. The **final agent** merges everything into a clean Markdown answer.
6. State is stored under a `thread_id`, so the user can keep refining the plan.

## MCP Integration

| Capability | MCP server | Transport |
| --- | --- | --- |
| Hotel and web search | Tavily (`https://mcp.tavily.com/mcp/`) | Remote HTTP |
| Flight data | AviationStack (`uvx aviationstack-mcp`) | Local stdio |
| Weather and forecast | `custom_weather_mcp_server.py` | Local stdio (custom) |

All MCP access is centralised in `mcp_client.py`, built on `langchain-mcp-adapters` and `mcp`. It exposes async helpers that the agents in `backend.py` call:

| Helper | Purpose |
| --- | --- |
| `tavily_mcp_search` | Hotel and web research through Tavily |
| `aviation_mcp_call` | Flight lookups through AviationStack |
| `weather_mcp_search` | Current weather for the destination |
| `forecast_mcp_search` | Forecast for the travel window |
| `extract_destination` | Pulls the destination out of the user's message |

## Tech Stack

| Layer | Technology |
| --- | --- |
| Backend | Python 3.10+, FastAPI |
| Agent orchestration | LangGraph, LangChain |
| Tool protocol | MCP via `langchain-mcp-adapters` and `mcp` |
| LLM | Groq |
| Persistence | PostgreSQL |
| Search and data | Tavily API, AviationStack API |
| Frontend | HTML, CSS, vanilla JavaScript, Jinja2 |

## Project Structure

```text
.
├── app.py                          # FastAPI app entry point
├── backend.py                      # LangGraph travel workflow
├── mcp_client.py                   # MCP client + async helper functions
├── custom_weather_mcp_server.py    # Custom weather MCP server
├── requirements.txt                # Python dependencies
├── static/                         # CSS and JS
└── templates/
    └── index.html                  # Main page template
```

## Getting Started

### Prerequisites

- Python **3.10** or newer
- A running **PostgreSQL** instance
- [`uv`](https://docs.astral.sh/uv/) installed (needed for `uvx aviationstack-mcp`)
- API keys for **Groq**, **Tavily** and **AviationStack**

### 1. Clone and install

```bash
git clone <your-repo-url>
cd <your-repo-folder>

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
DATABASE_URL=postgresql://user:password@localhost:5432/travel_db
GROQ_API_KEY=your_groq_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
TAVILY_API_KEY=your_tavily_api_key
DEFAULT_ORIGIN_IATA=HYD
```

> Never commit your `.env` file. Add it to `.gitignore`.

### 3. Run the app

```bash
python app.py
```

Open **http://127.0.0.1:8000/**. The local MCP servers (AviationStack and weather) are launched as subprocesses by the MCP client, so there is nothing extra to start.

## API Reference

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/health` | Health check |
| `POST` | `/api/travel` | Submit a travel request |

```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d '{"message":"Plan a 3-day trip to Tokyo with a budget of $1200","thread_id":null}'
```

Pass the `thread_id` from a previous response to continue a conversation.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| MCP connection errors | Confirm `uv` is installed and `uvx aviationstack-mcp` runs on its own |
| No flight results | Check your AviationStack key and plan limits (free tiers are restricted) |
| Empty hotel results | Check your Tavily key and quota |
| Weather tool failing | Run `python custom_weather_mcp_server.py` directly to see its errors |
| Database errors | Verify `DATABASE_URL` and that PostgreSQL is reachable |

## Roadmap

- [ ] Streaming responses
- [ ] Multi-city and round-trip planning
- [ ] Currency conversion and budget breakdown
- [ ] Map view for itinerary stops
- [ ] More MCP servers (trains, visas, local events)
- [ ] Docker setup for one-command start

## Contributing

Contributions are welcome.

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes
4. Push the branch and open a pull request

## Related Project

TripMate AI is the MCP-based extension of **TripPilot AI**, which used direct API integrations. See the original at https://travel-planner-gww0.onrender.com.

## Acknowledgments

Built with [LangGraph](https://github.com/langchain-ai/langgraph), [LangChain](https://github.com/langchain-ai/langchain), [FastAPI](https://fastapi.tiangolo.com/), [Groq](https://groq.com/), [Tavily](https://tavily.com/), [AviationStack](https://aviationstack.com/) and the [Model Context Protocol](https://modelcontextprotocol.io/).

> **Disclaimer:** Prices, schedules, weather and availability come from third-party APIs and AI-generated content, and can be incomplete or out of date. Always verify details with the airline or hotel before booking.
