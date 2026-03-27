<!-- navegación -->
> **[← Tools](10_tools.md)** | **[← Inicio](../README.md)** | **[Siguiente: Advisors →](12_advisors.md)**

---

# Capítulo 11 — Memory

> Cómo mantener el historial de conversación para que el modelo
> "recuerde" entre peticiones. Implementaciones en memoria y persistente (BD).

## 11. Memory / Conversation History — contexto de conversación

Por defecto los modelos de IA son **stateless**: cada petición es independiente y el
modelo no recuerda conversaciones anteriores. Para hacer un chatbot real necesitas
mantener el historial.

### 11.1 El problema — stateless por diseño

```
Sin memory:
  User:  "Me llamo Carlos"
  Bot:   "Hola Carlos, ¿en qué te puedo ayudar?"
  User:  "¿Cómo me llamo?"
  Bot:   "No sé cómo te llamas."   ← perdió el contexto completamente
```

**¿Por qué son stateless?** Los modelos reciben un bloque de texto (el prompt) y generan
una respuesta. Punto. No hay conexión persistente, no hay sesión, no hay memoria interna
entre llamadas. **Cada llamada es como la primera vez para el modelo.**

La solución es sencilla conceptualmente: **incluir el historial de la conversación en
cada nuevo prompt**. El modelo "recuerda" porque se lo contamos nosotros cada vez.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CÓMO FUNCIONA LA MEMORIA — lo que se envía al modelo en cada turno         │
│                                                                              │
│  Turno 1 (primer mensaje):                                                   │
│  ┌─────────────────────────────────┐                                         │
│  │ [SYSTEM] Eres un asistente...  │                                         │
│  │ [USER] "Me llamo Carlos"       │                                         │
│  └─────────────────────────────────┘                                         │
│  Respuesta: "Hola Carlos!"                                                   │
│                                                                              │
│  Turno 2 (el historial se añade):                                            │
│  ┌──────────────────────────────────────────────┐                            │
│  │ [SYSTEM]    Eres un asistente...             │                            │
│  │ [USER]      "Me llamo Carlos"   ← del turno 1│                            │
│  │ [ASSISTANT] "Hola Carlos!"      ← del turno 1│                            │
│  │ [USER]      "¿Cómo me llamo?"   ← turno 2   │                            │
│  └──────────────────────────────────────────────┘                            │
│  Respuesta: "Te llamas Carlos."  ✓                                           │
│                                                                              │
│  Turno 5 (historial crece):                                                  │
│  ┌──────────────────────────────────────────────┐                            │
│  │ [SYSTEM]    ...                              │  ← siempre                 │
│  │ [USER]      turno 1                          │  ← historia                │
│  │ [ASSISTANT] turno 1                          │                            │
│  │ [USER]      turno 2                          │                            │
│  │ [ASSISTANT] turno 2                          │                            │
│  │ [USER]      turno 3                          │                            │
│  │ [ASSISTANT] turno 3                          │                            │
│  │ [USER]      turno 4                          │                            │
│  │ [ASSISTANT] turno 4                          │                            │
│  │ [USER]      "nueva pregunta"                 │  ← actual                  │
│  └──────────────────────────────────────────────┘                            │
│                                                                              │
│  Cada turno = más tokens = más caro y más lento                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 El problema de la ventana de contexto y el coste

Este enfoque tiene un problema real que debes conocer:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PROBLEMA: el historial crece sin límite                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Conversación de 50 turnos ≈ 20.000 tokens (solo el historial)              │
│                                                                              │
│  Consecuencias:                                                              │
│  💰 Coste: pagas por TODOS los tokens del historial en CADA llamada          │
│     → turno 50: pagas los 20.000 tokens de historial + la nueva pregunta    │
│                                                                              │
│  ⏱️  Latencia: más tokens = más lento                                         │
│                                                                              │
│  🚧 Límite: GPT-4o tiene 128K tokens. Si el historial supera ese límite,    │
│     la llamada falla con error                                               │
│                                                                              │
│  Solución: truncar el historial — solo mantener los últimos N turnos        │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 11.3 Solución con MessageChatMemoryAdvisor

