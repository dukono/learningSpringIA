<!-- navegación -->
> **[← Observabilidad](14_observabilidad.md)** | **[← Inicio](../README.md)** | **[Siguiente: Comparativa →](16_comparativa_y_cheatsheet.md)**

---

# Capítulo 15 — Proyecto completo real

> Chatbot de documentación técnica que integra todos los conceptos del curso:
> RAG, memoria de conversación, streaming, advisors, multiprofiles y observabilidad.

## 15.1 Qué construimos y por qué

Este capítulo integra todos los conceptos anteriores en un proyecto coherente y deployable.
El caso de uso es un **asistente de documentación técnica** que responde preguntas sobre
los propios documentos de la empresa, con memoria por usuario y streaming.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  QUÉ INTEGRA ESTE PROYECTO                                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RAG              → respuestas basadas en TUS documentos, no en el modelo   │
│  Memory           → el usuario puede tener conversaciones multi-turno       │
│  Streaming        → el usuario ve la respuesta token a token                │
│  Advisors         → RAG + Memory + Logging declarativos, en un solo lugar   │
│  Profiles         → Ollama en dev, OpenAI en prod, sin cambiar código       │
│  Observabilidad   → métricas de tokens y latencia en /actuator/metrics      │
│                                                                              │
│  El código Java es idéntico en dev y en prod.                                │
│  Solo cambia la configuración (application-{perfil}.yml).                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  FLUJO DE UNA PETICIÓN EN EL SISTEMA                                         │
│                                                                              │
│  Usuario                                                                     │
│    │  POST /api/chat {"conversationId":"u1", "mensaje":"¿cómo uso JWT?"}     │
│    ▼                                                                         │
│  ChatController → ChatService                                                │
│    │                                                                         │
│    ▼                                                                         │
│  [MessageChatMemoryAdvisor]  recupera historial de "u1" de ChatMemory        │
│    │                                                                         │
│    ▼                                                                         │
│  [QuestionAnswerAdvisor]  convierte la pregunta en vector, busca en          │
│    │                      VectorStore, añade 5 chunks de documentos al prompt│
│    ▼                                                                         │
│  [SimpleLoggerAdvisor]   loguea el prompt final completo                     │
│    │                                                                         │
│    ▼                                                                         │
│  LLM (Ollama/OpenAI según perfil)  genera la respuesta                       │
│    │                                                                         │
│    ▼                                                                         │
│  [MessageChatMemoryAdvisor]  guarda la respuesta en el historial de "u1"    │
│    │                                                                         │
│    ▼                                                                         │
│  Usuario recibe: {"respuesta": "Para configurar JWT en Spring Security..."}  │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 15.2 Proyecto completo real — Chatbot con RAG

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
                    SearchRequest.builder().topK(5).build()
                ),
                new SimpleLoggerAdvisor()
            )
            .build();
    }
}
```

### SecurityConfig.java

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // Permite acceso libre a la API de chat y actuator desde cualquier origen.
    // En producción: sustituir .permitAll() por autenticación JWT u OAuth2.
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())   // necesario para POST desde clientes externos
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/chat/**").permitAll()        // chat público
                .requestMatchers("/api/documentos/**").authenticated() // subida protegida
                .requestMatchers("/actuator/health").permitAll()    // healthcheck para K8s
                .requestMatchers("/actuator/**").hasRole("ADMIN")   // métricas solo admin
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults()); // básico para demo; en prod → JWT/OAuth2
        return http.build();
    }

    // Usuario de demo para desarrollo. En producción usar UserDetailsService real.
    @Bean
    @Profile("dev")
    public UserDetailsService devUsers() {
        UserDetails admin = User.withDefaultPasswordEncoder()
            .username("admin")
            .password("admin")
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(admin);
    }
}
```

> ⚠️ **En producción** reemplaza `httpBasic` por JWT o OAuth2 (Spring Security 6).
> El patrón recomendado es un `JwtAuthenticationFilter` que extraiga el `userId`
> del token y lo inyecte en el `ToolContext` para los tools que necesiten saber
> quién está preguntando (ver Cap. 10 — seguridad en tools).

---

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

## 15.3 Errores comunes al integrar todo

Cuando se combinan varios componentes en un proyecto real, los errores más frecuentes
son de integración — no de un componente individual:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES EN PROYECTOS INTEGRADOS                                  │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  "No unique bean of type       │  Hay dos proveedores configurados en el     │
│  ChatModel" al arrancar        │  pom.xml (ej: ollama + openai) y Spring    │
│                                │  no sabe cuál usar para el ChatClient.     │
│                                │  ✅ Usar @Profile en los beans de           │
│                                │  ChatClient para activar solo uno por       │
│                                │  entorno, o excluir la autoconfiguración   │
│                                │  del proveedor no activo en el YAML.        │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El RAG devuelve resultados    │  Los documentos se indexaron con un         │
│  irrelevantes en producción    │  EmbeddingModel (Ollama, 768 dims) y en    │
│  pero correctos en dev         │  prod se busca con otro (OpenAI, 1536 dims).│
│                                │  Los vectores son incomparables.            │
│                                │  ✅ Reindexar los documentos con el         │
│                                │  EmbeddingModel de producción, o asegurarse │
│                                │  de que dev y prod usan el mismo modelo.    │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El chatbot no recuerda        │  conversationId no se está propagando       │
│  conversaciones aunque memory  │  correctamente desde el cliente. Si el      │
│  está configurado              │  frontend genera un ID nuevo en cada        │
│                                │  petición, cada mensaje es una conversación │
│                                │  nueva para el advisor.                     │
│                                │  ✅ El cliente debe guardar y reusar el     │
│                                │  mismo conversationId durante toda la       │
│                                │  sesión del usuario.                        │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  En producción el streaming    │  El proxy inverso (Nginx, API Gateway) no   │
│  falla o se corta              │  está configurado para SSE/streaming.       │
│                                │  Por defecto, muchos proxies almacenan la   │
│                                │  respuesta completa antes de enviarla.      │
│                                │  ✅ Configurar en Nginx:                    │
│                                │  proxy_buffering off;                       │
│                                │  proxy_read_timeout 300s;                   │
│                                │  X-Accel-Buffering: no;                     │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  SimpleVectorStore y           │  El proyecto de ejemplo usa SimpleVectorStore│
│  InMemoryChatMemory en prod    │  e InMemoryChatMemory para simplicidad.      │
│  — pérdida de datos            │  En producción, todo se pierde al reiniciar.│
│                                │  ✅ Reemplazar antes del primer deploy:      │
│                                │  SimpleVectorStore → pgvector (Capítulo 08) │
│                                │  InMemoryChatMemory → JdbcChatMemory        │
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Observabilidad](14_observabilidad.md)** | **[← Inicio](../README.md)** | **[Siguiente: Comparativa →](16_comparativa_y_cheatsheet.md)**
