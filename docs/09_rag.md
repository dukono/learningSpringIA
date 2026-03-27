<!-- navegación -->
> **[← Vector Store](08_vector_store.md)** | **[← Inicio](../README.md)** | **[Siguiente: Tools →](10_tools.md)**

---

# Capítulo 09 — RAG

> La técnica más importante para aplicaciones empresariales con IA.
> Permite que el modelo responda sobre tus propios datos sin alucinaciones.

## 9. RAG — Retrieval Augmented Generation

**RAG** es la técnica más importante de Spring AI para aplicaciones empresariales.
Resuelve el problema más crítico de los LLMs en producción: **los modelos no conocen
tus datos, y si les preguntas lo que no saben, inventan respuestas plausibles pero falsas**.

### El problema real: la alucinación

Los modelos de IA como GPT-4o fueron entrenados con datos hasta cierta fecha y con textos
públicos de internet. Esto tiene consecuencias muy concretas:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  QUÉ SABE Y QUÉ NO SABE UN LLM                                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SÍ SABE (conocimiento general público):                                     │
│  ✅ "¿Qué es el patrón Singleton?"                                            │
│  ✅ "¿Cuál es la capital de Alemania?"                                        │
│  ✅ "Explícame qué es REST"                                                   │
│                                                                              │
│  NO SABE (datos privados o recientes):                                       │
│  ❌ "¿Cuál es la política de vacaciones de mi empresa?"                       │
│  ❌ "¿Qué dice el contrato del cliente Acme Corp?"                            │
│  ❌ "¿Cuánto stock tenemos del producto X?"                                   │
│  ❌ "¿Qué dijo el CEO en la reunión del lunes?"                               │
│  ❌ "¿Cuál es el error en el ticket JIRA-4521?"                               │
│                                                                              │
│  RIESGO — ALUCINACIÓN:                                                       │
│  Si le preguntas algo que no sabe, el modelo NO dice "no sé".               │
│  INVENTA una respuesta que suena plausible pero es falsa.                    │
│                                                                              │
│  Ejemplo real:                                                               │
│  Usuario: "¿Cuántos días de vacaciones tengo?"                               │
│  GPT-4o (sin RAG): "Según las políticas laborales estándar, los empleados   │
│  tienen derecho a 22 días laborables de vacaciones anuales..."               │
│  → Inventó un número. Puede ser 15, 25, 30... según TU empresa.             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### La solución RAG — principio fundamental

RAG soluciona esto en 3 pasos simples de entender pero poderosos en la práctica:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PRINCIPIO DE RAG — en lenguaje simple                                       │
│                                                                              │
│  1. ANTES de preguntar al modelo:                                            │
│     → busca en TUS documentos qué fragmentos son relevantes para la pregunta │
│                                                                              │
│  2. INCLUYE esos fragmentos en el prompt como "contexto":                    │
│     → el modelo ya NO necesita inventar — lee la respuesta de TUS docs       │
│                                                                              │
│  3. EL MODELO responde basándose en TUS datos, no en su conocimiento:        │
│     → respuestas precisas, verificables, sin alucinaciones                  │
│                                                                              │
│  Es como dar un examen "open book":                                          │
│  Sin RAG → el modelo intenta responder de memoria                            │
│  Con RAG → el modelo puede "consultar los apuntes" antes de responder        │
└──────────────────────────────────────────────────────────────────────────────┘
```

```
Con RAG:
  1. Almacenas tus documentos (PDFs, docs, textos) en un Vector Store
  2. Cuando llega una pregunta, PRIMERO buscas documentos relevantes
  3. Incluyes esos documentos en el prompt como contexto
  4. El modelo responde CON TUS DATOS

  Usuario: "¿Cuál es la política de devoluciones?"
    → Busca en Vector Store → encuentra chunks de "politica_devoluciones.pdf"
    → Los añade al prompt
  GPT-4o: "Según vuestra política, las devoluciones se aceptan en 30 días..."
