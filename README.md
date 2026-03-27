# Spring AI — Guía Completa de Referencia

> **Documentación exhaustiva de Spring AI 1.1.0** orientada a llevar al lector desde
> cero hasta nivel profesional certificado. Cubre teoría, práctica, ejemplos reales
> y todos los componentes del ecosistema.

---

## ¿Qué encontrarás en esta guía?

Esta guía cubre Spring AI en su totalidad, sin asumir conocimientos previos de IA.
Sí se asume conocimiento básico de **Java** y **Spring Boot**. Si sabes crear un
`@RestController` y entiendes qué es un `@Bean`, puedes seguir esta guía completa.

**Objetivo final:** al terminar esta guía serás capaz de:
- Diseñar e implementar aplicaciones empresariales con IA usando Spring AI
- Integrar cualquier proveedor de IA (OpenAI, Anthropic, Gemini, Ollama...)
- Construir sistemas RAG (búsqueda semántica en tus documentos)
- Implementar Tools (la IA ejecuta código de tu aplicación)
- Configurar memoria de conversación, streaming, multimodalidad
- Aplicar buenas prácticas de seguridad, observabilidad y testing
- Responder con soltura cualquier pregunta técnica o empresarial sobre Spring AI

---

## Ruta de aprendizaje recomendada

```
NIVEL FUNDAMENTOS (obligatorio — no saltar)
─────────────────────────────────────────────────────────────────────────
  Cap. 01 → Introducción y conceptos fundamentales de IA
  Cap. 02 → Setup del proyecto, configuración y entornos
  Cap. 03 → ChatClient — el núcleo de toda la integración
  Cap. 04 → Prompts y PromptTemplate — cómo hablarle a la IA
  Cap. 05 → Structured Output — respuestas tipadas en Java
  Cap. 06 → Streaming — respuesta en tiempo real

NIVEL INTERMEDIO (los pilares de las apps empresariales)
─────────────────────────────────────────────────────────────────────────
  Cap. 07 → Embeddings — convertir texto en vectores numéricos
  Cap. 08 → Vector Store — bases de datos para búsqueda semántica
  Cap. 09 → RAG — Retrieval Augmented Generation
  Cap. 10 → Tools / Function Calling — la IA ejecuta tu código
  Cap. 11 → Memory — historial y contexto de conversación
  Cap. 12 → Advisors — middleware de IA (interceptores)

NIVEL AVANZADO (casos de uso especializados)
─────────────────────────────────────────────────────────────────────────
  Cap. 13 → Multimodalidad — imágenes, audio, generación, TTS
  Cap. 14 → Observabilidad — métricas, trazas, logs

NIVEL PROFESIONAL (producción y calidad)
─────────────────────────────────────────────────────────────────────────
  Cap. 17 → Testing — pruebas unitarias e integración con Spring AI
  Cap. 18 → Evaluación RAG — calidad de respuestas con evaluadores LLM
  Cap. 15 → Proyecto completo — chatbot empresarial real
  Cap. 16 → Comparativa, cheatsheet y referencia rápida
```

---

## Índice completo de capítulos

### Fundamentos

| Capítulo | Título | Qué aprenderás |
|---|---|---|
| [01](docs/01_introduccion.md) | Introducción a Spring AI | Qué es, por qué usarlo, arquitectura, tokens, roles de mensajes, proveedores soportados |
| [02](docs/02_setup_y_entornos.md) | Setup y entornos | Crear proyecto, dependencias Maven, configuración dev/prod, Ollama local, API keys |
| [03](docs/03_chatclient.md) | ChatClient en profundidad | Fluent API completa, ChatModel vs ChatClient, opciones por proveedor, patrones avanzados |
| [04](docs/04_prompts_y_templates.md) | Prompts y PromptTemplate | Diseño de prompts, PromptTemplate con variables, roles, few-shot prompting |
| [05](docs/05_structured_output.md) | Structured Output | BeanOutputConverter, MapOutputConverter, validación, errores de parseo |
| [06](docs/06_streaming.md) | Streaming | Flux, Server-Sent Events, WebFlux, backpressure, integración con frontend |

### Componentes principales

| Capítulo | Título | Qué aprenderás |
|---|---|---|
| [07](docs/07_embeddings.md) | Embeddings | Qué son los vectores, EmbeddingModel, distancia coseno, casos de uso |
| [08](docs/08_vector_store.md) | Vector Store | pgvector, Chroma, Weaviate, indexar y buscar documentos |
| [09](docs/09_rag.md) | RAG | Alucinación, indexación, retrieval, QuestionAnswerAdvisor, RAG avanzado |
| [10](docs/10_tools.md) | Tools / Function Calling | @Tool, @ToolParam, ToolContext, tools encadenadas, seguridad |
| [11](docs/11_memory.md) | Memory | ChatMemory, InMemoryChatMemory, JdbcChatMemory, ventana de contexto |
| [12](docs/12_advisors.md) | Advisors | Arquitectura de advisors, built-in advisors, StreamAroundAdvisor, orden de ejecución |

