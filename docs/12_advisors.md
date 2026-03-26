<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 16. Advisors — middleware de IA

Los **Advisors** son interceptores que se ejecutan antes y después de cada llamada al
modelo. Son el equivalente a los `Filter` o `Interceptor` de Spring MVC pero para IA.

### ¿Por qué existen los Advisors?

Sin Advisors, para añadir funcionalidades como memoria, RAG o logging tendrías que modificar
cada llamada al modelo manualmente. Los Advisors permiten añadir esas funcionalidades de
forma **declarativa y reutilizable**, una sola vez en la configuración.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SIN ADVISORS — código repetido en cada llamada:                             │
│                                                                              │
│  public String chat(String msg, String userId) {                             │
│      // 1. recuperar historial manualmente                                   │
│      // 2. buscar documentos en vectorStore manualmente                      │
│      // 3. construir el prompt a mano                                        │
│      // 4. llamar al modelo                                                  │
│      // 5. guardar en historial manualmente                                  │
│      // 6. loguear manualmente                                               │
│      // 7. devolver respuesta                                                │
│  }                                                                           │
│                                                                              │
│  CON ADVISORS — configuración única, código limpio:                          │
│                                                                              │
│  public String chat(String msg) {                                            │
│      return chatClient.prompt().user(msg).call().content();  // ← solo esto  │
│  }                                                                           │
│  // los advisors hacen todo lo demás automáticamente                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### El patrón de cadena — cómo fluye la petición

```
Petición → [Advisor 1] → [Advisor 2] → [Advisor 3] → [Modelo] → [Advisor 3] → [Advisor 2] → [Advisor 1] → Respuesta
             ANTES          ANTES          ANTES                   DESPUÉS        DESPUÉS        DESPUÉS

Ejemplo real con los 3 advisors más comunes:

Petición → [SimpleLoggerAdvisor] → [MessageChatMemoryAdvisor] → [QuestionAnswerAdvisor] → [Modelo]
             ANTES: loguea          ANTES: añade historial         ANTES: añade contexto RAG
             DESPUÉS: loguea        DESPUÉS: guarda en memoria      DESPUÉS: (nada)
```

### El orden de ejecución es CRÍTICO

El orden en que registras los advisors importa. Un error común es ponerlos en orden incorrecto:

```java
// ❌ MAL — SimpleLoggerAdvisor loguea ANTES de que el RAG añada el contexto:
// El log no incluirá los chunks del vector store
builder.defaultAdvisors(
    new SimpleLoggerAdvisor(),                    // orden 1 (primero)
    new QuestionAnswerAdvisor(vectorStore, ...),  // orden 2
    new MessageChatMemoryAdvisor(memory)          // orden 3
)

// ✅ BIEN — orden recomendado:
builder.defaultAdvisors(
    new MessageChatMemoryAdvisor(memory),         // 1. Añade historial
    new QuestionAnswerAdvisor(vectorStore, ...),  // 2. Añade contexto RAG
    new SimpleLoggerAdvisor()                     // 3. Loguea el prompt final completo
)
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ORDEN RECOMENDADO DE ADVISORS                                               │
├────┬─────────────────────────────┬────────────────────────────────────────── │
│ Nº │  Advisor                    │  Por qué en este orden                   │
├────┼─────────────────────────────┼──────────────────────────────────────────┤
│ 1  │  AuditAdvisor (custom)      │  Captura la petición original del usuario│
│ 2  │  SafeGuardAdvisor           │  Filtra contenido antes de procesar      │
│ 3  │  MessageChatMemoryAdvisor   │  Añade historial al prompt               │
│ 4  │  QuestionAnswerAdvisor      │  Añade contexto RAG al prompt            │
│ 5  │  SimpleLoggerAdvisor        │  Loguea el prompt final completo         │
└────┴─────────────────────────────┴──────────────────────────────────────────┘
```

### Advisors integrados en Spring AI

```java
// 1. QuestionAnswerAdvisor — hace RAG automáticamente
//    ANTES: convierte la pregunta en vector, busca en vectorStore, añade chunks al prompt
//    DESPUÉS: (nada adicional)
new QuestionAnswerAdvisor(vectorStore, SearchRequest.defaults())

// 2. MessageChatMemoryAdvisor — añade historial de conversación como mensajes
//    ANTES: recupera historial del ChatMemory y lo añade como mensajes [USER]/[ASSISTANT]
//    DESPUÉS: guarda la nueva pregunta y respuesta en el ChatMemory
new MessageChatMemoryAdvisor(chatMemory)

// 3. PromptChatMemoryAdvisor — añade historial dentro del system prompt (texto)
//    Alternativa a MessageChatMemoryAdvisor para modelos que manejan mejor texto plano
new PromptChatMemoryAdvisor(chatMemory)

// 4. SafeGuardAdvisor — filtra contenido peligroso
//    ANTES: revisa si la pregunta contiene palabras prohibidas → lanza excepción
//    DESPUÉS: (nada)
new SafeGuardAdvisor(List.of("hack", "explosivo", "veneno"))

// 5. SimpleLoggerAdvisor — loguea todas las peticiones/respuestas
//    Nivel de log: DEBUG por defecto
//    ANTES: loguea el prompt completo que se envía al modelo
//    DESPUÉS: loguea la respuesta completa recibida
new SimpleLoggerAdvisor()
```

### Crear tu propio Advisor

```java
@Component
public class AuditAdvisor implements CallAroundAdvisor {

    @Autowired AuditRepository auditRepo;

    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        String usuario = (String) request.adviseContext().get("userId");
        String pregunta = request.userText();
        long inicio = System.currentTimeMillis();

        // ANTES de enviar al modelo
        log.info("Usuario {} pregunta: {}", usuario, pregunta);

        // Continúa la cadena (llama al modelo y a los siguientes advisors)
        AdvisedResponse response = chain.nextAroundCall(request);

        // DESPUÉS de recibir la respuesta
        long duracion = System.currentTimeMillis() - inicio;
        String respuesta = response.response().getResult().getOutput().getContent();

        // Guardar auditoría en BD
        auditRepo.save(new AuditLog(
            usuario, pregunta, respuesta,
            duracion,            // cuánto tardó en ms
            LocalDateTime.now()
        ));

        return response;
    }

    @Override
    public String getName() {
        return "AuditAdvisor";
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;  // ejecuta primero (captura todo)
    }
}
```

### Pasar datos de contexto a los Advisors

Puedes pasar datos desde el controller hasta los advisors usando `adviseContext`:

```java
// En el controller — pasas el userId en el contexto
public String chat(String mensaje, String userId) {
    return chatClient.prompt()
        .user(mensaje)
        .advisors(a -> a
            .param(AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, userId)
            .param("userId", userId)          // disponible para tus advisors custom
            .param("tenant", "empresa-acme")  // cualquier dato que necesites
        )
        .call()
        .content();
}

// En tu advisor custom — lees el contexto
String userId  = (String) request.adviseContext().get("userId");
String tenant  = (String) request.adviseContext().get("tenant");
```

---


---

> **[← Volver al índice](00_indice.md)**
