# 🤖 Guía para Agentes de IA (AGENTS.md) - MCP Gmail Assistant

Este archivo establece las directrices, estructura del proyecto, comandos y convenciones técnicas para **Agentes de IA** (como Google Antigravity, Cursor, Claude, Copilot Workspace, etc.) que colaboren en el mantenimiento, desarrollo y refactorización de este repositorio.

---

## 📌 Visión General del Proyecto

**MCP Gmail Assistant** es un asistente inteligente basado en el **Model Context Protocol (MCP)** mediante **FastMCP**, que integra la API de Gmail con modelos de lenguaje locales/nube a través de **Ollama**, con una interfaz web construida en **Streamlit**.

### Componentes Principales:
- **`gmail_mcp_server.py`**: Servidor MCP que expone herramientas (`list_emails`, `send_email`), recursos estáticos (`gmail://profile`), plantillas de recursos (`docs://setup-manual/{version}`) y prompts predefinidos (`daily_email_summary`, `compose_professional_email`).
- **`client.py`**: Cliente MCP asíncrono que descubre dinámicamente capacidades MCP, las transforma al formato de herramientas de LLM (*Function Calling*) y gestiona el ciclo de interacción con **Ollama**.
- **`app.py`**: Aplicación de interfaz de usuario interactiva desarrollada en **Streamlit** que visualiza el chat, paneles informativos y respuestas formateadas.

---

## 📁 Estructura del Repositorio

```text
mcp-gmail-assistant/
├── .env                      # Variables de entorno (OLLAMA_MODEL, OLLAMA_HOST, GOOGLE_API_KEY)
├── app.py                    # Interfaz web interactiva en Streamlit
├── client.py                 # Cliente asíncrono MCP y conector Ollama
├── gmail_mcp_server.py       # Servidor MCP (FastMCP) con herramientas y recursos de Gmail
├── credentials.json          # Credenciales del cliente OAuth 2.0 (Google Cloud Console)
├── token.pickle              # Token OAuth persistido tras autenticación inicial
├── requirements.txt          # Dependencias del proyecto
├── AGENTS.md                 # Instrucciones y reglas para Agentes de IA (este archivo)
├── README.md                 # Documentación principal del proyecto
├── docs/                     # Documentación técnica de referencia y resúmenes de sesión
└── manuals/                  # Manuales PDF consumidos como recursos MCP
```

---

## 🛠️ Comandos de Ejecución y Entorno

Los agentes deben utilizar y recomendar los siguientes comandos estándar:

### 1. Gestión de Entornos y Dependencias
```powershell
# Crear y activar venv convencional (Windows PowerShell)
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# Instalación de dependencias
pip install -r requirements.txt

# (Opcional) Uso de uv para rendimiento optimizado
uv venv
uv pip install -r requirements.txt
```

### 2. Ejecución del Servidor Ollama
```bash
ollama run nemotron-3-super:cloud
# O el modelo configurado en .env (ej. gemma4:31b-cloud)
```

### 3. Ejecución y Depuración del Servidor MCP (FastMCP)
```bash
# Inspección de capacidades en consola
fastmcp inspect gmail_mcp_server.py

# Interfaz interactiva de pruebas en navegador (Inspector MCP)
fastmcp dev gmail_mcp_server.py

# Ejecución directa del servidor MCP
fastmcp run gmail_mcp_server.py
```

### 4. Ejecución de la Aplicación Streamlit
```bash
streamlit run app.py

# Con uv:
uv run streamlit run app.py
```

---

## 🔑 Variables de Entorno y Credenciales

- **`.env`**: Debe declarar `OLLAMA_MODEL` y `OLLAMA_HOST`.
- **`credentials.json`**: Se obtiene de la consola de Google Cloud (Desktop Application).
- **`token.pickle`**: Generado dinámicamente tras el consentimiento inicial del usuario con los ámbitos:
  - `https://www.googleapis.com/auth/gmail.readonly`
  - `https://www.googleapis.com/auth/gmail.send`

> [!CAUTION]
> **Seguridad:** Los agentes de IA NUNCA deben incluir `credentials.json`, `token.pickle` o secretos de `.env` en repositorios públicos o commits.

---

## 📏 Convenciones de Código y Arquitectura

1. **Python & Tipado:**
   - Escribir código compatible con **Python 3.10+**.
   - Incluir type hints en funciones y métodos (`-> dict`, `-> list[dict]`, etc.).
   - Utilizar `async/await` para llamadas de red o clientes de FastMCP en `client.py`.

2. **Extensión del Servidor MCP (`gmail_mcp_server.py`):**
   - Utilizar el decorador `@mcp.tool()` para funciones ejecutables.
   - Incluir docstrings detallados en cada herramienta, ya que FastMCP los utiliza para generar la descripción que lee el LLM.
   - Usar `@mcp.resource()` para URIs estáticas y `@mcp.resource("template://...")` para recursos parametrizados.

3. **Interfaz Streamlit (`app.py`):**
   - Usar `@st.cache_resource` para instanciar clientes o servicios pesados.
   - Mantener el estado de la interfaz en `st.session_state`.
   - Validar respuestas nulas o estructuradas antes de renderizar componentes visuales.

4. **Manejo Defensivo de Errores:**
   - Capturar excepciones específicas (`HttpError` de la API de Google, errores de conexión con Ollama).
   - No enmascarar ni silenciar errores críticamente sin registrar detalles significativos en logs o mensajes para el usuario.

---

## 🤖 Reglas de Trabajo para Agentes de IA

- **Verificación Atómica:** Siempre ejecutar la sintaxis del código (`python -m py_compile <archivo.py>`) o probar la ejecución tras cualquier modificación.
- **Respeto a Contratos Existentes:** No romper esquemas de entrada de herramientas MCP existentes a menos que se indique explícitamente.
- **Actualización de Documentación:** Si se modifican o añaden nuevas herramientas/recursos en `gmail_mcp_server.py`, actualizar `README.md` y `AGENTS.md`.
- **Estilo de Respuestas:** Mantener comunicaciones concisas, técnicas y estructuradas en Markdown.