```java
@Configuration
public class ChatConfig {

    @Bean
    public ChatMemory chatMemory() {
        // Memoria en RAM — se pierde al reiniciar la app
        return new InMemoryChatMemory();
    }

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder, ChatMemory memory) {
        return builder
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(memory)
                    .conversationHistoryWindowSize(10)  // máximo 10 turnos (20 mensajes)
                    .build()
            )
            .build();
    }
}

@RestController
public class ChatbotController {

    @Autowired ChatClient chatClient;

    @PostMapping("/chat")
    public String chat(@RequestBody ChatRequest request) {
        return chatClient.prompt()
            .user(request.mensaje())
            .advisors(a -> a.param(
                AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY,
                request.conversationId()    // ID único por conversación/usuario
            ))
            .call()
            .content();
    }
}

record ChatRequest(String conversationId, String mensaje) {}
```

### 11.4 Los dos tipos de advisor de memoria — diferencias importantes

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  MessageChatMemoryAdvisor vs PromptChatMemoryAdvisor                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MessageChatMemoryAdvisor:                                                   │
│  → Añade el historial como mensajes [USER]/[ASSISTANT] separados            │
│  → El modelo los ve como una conversación real                               │
│  → Formato nativo de chat — RECOMENDADO para chat                           │
│                                                                              │
│  Lo que se envía al modelo:                                                  │
│  [SYSTEM] instrucciones...                                                   │
│  [USER] "hola soy Carlos"         ← del historial                           │
│  [ASSISTANT] "hola Carlos!"       ← del historial                           │
│  [USER] "¿me recuerdas?"          ← pregunta actual                         │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────── │
│                                                                              │
│  PromptChatMemoryAdvisor:                                                    │
│  → Añade el historial como texto dentro del SYSTEM prompt                   │
│  → Útil para modelos que no soportan bien el formato de chat multi-turn     │
│                                                                              │
│  Lo que se envía al modelo:                                                  │
│  [SYSTEM] "instrucciones...                                                  │
│   Historial de la conversación:                                              │
│   - Usuario dijo: hola soy Carlos                                            │
│   - Asistente respondió: hola Carlos!                                        │
│  "                                                                           │
│  [USER] "¿me recuerdas?"          ← pregunta actual                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 11.5 Cómo funciona internamente

```
Conversación ID: "user-123"

Turno 1:
  Enviado al modelo:
    [USER: "Me llamo Carlos"]
  Respuesta: "Hola Carlos!"
  Guardado en InMemoryChatMemory["user-123"]:
    [{role:USER, msg:"Me llamo Carlos"}, {role:ASSISTANT, msg:"Hola Carlos!"}]

Turno 2:
  El advisor recupera el historial de "user-123"
  y lo añade ANTES de la nueva pregunta:
    [USER: "Me llamo Carlos"]       ← historial
    [ASSISTANT: "Hola Carlos!"]     ← historial
    [USER: "¿Cómo me llamo?"]       ← nueva pregunta
  Respuesta: "Te llamas Carlos."  ✓
```

### 11.6 Memoria persistente con base de datos — JdbcChatMemory

`JdbcChatMemory` guarda el historial en una tabla SQL, por lo que sobrevive a reinicios
y es compatible con despliegues de múltiples instancias.

#### Dependencia