```

### Los tres componentes de RAG que debes entender

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  COMPONENTE 1: INDEXACIÓN (preparación)                                      │
│  → Procesar tus documentos y almacenarlos como vectores                      │
│  → Se hace UNA VEZ (o cuando añades documentos nuevos)                       │
│  → Resultado: VectorStore lleno de chunks vectorizados                       │
├──────────────────────────────────────────────────────────────────────────────┤
│  COMPONENTE 2: RECUPERACIÓN (retrieval)                                      │
│  → Para cada pregunta: buscar los chunks más relevantes en el VectorStore    │
│  → Se hace EN TIEMPO REAL para cada consulta                                 │
│  → Resultado: lista de fragmentos de texto relevantes                        │
├──────────────────────────────────────────────────────────────────────────────┤
│  COMPONENTE 3: GENERACIÓN (generation)                                       │
│  → Construir un prompt con los fragmentos encontrados + la pregunta          │
│  → Enviarlo al LLM para que genere la respuesta final                        │
│  → Resultado: respuesta basada en TUS documentos                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Diagrama completo de RAG

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         FLUJO RAG                                            │
│                                                                              │
│  FASE 1 — INDEXACION (una vez, cuando cargas documentos)                     │
│                                                                              │
│  [Tus PDFs/Docs]                                                             │
│       │                                                                      │
│       ▼                                                                      │
│  [DocumentReader]  lee el contenido del fichero                              │
│       │                                                                      │
│       ▼                                                                      │
│  [TextSplitter]  divide en chunks de ~500 tokens cada uno                    │
│       │                                                                      │
│       ▼                                                                      │
│  [EmbeddingModel]  convierte cada chunk en un vector                         │
│       │                                                                      │
│       ▼                                                                      │
│  [VectorStore]  guarda los vectores con el texto original                    │
│                                                                              │
│  FASE 2 — CONSULTA (en tiempo real, por cada pregunta)                       │
│                                                                              │
│  [Pregunta usuario: "¿Cómo configuro JWT?"]                                  │
│       │                                                                      │
│       ├──→ [EmbeddingModel]  convierte la pregunta en vector                 │
│       │           │                                                          │
│       │           ▼                                                          │
│       │    [VectorStore.search(vector)]  encuentra los chunks más similares  │
│       │           │                                                          │
│       │           ▼  chunks relevantes encontrados                           │
│       └──────────────────────────────────────────────────────────┐          │
│                                                                   ▼          │
│                      [Prompt final enviado al modelo]                        │
│                      SYSTEM: "Eres un asistente..."                          │
│                      CONTEXT:  "Fragmento del PDF sobre JWT..."              │
│                      USER: "¿Cómo configuro JWT?"                            │
│                                      │                                       │
│                                      ▼                                       │
│                               [ChatModel]  respuesta basada en TUS docs      │
└──────────────────────────────────────────────────────────────────────────────┘
```

### El prompt real que Spring AI envía al modelo

Esto es lo más importante para entender RAG: **qué exactamente recibe el modelo**. El
`QuestionAnswerAdvisor` construye automáticamente este prompt:

```
[SYSTEM]
Eres un asistente técnico experto. Solo respondes con la información del contexto.
Si no encuentras la información, dilo claramente.

[USER — construido automáticamente por QuestionAnswerAdvisor]

Contexto relevante encontrado en la documentación:
---
[CHUNK 1 — encontrado con similitud 0.91]
"Para configurar JWT en Spring Security, primero añade la dependencia
spring-security-oauth2-resource-server en tu pom.xml. Luego configura
SecurityFilterChain con .oauth2ResourceServer(oauth2 -> oauth2.jwt(...))"

[CHUNK 2 — encontrado con similitud 0.85]
"La clave pública para verificar JWT se configura en application.yml:
spring.security.oauth2.resourceserver.jwt.public-key-location=classpath:public.pem"

[CHUNK 3 — encontrado con similitud 0.79]
"Para generar un JWT en el login, inyecta JwtEncoder y llama a encode() con
los claims del usuario autenticado."
---

Pregunta del usuario: ¿Cómo configuro JWT en mi app Spring Boot?

Responde basándote SOLO en el contexto anterior.
Si la información no está en el contexto, indica que no dispones de esa información.
```

**¿Qué hace el modelo con esto?** Lee los chunks como si fueran apuntes y redacta una
respuesta coherente combinando la información. No inventa nada porque tiene la fuente delante.

