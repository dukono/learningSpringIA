<!-- navegación -->
> **[← Inicio](00_indice.md)** | **[Siguiente: Setup y entornos →](02_setup_y_entornos.md)**

---

# Capítulo 01 — Introducción a Spring AI

> Este capítulo explica qué es Spring AI, por qué usarlo, su arquitectura interna
> y los conceptos fundamentales que debes dominar antes de escribir una línea de código.
> **Sin estos fundamentos el resto de la guía no tiene contexto.**

## Contenido de este capítulo

1. [¿Qué es Spring AI?](#1-qué-es-spring-ai)
2. [¿Por qué Spring AI y no llamar a la API directamente?](#2-por-qué-spring-ai-y-no-llamar-a-la-api-directamente)
3. [Arquitectura y conceptos clave](#3-arquitectura-y-conceptos-clave)
4. [Modelos de IA soportados](#4-modelos-de-ia-soportados)

---

## 1. ¿Qué es Spring AI?

Spring AI es el módulo oficial del ecosistema Spring para **integrar modelos de inteligencia
artificial** en aplicaciones Java. Fue creado por el equipo de Spring (VMware/Broadcom) y
lanzado en versión estable en 2024.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ¿QUÉ ES SPRING AI?                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Tu App Spring Boot                                                        │
│        │                                                                    │
│        ▼                                                                    │
│   ┌──────────────┐                                                          │
│   │  Spring AI   │  ← una capa de abstracción                              │
│   └──────┬───────┘                                                          │
│          │                                                                  │
│    ┌─────┼──────────────────────────┐                                       │
│    ▼     ▼                          ▼                                       │
│  OpenAI  Anthropic  Azure OpenAI  Google Gemini  Ollama (local) ...         │
│                                                                             │
│  La misma API de Spring AI → cualquier proveedor de IA                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### La idea central

Spring AI hace con los modelos de IA lo mismo que **Spring Data** hace con las bases de datos:
- Con Spring Data escribes `UserRepository` y funciona con MySQL, PostgreSQL, MongoDB...
- Con Spring AI escribes `ChatClient` y funciona con OpenAI, Claude, Gemini, Ollama...

**Cambia el proveedor → cambia solo la dependencia y la config. El código no cambia.**

### ¿Qué puede hacer Spring AI?

| Capacidad            | Descripción                                                    |
|----------------------|----------------------------------------------------------------|
| **Chat / LLM**       | Conversaciones con GPT-4, Claude, Gemini, Llama...             |
| **Embeddings**       | Convertir texto en vectores numéricos para búsqueda semántica  |
| **Generación imagen**| Crear imágenes con DALL-E, Stable Diffusion                    |
| **Transcripción**    | Audio → texto con Whisper                                      |
| **Text-to-Speech**   | Texto → audio con la voz que elijas                           |
| **RAG**              | Buscar en tus documentos y responder con contexto propio       |
| **Function Calling** | La IA decide cuándo llamar a tu código Java                    |
| **Streaming**        | Recibir la respuesta palabra a palabra (como ChatGPT)          |

---

## 2. ¿Por qué Spring AI y no llamar a la API directamente?

### Opción A — Sin Spring AI (crudo)

```java
// Tienes que hacer esto manualmente para cada proveedor:
HttpClient client = HttpClient.newHttpClient();
String body = """
    {
        "model": "gpt-4o",
        "messages": [{"role": "user", "content": "Hola"}]
    }
    """;
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.openai.com/v1/chat/completions"))
    .header("Authorization", "Bearer " + apiKey)
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(body))
    .build();
// parsear JSON manualmente...
// manejar errores de red...
// reintentos...
// cada proveedor tiene un JSON diferente...
```

### Opción B — Con Spring AI

```java
// Esto funciona con CUALQUIER proveedor
@Autowired
ChatClient chatClient;

String respuesta = chatClient.prompt()
    .user("Hola")
    .call()
    .content();
```

### Ventajas reales

```
┌──────────────────────────────────────────────────────────────────────────┐
│  SIN Spring AI                     │  CON Spring AI                      │
├────────────────────────────────────┼─────────────────────────────────────┤
│  HTTP manual para cada proveedor   │  API unificada                       │
│  Parsear JSON a mano               │  Objetos Java automáticamente        │
│  Gestionar reintentos tú mismo     │  Retry automático configurable       │
│  Streaming complejo con SSE        │  .stream() y ya                     │
│  Sin integración con Spring DI     │  @Autowired como cualquier bean      │
│  Sin observabilidad                │  Micrometer + métricas incluidas     │
│  Vector stores → cada uno distinto │  API unificada para todos            │
│  Function calling → código manual  │  @Description en métodos Java        │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Arquitectura y conceptos clave

Antes de escribir código, es fundamental entender el vocabulario de Spring AI:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONCEPTOS CLAVE DE SPRING AI                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ChatClient (o ChatModel)                                           │   │
│  │  ↳ El objeto principal. Envía mensajes y recibe respuestas.        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  Prompt          │  │  Message         │  │  ChatOptions             │  │
│  │  ↳ La petición   │  │  ↳ Rol + texto   │  │  ↳ temperatura, tokens  │  │
│  │    completa      │  │    (user/system/ │  │    máximos, modelo...   │  │
│  │                  │  │     assistant)   │  │                          │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘  │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  ChatResponse    │  │  EmbeddingModel  │  │  VectorStore             │  │
│  │  ↳ La respuesta  │  │  ↳ Convierte     │  │  ↳ BD de vectores        │  │
│  │    del modelo    │  │    texto→vector  │  │    para búsqueda IA      │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘  │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  Advisor         │  │  PromptTemplate  │  │  Memory                  │  │
│  │  ↳ Interceptor   │  │  ↳ Plantilla de  │  │  ↳ Historial de         │  │
│  │    de la IA      │  │    prompt        │  │    conversación         │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **`ChatClient` vs `ChatModel` — diferencia clave que debes entender:**
>
> `ChatModel` es la interfaz de bajo nivel, directamente ligada al proveedor.
> `ChatClient` es una capa superior que envuelve a `ChatModel` y le añade:
> Advisors, memoria de conversación, RAG automático, defaults de system prompt, etc.
>
> **Regla práctica: usa siempre `ChatClient`. `ChatModel` solo si necesitas acceso
> directo a la API sin ninguna capa intermedia.**

### ¿Qué es un Token? — concepto fundamental

Un **token** es la unidad mínima que procesa el modelo. NO es una palabra ni un carácter:
es un fragmento de texto. **Debes entenderlo porque determina el coste y los límites.**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CÓMO SE DIVIDEN LOS TOKENS                                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  "Hello world"           → ["Hello", " world"]            = 2 tokens    │
│  "Spring Boot"           → ["Spring", " Boot"]            = 2 tokens    │
│  "incomprehensibilities" → ["inc","ompreh","ens","ibilities"] = 4 tokens │
│                            (palabras largas/raras → más tokens)          │
│                                                                          │
│  REGLA APROXIMADA:                                                       │
│  1 token ≈ 0.75 palabras en inglés                                       │
│  1 token ≈ 0.50 palabras en español  (el español gasta más tokens)       │
│  1000 tokens ≈ 750 palabras ≈ 3 páginas de texto                         │
│                                                                          │
│  LÍMITES DE CONTEXTO (context window = todo lo que el modelo "ve"):      │
│  GPT-4o:          128.000 tokens                                         │
│  Claude 3.5:      200.000 tokens                                         │
│  Gemini 1.5 Pro: 1.000.000 tokens                                        │
│                                                                          │
│  IMPACTO EN COSTES (2026):                                               │
│  GPT-4o-mini:  $0.15 / 1M tokens entrada  +  $0.60 / 1M salida          │
│  GPT-4o:       $5.00 / 1M tokens entrada  + $15.00 / 1M salida          │
│  Claude 3.5:   $3.00 / 1M tokens entrada  + $15.00 / 1M salida          │
│                                                                          │
│  Ejemplo real: 10 turnos de chat ≈ 2000 tokens ≈ $0.001 con GPT-4o-mini │
└──────────────────────────────────────────────────────────────────────────┘
```

**¿Por qué importa en la práctica?**
- El historial de conversación ocupa tokens → puede llegar al límite y encarecer
- Un PDF de 100 páginas ≈ 50.000 tokens → no puedes meterlo entero en un prompt
- Por eso existe RAG: para no meter todo el documento, solo los fragmentos relevantes

### Los tres roles de un mensaje

```
USER      → Lo que escribe el usuario humano
SYSTEM    → Instrucciones del sistema (el "comportamiento" del asistente)
ASSISTANT → Respuestas previas del modelo (para dar contexto/historia)
```

Cómo se mapea lo que ves en ChatGPT con lo que envía Spring AI al modelo:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  LO QUE VES EN CHATGPT        │  LO QUE ENVÍA Spring AI AL MODELO       │
├───────────────────────────────┼──────────────────────────────────────────┤
│  (invisible al usuario)       │  [SYSTEM]                                │
│                               │  "Eres un asistente experto en Java..."  │
│                               │                                          │
│  Tú: "¿Qué es un Stream?"    │  [USER]                                  │
│                               │  "¿Qué es un Stream en Java?"            │
│                               │                                          │
│  Bot: "Un Stream es..."       │  [ASSISTANT]  (generado por el modelo)  │
│                               │  "Un Stream es una secuencia..."         │
│                               │                                          │
│  Tú: "¿Cómo uso filter()?"   │  [USER]  (nueva pregunta con historial)  │
│                               │  "¿Y cómo se usa filter()?"              │
└───────────────────────────────┴──────────────────────────────────────────┘
```

---

## 4. Modelos de IA soportados

Spring AI soporta múltiples proveedores con el mismo código:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    PROVEEDORES SOPORTADOS (2026)                         │
├────────────────────┬────────────────────────────────────────────────────┤
│  PROVEEDOR         │  MODELOS DESTACADOS                                 │
├────────────────────┼────────────────────────────────────────────────────┤
│  OpenAI            │  GPT-4o, GPT-4o-mini, o1, o3-mini                  │
│  Anthropic         │  Claude 3.5 Sonnet, Claude 3 Opus/Haiku            │
│  Google            │  Gemini 2.0 Flash, Gemini 1.5 Pro                  │
│  Azure OpenAI      │  Todos los modelos OpenAI vía Azure                │
│  AWS Bedrock       │  Claude, Llama, Titan, Mistral                     │
│  Ollama            │  Llama 3.2, Mistral, Gemma, Phi-3 (LOCAL)         │
│  Mistral AI        │  Mistral Large, Mixtral 8x7B                       │
│  Groq              │  Llama ultra rápido (inferencia acelerada)          │
│  Hugging Face      │  Modelos open source                                │
│  ONNX              │  Modelos locales optimizados para CPU/GPU           │
└────────────────────┴────────────────────────────────────────────────────┘
```

> **Ollama** es el más importante para desarrollo local: descarga modelos y los ejecuta
> en tu máquina sin coste ni internet. Perfecto para pruebas y desarrollo.
>
> Para instalar Ollama y descargar un modelo:
> ```bash
> # Instalar Ollama (Linux/Mac)
> curl -fsSL https://ollama.com/install.sh | sh
>
> # Descargar y arrancar un modelo
> ollama pull llama3.2       # ~2GB — modelo general
> ollama pull nomic-embed-text  # modelo de embeddings
>
> # Verificar que funciona
> ollama run llama3.2 "Hola, ¿qué tal?"
> ```

---


---

> **[← Inicio](00_indice.md)** | **[Siguiente: Setup y entornos →](02_setup_y_entornos.md)**
