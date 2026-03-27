<!-- navegación -->
> **[← Memory](11_memory.md)** | **[← Inicio](../README.md)** | **[Siguiente: Multimodalidad →](13_multimodalidad.md)**

---

# Capítulo 12 — Advisors

> Interceptores que se ejecutan antes y después de cada llamada al modelo.
> Equivalente a los Filter/Interceptor de Spring MVC, pero para IA.

## 12. Advisors — middleware de IA

Los **Advisors** son interceptores que se ejecutan antes y después de cada llamada al
modelo. Son el equivalente a los `Filter` o `Interceptor` de Spring MVC pero para IA.

### 12.1 ¿Por qué existen los Advisors?

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

### 12.2 El patrón de cadena — cómo fluye la petición

```
Petición → [Advisor 1] → [Advisor 2] → [Advisor 3] → [Modelo] → [Advisor 3] → [Advisor 2] → [Advisor 1] → Respuesta
             ANTES          ANTES          ANTES                   DESPUÉS        DESPUÉS        DESPUÉS

Ejemplo real con los 3 advisors más comunes:

Petición → [SimpleLoggerAdvisor] → [MessageChatMemoryAdvisor] → [QuestionAnswerAdvisor] → [Modelo]
             ANTES: loguea          ANTES: añade historial         ANTES: añade contexto RAG
             DESPUÉS: loguea        DESPUÉS: guarda en memoria      DESPUÉS: (nada)
```

### 12.3 El orden de ejecución es CRÍTICO

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

### 12.4 Advisors integrados en Spring AI

```java
// 1a. QuestionAnswerAdvisor — hace RAG automáticamente (API clásica)
//     ANTES: convierte la pregunta en vector, busca en vectorStore, añade chunks al prompt
//     DESPUÉS: (nada adicional)
new QuestionAnswerAdvisor(vectorStore, SearchRequest.builder().build())

// 1b. RetrievalAugmentationAdvisor — API modular de RAG (1.1.x, recomendada)
//     Igual que QuestionAnswerAdvisor pero con DocumentRetriever intercambiable.
//     Permite usar fuentes de datos distintas a un VectorStore (BBDD, API, ficheros...).
RetrievalAugmentationAdvisor.builder()
    .documentRetriever(
        VectorStoreDocumentRetriever.builder()
            .vectorStore(vectorStore)
            .searchRequest(SearchRequest.builder().topK(5).build())
            .build()
    )
    .build()

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

### 12.5 Crear tu propio Advisor

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

### 12.5b StreamAroundAdvisor — advisors para streaming

`CallAroundAdvisor` funciona con `.call()` (respuesta bloqueante). Para `.stream()` existe
la interfaz equivalente: `StreamAroundAdvisor`. Un advisor que necesite operar en ambos
modos debe implementar **las dos interfaces**.

```java
@Component
public class StreamingAuditAdvisor implements CallAroundAdvisor, StreamAroundAdvisor {

    // --- Modo bloqueante (.call()) ---
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        log.info("[CALL] pregunta: {}", request.userText());
        AdvisedResponse response = chain.nextAroundCall(request);
        log.info("[CALL] respuesta recibida");
        return response;
    }

    // --- Modo streaming (.stream()) ---
    @Override
    public Flux<AdvisedResponse> aroundStream(
            AdvisedRequest request, StreamAroundAdvisorChain chain) {

        log.info("[STREAM] pregunta: {}", request.userText());
        long inicio = System.currentTimeMillis();

        return chain.nextAroundStream(request)
            .doOnComplete(() ->
                log.info("[STREAM] completado en {}ms",
                         System.currentTimeMillis() - inicio));
    }

    @Override public String getName()  { return "StreamingAuditAdvisor"; }
    @Override public int    getOrder() { return Ordered.HIGHEST_PRECEDENCE; }
}
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CallAroundAdvisor vs StreamAroundAdvisor                                    │
├──────────────────────────┬───────────────────────────────────────────────────┤
│  CallAroundAdvisor       │  Intercepta .call() — respuesta completa de golpe │
│  StreamAroundAdvisor     │  Intercepta .stream() — Flux<AdvisedResponse>     │
│  Ambas interfaces        │  Advisor compatible con los dos modos              │
└──────────────────────────┴───────────────────────────────────────────────────┘

Nota: QuestionAnswerAdvisor y MessageChatMemoryAdvisor implementan AMBAS interfaces
internamente — por eso funcionan igual en .call() y en .stream().
```

> ⚠️ Si implementas solo `CallAroundAdvisor` y lo usas con `.stream()`, Spring AI
> lanza `IllegalStateException: no StreamAroundAdvisor found`. Implementa siempre
> las dos si tu advisor puede usarse en ambos contextos.

---

### 12.6 Pasar datos de contexto a los Advisors

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

## 12.7 Errores comunes

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES CON ADVISORS                                             │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El SimpleLoggerAdvisor no     │  El orden de los advisors es incorrecto.    │
│  loguea el contexto RAG ni     │  Si el logger va antes que                  │
│  el historial de memoria        │  QuestionAnswerAdvisor y                   │
│                                │  MessageChatMemoryAdvisor, loguea el prompt │
│                                │  original sin los chunks ni el historial.   │
│                                │  ✅ Poner SimpleLoggerAdvisor el último,    │
│                                │  así loguea el prompt final completo.       │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Un Advisor custom no recibe   │  El Advisor no está registrado como bean de │
│  las dependencias de Spring    │  Spring. Si se instancia con new en el      │
│  (@Autowired no funciona)      │  builder, Spring no inyecta dependencias.   │
│                                │  ✅ Declararlo como @Component y pasarlo    │
│                                │  como bean inyectado al builder, no como    │
│                                │  new MiAdvisor() sin dependencias.          │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El advisor de seguridad       │  SafeGuardAdvisor está posicionado después  │
│  (SafeGuard) no bloquea        │  de un advisor que ya modificó el prompt.   │
│  contenido que debería         │  Evalúa el texto modificado, no el original.│
│  bloquear                      │  ✅ Poner SafeGuardAdvisor el primero,      │
│                                │  antes de cualquier otro advisor.           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El advisor custom no se       │  getOrder() devuelve el mismo valor que     │
│  ejecuta en el orden esperado  │  otro advisor ya registrado. Spring ejecuta │
│                                │  advisors con el mismo orden en orden        │
│                                │  indefinido.                                │
│                                │  ✅ Usar valores de orden explícitos y      │
│                                │  únicos: Ordered.HIGHEST_PRECEDENCE,        │
│                                │  Ordered.LOWEST_PRECEDENCE, o valores como  │
│                                │  100, 200, 300...                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Los advisors registrados en   │  Los advisors en defaultAdvisors() se usan  │
│  defaultAdvisors() se duplican │  en TODAS las llamadas. Si además los añades│
│  en algunas peticiones         │  con .advisors() en la petición concreta,   │
│                                │  se ejecutan dos veces: historial duplicado, │
│                                │  RAG duplicado, etc.                        │
│                                │  ✅ Los advisors de configuración global van │
│                                │  en defaultAdvisors(). Los temporales o de  │
│                                │  contexto van en .advisors() de la petición.│
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Memory](11_memory.md)** | **[← Inicio](../README.md)** | **[Siguiente: Multimodalidad →](13_multimodalidad.md)**