### ¿Qué es un chunk y por qué importa su tamaño?

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHUNK = fragmento de texto en que se divide un documento                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ¿Por qué dividir? Dos razones:                                          │
│  1. Un PDF de 100 páginas = ~50.000 tokens → no cabe en el prompt        │
│  2. La búsqueda por similitud es más precisa en fragmentos pequeños      │
│                                                                          │
│  Chunk demasiado GRANDE (2000+ tokens):                                  │
│  + Más contexto por fragmento                                            │
│  - Menos preciso en la búsqueda por similitud                            │
│  - Ocupa más tokens del prompt = más caro                                │
│                                                                          │
│  Chunk demasiado PEQUEÑO (50 tokens):                                    │
│  + Búsqueda muy precisa                                                  │
│  - Puede perder contexto (la frase queda sin contexto)                   │
│  - Muchos más chunks que gestionar                                       │
│                                                                          │
│  Chunk OPTIMO: 200-500 tokens con overlap de 50-100 tokens               │
│                                                                          │
│  El OVERLAP es la superposición entre chunks consecutivos:               │
│                                                                          │
│  Documento: [----Chunk 1----][----Chunk 2----][----Chunk 3----]          │
│  Con overlap:[----Chunk 1------]                                         │
│                          [------Chunk 2------]                           │
│                                          [------Chunk 3----]             │
│                           ↑ zona solapada = misma info en ambos chunks  │
│                                                                          │
│  Evita que una idea quede partida entre dos chunks sin contexto          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Implementación completa de RAG

> **📌 Dependencias adicionales:** `PagePdfDocumentReader` requiere un artefacto
> separado que **no viene incluido** en los starters de chat o embeddings. Añade
> según el tipo de documento que necesites leer:
>
> ```xml
> <!-- Leer PDFs -->
> <dependency>
>     <groupId>org.springframework.ai</groupId>
>     <artifactId>spring-ai-pdf-document-reader</artifactId>
> </dependency>
>
> <!-- Leer HTML / web scraping -->
> <dependency>
>     <groupId>org.springframework.ai</groupId>
>     <artifactId>spring-ai-jsoup-document-reader</artifactId>
> </dependency>
> ```

```java
// PASO 1: Cargar y almacenar documentos
@Service
public class DocumentIngestionService {

    @Autowired VectorStore vectorStore;

    public int cargarPDF(String rutaPDF) {
        // Leer el PDF
        DocumentReader reader = new PagePdfDocumentReader(rutaPDF);
        List<Document> documentos = reader.get();

        // Dividir en chunks de 400 tokens con overlap de 80 tokens
        TokenTextSplitter splitter = new TokenTextSplitter(
            400,   // tamaño del chunk
            80,    // overlap (superposición entre chunks)
            5,     // mínimo de tokens para un chunk
            10000, // máximo de tokens totales
            true   // keepSeparator
        );
        List<Document> chunks = splitter.apply(documentos);

        // Añadir metadata para poder filtrar después
        String nombreFichero = Path.of(rutaPDF).getFileName().toString();
        chunks.forEach(d -> {
            d.getMetadata().put("filename",    nombreFichero);
            d.getMetadata().put("uploadDate",  LocalDate.now().toString());
            d.getMetadata().put("contentType", "pdf");
        });

        // Guardar en vector store (genera embeddings automáticamente)
        vectorStore.add(chunks);

        System.out.println("PDF dividido en " + chunks.size() + " chunks y almacenado");
        return chunks.size();
    }
}

// PASO 2: Hacer RAG con QuestionAnswerAdvisor
@Service
public class RagChatService {

    private final ChatClient chatClient;

    public RagChatService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
            .defaultAdvisors(
                // QuestionAnswerAdvisor hace el RAG completo automáticamente
                new QuestionAnswerAdvisor(
                    vectorStore,
                    SearchRequest.builder()
                        .topK(5)                    // top 5 chunks más relevantes
                        .similarityThreshold(0.65)  // mínimo de similitud
                        .build()
                )
            )
            .build();
    }

    public String pregunta(String consulta) {
        return chatClient.prompt()
            .user(consulta)
            .call()
            .content();
        // Spring AI automáticamente:
        // 1. Convierte la consulta en vector
        // 2. Busca en vectorStore los chunks más similares
        // 3. Los añade al prompt como contexto [CONTEXT]
        // 4. Envía todo al modelo
        // 5. Devuelve la respuesta
    }
}
```

