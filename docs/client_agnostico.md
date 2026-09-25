# Guía de Implementación: `client.py` Agnóstico a Proveedores de IA

Este documento explica paso a paso cómo transformar el cliente MCP de Gmail ([`client.py`](../client.py)) para que sea **completamente agnóstico a proveedores de modelos de lenguaje (LLMs)**.

Actualmente, [`client.py`](../client.py) está fuertemente acoplado a la librería oficial de Ollama (`import ollama` y `ollama.chat(...)`). Siguiendo esta guía podrás alternar dinámicamente entre **Ollama (local/gratuito)**, **Google Gemini (económico/cloud)**, **OpenAI** y en un futuro **Anthropic**, optimizando costos y garantizando alta disponibilidad con fallbacks.

---

## 📑 Tabla de Contenidos
1. [Diagnóstico del Acoplamiento Actual](#1-diagnóstico-del-acoplamiento-actual)
2. [Instalación de Dependencias](#2-instalación-de-dependencias)
3. [Configuración de Variables de Entorno (`.env`)](#3-configuración-de-variables-de-entorno-env)
4. [Importaciones e Inicializaciones por Proveedor](#4-importaciones-e-inicializaciones-por-proveedor)
5. [Capa de Abstracción Unificada (`LLMAdapter`)](#5-capa-de-abstracción-unificada-llmadapter)
6. [Patrón 1: Router de IA (Enrutamiento por Reglas/Costos)](#6-patrón-1-router-de-ia-enrutamiento-por-reglascostos)
7. [Patrón 2: Fallback de Alta Disponibilidad](#7-patrón-2-fallback-de-alta-disponibilidad)
8. [Patrón 3: Capa de Evaluación Inteligente (LLM-as-a-Judge)](#8-patrón-3-capa-de-evaluación-inteligente-llm-as-a-judge)
9. [Integración Completa en `client.py`](#9-integración-completa-en-clientpy)
10. [Ruta de Implementación y Recomendaciones](#10-ruta-de-implementación-y-recomendaciones)

---

## 1. Diagnóstico del Acoplamiento Actual

En el archivo original [`client.py`](../client.py), observamos los siguientes puntos de fricción:
- **Línea 3:** `import ollama` — biblioteca propietaria fija.
- **Líneas 40-49:** Las herramientas se formatean al esquema de funciones de OpenAI (`{"type": "function", "function": {...}}`), que Ollama interpreta bien, pero Gemini y Anthropic manejan sus propias estructuras para Tool Calling.
- **Líneas 142-148:** `ollama.chat(model=self.ollama_model, messages=messages, tools=all_tools)` invoca directamente a Ollama.
- **Líneas 163-165:** El parseo de los tool calls asume la estructura retornada por el cliente de Ollama (`tool_call['function']['name']` y `tool_call['function']['arguments']`).

Para ser agnósticos necesitamos desacoplar la **generación de respuestas**, el **llamado a herramientas** y el **manejo de credenciales**.

---

## 2. Instalación de Dependencias

Ejecuta con tu gestor (`uv` o `pip`):

```bash
# Ya instalados en el proyecto:
uv pip install fastmcp python-dotenv

# Proveedor Local (Ollama):
uv pip install ollama

# Google Gemini (nuevo SDK oficial google-genai):
uv pip install google-genai

# OpenAI:
uv pip install openai

# Anthropic (para soporte futuro):
uv pip install anthropic
```

---

## 3. Configuración de Variables de Entorno (`.env`)

En tu archivo [`.env`](../.env), define las credenciales y las configuraciones de cada proveedor:

```ini
# --- Proveedor por defecto ---
# Opciones: "ollama" | "gemini" | "openai" | "anthropic"
DEFAULT_LLM_PROVIDER=ollama

# --- Ollama (Local - Gratuito) ---
OLLAMA_MODEL=gemma3:4b
OLLAMA_HOST=http://localhost:11434

# --- Google Gemini (Cloud - Muy Económico) ---
GEMINI_API_KEY=tu_gemini_api_key_aqui
GEMINI_MODEL=gemini-2.0-flash

# --- OpenAI (Cloud) ---
OPENAI_API_KEY=tu_openai_api_key_aqui
OPENAI_MODEL=gpt-4o-mini

# --- Anthropic (Cloud - Futuro) ---
ANTHROPIC_API_KEY=tu_anthropic_api_key_aqui
ANTHROPIC_MODEL=claude-3-5-haiku-20241022

# --- Configuración del Evaluador Inteligente ---
EVALUATOR_PROVIDER=ollama
EVALUATOR_MODEL=gemma3:4b
```

---

## 4. Importaciones e Inicializaciones por Proveedor

A continuación se detalla cómo inicializar y consumir cada SDK de forma aislada:

### A. OpenAI
Usa la librería estándar `openai`.

```python
import os
from openai import OpenAI

client_openai = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY")
)

# Invocación básica:
# response = client_openai.chat.completions.create(
#     model=os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
#     messages=[{"role": "user", "content": "Hola"}],
#     tools=tools_openai_format
# )
```

### B. Google Gemini (`google-genai`)
Google introdujo el nuevo SDK unificado `google-genai` (reemplazando al antiguo `google-generativeai`).

```python
import os
from google import genai
from google.genai import types

client_gemini = genai.Client(
    api_key=os.getenv("GEMINI_API_KEY")
)

# Invocación básica:
# response = client_gemini.models.generate_content(
#     model=os.getenv("GEMINI_MODEL", "gemini-2.0-flash"),
#     contents="Hola"
# )
# text_response = response.text
```

### C. Ollama
Uso local sin costo ni claves de API:

```python
import os
import ollama

# ollama usa directamente llamadas al módulo o a su Client:
# response = ollama.chat(
#     model=os.getenv("OLLAMA_MODEL", "gemma3:4b"),
#     messages=[{"role": "user", "content": "Hola"}]
# )
```

### D. Anthropic
Para soportar modelos como Claude 3.5 Haiku o Sonnet:

```python
import os
import anthropic

client_anthropic = anthropic.Anthropic(
    api_key=os.getenv("ANTHROPIC_API_KEY")
)

# Invocación básica:
# response = client_anthropic.messages.create(
#     model=os.getenv("ANTHROPIC_MODEL", "claude-3-5-haiku-20241022"),
#     max_tokens=1024,
#     messages=[{"role": "user", "content": "Hola"}]
# )
```

---

## 5. Capa de Abstracción Unificada (`LLMAdapter`)

Para evitar llenar [`client.py`](../client.py) de bloques `if/elif/else`, creamos un archivo puente `llm_adapter.py`. 

Este adaptador expone un método común `.chat(messages, tools=None)` que devuelve una respuesta con una estructura uniforme:

```python
# Tema_04/llm_adapter.py
import os
import json
from dotenv import load_dotenv

load_dotenv()

class LLMAdapter:
    """
    Abstracción agnóstica para interactuar con Ollama, Gemini, OpenAI y Anthropic.
    """
    def __init__(self, provider: str = None, model: str = None):
        self.provider = (provider or os.getenv("DEFAULT_LLM_PROVIDER", "ollama")).lower()
        self.model = model or self._default_model_for_provider(self.provider)
        self.client = self._init_client()

    def _default_model_for_provider(self, provider: str) -> str:
        defaults = {
            "ollama": os.getenv("OLLAMA_MODEL", "gemma3:4b"),
            "gemini": os.getenv("GEMINI_MODEL", "gemini-2.0-flash"),
            "openai": os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
            "anthropic": os.getenv("ANTHROPIC_MODEL", "claude-3-5-haiku-20241022"),
        }
        return defaults.get(provider, "gemma3:4b")

    def _init_client(self):
        if self.provider == "ollama":
            import ollama
            return ollama
        elif self.provider == "gemini":
            from google import genai
            return genai.Client(api_key=os.getenv("GEMINI_API_KEY"))
        elif self.provider == "openai":
            from openai import OpenAI
            return OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        elif self.provider == "anthropic":
            import anthropic
            return anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
        else:
            raise ValueError(f"Proveedor '{self.provider}' no soportado.")

    def chat(self, messages: list, tools: list = None) -> dict:
        """
        Ejecuta la llamada y retorna un diccionario normalizado:
        {
            "content": str,
            "tool_calls": [{"name": str, "arguments": dict}]
        }
        """
        if self.provider == "ollama":
            return self._call_ollama(messages, tools)
        elif self.provider == "openai":
            return self._call_openai(messages, tools)
        elif self.provider == "gemini":
            return self._call_gemini(messages, tools)
        elif self.provider == "anthropic":
            return self._call_anthropic(messages, tools)

    def _call_ollama(self, messages: list, tools: list = None) -> dict:
        kwargs = {"model": self.model, "messages": messages}
        if tools:
            kwargs["tools"] = tools
        response = self.client.chat(**kwargs)
        msg = response.get("message", {})
        
        parsed_tools = []
        for call in msg.get("tool_calls", []):
            parsed_tools.append({
                "name": call["function"]["name"],
                "arguments": call["function"]["arguments"]
            })
        return {"content": msg.get("content", "") or "", "tool_calls": parsed_tools}

    def _call_openai(self, messages: list, tools: list = None) -> dict:
        kwargs = {"model": self.model, "messages": messages}
        if tools:
            kwargs["tools"] = tools
        response = self.client.chat.completions.create(**kwargs)
        choice = response.choices[0].message
        
        parsed_tools = []
        if choice.tool_calls:
            for call in choice.tool_calls:
                parsed_tools.append({
                    "name": call.function.name,
                    "arguments": json.loads(call.function.arguments) if isinstance(call.function.arguments, str) else call.function.arguments
                })
        return {"content": choice.content or "", "tool_calls": parsed_tools}

    def _call_gemini(self, messages: list, tools: list = None) -> dict:
        # Conversión básica de historial a string concatenado o formato Gemini
        last_user_message = next((m["content"] for m in reversed(messages) if m["role"] == "user"), "")
        response = self.client.models.generate_content(
            model=self.model,
            contents=last_user_message
        )
        return {"content": response.text or "", "tool_calls": []}

    def _call_anthropic(self, messages: list, tools: list = None) -> dict:
        user_msgs = [m for m in messages if m["role"] != "system"]
        system_msg = next((m["content"] for m in messages if m["role"] == "system"), "")
        
        kwargs = {
            "model": self.model,
            "max_tokens": 1500,
            "messages": user_msgs
        }
        if system_msg:
            kwargs["system"] = system_msg
            
        response = self.client.messages.create(**kwargs)
        content_text = response.content[0].text if response.content else ""
        return {"content": content_text, "tool_calls": []}
```

---

## 6. Patrón 1: Router de IA (Enrutamiento por Reglas/Costos)

El **Router de IA** inspecciona la solicitud antes de disparar llamadas y elige el proveedor más conveniente según métricas como número de caracteres o tokens estimados:

```python
class AIRouter:
    @staticmethod
    def select_adapter(messages: list) -> LLMAdapter:
        """
        Regla de negocio:
        - Si el mensaje tiene < 200 caracteres -> Modelo Local Ollama (Gratis y rápido)
        - Si el mensaje tiene entre 200 y 1000 caracteres -> Gemini Flash (Económico en cloud)
        - Si es > 1000 caracteres o pide análisis avanzado -> OpenAI / Anthropic
        """
        last_message = next((m["content"] for m in reversed(messages) if m["role"] == "user"), "")
        length = len(last_message)

        if length < 200:
            print(f"[Router] Solicitud corta ({length} chars) -> Enrutando a Ollama Local")
            return LLMAdapter(provider="ollama")
        elif length <= 1000:
            print(f"[Router] Solicitud mediana ({length} chars) -> Enrutando a Gemini Flash")
            return LLMAdapter(provider="gemini")
        else:
            print(f"[Router] Solicitud compleja ({length} chars) -> Enrutando a OpenAI")
            return LLMAdapter(provider="openai")
```

---

## 7. Patrón 2: Fallback de Alta Disponibilidad

Si Ollama no está encendido o da un error de conexión, o si la API de Gemini devuelve un error `429 Too Many Requests`, el patrón **Fallback** prueba automáticamente con el siguiente proveedor de la cadena sin interrumpir al usuario.

```python
class FallbackRunner:
    # Cadena de prioridad ordenada por costo y conveniencia:
    CHAIN = [
        {"provider": "ollama", "model": os.getenv("OLLAMA_MODEL", "gemma3:4b")},
        {"provider": "gemini", "model": os.getenv("GEMINI_MODEL", "gemini-2.0-flash")},
        {"provider": "openai", "model": os.getenv("OPENAI_MODEL", "gpt-4o-mini")},
    ]

    @classmethod
    def execute_with_fallback(cls, messages: list, tools: list = None) -> dict:
        errores = []
        for step in cls.CHAIN:
            provider = step["provider"]
            model = step["model"]
            try:
                print(f"[Fallback] Probando con {provider} ({model})...")
                adapter = LLMAdapter(provider=provider, model=model)
                response = adapter.chat(messages, tools=tools)
                print(f"[Fallback] ✅ Éxito con {provider}")
                return response
            except Exception as e:
                print(f"[Fallback] ⚠️ {provider} falló: {e}")
                errores.append(f"{provider}: {str(e)}")
                continue

        raise RuntimeError(f"Todos los proveedores de la cadena fallaron:\n" + "\n".join(errores))
```

---

## 8. Patrón 3: Capa de Evaluación Inteligente (LLM-as-a-Judge)

En vez de contar caracteres con reglas rígidas, se usa un modelo ultra-liviano (por ejemplo, `gemma3:4b` en Ollama o `gemini-2.0-flash`) como juez para clasificar la dificultad semántica del prompt.

### Prompt de Clasificación del Juez:
```text
Eres un evaluador de complejidad de tareas de IA.
Analiza la intención del usuario y clasifícala estrictamente en:
- "FACIL": Saludos, preguntas directas de correo, consultas puntuales de perfil, confirmaciones.
- "DIFICIL": Análisis profundo de hilos de correos, resúmenes ejecutivos con múltiples criterios, redacción formal persuasiva o deducción de tareas pendientes.

Responde ÚNICAMENTE un JSON: {"dificultad": "FACIL"} o {"dificultad": "DIFICIL"}
```

### Implementación del Evaluador Inteligente:

```python
class SmartJudge:
    def __init__(self):
        # El juez siempre debe ser el modelo más veloz y económico
        eval_provider = os.getenv("EVALUATOR_PROVIDER", "ollama")
        eval_model = os.getenv("EVALUATOR_MODEL", "gemma3:4b")
        self.judge_adapter = LLMAdapter(provider=eval_provider, model=eval_model)

    def classify_task(self, prompt: str) -> str:
        messages = [
            {
                "role": "system",
                "content": (
                    'Clasifica la solicitud en "FACIL" o "DIFICIL". '
                    'Responde SOLO un JSON con formato: {"dificultad": "FACIL"} o {"dificultad": "DIFICIL"}'
                )
            },
            {"role": "user", "content": prompt}
        ]
        try:
            res = self.judge_adapter.chat(messages)
            text = res["content"].strip()
            # Limpiar posibles bloques ```json ```
            if "```" in text:
                text = text.split("```")[1].replace("json", "").strip()
            data = json.loads(text)
            return data.get("dificultad", "FACIL").upper()
        except Exception as e:
            print(f"[Judge] Error clasificando ({e}), usando FACIL por defecto.")
            return "FACIL"

    def select_best_model(self, user_prompt: str) -> LLMAdapter:
        dificultad = self.classify_task(user_prompt)
        print(f"[Judge] Veredicto de dificultad: {dificultad}")

        if dificultad == "FACIL":
            # Usar modelo local gratuito o Gemini Flash
            return LLMAdapter(provider="ollama")
        else:
            # Escalar a OpenAI o Gemini 2.0 Flash
            return LLMAdapter(provider="openai")
```

---

## 9. Integración Completa en `client.py`

Veamos cómo queda el método `chat()` en [`client.py`](../client.py) sustituyendo la dependencia directa de `ollama`:

```python
# Fragmento refactorizado de client.py
from llm_adapter import LLMAdapter, AIRouter, FallbackRunner, SmartJudge

class GmailMCPClient:
    def __init__(self):
        self.mcp_server_path = "C:\\Users\\USER\\Documents\\wplanchez\\Portafolio\\Repositorios\\Langchain\\curso_mcp\\Tema_04\\gmail_mcp_server.py"
        self.judge = SmartJudge()

    # ... [métodos get_system_info, get_tools_for_llm, get_resources_as_tools permanecen iguales] ...

    async def chat(self, messages: list, strategy: str = "smart") -> str:
        """
        Procesa una conversación soportando múltiples proveedores.

        Estrategias disponibles:
        - "smart": Clasifica con el evaluador y elige el modelo óptimo.
        - "fallback": Usa cadena de redundancia (Ollama -> Gemini -> OpenAI).
        - "router": Enrutador por tamaño.
        - "default": Usa el proveedor definido en DEFAULT_LLM_PROVIDER.
        """
        async with await self._get_mcp_client() as mcp:
            tools, _ = await self.get_tools_for_llm()
            resource_tools, resource_map = await self.get_resources_as_tools()
            all_tools = tools + resource_tools

            # 1. Obtener el adaptador según la estrategia elegida
            last_prompt = next((m["content"] for m in reversed(messages) if m["role"] == "user"), "")
            
            if strategy == "smart":
                adapter = self.judge.select_best_model(last_prompt)
                response_dict = adapter.chat(messages, tools=all_tools)
            elif strategy == "fallback":
                response_dict = FallbackRunner.execute_with_fallback(messages, tools=all_tools)
            elif strategy == "router":
                adapter = AIRouter.select_adapter(messages)
                response_dict = adapter.chat(messages, tools=all_tools)
            else:
                adapter = LLMAdapter()
                response_dict = adapter.chat(messages, tools=all_tools)

            tool_calls = response_dict.get("tool_calls", [])

            # 2. Si no hubo invocación de herramientas, retornar texto directo
            if not tool_calls:
                return response_dict.get("content", "")

            # 3. Procesar llamadas a Tools o Resources del MCP
            messages.append({
                "role": "assistant",
                "content": response_dict.get("content", ""),
                "tool_calls": tool_calls
            })

            for tool_call in tool_calls:
                function_name = tool_call["name"]
                function_args = tool_call["arguments"]

                if function_name in resource_map:
                    resource_info = resource_map[function_name]
                    if "template" in resource_info:
                        uri = resource_info["template"]
                        for param in resource_info["params"]:
                            uri = uri.replace(f"{{{param}}}", str(function_args.get(param, "")))
                    else:
                        uri = resource_info["uri"]
                    function_response = await self.get_resource(uri, mcp)
                else:
                    function_response = await self.call_tool(function_name, function_args, mcp)

                messages.append({
                    "role": "tool",
                    "content": "Tool response to add to context: " + function_response,
                    "name": function_name,
                })

            # 4. Segunda llamada para que el modelo interprete el resultado de la herramienta
            # Reutilizamos el mismo adaptador o proveedor por defecto
            final_response = adapter.chat(messages)
            return final_response.get("content", "")
```

---

## 10. Ruta de Implementación y Recomendaciones

Si deseas implementar esta mejora progresivamente, te sugerimos esta ruta:

| Etapa | Meta | Proveedores involucrados | Beneficio |
|:---:|:---|:---|:---|
| **Fase 1** | Crear `llm_adapter.py` básico | Ollama + Gemini | Desacopla `ollama.chat` y permite probar Gemini 2.0 Flash sin costo elevado. |
| **Fase 2** | Incorporar `FallbackRunner` | Ollama $\rightarrow$ Gemini | Si Ollama local está apagado, la aplicación en Streamlit no crashea. |
| **Fase 3** | Incorporar `SmartJudge` | Ollama (Juez) + OpenAI / Gemini | Ahorro sustancial de tokens reservando OpenAI solo para tareas de alta dificultad. |
| **Fase 4** | Integrar Anthropic | Claude 3.5 Haiku / Sonnet | Capacidad analítica y razonamiento avanzado para correos ejecutivos complejos. |