### Capacidades avanzadas

| Capítulo | Título | Qué aprenderás |
|---|---|---|
| [13](docs/13_multimodalidad.md) | Multimodalidad | Visión, generación de imágenes (DALL-E), transcripción (Whisper), TTS, Ollama visión |
| [14](docs/14_observabilidad.md) | Observabilidad | Micrometer, trazas, métricas de tokens, Prometheus, Grafana |

### Producción y referencia

| Capítulo | Título | Qué aprenderás |
|---|---|---|
| [17](docs/17_testing.md) | Testing con Spring AI | MockChatModel, @TestConfiguration, TestContainers con pgvector, testear advisors y tools |
| [18](docs/18_evaluacion_rag.md) | Evaluación de respuestas RAG | RelevancyEvaluator, FaithfulnessEvaluator, EvaluationRequest, suite de evaluación automática |
| [15](docs/15_proyecto_completo.md) | Proyecto completo | Chatbot empresarial real: RAG + Tools + Memory + Streaming + Observabilidad |
| [16](docs/16_comparativa_y_cheatsheet.md) | Comparativa y cheatsheet | Spring AI vs alternativas, jerarquía de clases, tabla de decisión, referencia rápida |

---

## Mapa de dependencias entre conceptos

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   MAPA DE DEPENDENCIAS                                       │
│                                                                              │
│  [01 Introducción]                                                           │
│       │                                                                      │
│       ▼                                                                      │
│  [02 Setup] ──────────────────────────────────────────────────────────────┐ │
│       │                                                                    │ │
│       ▼                                                                    │ │
│  [03 ChatClient] ──────────────────────────────────────────────────────┐  │ │
│       │                │                  │                             │  │ │
│       ▼                ▼                  ▼                             │  │ │
│  [04 Prompts]    [05 Structured]    [06 Streaming]                      │  │ │
│                        │                                                │  │ │
│                        ▼                                                │  │ │
│  [07 Embeddings] ──→ [08 VectorStore] ──→ [09 RAG]                     │  │ │
│                                              │                          │  │ │
│  [10 Tools] ────────────────────────────────┤                          │  │ │
│  [11 Memory] ───────────────────────────────┤                          │  │ │
│  [12 Advisors] ─────────────────────────────┘                          │  │ │
│                              │                                          │  │ │
│                              ▼                                          │  │ │
│  [13 Multimodalidad]    [14 Observabilidad]    [17 Testing]             │  │ │
│                                                      │                  │  │ │
│                                               [18 Eval. RAG]           │  │ │
│                              │                                          │  │ │
│                              ▼                                          ▼  ▼ │
│                     [15 Proyecto Completo] ◄────────────────────────────┘  │ │
│                              │                                              │ │
│                              ▼                                              │ │
│                     [16 Comparativa / Cheatsheet] ◄─────────────────────── ┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerrequisitos

### Conocimientos necesarios
- **Java 17+**: clases, interfaces, generics, lambdas, records
- **Spring Boot básico**: `@SpringBootApplication`, `@RestController`, `@Service`, `@Bean`, inyección de dependencias
- **Maven**: estructura de `pom.xml`, dependencias, compilación

### Conocimientos NO necesarios
- No necesitas saber matemáticas ni álgebra lineal
- No necesitas haber trabajado con IA anteriormente
- No necesitas conocer Python ni librerías de ML

### Entorno de trabajo recomendado
```
JDK 21+
Maven 3.9+
IDE: IntelliJ IDEA (recomendado) o VS Code con extensión Java
Ollama instalado (para desarrollo local sin coste)
```

---

## Cómo instalar Ollama (desarrollo local)

Ollama permite ejecutar modelos de IA en tu máquina **sin coste ni internet**.
Es la opción recomendada para aprender y desarrollar.

```bash
# Linux / WSL
curl -fsSL https://ollama.com/install.sh | sh

# Descargar modelos necesarios
ollama pull llama3.2          # modelo de chat ~2GB
ollama pull nomic-embed-text  # modelo de embeddings ~274MB

# Verificar que funciona
ollama run llama3.2 "Hola, ¿funcionas?"

# Ollama expone una API REST en:
# http://localhost:11434
```

---

## Versiones cubiertas en esta guía

| Componente | Versión |
|---|---|
| Spring AI | **1.1.0** |
| Spring Boot | 3.3+ |
| Java | 21 |
| Ollama | 0.3+ (para desarrollo local) |

> Esta guía cubre la API estable de Spring AI 1.1.0. Los cambios más importantes
> respecto a versiones anteriores se indican donde corresponde con marcas **[1.1.0]**.

---

## Convenciones de esta guía

```
💡 Consejo o truco útil
⚠️  Advertencia — error frecuente o comportamiento no obvio
✅  Buena práctica recomendada
❌  Anti-patrón — lo que NO debes hacer
🔑  Concepto clave que debes memorizar
📌  Referencia a otro capítulo relacionado
```

---

*Stack: Java 21 · Spring Boot 3.x · Spring AI 1.1.0 · Maven · Ollama*