### Cómo debuggear RAG — problemas frecuentes y soluciones

Esta sección es crítica. RAG parece no funcionar en muchos casos por razones concretas:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PROBLEMA 1: "No encuentra los documentos relevantes"                        │
├──────────────────────────────────────────────────────────────────────────────┤
│  Síntoma: el modelo dice "no tengo información" aunque subiste el documento  │
│                                                                              │
│  Diagnóstico — comprueba la similitud directamente:                          │
│                                                                              │
│  List<Document> chunks = vectorStore.similaritySearch(                       │
│      SearchRequest.builder().query("tu pregunta aquí")                                 │
│          .topK(10)                                                       │
│          .similarityThreshold(0.0)   // sin umbral para ver todos        │
│          .build()                                                        │
│  );                                                                          │
│  chunks.forEach(c ->                                                         │
│      System.out.println("Similitud: " + c.getScore()                        │
│                        + " | " + c.getContent().substring(0, 100)));        │
│                                                                              │
│  Si los scores son todos < 0.5:                                              │
│  → El documento no se indexó correctamente, o                                │
│  → Estás usando un modelo de embedding distinto al de indexación             │
│                                                                              │
│  SOLUCIONES:                                                                 │
│  ✅ Baja el threshold: .similarityThreshold(0.5)  (era 0.7)             │
│  ✅ Verifica que usas el mismo EmbeddingModel en indexación y búsqueda       │
│  ✅ Re-indexa los documentos                                                  │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  PROBLEMA 2: "El modelo ignora el contexto y responde con su conocimiento"   │
├──────────────────────────────────────────────────────────────────────────────┤
│  Síntoma: el modelo responde con información genérica, no de tus documentos  │
│                                                                              │
│  SOLUCIONES:                                                                 │
│  ✅ Añade en el system prompt: "SOLO responde con la información del          │
│     contexto proporcionado. Si no está en el contexto, di que no tienes     │
│     esa información. Nunca uses tu conocimiento general."                    │
│  ✅ Aumenta topK: .topK(10)  (estaba en 3)                               │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  PROBLEMA 3: "El modelo mezcla información de distintos documentos"          │
├──────────────────────────────────────────────────────────────────────────────┤
│  Síntoma: la respuesta combina partes de documentos no relacionados          │
│                                                                              │
│  SOLUCIÓN: usar filtros de metadata para acotar la búsqueda                  │
│  SearchRequest.builder().query(consulta)                                     │
│      .filterExpression("filename == '" + nombreDoc + "'").build()        │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  PROBLEMA 4: "Respuestas incorrectas en preguntas que cruzan varios chunks"  │
├──────────────────────────────────────────────────────────────────────────────┤
│  Síntoma: la pregunta necesita info de 3 párrafos distintos del documento    │
│  pero cada párrafo está en un chunk diferente y el modelo solo recibe 2      │
│                                                                              │
│  SOLUCIONES:                                                                 │
│  ✅ Aumenta el tamaño del chunk (400 → 600 tokens)                           │
│  ✅ Aumenta el overlap (80 → 150 tokens)                                     │
│  ✅ Aumenta topK para recuperar más contexto                                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### RAG naïve vs RAG avanzado

