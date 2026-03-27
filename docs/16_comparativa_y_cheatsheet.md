<!-- navegación -->
> **[← Proyecto Completo](15_proyecto_completo.md)** | **[← Inicio](../README.md)** | **[Siguiente: Testing →](17_testing.md)**

---

# Capítulo 16 — Referencia rápida y comparativa

> Cheatsheet completo de Spring AI 1.1.0: jerarquía de clases, implementaciones,
> firmas de API, tabla de decisión y comparativa con otras librerías.

---

## 16.1 Jerarquía de interfaces y clases clave

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  INTERFACES PRINCIPALES                                                       │
├──────────────────────────┬────────────────────────────────────────────────── │
│  ChatClient              │  Punto de entrada. Fluent API. SIEMPRE usar este.  │
│  ChatModel               │  Low-level. Liga al proveedor. Rara vez directo.   │
│  EmbeddingModel          │  Convierte texto → float[]. Un impl. por proveedor.│
│  VectorStore             │  Almacena y busca vectores. Múltiples impls.       │
│  ChatMemory              │  Historial de conversación. Múltiples impls.       │
│  DocumentReader          │  Lee ficheros y devuelve List<Document>.            │
│  DocumentTransformer     │  Transforma List<Document> → List<Document>.        │
│  Advisor                 │  Interceptor (CallAroundAdvisor/StreamAroundAdvisor)│
└──────────────────────────┴────────────────────────────────────────────────── ┘

RELACIÓN ChatClient → ChatModel:
  ChatClient.Builder  ←autowired por Spring AI
       │ build()
       ▼
  ChatClient  (usa internamente ChatModel para llamar al proveedor)
       │ .prompt().user("...").call()
       ▼
  ChatModel.call(Prompt)  →  ChatResponse
```

---

## 16.2 Implementaciones disponibles

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  VectorStore — implementaciones                                              │
├──────────────────────────┬────────────────────────────────────────────────── │
│  SimpleVectorStore        │  En memoria. Solo tests/demos.                   │
│  PgVectorStore            │  PostgreSQL + extensión pgvector. Recomendado.   │
│  ChromaVectorStore        │  Chroma OS. Ideal para desarrollo.               │
│  PineconeVectorStore      │  Cloud managed. Alta escala.                     │
│  WeaviateVectorStore      │  Cloud y self-hosted.                            │
│  RedisVectorStore         │  RedisSearch. Baja latencia.                     │
│  QdrantVectorStore        │  Open source. Alto rendimiento.                  │
│  MilvusVectorStore        │  Enterprise. Millones de vectores.               │
└──────────────────────────┴────────────────────────────────────────────────── ┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  ChatMemory — implementaciones                                               │
├──────────────────────────┬────────────────────────────────────────────────── │
│  InMemoryChatMemory       │  RAM. Pierde todo al reiniciar. Tests/demos.     │
│  JdbcChatMemory           │  SQL (PostgreSQL, MySQL, H2). Producción.        │
│  CassandraChatMemory      │  Apache Cassandra. Alta escala.                  │
│  RedisChatMemory          │  Redis. TTL configurable. Baja latencia.         │
└──────────────────────────┴────────────────────────────────────────────────── ┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  EmbeddingModel — implementaciones                                           │
├──────────────────────────┬────────────────────────────────────────────────── │
│  OpenAiEmbeddingModel     │  text-embedding-3-small (1536 dims, recomendado) │
│                           │  text-embedding-3-large (3072 dims)              │
│  OllamaEmbeddingModel     │  nomic-embed-text (768), mxbai-embed-large (1024)│
│  AnthropicEmbeddingModel  │  Pendiente soporte nativo en 1.1.0               │
│  AzureOpenAiEmbeddingModel│  text-embedding-ada-002 vía Azure                │
└──────────────────────────┴────────────────────────────────────────────────── ┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  Advisors integrados en Spring AI                                            │
├──────────────────────────┬────────────────────────────────────────────────── │
│  QuestionAnswerAdvisor    │  RAG automático. Busca en VectorStore y añade     │
│                           │  chunks al prompt como contexto.                 │
│  MessageChatMemoryAdvisor │  Memoria como mensajes [USER]/[ASSISTANT].        │
│  PromptChatMemoryAdvisor  │  Memoria como texto dentro del SYSTEM prompt.    │
│  SafeGuardAdvisor         │  Filtra palabras prohibidas en el input.          │
│  SimpleLoggerAdvisor      │  Log DEBUG de prompt y respuesta completos.      │
└──────────────────────────┴────────────────────────────────────────────────── ┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  DocumentReader — lectores disponibles                                       │
├──────────────────────────┬────────────────────────────────────────────────── │
│  PagePdfDocumentReader    │  PDF, página a página. spring-ai-pdf-document-reader│
│  TextReader               │  Texto plano (.txt).                             │
│  MarkdownDocumentReader   │  Markdown. spring-ai-markdown-document-reader.   │
│  JsoupDocumentReader      │  HTML. spring-ai-jsoup-document-reader.          │
│  JsonDocumentReader       │  JSON.                                           │
└──────────────────────────┴────────────────────────────────────────────────── ┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  DocumentTransformer — transformadores de ingesta                            │
├──────────────────────────┬────────────────────────────────────────────────── │
│  TokenTextSplitter        │  Divide documentos en chunks por tokens.         │
│  KeywordMetadataEnricher  │  Extrae keywords con LLM y las añade a metadata. │
│  SummaryMetadataEnricher  │  Genera resumen con LLM y lo añade a metadata.   │
│  [custom]                 │  implements DocumentTransformer { apply(docs) }  │
└──────────────────────────┴────────────────────────────────────────────────── ┘
```

