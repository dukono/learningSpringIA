<!-- navegación -->
> **[← Setup y entornos](02_setup_y_entornos.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Prompts →](04_prompts_y_templates.md)**

---

# Capítulo 03 — ChatClient en profundidad

> `ChatClient` es el componente central de Spring AI. Todo pasa por él.
> Este capítulo cubre desde el uso básico hasta patrones avanzados de producción.

## Contenido

- [6.1 Inyección y uso básico](#61-inyección-y-uso-básico)
- [6.2 System Prompt global](#62-chatclient-con-system-prompt-global)
- [6.3 Estructura completa de una petición](#63-estructura-completa-de-una-petición)
- [6.4 ChatResponse — respuesta completa con metadata](#64-chatresponse--respuesta-completa-con-metadata)
- [6.5 ChatOptions — todos los parámetros explicados](#65-chatoptions--todos-los-parámetros-explicados)
- [6.6 Múltiples ChatClients para distintos propósitos](#66-múltiples-chatclients-para-distintos-propósitos)
- [6.7 Manejo de errores y excepciones](#67-manejo-de-errores-y-excepciones)
- [6.8 ChatClient vs ChatModel — cuándo usar cada uno](#68-chatclient-vs-chatmodel--cuándo-usar-cada-uno)

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

## 6.5 ChatOptions — todos los parámetros explicados

`ChatOptions` controla cómo genera texto el modelo. Estos parámetros son los más importantes
que debes entender y ajustar según la tarea.

### Parámetros universales (todos los proveedores)

```java
// Con OpenAI como ejemplo — cada proveedor tiene su clase Options propia
OpenAiChatOptions options = OpenAiChatOptions.builder()
    .withModel("gpt-4o-mini")
    .withTemperature(0.7f)
    .withMaxTokens(2000)
    .withTopP(1.0f)
    .withFrequencyPenalty(0.0f)
    .withPresencePenalty(0.0f)
    .withStop(List.of("FIN", "---"))   // tokens que detienen la generación
    .build();
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PARÁMETROS CLAVE DE ChatOptions — qué hace cada uno                        │
├──────────────────────────┬──────────────────────────────────────────────────┤
│  model                   │  Qué modelo usar (string exacto del proveedor)   │
│  temperature (0.0-2.0)   │  Aleatoriedad de la respuesta                   │
│                          │  0.0 = siempre la respuesta más probable         │
│                          │  1.0 = variado pero coherente (defecto habitual) │
│                          │  2.0 = caótico, poco recomendable               │
│  maxTokens               │  Límite de tokens EN LA RESPUESTA (no el prompt) │
│                          │  Si la respuesta se corta → sube este valor      │
│  topP (0.0-1.0)          │  Nucleus sampling: solo considera los tokens     │
│                          │  cuya probabilidad acumulada suma ≤ topP         │
│                          │  Alternativa a temperature; no uses ambos        │
│  frequencyPenalty (-2/+2)│  Penaliza repetir tokens ya usados EN LA RESPUESTA│
│                          │  +1.0 = reduce mucho la repetición léxica        │
│  presencePenalty (-2/+2) │  Penaliza usar tokens que ya han aparecido       │
│                          │  +1.0 = fuerza al modelo a hablar de cosas nuevas│
│  stop                    │  Lista de strings que detienen la generación      │
│                          │  Útil para formatos estructurados                │
└──────────────────────────┴──────────────────────────────────────────────────┘
```

### Reglas prácticas de temperature

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  TEMPERATURE CORRECTA SEGÚN LA TAREA                                         │
├─────────────────────┬────────────────────────────────────────────────────────┤
│  0.0                │  Tests, código determinista, clasificación binaria     │
│  0.1 – 0.3          │  SQL, extracción de datos, análisis, parsing JSON      │
│  0.4 – 0.6          │  Resumen, traducción, documentación técnica            │
│  0.7 – 0.9          │  Chat general, asistentes, explicaciones didácticas    │
│  1.0 – 1.5          │  Brainstorming, generación de ideas, creatividad       │
│  > 1.5              │  Escritura muy creativa — respuestas impredecibles     │
└─────────────────────┴────────────────────────────────────────────────────────┘

⚠️  Error frecuente: usar temperature=0.9 para extracción de datos JSON.
    El modelo "varía" el formato → el parser falla → excepción en runtime.
    Para extracción de datos: temperature=0.0 o 0.1, siempre.
```

### Configurar opciones por defecto en el Builder

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("Eres un asistente técnico experto en Java.")
            .defaultOptions(OpenAiChatOptions.builder()
                .withModel("gpt-4o-mini")
                .withTemperature(0.3f)   // preciso para preguntas técnicas
                .withMaxTokens(1500)
                .build())
            .build();
    }
}
```

### Sobreescribir opciones en una llamada específica

```java
// Las opciones por llamada SOBREESCRIBEN las del builder (no se fusionan)
String respuestaCreativa = chatClient.prompt()
    .user("Dame 10 ideas de nombres para una startup de IA")
    .options(OpenAiChatOptions.builder()
        .withTemperature(1.2f)    // ← sobreescribe el 0.3 del builder para esta llamada
        .withMaxTokens(300)
        .build())
    .call()
    .content();
```

---

## 6.6 Múltiples ChatClients para distintos propósitos

En una aplicación real es frecuente necesitar varios `ChatClient` con configuraciones
distintas: uno para chat de usuario, otro para análisis preciso de datos, otro para
generación creativa.

```java
@Configuration
public class AiConfig {

    // ChatClient para conversación con usuarios — moderado y equilibrado
    @Bean("chatClientUsuario")
    public ChatClient chatClientUsuario(ChatClient.Builder builder) {
        return builder
            .defaultSystem("Eres un asistente amable que responde en español.")
            .defaultOptions(OpenAiChatOptions.builder()
                .withModel("gpt-4o-mini")
                .withTemperature(0.7f)
                .withMaxTokens(800)
                .build())
            .build();
    }

    // ChatClient para extracción/clasificación — muy preciso
    @Bean("chatClientAnalisis")
    public ChatClient chatClientAnalisis(ChatClient.Builder builder) {
        return builder
            .defaultSystem("Extrae datos exactos. Responde SOLO en JSON válido.")
            .defaultOptions(OpenAiChatOptions.builder()
                .withModel("gpt-4o")       // modelo más potente para análisis
                .withTemperature(0.0f)      // determinista
                .withMaxTokens(500)
                .build())
            .build();
    }

    // ChatClient para generación creativa — máxima variedad
    @Bean("chatClientCreativo")
    public ChatClient chatClientCreativo(ChatClient.Builder builder) {
        return builder
            .defaultOptions(OpenAiChatOptions.builder()
                .withModel("gpt-4o")
                .withTemperature(1.1f)
                .withMaxTokens(2000)
                .build())
            .build();
    }
}

// Inyección por nombre con @Qualifier
@Service
public class ContentService {

    private final ChatClient chatUsuario;
    private final ChatClient chatAnalisis;
    private final ChatClient chatCreativo;

    public ContentService(
            @Qualifier("chatClientUsuario")  ChatClient chatUsuario,
            @Qualifier("chatClientAnalisis") ChatClient chatAnalisis,
            @Qualifier("chatClientCreativo") ChatClient chatCreativo) {
        this.chatUsuario  = chatUsuario;
        this.chatAnalisis = chatAnalisis;
        this.chatCreativo = chatCreativo;
    }
}
```

---

## 6.7 Manejo de errores y excepciones

Las APIs de IA pueden fallar. Debes conocer qué excepciones puede lanzar `ChatClient`
y cómo manejarlas correctamente.

### Jerarquía de excepciones en Spring AI

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  EXCEPCIONES DE SPRING AI                                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  NonTransientAiException  — errores que NO tienen sentido reintentar:       │
│    ├── BadRequestException      → 400: prompt malformado, parámetros malos  │
│    ├── AuthenticationException  → 401: API key incorrecta o expirada        │
│    ├── ContentFilterException   → prompt rechazado por filtro de contenido  │
│    └── ModelNotAvailableException → el modelo especificado no existe         │
│                                                                              │
│  TransientAiException  — errores temporales que SÍ vale la pena reintentar: │
│    ├── RateLimitException       → 429: demasiadas peticiones                │
│    ├── ResourceUnavailableException → 503: servicio caído temporalmente     │
│    └── AiConnectionException   → fallo de red / timeout                    │
│                                                                              │
│  Ambas heredan de: AiException → RuntimeException                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Cómo manejarlas en la práctica

```java
@Service
public class ChatService {

    @Autowired ChatClient chatClient;

    public String preguntar(String mensaje) {
        try {
            return chatClient.prompt()
                .user(mensaje)
                .call()
                .content();

        } catch (AuthenticationException e) {
            // API key incorrecta — error de configuración, no reintentar
            log.error("API key inválida: {}", e.getMessage());
            throw new RuntimeException("Error de configuración de IA", e);

        } catch (ContentFilterException e) {
            // El prompt activó el filtro de contenido del proveedor
            log.warn("Prompt rechazado por filtro: {}", e.getMessage());
            return "Lo siento, no puedo procesar esa solicitud.";

        } catch (RateLimitException e) {
            // Rate limit — Spring AI tiene retry automático configurado
            // Si llegamos aquí, los reintentos se agotaron
            log.error("Rate limit agotado después de reintentos: {}", e.getMessage());
            return "El servicio está saturado. Inténtalo en unos minutos.";

        } catch (AiException e) {
            // Cualquier otro error de Spring AI
            log.error("Error inesperado de IA [{}]: {}", e.getClass().getSimpleName(), e.getMessage());
            return "Error temporal. Inténtalo de nuevo.";
        }
    }
}
```

### Timeouts — configuración

```yaml
# application.yml — controlar cuánto esperas la respuesta
spring:
  ai:
    openai:
      # Tiempo máximo esperando la conexión con la API (ms)
      connect-timeout: 5000
      # Tiempo máximo esperando LA RESPUESTA COMPLETA (ms)
      # Para respuestas largas o modelos lentos: aumentar este valor
      read-timeout: 60000
```

```
⚠️  El read-timeout es el tiempo total de espera, incluyendo el tiempo de
    generación. Un prompt que genera 2000 tokens puede tardar 20-30 segundos
    con GPT-4o. Ajusta según los maxTokens que configures.
```

---

## 6.8 ChatClient vs ChatModel — cuándo usar cada uno

Esta es la pregunta más frecuente al empezar. La respuesta corta: **usa siempre `ChatClient`**.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ChatClient  vs  ChatModel                                                   │
├──────────────────────────────────────┬───────────────────────────────────────┤
│  ChatClient                          │  ChatModel                            │
├──────────────────────────────────────┼───────────────────────────────────────┤
│  API fluida de alto nivel            │  API de bajo nivel, directa           │
│  Soporta Advisors (RAG, Memory...)   │  Sin advisors                         │
│  Soporta defaults (system, options)  │  Sin defaults — todo manual           │
│  Maneja historial con Advisors       │  Sin gestión de historial             │
│  .call().content()  o  .entity()     │  chatModel.call(prompt) → ChatResponse│
│  Recomendado para el 99% de los casos│  Solo casos muy específicos           │
└──────────────────────────────────────┴───────────────────────────────────────┘

¿Cuándo usar ChatModel directamente?
→ Solo si necesitas acceso a funcionalidades muy específicas del proveedor
  que ChatClient.Builder no expone, o si estás construyendo tu propio advisor.
→ Si dudas: usa ChatClient.
```

```java
// ✅ Forma correcta — siempre inyecta ChatClient.Builder, no ChatClient directamente
// Spring Boot auto-configura el Builder; de él construyes tu ChatClient
@Service
public class MiServicio {

    private final ChatClient chatClient;

    public MiServicio(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("Eres un asistente de Java.")
            .build();
    }
}

// ❌ No inyectes ChatModel directamente salvo que sea imprescindible
@Service
public class OtroServicio {
    @Autowired ChatModel chatModel;  // ← solo si tienes razón específica
}
```

---

> **[← Setup y entornos](02_setup_y_entornos.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Prompts →](04_prompts_y_templates.md)**