La implementación que vimos arriba es **RAG naïve** (básico). Funciona bien para el 70%
de los casos. Para casos más exigentes existe **RAG avanzado**:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  RAG NAÏVE (lo que hemos implementado)                                       │
│  Flujo: pregunta → embedding → búsqueda → prompt → respuesta                 │
│  Ventaja: simple, rápido de implementar                                      │
│  Limitación: si la pregunta está mal formulada, la búsqueda falla            │
├──────────────────────────────────────────────────────────────────────────────┤
│  RAG AVANZADO — técnicas para mejorar la calidad                             │
│                                                                              │
│  1. QUERY REWRITING: el LLM reformula la pregunta antes de buscar           │
│     "¿Cuándo fue fundada mi empresa?" →                                      │
│     LLM reescribe a: "fecha de fundación historia origen empresa"            │
│     (mejor para búsqueda semántica)                                          │
│                                                                              │
│  2. HYDE (Hypothetical Document Embeddings):                                 │
│     El LLM genera una respuesta hipotética → se usa ESA para buscar         │
│     La respuesta hipotética está formulada como un documento, no pregunta   │
│     → mejora la similitud con los chunks reales                              │
│                                                                              │
│  3. RE-RANKING: recuperar 20 chunks, luego un modelo clasifica los mejores  │
│     topK(20) → reranker → top 5 reales → prompt                             │
│     → más preciso pero más lento y costoso                                   │
│                                                                              │
│  4. PARENT DOCUMENT RETRIEVAL:                                               │
│     Indexas chunks pequeños para búsqueda precisa,                           │
│     pero al recuperar devuelves el documento padre (más grande)              │
│     → más contexto sin perder precisión en búsqueda                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### RetrievalAugmentationAdvisor — la API modular de RAG (1.1.x)

`QuestionAnswerAdvisor` es el advisor de RAG clásico y funciona bien para la mayoría
de casos. Spring AI 1.1.x introduce `RetrievalAugmentationAdvisor` como su evolución
más modular y composable, donde **el retriever es una pieza intercambiable**:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  QuestionAnswerAdvisor (clásico)                                             │
│  → VectorStore acoplado directamente dentro del advisor                      │
│  → Menos flexible para escenarios avanzados                                  │
│                                                                              │
│  RetrievalAugmentationAdvisor (modular, 1.1.x)                              │
│  → DocumentRetriever como interfaz separada e inyectable                     │
│  → VectorStoreDocumentRetriever como implementación por defecto              │
│  → Permite custom retrievers (BDD, API externa, ficheros, etc.)              │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### Uso básico — equivalente a QuestionAnswerAdvisor

```java
// VectorStoreDocumentRetriever envuelve el VectorStore con SearchRequest configurable
DocumentRetriever retriever = VectorStoreDocumentRetriever.builder()
    .vectorStore(vectorStore)
    .searchRequest(SearchRequest.builder()
        .topK(5)
        .similarityThreshold(0.65)
        .build())
    .build();

ChatClient chatClient = builder
    .defaultAdvisors(
        RetrievalAugmentationAdvisor.builder()
            .documentRetriever(retriever)
            .build()
    )
    .build();

// El uso es idéntico al de QuestionAnswerAdvisor
String respuesta = chatClient.prompt()
    .user("¿Cómo configuro JWT?")
    .call()
    .content();
```

#### DocumentRetriever custom — recuperar desde fuente propia

La ventaja clave: puedes implementar `DocumentRetriever` para recuperar documentos
desde **cualquier fuente**, no solo un VectorStore:

```java
// Retriever custom — recupera desde tu propio sistema
@Component
public class JiraDocumentRetriever implements DocumentRetriever {

    @Autowired JiraClient jiraClient;

    @Override
    public List<Document> retrieve(String query) {
        // Buscar tickets de Jira relacionados con la consulta
        List<JiraIssue> issues = jiraClient.search("text ~ \"" + query + "\"", 5);
        return issues.stream()
            .map(issue -> new Document(
                issue.getSummary() + "\n" + issue.getDescription(),
                Map.of("jiraKey", issue.getKey(), "status", issue.getStatus())
            ))
            .toList();
    }
}

// Usar el retriever custom en el advisor
ChatClient chatClient = builder
    .defaultAdvisors(
        RetrievalAugmentationAdvisor.builder()
            .documentRetriever(jiraDocumentRetriever)
            .build()
    )
    .build();
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  QuestionAnswerAdvisor vs RetrievalAugmentationAdvisor                       │
├──────────────────────────────┬───────────────────────────────────────────────┤
│  QuestionAnswerAdvisor       │  Más simple. Toma VectorStore + SearchRequest. │
│                              │  Suficiente para el 80% de los casos.         │
│  RetrievalAugmentationAdvisor│  Más modular. DocumentRetriever intercambiable.│
│                              │  Usa cuando necesitas fuente de datos custom.  │
└──────────────────────────────┴───────────────────────────────────────────────┘
```

