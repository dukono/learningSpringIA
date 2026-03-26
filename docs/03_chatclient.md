<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 6. ChatClient — el núcleo de Spring AI

`ChatClient` es el punto de entrada principal. Tiene una API fluida (Fluent API) muy limpia.

### 6.1 Inyección y uso básico

```java
@RestController
@RequestMapping("/chat")
public class ChatController {

    private final ChatClient chatClient;

    // Spring AI auto-configura ChatClient.Builder via @AutoConfiguration
    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping
    public String chat(@RequestParam String pregunta) {
        return chatClient.prompt()
                .user(pregunta)          // mensaje del usuario
                .call()                  // envía la petición (bloqueante)
                .content();              // extrae solo el texto
    }
}
```

Resultado visual cuando llamas a `GET /chat?pregunta=Hola`:

```
Petición  ──→  [Tu App] ──→  [Spring AI] ──→  [OpenAI API]
                                                     │
Respuesta ←──  [Tu App] ←──  [Spring AI] ←──────────┘
"¡Hola! ¿En qué puedo ayudarte hoy?"
```

### 6.2 ChatClient con System Prompt global

El **system prompt** define el "carácter" o "rol" del asistente. Se puede configurar una vez
para todo el cliente:

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("""
                Eres un asistente experto en desarrollo Java y Spring Boot.
                Respondes siempre en español.
                Tus respuestas son concisas, directas y con ejemplos de código.
                Nunca inventas APIs que no existen.
                """)
            .build();
    }
}
```

Ahora **todos los usos** de este `ChatClient` tendrán ese comportamiento automáticamente.

### 6.3 Estructura completa de una petición

```java
String respuesta = chatClient.prompt()
    // --- Mensajes ---
    .system("Eres un experto en SQL")           // override del system prompt
    .user("Explícame un JOIN")                  // pregunta del usuario

    // --- Opciones del modelo (override de la config) ---
    .options(OpenAiChatOptions.builder()
        .withModel("gpt-4o")
        .withTemperature(0.2f)                  // más preciso para SQL
        .withMaxTokens(500)
        .build())

    // --- Llamada ---
    .call()
    .content();
```

### 6.4 ChatResponse — respuesta completa con metadata

```java
ChatResponse response = chatClient.prompt()
    .user("¿Cuánto es 2+2?")
    .call()
    .chatResponse();                            // respuesta completa

// Texto de la respuesta
String texto = response.getResult().getOutput().getContent();

// Metadata de uso (tokens consumidos) — muy útil para control de costes
Usage usage = response.getMetadata().getUsage();
System.out.println("Tokens usados: " + usage.getTotalTokens());
System.out.println("  → Prompt:    " + usage.getPromptTokens());
System.out.println("  → Respuesta: " + usage.getGenerationTokens());

// Modelo que respondió
String modelo = response.getMetadata().getModel();

// Razón por la que el modelo paró de generar
// "stop" = terminó normalmente
// "length" = llegó al límite de max-tokens (respuesta cortada)
// "content_filter" = filtro de contenido activado
String finishReason = response.getResult().getMetadata().getFinishReason();
System.out.println("Razón de parada: " + finishReason);
```

---


---

> **[← Volver al índice](00_indice.md)**
