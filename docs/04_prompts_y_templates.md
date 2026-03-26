<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 7. Prompts — cómo hablarle a la IA

Un **Prompt** es el objeto que contiene todos los mensajes que envías al modelo.

### 7.1 Tipos de mensajes

```java
import org.springframework.ai.chat.messages.*;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.model.ChatModel;

// Mensaje del sistema (comportamiento del asistente)
Message system = new SystemMessage("Eres un poeta que rima todo lo que dice.");

// Mensaje del usuario (la pregunta actual)
Message user = new UserMessage("Háblame del café");

// Mensaje del asistente (para simular historia de conversación)
Message assistant = new AssistantMessage("El café es la bebida del programador...");

// Construir el Prompt con varios mensajes
Prompt prompt = new Prompt(List.of(system, user));

// Enviarlo con ChatModel (API de bajo nivel — solo si no usas ChatClient)
@Autowired ChatModel chatModel;
ChatResponse response = chatModel.call(prompt);
String texto = response.getResult().getOutput().getContent();
```

> **Nota:** Si usas `ChatClient` (recomendado) no necesitas construir `Prompt` manualmente.
> La API fluida `.system()`, `.user()` lo hace por ti internamente.

### 7.2 Visualización de los roles

```
┌──────────────────────────────────────────────────────────────────┐
│                    COMO LO VE EL MODELO                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [SYSTEM]                                                        │
│  "Eres un experto en Java. Responde siempre con código."        │
│                                                                  │
│  [USER]                                                          │
│  "¿Cómo hago un singleton?"                                     │
│                                                                  │
│  [ASSISTANT]  generado por el modelo                            │
│  "Aquí tienes el patrón Singleton en Java:                      │
│   public class Singleton { ... }"                               │
│                                                                  │
│  [USER]                                                          │
│  "¿Y cómo lo hago thread-safe?"   nueva pregunta con contexto   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 7.3 Ingeniería de prompts — cómo escribir prompts efectivos

La calidad de la respuesta depende directamente de la calidad del prompt. Esto se llama
**Prompt Engineering** y es una habilidad fundamental para cualquier desarrollador que
trabaje con IA.

#### Prompts malos vs buenos

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PROMPT MALO:                    │  PROMPT BUENO:                            │
│  "Analiza esto"                  │  "Analiza el siguiente log de error Java   │
│                                  │   e identifica: (1) la causa raíz,        │
│                                  │   (2) en qué línea ocurre, y              │
│                                  │   (3) cómo solucionarlo.                   │
│                                  │   Responde en español con código           │
│                                  │   corregido."                              │
├──────────────────────────────────┼────────────────────────────────────────── │
│  "Resume"                        │  "Resume el siguiente texto en máximo 3   │
│                                  │   puntos clave. Cada punto = 1 frase.     │
│                                  │   Usa lenguaje formal y técnico."          │
├──────────────────────────────────┼────────────────────────────────────────── │
│  "Traduce al inglés"             │  "Traduce al inglés americano formal.      │
│                                  │   Mantén los términos técnicos sin         │
│                                  │   traducir (nombres de clases Java)."      │
└──────────────────────────────────┴──────────────────────────────────────────┘

Regla: cuanto más específico y concreto, mejor resultado.
```

#### Técnica 1 — Few-Shot: dar ejemplos dentro del prompt

En lugar de explicar qué quieres, **muestra ejemplos de entrada→salida**. El modelo aprende
el patrón de los ejemplos:

```java
String systemPrompt = """
    Eres un clasificador de tickets de soporte.
    Clasificas cada ticket en: BUG, FEATURE, QUESTION o BILLING.

    Ejemplos:
    Ticket: "La app se cierra cuando hago login"         → BUG
    Ticket: "¿Podéis añadir modo oscuro?"                → FEATURE
    Ticket: "¿Cómo exporto mis datos en CSV?"            → QUESTION
    Ticket: "Mi factura de enero tiene un importe mal"   → BILLING

    Responde SOLO con la categoría, sin explicación.
    """;

String resultado = chatClient.prompt()
    .system(systemPrompt)
    .user("La página de reportes no carga ningún dato desde ayer")
    .call()
    .content();
// resultado: "BUG"
```

#### Técnica 2 — Chain of Thought: pedir razonamiento paso a paso