```xml
<!-- Ya incluida si tienes spring-boot-starter-data-jdbc o spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

#### Schema de base de datos requerido

Spring AI **no crea la tabla automáticamente**. Debes crearla tú con Flyway, Liquibase
o directamente en el script de inicialización:

```sql
-- Para PostgreSQL / MySQL / H2 (compatible con todos)
CREATE TABLE IF NOT EXISTS spring_ai_chat_memory (
    id            BIGSERIAL PRIMARY KEY,       -- PostgreSQL; en MySQL: BIGINT AUTO_INCREMENT
    conversation_id VARCHAR(255) NOT NULL,     -- ID de la conversación (tu userId o sessionId)
    content       TEXT          NOT NULL,      -- contenido del mensaje (JSON serializado)
    type          VARCHAR(50)   NOT NULL,      -- "USER", "ASSISTANT" o "SYSTEM"
    created_at    TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Índice para acelerar la consulta por conversationId (imprescindible en producción)
CREATE INDEX IF NOT EXISTS idx_chat_memory_conversation
    ON spring_ai_chat_memory (conversation_id, created_at);
```

> 💡 Con H2 en tests no necesitas crear la tabla manualmente si activas
> `spring.sql.init.mode=always` junto con un archivo `schema.sql` en `src/test/resources/`.

#### Configuración del bean

```java
@Configuration
public class ChatMemoryConfig {

    // Desarrollo: en memoria (se pierde al reiniciar)
    @Bean
    @Profile("dev")
    public ChatMemory inMemoryChatMemory() {
        return new InMemoryChatMemory();
    }

    // Producción: persistente en BD
    @Bean
    @Profile("prod")
    public ChatMemory jdbcChatMemory(JdbcTemplate jdbcTemplate) {
        return new JdbcChatMemory(jdbcTemplate);
    }
}
```

#### Cómo limpiar el historial de una conversación

Cuando el usuario hace "nuevo chat" debes borrar el historial para que el modelo
no arrastre el contexto anterior:

```java
@Service
public class ChatService {

    @Autowired ChatClient  chatClient;
    @Autowired ChatMemory  chatMemory;
    @Autowired JdbcTemplate jdbcTemplate;  // solo si usas JdbcChatMemory

    public String chat(String conversationId, String mensaje) {
        return chatClient.prompt()
            .user(mensaje)
            .advisors(a -> a.param(
                AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId))
            .call().content();
    }

    // Llamar cuando el usuario pulsa "Nueva conversación"
    public void limpiarHistorial(String conversationId) {
        // Funciona tanto con InMemoryChatMemory como con JdbcChatMemory
        chatMemory.clear(conversationId);
    }

    // Alternativa directa en BD (más eficiente para JdbcChatMemory en prod)
    public void limpiarHistorialBD(String conversationId) {
        jdbcTemplate.update(
            "DELETE FROM spring_ai_chat_memory WHERE conversation_id = ?",
            conversationId
        );
    }
}
```

#### Tabla de decisión

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ¿QUÉ IMPLEMENTACIÓN DE ChatMemory USAR?                                     │
├────────────────────────┬─────────────────────────────────────────────────────┤
│  InMemoryChatMemory    │  Tests, demos locales. Pierde todo al reiniciar.    │
│  JdbcChatMemory        │  Producción. Requiere tabla en BD. Persistente.     │
│  CassandraChatMemory   │  Alta escala. Requiere Apache Cassandra.            │
│  RedisChatMemory       │  Baja latencia. Requiere Redis. TTL configurable.   │
└────────────────────────┴─────────────────────────────────────────────────────┘

Regla: siempre configura conversationHistoryWindowSize para controlar el coste.
```

## 11.7 Errores comunes

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES CON MEMORY                                               │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El chatbot "olvida" todo       │  No se está pasando el conversationId en    │
│  entre mensajes del mismo       │  cada petición. Sin ese ID, el advisor      │
│  usuario                        │  no sabe qué historial recuperar y cada     │
│                                 │  mensaje se trata como conversación nueva.  │
│                                 │  ✅ Siempre pasar un ID estable por usuario:│
│                                 │  .advisors(a -> a.param(                    │
│                                 │    CHAT_MEMORY_CONVERSATION_ID_KEY, userId))│
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El coste de la API se          │  No se ha configurado                       │
│  dispara con conversaciones     │  conversationHistoryWindowSize. Por defecto │
│  largas                         │  Spring AI incluye TODO el historial.       │
│                                 │  Con 50 turnos y respuestas largas, cada    │
│                                 │  llamada incluye 20K+ tokens de historial.  │
│                                 │  ✅ Configurar un tamaño máximo:            │
│                                 │  MessageChatMemoryAdvisor.builder(memory)   │
│                                 │    .conversationHistoryWindowSize(10)       │
│                                 │    .build()                                 │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  InMemoryChatMemory en prod:    │  InMemoryChatMemory guarda el historial en  │
│  los usuarios pierden la        │  la JVM. Al reiniciar la app (deploy, crash,│
│  conversación al reiniciar      │  escalado) se pierde todo el historial.     │
│                                 │  ✅ En producción usar JdbcChatMemory o     │
│                                 │  cualquier implementación persistente.      │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  InMemoryChatMemory crece       │  Sin ningún mecanismo de limpieza, la       │
│  sin límite y consume           │  memoria RAM crece indefinidamente si hay   │
│  demasiada RAM                  │  muchos usuarios con historiales largos.    │
│                                 │  ✅ Configurar windowSize para truncar.     │
│                                 │  En producción, JdbcChatMemory + una tarea  │
│                                 │  programada que borre historiales antiguos. │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Error: múltiples beans         │  Si tienes varios ChatClient con distintos  │
│  ChatMemory y Spring no sabe    │  ChatMemory, Spring no puede inyectar por   │
│  cuál inyectar                  │  tipo. Necesitas cualificarlos.             │
│                                 │  ✅ Usar @Qualifier("memoriaInmediata") o   │
│                                 │  @Bean con nombre explícito.                │
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Tools](10_tools.md)** | **[← Inicio](../README.md)** | **[Siguiente: Advisors →](12_advisors.md)**
