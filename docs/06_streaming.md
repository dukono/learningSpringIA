<!-- navegación -->
> **[← Structured Output](05_structured_output.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Embeddings →](07_embeddings.md)**

---

# Capítulo 06 — Streaming

> El streaming permite enviar tokens al cliente en tiempo real mientras el modelo
> genera la respuesta. Esencial para buena UX en aplicaciones de chat.

## Contenido

- [10.1 Dependencia necesaria](#dependencia-necesaria)
- [10.2 Implementación básica](#implementación)
- [10.3 Manejo de errores en streaming](#manejo-de-errores-en-streaming)
- [10.4 Streaming con Advisors (RAG + Memory)](#104-streaming-con-advisors-rag--memory)
- [10.5 Streaming con Tools — cómo funciona](#105-streaming-con-tools--cómo-funciona)
- [10.6 Operaciones Flux útiles](#106-operaciones-flux-útiles)
- [10.7 Cliente JavaScript (SSE)](#cliente-javascript-para-consumir-el-stream-sse)

---

## 10. Streaming — respuesta en tiempo real

El **streaming** es como funciona ChatGPT: la respuesta llega token a token en lugar
de esperar a que termine toda la generación.

### Dependencia necesaria

El streaming en Spring AI usa **Project Reactor** (WebFlux). Asegúrate de tenerlo:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### Implementación

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String pregunta) {
    return chatClient.prompt()
        .user(pregunta)
        .stream()                    // en lugar de .call()
        .content();                  // Flux<String> — cada elemento = un token
}
```

### Visualización del streaming

```
Sin streaming (bloqueante):
  Petición → [esperas 3-10 segundos en silencio] → "Aquí tienes la respuesta completa..."

Con streaming (reactivo):
  Petición → "Aquí" → " tienes" → " la" → " respuesta" → " completa..."
              ↑ cada token llega en milisegundos → el usuario ve texto aparecer
```

### Manejo de errores en streaming

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String pregunta) {
    return chatClient.prompt()
        .user(pregunta)
        .stream()
        .content()
        .onErrorResume(e -> {
            // Si la API falla a mitad del stream
            log.error("Error en streaming: {}", e.getMessage());
            return Flux.just("[Error: no se pudo completar la respuesta]");
        })
        .doOnComplete(() -> log.info("Stream completado"));
}
```

### Cliente JavaScript para consumir el stream (SSE)

```javascript
// Server-Sent Events — el navegador recibe los tokens en tiempo real
const eventSource = new EventSource('/chat/stream?pregunta=Explicame%20Java');
const div = document.getElementById('respuesta');

eventSource.onmessage = (event) => {
    div.textContent += event.data;  // cada evento = un token
};

eventSource.onerror = () => {
    console.log('Stream terminado o error');
    eventSource.close();
};
```

---

## 10.4 Streaming con Advisors (RAG + Memory)

El streaming es totalmente compatible con la cadena de Advisors. El comportamiento es:
el Advisor **procesa la petición** antes de enviarla al modelo (de forma bloqueante),
y el modelo **responde en streaming**. El usuario empieza a ver tokens mientras el RAG
ya encontró los documentos relevantes.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  FLUJO: RAG + Memory + Streaming                                             │
│                                                                              │
│  Petición del usuario                                                        │
│          │                                                                   │
│          ▼ (bloqueante — antes del modelo)                                   │
│  [MessageChatMemoryAdvisor]  recupera historial de la conversación          │
│          │                                                                   │
│          ▼ (bloqueante — antes del modelo)                                   │
│  [QuestionAnswerAdvisor]  busca chunks en VectorStore, los añade al prompt  │
│          │                                                                   │
│          ▼ (streaming — desde el modelo)                                     │
│  [Modelo]  genera la respuesta token a token                                 │
│          │                                                                   │
│          ▼                                                                   │
│  [MessageChatMemoryAdvisor]  guarda respuesta en memoria cuando termina     │
│          │                                                                   │
│          ▼                                                                   │
│  Flux<String>  → cliente recibe tokens en tiempo real                        │
└──────────────────────────────────────────────────────────────────────────────┘
```

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClientConStreaming(
            ChatClient.Builder builder,
            VectorStore vectorStore,
            ChatMemory memory) {

        return builder
            .defaultSystem("Eres un asistente técnico. Responde basándote en la documentación.")
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(memory),
                new QuestionAnswerAdvisor(vectorStore, SearchRequest.defaults().withTopK(4))
            )
            .build();
    }
}

@RestController
@RequestMapping("/chat")
public class ChatStreamController {

    @Autowired ChatClient chatClient;

    // Streaming con RAG + Memory en un solo endpoint
    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chatConRagYMemoria(@RequestBody ChatRequest request) {
        return chatClient.prompt()
            .user(request.mensaje())
            .advisors(a -> a.param(
                AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY,
                request.conversationId()
            ))
            .stream()       // en lugar de .call()
            .content();     // Flux<String>
        // Los advisors hacen el RAG y la memoria automáticamente
        // El usuario recibe tokens en tiempo real
    }
}
```

> **Nota sobre Memory + Streaming:** el `MessageChatMemoryAdvisor` guarda el historial
> correctamente aunque uses streaming — espera a que el Flux complete antes de persistir
> el mensaje del asistente. No necesitas hacer nada especial.

---

## 10.5 Streaming con Tools — cómo funciona

Cuando el modelo necesita llamar a Tools durante el streaming, el comportamiento es
diferente a un streaming puro: el stream "pausa" mientras se ejecutan los tools y
"reanuda" cuando el modelo genera la respuesta final.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  STREAMING + TOOLS — lo que ocurre internamente                              │
│                                                                              │
│  Pregunta: "¿Qué tiempo hace en Madrid?"                                    │
│                                                                              │
│  1. Spring AI envía la pregunta al modelo en modo streaming                 │
│  2. El modelo detecta que necesita el tool getWeather("Madrid")              │
│     → devuelve una "tool call" (no texto) → el stream para                  │
│  3. Spring AI ejecuta getWeather("Madrid") en tu JVM (bloqueante)           │
│  4. Spring AI re-envía al modelo con el resultado del tool                  │
│  5. El modelo genera la respuesta final en streaming                        │
│     → "En Madrid hay 22°C y está soleado."  (token a token)                 │
│                                                                              │
│  El usuario ve: [pausa de ~50-200ms] → "En Madrid hay 22°C y está soleado." │
└──────────────────────────────────────────────────────────────────────────────┘
```

```java
@RestController
public class AssistantStreamController {

    @Autowired ChatClient chatClient;
    @Autowired WeatherTools weatherTools;

    @GetMapping(value = "/assistant/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> preguntar(@RequestParam String mensaje) {
        return chatClient.prompt()
            .user(mensaje)
            .tools(weatherTools)   // registra los tools para esta llamada
            .stream()
            .content();
        // Si el modelo usa tools: pausa → ejecuta tool → stream de la respuesta final
        // Si el modelo no usa tools: stream directo sin pausa
    }
}
```

---

## 10.6 Operaciones Flux útiles

`Flux<String>` es una secuencia reactiva. Puedes aplicar operaciones antes de enviarlo
al cliente.

```java
@Service
public class ChatStreamService {

    @Autowired ChatClient chatClient;

    // Obtener el stream de tokens
    public Flux<String> stream(String pregunta) {
        return chatClient.prompt()
            .user(pregunta)
            .stream()
            .content();
    }

    // Acumular todos los tokens en un String completo (si necesitas el texto entero)
    public Mono<String> streamComoString(String pregunta) {
        return chatClient.prompt()
            .user(pregunta)
            .stream()
            .content()
            .collect(Collectors.joining());  // Flux<String> → Mono<String>
    }

    // Filtrar o transformar tokens antes de enviar al cliente
    public Flux<String> streamFiltrado(String pregunta) {
        return chatClient.prompt()
            .user(pregunta)
            .stream()
            .content()
            .filter(token -> !token.isBlank())   // ignorar tokens vacíos
            .map(token -> token.replace("```", ""));  // eliminar markdown
    }

    // Añadir un token de finalización para que el cliente sepa cuándo paró
    public Flux<String> streamConFin(String pregunta) {
        return chatClient.prompt()
            .user(pregunta)
            .stream()
            .content()
            .concatWith(Flux.just("[FIN]"));   // último elemento marca el fin
    }

    // Obtener el ChatResponse completo (con metadata de tokens) al finalizar el stream
    public Flux<ChatResponse> streamConMetadata(String pregunta) {
        return chatClient.prompt()
            .user(pregunta)
            .stream()
            .chatResponse();  // Flux<ChatResponse> en lugar de Flux<String>
        // Cada ChatResponse contiene el token generado + metadata acumulada
    }
}
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CUÁNDO USAR CADA VARIANTE                                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│  .stream().content()          → Flux<String> — más común para SSE al browser│
│  .stream().chatResponse()     → Flux<ChatResponse> — si necesitas metadata   │
│                                 (tokens usados, finish reason) en el stream  │
│  .collect(Collectors.joining())→ bloquea hasta tener la respuesta completa   │
│                                  útil en @Service sin frontend reactivo      │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

> **[← Structured Output](05_structured_output.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Embeddings →](07_embeddings.md)**