---

## 16.3 Tabla de decisión — ¿qué componente usar?

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  NECESITO...                          USAR...                                │
├──────────────────────────────────────┬───────────────────────────────────────┤
│  Chatbot básico                      │  ChatClient + InMemoryChatMemory       │
│  Chatbot que recuerde conversación   │  ChatClient + MessageChatMemoryAdvisor │
│  Chatbot sobre mis documentos        │  ChatClient + QuestionAnswerAdvisor    │
│                                      │  + VectorStore + EmbeddingModel        │
│  Chatbot multi-usuario en producción │  ChatClient + JdbcChatMemory           │
│                                      │  + conversationId por usuario          │
│  Respuesta estructurada (JSON/objeto)│  .call().entity(MiClase.class)         │
│  Respuesta en tiempo real (streaming)│  .stream().content() → Flux<String>    │
│  Llamar a funciones/APIs externas    │  @Tool en métodos + .defaultTools()    │
│  Guardar vectores para búsqueda      │  VectorStore + EmbeddingModel          │
│  Buscar documentos similares         │  VectorStore.similaritySearch()         │
│  Filtrar por categoría/tenant        │  SearchRequest.filterExpression(...)   │
│  Métricas de tokens y latencia       │  Micrometer (automático con starter)   │
│  Test sin llamadas reales a la API   │  MockChatModel (spring-ai-test)        │
│  Dev local sin API key               │  Ollama + llama3.2                     │
│  Prod con máxima calidad             │  OpenAI gpt-4o / Anthropic claude-3-5  │
└──────────────────────────────────────┴───────────────────────────────────────┘
```

---

## 16.4 API de ChatClient — firmas completas

```java
// ─── CONSTRUCCIÓN ────────────────────────────────────────────────────────────

// Inyectar el Builder (auto-configurado por Spring AI)
@Autowired ChatClient.Builder builder;

ChatClient client = builder
    .defaultSystem("Instrucciones permanentes del sistema")
    .defaultAdvisors(
        new MessageChatMemoryAdvisor(memory),
        new QuestionAnswerAdvisor(vectorStore),
        new SimpleLoggerAdvisor()
    )
    .defaultTools(misTools)           // instancia con @Tool
    .defaultOptions(OpenAiChatOptions.builder()
        .model("gpt-4o-mini")
        .temperature(0.7)
        .maxTokens(2048)
        .build())
    .build();

// ─── LLAMADA BÁSICA ──────────────────────────────────────────────────────────

String texto = client.prompt()
    .system("Rol específico para esta llamada")   // sobreescribe defaultSystem
    .user("Pregunta del usuario")
    .call()
    .content();                                    // → String

// ─── RESPUESTA COMPLETA (tokens, modelo, finishReason) ───────────────────────

ChatResponse response = client.prompt()
    .user("Pregunta")
    .call()
    .chatResponse();                               // → ChatResponse

String contenido   = response.getResult().getOutput().getContent();
String finishReason = response.getResult().getMetadata().getFinishReason();
Long inputTokens   = response.getMetadata().getUsage().getPromptTokens();
Long outputTokens  = response.getMetadata().getUsage().getGenerationTokens();

// ─── RESPUESTA TIPADA ────────────────────────────────────────────────────────

MiRecord obj = client.prompt().user("...").call()
    .entity(MiRecord.class);                       // → MiRecord

List<MiRecord> lista = client.prompt().user("...").call()
    .entity(new ParameterizedTypeReference<List<MiRecord>>() {}); // → List<MiRecord>

// ─── STREAMING ───────────────────────────────────────────────────────────────

Flux<String> tokens = client.prompt()
    .user("Pregunta")
    .stream()
    .content();                                    // → Flux<String>