> 💡 Si solo tienes un VectorStore estándar, `QuestionAnswerAdvisor` sigue siendo
> la opción más directa. Usa `RetrievalAugmentationAdvisor` cuando el origen de los
> documentos de contexto sea algo distinto a un VectorStore de Spring AI.

---

### Query Rewriting — ejemplo con código

La técnica más práctica del RAG avanzado. Antes de buscar en el vector store,
se pide al modelo que reformule la pregunta del usuario en términos más adecuados
para búsqueda semántica.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  POR QUÉ FUNCIONA MEJOR                                                      │
│                                                                              │
│  Pregunta original:   "¿cómo lo activo?"                                    │
│  → Embeddings de "cómo lo activo" → vector muy genérico → chunks irrelevantes│
│                                                                              │
│  Pregunta reescrita:  "pasos para activar funcionalidad en panel de admin"  │
│  → Embeddings del texto expandido → vector más específico → mejores chunks  │
└──────────────────────────────────────────────────────────────────────────────┘
```

```java
@Service
public class QueryRewritingRagService {

    @Autowired ChatClient  chatClient;
    @Autowired VectorStore vectorStore;

    // Prompt dedicado para reescribir — usa un modelo rápido y barato
    private final ChatClient rewriterClient;

    public QueryRewritingRagService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.vectorStore = vectorStore;
        // Cliente específico para reescritura — sin advisors, sin historial
        this.rewriterClient = builder
            .defaultSystem("""
                Tu única tarea es reformular la pregunta del usuario para mejorar
                la búsqueda semántica en una base de documentos técnicos.
                Devuelve SOLO la pregunta reformulada, sin explicación ni comillas.
                Usa términos técnicos específicos. Si la pregunta ya es específica,
                devuélvela igual.
                """)
            .build();
        this.chatClient = builder.build();
    }

    public String preguntaConReescritura(String preguntaOriginal) {

        // PASO 1: reescribir la pregunta para mejorar la búsqueda
        String preguntaOptimizada = rewriterClient.prompt()
            .user("Reformula para búsqueda semántica: " + preguntaOriginal)
            .call()
            .content();

        log.debug("Query original:   '{}'", preguntaOriginal);
        log.debug("Query reescrita:  '{}'", preguntaOptimizada);

        // PASO 2: buscar con la pregunta optimizada, no con la original
        List<Document> chunks = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(preguntaOptimizada)  // ← pregunta reformulada
                .topK(5)
                .similarityThreshold(0.6)
                .build()
        );

        if (chunks.isEmpty()) {
            return "No encontré información relevante en la documentación.";
        }

        // PASO 3: construir el prompt con el contexto encontrado
        String contexto = chunks.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n---\n"));

        // PASO 4: respuesta final con el contexto real
        return chatClient.prompt()
            .system("""
                Responde usando SOLO el contexto proporcionado.
                Si la respuesta no está en el contexto, dilo claramente.
                """)
            .user("Contexto:\n" + contexto + "\n\nPregunta: " + preguntaOriginal)
            .call()
            .content();
    }
}
```

> 💡 Este patrón manual da más control que `QuestionAnswerAdvisor` pero requiere
> más código. Úsalo cuando el RAG naïve devuelva resultados pobres en preguntas
> cortas, conversacionales o con referencias implícitas ("¿cómo lo hago?", "¿y si falla?").

---

### Lectores de documentos disponibles

```java
// PDF — page by page
new PagePdfDocumentReader("classpath:documento.pdf")

// Texto plano
new TextReader("classpath:faq.txt")

// Markdown (requiere spring-ai-markdown-document-reader)
new MarkdownDocumentReader("classpath:wiki.md")

// HTML, extrae texto del HTML (requiere spring-ai-jsoup-document-reader)
new JsoupDocumentReader("classpath:pagina.html")

