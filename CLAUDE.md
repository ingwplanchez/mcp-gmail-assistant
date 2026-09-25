# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**MCP Gmail Assistant** is an intelligent email management assistant built on the **Model Context Protocol (MCP)** using **FastMCP**, integrating the Gmail API with local/cloud LLMs via **Ollama**, with a **Streamlit** web interface.

### Core Components

| File | Purpose |
|------|---------|
| `gmail_mcp_server.py` | FastMCP server exposing Gmail tools (`list_emails`, `send_email`), static resource (`gmail://profile`), resource templates (`docs://setup-manual/{version}`), and prompts (`daily_email_summary`, `compose_professional_email`) |
| `client.py` | Async MCP client that dynamically discovers MCP capabilities, converts them to LLM function-calling format, and manages the Ollama interaction loop |
| `app.py` | Streamlit web UI with chat interface, sidebar showing real-time MCP catalog, quick-prompt buttons, and formatted responses |

---

## Commands

### Environment Setup
```powershell
# Create and activate venv (Windows PowerShell)
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# Install dependencies
pip install -r requirements.txt

# Or with uv (faster)
uv venv
uv pip install -r requirements.txt
```

### Running the Application
```bash
# 1. Start Ollama with the model from .env
ollama run nemotron-3-super:cloud

# 2. Run Streamlit app
streamlit run app.py
# Or with uv:
uv run streamlit run app.py
```

### MCP Server Development & Debugging
```bash
# Run MCP server directly
fastmcp run gmail_mcp_server.py

# Interactive browser inspector (recommended for development)
fastmcp dev gmail_mcp_server.py

# Console inspection (summary/JSON)
fastmcp inspect gmail_mcp_server.py

# Test the client directly
python client.py
```

### Syntax Validation
```bash
# Validate Python syntax after changes
python -m py_compile gmail_mcp_server.py
python -m py_compile client.py
python -m py_compile app.py
```

---

## Architecture

### Data Flow
```
User → Streamlit (app.py) → GmailMCPClient (client.py) → FastMCP Server (gmail_mcp_server.py) 
                                                    → Gmail API / PDF Manuals
                                                    → Ollama LLM
```

1. **User Interaction**: Chat input or quick-prompt button in Streamlit
2. **MCP Discovery**: Client connects to server, lists tools/resources/prompts dynamically
3. **LLM Inference**: Client sends messages + tool definitions to Ollama
4. **Tool/Resource Execution**: If LLM calls a tool, client executes it against MCP server
5. **External Services**: MCP server calls Gmail API (OAuth 2.0) or reads PDF manuals
6. **Response**: Results flow back through client → Streamlit for rendering

### Key Design Patterns

- **Dynamic MCP Discovery**: `GmailMCPClient` discovers capabilities at runtime via `list_tools()`, `list_resources()`, `list_resource_templates()`, `list_prompts()` — no hardcoded schemas
- **Resources as Tools**: Static resources and resource templates are converted to function-calling format so the LLM can invoke them uniformly
- **OAuth 2.0 Persistence**: `token.pickle` caches credentials; refreshed automatically on expiry
- **Streamlit Session State**: Chat history and prompt triggers stored in `st.session_state`

---

## Key Files Reference

### `gmail_mcp_server.py`
- **Tools**: `list_emails(max_results, query)`, `send_email(to, subject, body)`
- **Resource**: `gmail://profile` → returns Markdown with email, message count, thread count
- **Resource Template**: `docs://setup-manual/{version}` → reads PDF from `manuals/` using PyPDF2
- **Prompts**: `daily_email_summary()`, `compose_professional_email(recipient, subject)`
- **Auth**: `get_gmail_service()` handles OAuth flow, token refresh, and `token.pickle` persistence

### `client.py`
- `GmailMCPClient` class with:
  - `get_system_info()` → returns catalog of tools/resources/templates/prompts
  - `get_tools_for_llm()` → converts MCP tools to OpenAI function schema
  - `get_resources_as_tools()` → wraps resources/templates as callable functions
  - `chat(messages)` → main interaction loop with Ollama, handles tool calls and re-evaluation
- Hardcoded MCP server path (line 13) — update if moving the project

### `app.py`
- `@st.cache_resource` caches `GmailMCPClient` instance
- Sidebar: quick prompts + expandable MCP catalog (tools, resources, templates, prompts)
- `display_message()` formats MCP resource responses (Markdown starting with `# `) in expanders
- Uses `asyncio.run()` for async client calls (Streamlit runs sync)

---

## Environment Variables (`.env`)

| Variable | Description | Example |
|----------|-------------|---------|
| `OLLAMA_MODEL` | Ollama model name | `nemotron-3-super:cloud` |
| `OLLAMA_HOST` | Ollama server URL | `http://localhost:11434` |
| `GOOGLE_API_KEY` | Optional Google API key | `AQ.Ab8RN6...` |

---

## Security Notes

- **Never commit** `credentials.json`, `token.pickle`, or `.env` with real secrets
- `credentials.json` from Google Cloud Console (Desktop Application type)
- Required Gmail scopes: `gmail.readonly`, `gmail.send`
- First run opens browser for OAuth consent; generates `token.pickle`

---

## Project Structure

```
mcp-gmail-assistant/
├── .env                      # Environment variables
├── app.py                    # Streamlit web UI
├── client.py                 # Async MCP client + Ollama connector
├── gmail_mcp_server.py       # FastMCP server with Gmail tools/resources/prompts
├── credentials.json          # Google OAuth client secret (not in git)
├── token.pickle              # Persisted OAuth token (not in git)
├── requirements.txt          # Python dependencies
├── AGENTS.md                 # AI agent guidelines (this repo's conventions)
├── README.md                 # Project documentation
├── docs/                     # Technical docs & session summaries
│   ├── doc_gmail_mcp_server.md
│   ├── doc_client.md
│   ├── doc_app.md
│   └── ...
└── manuals/                  # PDF manuals for resource templates
    ├── manual_v1.pdf
    ├── manual_v2.pdf
    └── manual_v3.pdf
```

---

## Conventions for AI Agents

- **Python 3.10+** with type hints (`-> dict`, `-> list[dict]`)
- **Async/await** for network calls in `client.py`
- **Docstrings on MCP tools** — FastMCP uses them for LLM descriptions
- **`@st.cache_resource`** for heavy client instantiation in Streamlit
- **Defensive error handling**: catch specific exceptions (Google `HttpError`, Ollama connection errors)
- **Atomic verification**: run `python -m py_compile <file.py>` after edits
- **Update docs** (`README.md`, `AGENTS.md`) when adding/modifying MCP tools/resources/prompts