Para problemas complejos, pedir al modelo que **piense paso a paso** antes de dar la
respuesta mejora drásticamente la precisión:

```java
// ✅ Con Chain of Thought:
String prompt = """
    Revisa el siguiente contrato e identifica cláusulas problemáticas.

    IMPORTANTE: Analiza el contrato párrafo a párrafo explicando tu razonamiento.
    Al final, da un resumen de los problemas encontrados.

    Contrato: {contrato}
    """;

// ❌ Sin Chain of Thought:
String promptSimple = "¿Hay cláusulas problemáticas en este contrato? {contrato}";
// → respuestas menos precisas y más superficiales
```

#### Técnica 3 — System prompt efectivo

El system prompt define quién es el asistente. Un buen system prompt incluye:

```java
String systemPrompt = """
    Eres un asistente técnico senior especializado en Spring Boot.

    TU ROL:
    - Ayudas a desarrolladores Java con problemas técnicos concretos
    - Das ejemplos de código funcionales, no pseudocódigo
    - Señalas errores comunes y cómo evitarlos

    CÓMO RESPONDES:
    - Siempre en español
    - Conciso: primero la respuesta, luego la explicación
    - Con código formateado en bloques ```java
    - Si algo puede fallar, lo adviertes con un comentario

    LO QUE NO HACES:
    - No inventas APIs que no existen
    - No das respuestas genéricas para preguntas técnicas
    - Si no sabes algo, dices "no estoy seguro" en vez de inventar
    """;
```

#### Temperatura correcta según la tarea

```
┌───────────────────────────────────────────────────────────────────┐
│  TEMPERATURA vs TIPO DE TAREA                                     │
├────────────────────────────┬──────────────────────────────────────┤
│  0.0 → 0.2                 │  Clasificación, extracción de datos, │
│  (muy determinista)        │  SQL, análisis de código, parsing     │
├────────────────────────────┼──────────────────────────────────────┤
│  0.3 → 0.5                 │  Traducción, resúmenes,              │
│  (ligeramente creativo)    │  documentación técnica               │
├────────────────────────────┼──────────────────────────────────────┤
│  0.6 → 0.8                 │  Chat general, respuestas didácticas,│
│  (moderadamente creativo)  │  explicaciones, asistencia           │
├────────────────────────────┼──────────────────────────────────────┤
│  0.9 → 1.0                 │  Escritura creativa, brainstorming,  │
│  (muy creativo/variado)    │  generación de ideas, storytelling   │
└────────────────────────────┴──────────────────────────────────────┘

Error común: usar temperature=1.0 para extracción de datos
→ el modelo "inventa" variaciones en los datos extraídos
```

#### Errores comunes en prompts

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERROR 1: Pedir demasiadas cosas en una sola llamada                         │
│  ❌ "Analiza el código, busca bugs, refactorízalo y añade tests"              │
│  ✅ Divide en llamadas separadas (primero analiza, luego refactoriza)         │
├──────────────────────────────────────────────────────────────────────────────┤
│  ERROR 2: No especificar el formato de salida                                │
│  ❌ "Dame los datos del cliente"                                              │
│  ✅ "Dame los datos en JSON con campos: nombre, email, telefono (string)"    │
├──────────────────────────────────────────────────────────────────────────────┤
│  ERROR 3: Instrucciones negativas ambiguas                                   │
│  ❌ "No respondas de forma incorrecta"                                        │
│  ✅ "Si no tienes datos suficientes, responde exactamente:                    │
│      'No dispongo de información para responder esto.'"                      │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

`PromptTemplate` funciona como un template de texto donde puedes inyectar variables,
igual que Thymeleaf o FreeMarker pero para prompts.

```java
@Service
public class ReviewService {

    private final ChatClient chatClient;

    public ReviewService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String analizarReview(String producto, String review) {

        // Plantilla con variables entre {}
        String template = """
            Analiza la siguiente review del producto "{producto}":
            
            Review: {review}
            
            Proporciona:
            1. Sentimiento (positivo/negativo/neutro)
            2. Puntos positivos mencionados
            3. Puntos negativos mencionados
            4. Puntuación del 1 al 10
            
            Responde en formato JSON.
            """;

        // Inyectar variables
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "producto", producto,
            "review", review
        ));

        return chatClient.prompt(prompt)
                .call()
                .content();
    }
}
```

> **Error común:** Si una variable del template no está en el Map, Spring AI lanza
> `IllegalArgumentException: "Variable {nombre} not found"`. Asegúrate de que
> todas las variables `{...}` del template tienen su clave en el Map.

Llamada de ejemplo y resultado:

```java
// Llamada
String resultado = reviewService.analizarReview(
    "Teclado mecánico XK-900",
    "Excelente calidad de construcción y el envío fue rapidísimo. Precio algo alto."
);
```

```json
{
  "sentimiento": "positivo",
  "puntos_positivos": ["excelente calidad de construcción", "envío rápido"],
  "puntos_negativos": ["precio alto"],
  "puntuacion": 8
}
```

### 8.2 StringTemplate — prompts con lógica condicional (spring-ai-template-st)

`PromptTemplate` con `{variables}` simples es suficiente para la mayoría de casos, pero
a veces necesitas **condicionales, bucles o formato dinámico** dentro del propio template.
Para eso existe **StringTemplate**, incluido en el artefacto `spring-ai-template-st`.

#### Dependencia

```xml
<!-- pom.xml — artefacto específico para StringTemplate -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-template-st</artifactId>
</dependency>
```

#### Diferencia fundamental

```
┌────────────────────────────────────────────────────────────────────────────┐
│  PromptTemplate (básico)          │  StringTemplate (avanzado)             │
├────────────────────────────────────┼────────────────────────────────────────┤
│  Variables simples: {nombre}       │  Condicionales: <if(condicion)>        │
│  Solo sustitución de texto        │  Bucles: <lista:{ item | <item> }>     │
│  Suficiente para el 80% de casos  │  Formato complejo de prompts           │
│  Más legible para prompts simples │  Para prompts muy dinámicos            │
└────────────────────────────────────┴────────────────────────────────────────┘
```

#### Ejemplo — prompt con condicional

```java
import org.springframework.ai.template.st.StPromptTemplateFactory;

@Service
public class ProductDescriptionService {

    private final ChatClient chatClient;
    private final StPromptTemplateFactory stFactory;

    public ProductDescriptionService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
        this.stFactory = new StPromptTemplateFactory();
    }

    public String generarDescripcion(String producto, boolean esParaNinos, List<String> caracteristicas) {

        // StringTemplate — usa <if(...)> y <lista:{...}> para lógica
        String template = """
            Genera una descripción de marketing para el producto: <producto>.
            
            <if(esParaNinos)>
            El público objetivo son NIÑOS de 3 a 10 años.
            Usa lenguaje simple, divertido y llamativo.
            <else>
            El público objetivo son ADULTOS.
            Usa lenguaje profesional y técnico.
            <endif>
            
            Características del producto:
            <caracteristicas:{ item | - <item>
            }>
            
            La descripción debe tener 2 párrafos.
            """;

        PromptTemplate promptTemplate = stFactory.create(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "producto",         producto,
            "esParaNinos",      esParaNinos,
            "caracteristicas",  caracteristicas
        ));

        return chatClient.prompt(prompt).call().content();
    }
}
```

#### Resultado para adultos vs niños

```
// Llamada para adultos:
generarDescripcion("Auriculares XK-Pro", false, List.of("40h batería", "Noise cancelling", "Bluetooth 5.3"))

// Prompt generado internamente:
"Genera una descripción de marketing para el producto: Auriculares XK-Pro.
El público objetivo son ADULTOS.
Usa lenguaje profesional y técnico.
Características del producto:
- 40h batería
- Noise cancelling
- Bluetooth 5.3
La descripción debe tener 2 párrafos."

// Llamada para niños:
generarDescripcion("Mochila Espacial", true, List.of("LED parpadeante", "Compartimento secreto", "Diseño cohete"))

// Prompt generado internamente:
"Genera una descripción de marketing para el producto: Mochila Espacial.
El público objetivo son NIÑOS de 3 a 10 años.
Usa lenguaje simple, divertido y llamativo.
Características del producto:
- LED parpadeante
- Compartimento secreto
- Diseño cohete
La descripción debe tener 2 párrafos."
```

> **¿Cuándo usar StringTemplate?** Cuando tu prompt cambia de estructura según condiciones
> (no solo de contenido). Si solo cambias valores de variables, `PromptTemplate` es suficiente
> y más legible. Para el 80% de los casos, no necesitas StringTemplate.

---


---

> **[← Volver al índice](00_indice.md)**