// JSON
new JsonDocumentReader("classpath:data.json")
```

> ⚠️ `GithubDocumentReader` requiere el artefacto `spring-ai-github-document-reader`
> y autenticación con token de GitHub. No está disponible en el starter estándar.

---

### 9.X DocumentTransformer — enriquecimiento de documentos antes de indexar

`DocumentTransformer` es la interfaz de Spring AI para **transformar documentos entre
el paso de lectura y el de indexación**. Un transformador recibe una lista de documentos
y devuelve una lista (posiblemente diferente) de documentos modificados.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PIPELINE DE INGESTA COMPLETO CON TRANSFORMADORES                             │
│                                                                              │
│  [DocumentReader]                                                            │
│       │  Lee el fichero — devuelve List<Document> sin procesar               │
│       ▼                                                                      │
│  [DocumentTransformer 1: KeywordMetadataEnricher]                            │
│       │  Usa el LLM para extraer keywords y añadirlas como metadata          │
│       ▼                                                                      │
│  [DocumentTransformer 2: SummaryMetadataEnricher]                            │
│       │  Usa el LLM para generar un resumen del documento y añadirlo         │
│       ▼                                                                      │
│  [DocumentTransformer 3: TokenTextSplitter]                                  │
│       │  Divide en chunks de ~400 tokens (también es un DocumentTransformer) │
│       ▼                                                                      │
│  [VectorStore.add()]                                                         │
│       Genera embeddings e indexa                                             │
└──────────────────────────────────────────────────────────────────────────────┘

El resultado: cada chunk en el vector store tiene metadata enriquecida con
keywords y resumen, lo que mejora significativamente la calidad del RAG.
```

#### KeywordMetadataEnricher

Usa el LLM para extraer palabras clave de cada documento y añadirlas como metadata.
Mejora la búsqueda cuando la pregunta usa términos relacionados pero no idénticos al texto.

```java
@Service
public class EnrichedIngestionService {

    @Autowired VectorStore vectorStore;
    @Autowired ChatModel  chatModel;   // usado internamente por los enrichers

    public void ingestarConEnriquecimiento(String rutaPDF) {
        // 1. Leer
        List<Document> docs = new PagePdfDocumentReader(rutaPDF).get();

        // 2. Enriquecer con keywords (llama al LLM por cada documento)
        KeywordMetadataEnricher keywordEnricher = new KeywordMetadataEnricher(chatModel, 5);
        List<Document> docsConKeywords = keywordEnricher.apply(docs);
        // Resultado: cada doc.metadata ahora tiene "excerpt_keywords": "JWT, Spring Security,
        //            autenticación, token, autorización"

        // 3. Enriquecer con resumen (llama al LLM por cada documento)
        SummaryMetadataEnricher summaryEnricher = new SummaryMetadataEnricher(
            chatModel,
            List.of(SummaryType.PREVIOUS, SummaryType.CURRENT, SummaryType.NEXT)
            // PREVIOUS: resumen del chunk anterior (contexto)
            // CURRENT:  resumen de este chunk
            // NEXT:     resumen del siguiente chunk (anticipo)
        );
        List<Document> docsConResumen = summaryEnricher.apply(docsConKeywords);
        // Resultado: metadata también tiene "section_summary": "Este fragmento describe
        //            cómo configurar JWT en Spring Security 6..."

        // 4. Dividir en chunks
        List<Document> chunks = new TokenTextSplitter().apply(docsConResumen);

        // 5. Indexar
        vectorStore.add(chunks);
    }
}
```

> ⚠️ `KeywordMetadataEnricher` y `SummaryMetadataEnricher` hacen **una llamada al LLM
> por cada documento**. Con 100 PDFs de 10 páginas cada uno → 1000 llamadas al LLM.
> Tenlo en cuenta para planificar el coste y el tiempo de ingesta.

#### DocumentTransformer custom

Si necesitas lógica de transformación propia (limpiar HTML, traducir, filtrar páginas),
implementa `DocumentTransformer`:

```java
@Component
public class LimpiezaHtmlTransformer implements DocumentTransformer {

    @Override
    public List<Document> apply(List<Document> docs) {
        return docs.stream()
            .map(doc -> {
                // Limpiar etiquetas HTML del contenido
                String textoLimpio = doc.getContent()
                    .replaceAll("<[^>]+>", " ")     // eliminar etiquetas
                    .replaceAll("\\s+", " ")         // normalizar espacios
                    .trim();

                // Añadir metadata de procesamiento
                Map<String, Object> metadata = new HashMap<>(doc.getMetadata());
                metadata.put("procesado", true);
                metadata.put("procesadoEn", LocalDateTime.now().toString());

                return new Document(textoLimpio, metadata);
            })
            .filter(doc -> doc.getContent().length() > 100)  // descartar fragmentos muy cortos
            .toList();
    }
}
```