Flux<ChatResponse> responses = client.prompt()
    .user("Pregunta")
    .stream()
    .chatResponse();                               // → Flux<ChatResponse> (con metadata)

// ─── MENSAJES EXPLÍCITOS ─────────────────────────────────────────────────────

String resultado = client.prompt()
    .messages(
        new SystemMessage("Eres un asistente"),
        new UserMessage("Hola"),
        new AssistantMessage("Hola, ¿en qué puedo ayudarte?"),
        new UserMessage("Tengo una duda sobre JWT")
    )
    .call()
    .content();

// ─── TOOLS EN UNA LLAMADA ────────────────────────────────────────────────────

String resultado = client.prompt()
    .user("¿Cuál es el precio de BTC?")
    .tools(misTools)                  // tool solo para esta llamada
    .toolContext(Map.of("userId", userId))  // contexto accesible desde los tools
    .call()
    .content();

// ─── ADVISORS EN UNA LLAMADA ─────────────────────────────────────────────────

String resultado = client.prompt()
    .user("Pregunta")
    .advisors(a -> a.param(
        AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId
    ))
    .call()
    .content();
```

---

## 16.5 FilterExpression — referencia de sintaxis

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  OPERADORES          EJEMPLO                                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│  ==   igual          filename == 'informe.pdf'                               │
│  !=   distinto       status != 'borrador'                                    │
│  >    mayor          year > 2023                                              │
│  >=   mayor/igual    year >= 2024                                             │
│  <    menor          priority < 5                                             │
│  <=   menor/igual    size <= 100                                              │
│  in   en lista       dept in ['RRHH', 'Legal']                               │
│  nin  no en lista    status nin ['borrador', 'obsoleto']                     │
│  &&   AND            dept == 'RRHH' && year >= 2024                          │
│  ||   OR             dept == 'RRHH' || dept == 'Legal'                       │
│  !    NOT            !(confidential == true)                                 │
└──────────────────────────────────────────────────────────────────────────────┘

String literal:  'valor'           (comillas simples)
Número:          year >= 2024      (sin comillas)
Booleano:        active == true    (sin comillas)
Lista:           dept in ['A','B'] (corchetes, strings con comillas simples)
```

```java
// String directo
SearchRequest.builder().query("...").filterExpression("dept == 'RRHH'").build()

// FilterExpressionBuilder (recomendado con valores dinámicos)
FilterExpressionBuilder b = new FilterExpressionBuilder();
Filter.Expression expr = b.and(b.eq("tenantId", tenantId), b.gte("year", 2024));
SearchRequest.builder().query("...").filterExpression(expr).build()
```

---

## 16.6 Anotaciones y clases de configuración rápida

```java
// ─── TOOLS ───────────────────────────────────────────────────────────────────

@Tool(description = "Descripción que lee el modelo — sé preciso")
public String miTool(@ToolParam(description = "qué es este parámetro") String param) {
    return "resultado";
}

// Registrar en ChatClient
builder.defaultTools(instanciaConTools)

// Acceder al ToolContext desde un tool
@Tool(description = "...")
public String toolConContexto(ToolContext ctx) {
    String userId = (String) ctx.getContext().get("userId");
    return "...";
}

// ─── STRUCTURED OUTPUT ───────────────────────────────────────────────────────

record Producto(String nombre, double precio, @JsonProperty("en_stock") boolean enStock) {}

// Bean output: Spring AI añade instrucciones de formato al prompt automáticamente
Producto p = chatClient.prompt().user("...").call().entity(Producto.class);
List<Producto> ps = chatClient.prompt().user("...").call()
    .entity(new ParameterizedTypeReference<List<Producto>>() {});

// ─── MEMORY ──────────────────────────────────────────────────────────────────

@Bean
public ChatMemory chatMemory() { return new InMemoryChatMemory(); }   // dev

@Bean @Profile("prod")
public ChatMemory jdbcMemory(JdbcTemplate jdbc) { return new JdbcChatMemory(jdbc); }

MessageChatMemoryAdvisor.builder(memory)
    .conversationHistoryWindowSize(10)   // últimos 10 turnos (20 mensajes)
    .build()

// ─── VECTOR STORE pgvector ────────────────────────────────────────────────────

// application.yml
// spring.ai.vectorstore.pgvector.initialize-schema: true
// spring.ai.vectorstore.pgvector.dimensions: 1536
// spring.ai.vectorstore.pgvector.index-type: HNSW
// spring.ai.vectorstore.pgvector.distance-type: COSINE_DISTANCE

// ─── OBSERVABILIDAD ──────────────────────────────────────────────────────────

// application.yml (Spring Boot 3.x)
// management.tracing.sampling.probability: 1.0
// management.zipkin.tracing.endpoint: http://localhost:9411/api/v2/spans

// Métricas disponibles en /actuator/metrics:
// gen_ai.client.token.usage  (tags: gen_ai.token.type=input|output)
// gen_ai.client.operation.duration

// ─── TESTING ─────────────────────────────────────────────────────────────────

// MockChatModel — siempre una respuesta
MockChatModel mockModel = new MockChatModel("respuesta fija");

// MockChatModel — respuestas secuenciales
MockChatModel mockModel = new MockChatModel(List.of("resp 1", "resp 2", "resp 3"));

// MockEmbeddingModel
MockEmbeddingModel mockEmbed = new MockEmbeddingModel(1536);  // dimensión

// TestConfiguration
@TestConfiguration
public class TestAiConfig {
    @Bean @Primary
    public ChatModel mockChatModel() { return new MockChatModel("respuesta"); }
}
```

