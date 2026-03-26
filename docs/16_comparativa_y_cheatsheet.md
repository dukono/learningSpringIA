<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 21. Comparativa Spring AI vs otras librerías

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SPRING AI vs ALTERNATIVAS                                                   │
├────────────────┬───────────────┬────────────────────────────────────────────┤
│  LIBRERÍA      │  LENGUAJE     │  CUÁNDO USARLA                             │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  Spring AI     │  Java         │  Apps Spring Boot existentes — integración │
│                │               │  natural, mismo ecosistema                 │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  LangChain4j   │  Java         │  Más maduro que Spring AI, más funciones   │
│                │               │  de agentes, pero más verboso              │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  LangChain     │  Python       │  El estándar en Python. Más documentación, │
│                │               │  más ejemplos, más librerías               │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  LlamaIndex    │  Python       │  Especializado en RAG y documentos          │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  Semantic      │  C# / Python  │  Microsoft stack, Azure-first               │
│  Kernel        │               │                                             │
├────────────────┼───────────────┼────────────────────────────────────────────┤
│  OpenAI SDK    │  Cualquiera   │  Directo a OpenAI, máximo control,          │
│                │               │  pero atado a un proveedor                  │
└────────────────┴───────────────┴────────────────────────────────────────────┘
```

**Regla simple**: Si ya usas Spring Boot → Spring AI. Si tu equipo es Python → LangChain.

---

## 22. Cheatsheet rápido

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        SPRING AI 1.1.0 — CHEATSHEET                          │
├──────────────────────────────────────────────────────────────────────────────┤
│  CHAT BÁSICO                                                                 │
│  chatClient.prompt().user("texto").call().content()                          │
│                                                                              │
│  CHAT CON SYSTEM                                                             │
│  chatClient.prompt().system("rol").user("texto").call().content()            │
│                                                                              │
│  STREAMING                                                                   │
│  chatClient.prompt().user("texto").stream().content()  → Flux<String>        │
│                                                                              │
│  RESPUESTA TIPADA                                                            │
│  .call().entity(MiClase.class)                                               │
│  .call().entity(new ParameterizedTypeReference<List<T>>() {})                │
│                                                                              │
│  RESPUESTA COMPLETA CON METADATA (tokens, modelo, finish reason)            │
│  .call().chatResponse()  → ChatResponse                                      │
│                                                                              │
│  EMBEDDINGS                                                                  │
│  embeddingModel.embed("texto")  → float[]                                    │
│                                                                              │
│  VECTOR STORE — GUARDAR                                                      │
│  vectorStore.add(List.of(new Document("texto")))                             │
│                                                                              │
│  VECTOR STORE — BUSCAR                                                       │
│  vectorStore.similaritySearch(SearchRequest.query("consulta").withTopK(5))   │
│                                                                              │
│  RAG AUTOMÁTICO                                                              │
│  builder.defaultAdvisors(new QuestionAnswerAdvisor(vectorStore))             │
│                                                                              │
│  MEMORY / HISTORIAL                                                          │
│  builder.defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))           │
│  + .advisors(a -> a.param(CONVERSATION_ID_KEY, "user-123"))                  │
│                                                                              │
│  TOOLS — API NUEVA (1.1.0, recomendada)                                     │
│  @Tool(description="...") en método + builder.defaultTools(instancia)        │
│  @ToolParam(description="...") en parámetros del método                     │
│  .toolContext(Map.of("userId", id))  para pasar contexto seguro             │
│                                                                              │
│  TOOLS — API ANTIGUA (compatible con 1.0.x)                                 │
│  @Bean @Description("...") + .functions("nombreBean")                        │
│                                                                              │
│  RETRY AUTOMÁTICO (configuración en yml)                                    │
│  spring.ai.retry.max-attempts: 3                                             │
│  spring.ai.retry.initial-interval: 1000                                      │
│  spring.ai.retry.multiplier: 2.0                                             │
│                                                                              │
│  STRING TEMPLATE (prompts con condicionales)                                 │
│  new StPromptTemplateFactory().create(template)                              │
│  Sintaxis: <if(cond)>...<else>...<endif>  y  <lista:{item | <item>}>        │
│                                                                              │
│  TEMPERATURA                                                                 │
│  0.0 = determinista/exacto   |   1.0 = creativo/variado                     │
│                                                                              │
│  TOKENS                                                                      │
│  ≈ 0.75 palabras en inglés   |   ≈ 0.5 palabras en español                  │
│  GPT-4o-mini: $0.15/1M input |   GPT-4o: $5/1M input (2026)                 │
│                                                                              │
│  OBSERVABILIDAD (métricas automáticas por tipo de modelo)                   │
│  Chat:     gen_ai.client.token.usage + gen_ai.client.operation.duration     │
│  Embeddings: mismas métricas con operation.name="embeddings"                │
│  Images:   gen_ai.client.operation.duration con operation.name="image_gen"  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

*Documento creado: Marzo 2025 — actualizado Marzo 2026 — Spring AI 1.1.0*

---


---

> **[← Volver al índice](00_indice.md)**