#### Composición de transformadores

```java
// Encadenar transformadores manualmente
List<Document> docs = reader.get();
docs = keywordEnricher.apply(docs);
docs = miLimpiezaCustom.apply(docs);
docs = splitter.apply(docs);
vectorStore.add(docs);
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ¿CUÁNDO USAR CADA TRANSFORMADOR?                                            │
├─────────────────────────────────┬────────────────────────────────────────────┤
│  KeywordMetadataEnricher        │  Cuando las preguntas de los usuarios usan  │
│                                 │  sinónimos o términos relacionados al texto │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  SummaryMetadataEnricher        │  Cuando los documentos son largos y         │
│                                 │  necesitas contexto entre chunks            │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  DocumentTransformer custom     │  Limpieza, traducción, normalización,       │
│                                 │  enriquecimiento con datos externos         │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  TokenTextSplitter              │  Siempre — divide documentos largos en      │
│                                 │  chunks que caben en el contexto del modelo │
└─────────────────────────────────┴────────────────────────────────────────────┘
```

---

## 9.1 Errores comunes en el pipeline RAG

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES EN RAG                                                   │
├─────────────────────────────────┬────────────────────────────────────────────┤
│  Error                          │  Causa y solución                          │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  El modelo responde "no tengo   │  topK demasiado bajo o similarityThreshold │
│  información" aunque el         │  demasiado alto → no recupera chunks       │
│  documento está indexado        │  relevantes. O el chunk está indexado con  │
│                                 │  un embedding diferente al de la consulta. │
│                                 │  ✅ Bajar similarityThreshold (0.5-0.65),  │
│                                 │  subir topK (5-10), y verificar que        │
│                                 │  EmbeddingModel de indexación y consulta   │
│                                 │  son el mismo modelo.                      │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  Respuestas basadas en datos    │  El VectorStore no se actualiza cuando     │
│  obsoletos aunque los           │  cambian los documentos fuente. Los chunks │
│  documentos han cambiado        │  antiguos permanecen indefinidamente.      │
│                                 │  ✅ Implementar un pipeline de             │
│                                 │  re-indexación: delete(ids) + add(nuevos   │
│                                 │  documentos) cuando cambie la fuente.      │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  Alucinaciones aunque hay RAG   │  El system prompt no instruye explícitamente│
│  configurado                    │  al modelo a responder solo con el         │
│                                 │  contexto proporcionado.                   │
│                                 │  ✅ Añadir en el system prompt: "Responde  │
│                                 │  ÚNICAMENTE con la información del         │
│                                 │  contexto proporcionado. Si no encuentras  │
│                                 │  la respuesta en el contexto, di           │
│                                 │  explícitamente que no lo sabes."          │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  Chunks recuperados irrelevantes│  El TokenTextSplitter usa un chunkSize     │
│  → contexto de baja calidad     │  inadecuado: demasiado pequeño (pierde     │
│                                 │  contexto semántico) o demasiado grande    │
│                                 │  (mezcla temas distintos en un chunk).     │
│                                 │  ✅ Ajustar defaultChunkSize (400-600      │
│                                 │  tokens) y defaultOverlapSize (50-100)     │
│                                 │  según el tipo de documento.               │
├─────────────────────────────────┼────────────────────────────────────────────┤
│  Alto coste al usar             │  KeywordMetadataEnricher y                 │
│  DocumentTransformer en         │  SummaryMetadataEnricher llaman al LLM     │
│  indexación masiva              │  por cada chunk. 10.000 chunks = 10.000    │
│                                 │  llamadas adicionales.                     │
│                                 │  ✅ Usar enriquecedores solo en            │
│                                 │  indexaciones incrementales, no en         │
│                                 │  re-indexaciones masivas. Considerar       │
│                                 │  modelos pequeños y baratos para           │
│                                 │  enriquecer metadata.                      │
└─────────────────────────────────┴────────────────────────────────────────────┘
```

---

> **[← Vector Store](08_vector_store.md)** | **[← Inicio](../README.md)** | **[Siguiente: Tools →](10_tools.md)**
