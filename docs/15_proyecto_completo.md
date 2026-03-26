<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 20. Proyecto completo real — Chatbot con RAG

Ejemplo de un **asistente de documentación técnica** que responde preguntas sobre tus
propios documentos.

### Estructura del proyecto

```
src/main/java/com/empresa/chatbot/
├── config/
│   ├── AiConfig.java           # Configura ChatClient, VectorStore, Memory
│   └── SecurityConfig.java     # Seguridad básica
├── controller/
│   ├── ChatController.java     # Endpoints de chat (REST + SSE)
│   └── DocumentController.java # Subir y gestionar documentos
├── service/
│   ├── ChatService.java        # Lógica de conversación
│   ├── DocumentService.java    # Ingesta y gestión de documentos
│   └── AuditService.java       # Auditoría de conversaciones
├── advisor/
│   └── AuditAdvisor.java       # Interceptor de auditoría
├── model/
│   ├── ChatRequest.java        # DTO de petición
│   └── ChatResponse.java       # DTO de respuesta
└── ChatbotApplication.java
```

### AiConfig.java

```java
@Configuration
public class AiConfig {

    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        // Producción: usar pgvector
        return SimpleVectorStore.builder(embeddingModel).build();
    }

    @Bean
    public ChatMemory chatMemory() {
        return new InMemoryChatMemory();
    }

    @Bean
    public ChatClient chatClient(
            ChatClient.Builder builder,
            VectorStore vectorStore,
            ChatMemory memory) {

        return builder
            .defaultSystem("""
                Eres un asistente técnico experto.
                Solo respondes preguntas basadas en la documentación proporcionada.
                Si no encuentras la información, dilo claramente.
                Responde siempre en español.
                """)
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(memory),
                new QuestionAnswerAdvisor(
                    vectorStore,
                    SearchRequest.defaults().withTopK(5)
                ),
                new SimpleLoggerAdvisor()
            )
            .build();
    }
}
```

### DocumentController.java

```java
@RestController
@RequestMapping("/api/documentos")
public class DocumentController {

    @Autowired DocumentService documentService;

    @PostMapping("/subir")
    public ResponseEntity<String> subirDocumento(@RequestParam MultipartFile archivo) {
        int chunks = documentService.procesar(archivo);
        return ResponseEntity.ok("Documento procesado en " + chunks + " fragmentos");
    }
}
```

### DocumentService.java

```java
@Service
public class DocumentService {

    @Autowired VectorStore vectorStore;

    public int procesar(MultipartFile archivo) {
        try {
            // Guardar temporalmente
            Path tmp = Files.createTempFile("doc", ".pdf");
            archivo.transferTo(tmp.toFile());

            // Leer
            DocumentReader reader = new PagePdfDocumentReader(tmp.toString());
            List<Document> documentos = reader.get();

            // Dividir en chunks de ~500 tokens
            TokenTextSplitter splitter = new TokenTextSplitter(500, 100, 5, 10000, true);
            List<Document> chunks = splitter.apply(documentos);

            // Añadir metadata útil
            chunks.forEach(d -> {
                d.getMetadata().put("filename", archivo.getOriginalFilename());
                d.getMetadata().put("uploadDate", LocalDate.now().toString());
            });

            // Guardar en vector store
            vectorStore.add(chunks);

            Files.delete(tmp);
            return chunks.size();

        } catch (IOException e) {
            throw new RuntimeException("Error procesando documento", e);
        }
    }
}
```

### ChatController.java

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    @Autowired ChatService chatService;

    // Chat normal (espera respuesta completa)
    @PostMapping
    public ResponseEntity<ChatResponseDto> chat(@RequestBody ChatRequestDto request) {
        String respuesta = chatService.chat(
            request.conversationId(),
            request.mensaje()
        );
        return ResponseEntity.ok(new ChatResponseDto(respuesta));
    }

    // Chat en streaming (respuesta token a token)
    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chatStream(@RequestBody ChatRequestDto request) {
        return chatService.chatStream(
            request.conversationId(),
            request.mensaje()
        );
    }
}

record ChatRequestDto(String conversationId, String mensaje) {}
record ChatResponseDto(String respuesta) {}
```

### ChatService.java

```java
@Service
public class ChatService {

    @Autowired ChatClient chatClient;

    public String chat(String conversationId, String mensaje) {
        return chatClient.prompt()
            .user(mensaje)
            .advisors(a -> a.param(
                AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY,
                conversationId
            ))
            .call()
            .content();
    }

    public Flux<String> chatStream(String conversationId, String mensaje) {
        return chatClient.prompt()
            .user(mensaje)
            .advisors(a -> a.param(
                AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY,
                conversationId
            ))
            .stream()
            .content();
    }
}
```

### Flujo completo de una petición

```
POST /api/chat
{
  "conversationId": "user-42",
  "mensaje": "¿Cómo configuro la seguridad JWT?"
}

1. ChatController.chat() recibe la petición
2. ChatService llama a chatClient.prompt().user(...)
3. MessageChatMemoryAdvisor añade historial previo del usuario-42
4. QuestionAnswerAdvisor:
   a. Convierte "¿Cómo configuro la seguridad JWT?" en vector
   b. Busca en VectorStore → encuentra 3 chunks de "security-guide.pdf"
   c. Los añade al prompt como contexto
5. Prompt enviado al modelo:
   [SYSTEM] Eres un asistente técnico...
   [CONTEXT] "Para configurar JWT en Spring Security debes..."
   [USER_HISTORY] (turnos anteriores de user-42)
   [USER] "¿Cómo configuro la seguridad JWT?"
6. GPT-4o responde con información específica de TU documentación
7. SimpleLoggerAdvisor loguea la petición y respuesta
8. Respuesta llega al usuario

{"respuesta": "Para configurar JWT según vuestra documentación, debes..."}
```

---


---

> **[← Volver al índice](00_indice.md)**