---

## 16.7 Diagrama de jerarquía de excepciones

```
AiException  (raíz)
├── NonTransientAiException         → NO reintentar (error permanente)
│   ├── BadRequestException         → Petición mal formada (parámetros inválidos, 400)
│   ├── AuthenticationException     → API key inválida o sin permisos (401/403)
│   └── ContentFilterException      → Contenido bloqueado por políticas del proveedor
└── TransientAiException            → SÍ reintentar (error temporal)
    ├── RateLimitException          → Límite de peticiones alcanzado (429)
    └── ModelNotAvailableException  → Modelo no disponible temporalmente (503)
```

---

## 16.8 Configuración YAML de referencia

```yaml
spring:
  ai:
    # ── Proveedor de chat ──────────────────────────────────────────────────────
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
          max-tokens: 2048
      embedding:
        options:
          model: text-embedding-3-small

    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
      embedding:
        options:
          model: nomic-embed-text

    # ── Retry automático ────────────────────────────────────────────────────────
    retry:
      max-attempts: 3
      initial-interval: 1000     # ms
      multiplier: 2.0
      max-interval: 60000        # ms

    # ── Vector Store (pgvector) ──────────────────────────────────────────────────
    vectorstore:
      pgvector:
        initialize-schema: true
        dimensions: 1536
        index-type: HNSW
        distance-type: COSINE_DISTANCE

    # ── Observabilidad ────────────────────────────────────────────────────────────
    chat:
      observations:
        include-prompt: false
        include-completion: false
    embedding:
      observations:
        include-input: false

management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

---

## 16.9 Comparativa Spring AI vs otras librerías

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SPRING AI vs ALTERNATIVAS                                                   │
├────────────────┬───────────────┬────────────────────────────────────────────┤
│  LIBRERÍA      │  LENGUAJE     │  CUÁNDO USARLA                             │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  Spring AI     │  Java         │  Apps Spring Boot existentes — integración │
│  1.1.0         │               │  natural, mismo ecosistema, mismo IoC      │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  LangChain4j   │  Java         │  Más maduro en funciones de agentes,       │
│                │               │  más explícito, más verbose. Alternativa   │
│                │               │  si los agentes son el core del producto.  │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  LangChain     │  Python       │  El estándar en Python. Más documentación, │
│                │               │  más ejemplos, más librerías externas.     │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  LlamaIndex    │  Python       │  Especializado en RAG y procesamiento de   │
│                │               │  documentos complejos.                     │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  Semantic      │  C# / Python  │  Microsoft stack, Azure-first. Ideal si    │
│  Kernel        │               │  el equipo ya está en .NET/Azure.          │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  OpenAI SDK    │  Cualquiera   │  Acceso directo a la API de OpenAI, máximo │
│                │               │  control, pero atado a un proveedor.       │
└────────────────┴───────────────┴────────────────────────────────────────────┘
```

**Diferencias API clave Spring AI vs LangChain4j:**

```
Spring AI                          LangChain4j
──────────────────────────────     ──────────────────────────────
ChatClient (fluent, por defecto)   AiServices (interfaces Java)
@Tool en método                    @Tool en interfaz
Advisor (interceptor)              AiServiceContext / callbacks
VectorStore (interfaz unificada)   EmbeddingStore (interfaz similar)
```

**Regla simple**: Si ya usas Spring Boot → Spring AI. Si los agentes son el core
del producto y necesitas más control → LangChain4j. Si el equipo es Python → LangChain.

---

*Actualizado Marzo 2026 — Spring AI 1.1.0 / Spring Boot 3.3+ / Java 21*

---

> **[← Proyecto Completo](15_proyecto_completo.md)** | **[← Inicio](../README.md)** | **[Siguiente: Testing →](17_testing.md)**

