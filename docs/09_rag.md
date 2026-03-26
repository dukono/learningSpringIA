<!-- navegación -->
> **[← Vector Store](08_vector_store.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Tools →](10_tools.md)**

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

### Lectores de documentos disponibles

```java
// PDF — page by page
new PagePdfDocumentReader("classpath:documento.pdf")

// Texto plano
new TextReader("classpath:faq.txt")

// Markdown
new MarkdownDocumentReader("classpath:wiki.md")

// HTML (extrae texto del HTML)
new JsoupDocumentReader("https://docs.spring.io/spring-ai/docs/current/reference/html/")

// JSON
new JsonDocumentReader("classpath:data.json")

// GitHub repositorio entero
new GithubDocumentReader(token, "usuario/repo")
```

---

> **[← Vector Store](08_vector_store.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Tools →](10_tools.md)**
