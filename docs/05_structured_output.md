<!-- navegación -->
> **[← Prompts](04_prompts_y_templates.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Streaming →](06_streaming.md)**

---

# Capítulo 05 — Structured Output

> En lugar de recibir texto y parsear JSON manualmente, Spring AI convierte
> automáticamente la respuesta del modelo en objetos Java tipados.

## Contenido

- [9.1 Con Records (recomendado en Java 21+)](#91-con-records-recomendado-en-java-21)
- [9.2 Con listas](#92-con-listas)
- [9.3 Con Map para datos libres](#93-con-map-para-datos-libres)
- [9.4 Cómo funciona internamente — BeanOutputConverter](#94-cómo-funciona-internamente--beanoutputconverter)
- [9.5 Manejo de errores de parsing](#95-manejo-de-errores-de-parsing)
- [9.6 Validación con Jakarta Bean Validation](#96-validación-con-jakarta-bean-validation)
- [9.7 Objetos anidados y grafos complejos](#97-objetos-anidados-y-grafos-complejos)

---

## 9. Structured Output — respuestas tipadas

En lugar de recibir texto y parsear JSON manualmente, Spring AI puede convertir
automáticamente la respuesta en un objeto Java.

### 9.1 Con Records (recomendado en Java 21+)

```java
// Define el record que quieres recibir
record AnalisisProducto(
    String sentimiento,           // "positivo", "negativo", "neutro"
    List<String> puntosPositivos,
    List<String> puntosNegativos,
    int puntuacion                // del 1 al 10
) {}

// Uso con .entity()
@Service
public class ReviewService {

    public AnalisisProducto analizar(String review) {
        return chatClient.prompt()
            .user("Analiza esta review: " + review)
            .call()
            .entity(AnalisisProducto.class);  // conversión automática
    }
}
```

```
┌───────────────────────────────────────────────────────────────────┐
│  LO QUE HACE Spring AI internamente con .entity():                │
│                                                                   │
│  1. Lee los campos de AnalisisProducto.class via reflexión        │
│  2. Genera instrucciones JSON automáticamente en el prompt:       │
│     "Responde SOLO con JSON con estos campos:                     │
│      sentimiento (String), puntosPositivos (array)..."            │
│  3. El modelo responde en JSON                                    │
│  4. Spring AI deserializa con Jackson a AnalisisProducto          │
│  5. Tú recibes un objeto Java tipado listo para usar              │
│                                                                   │
│  Si el modelo no devuelve JSON válido → lanza excepción           │
│  (poco frecuente con GPT-4o, más frecuente con modelos pequeños)  │
└───────────────────────────────────────────────────────────────────┘
```

### 9.2 Con listas

```java
record Receta(String nombre, List<String> ingredientes, int minutosPreparacion) {}

List<Receta> recetas = chatClient.prompt()
    .user("Dame 3 recetas fáciles con pollo")
    .call()
    .entity(new ParameterizedTypeReference<List<Receta>>() {});

// recetas.get(0).nombre()       → "Pollo al ajillo"
// recetas.get(0).ingredientes() → ["pollo", "ajo", "aceite", "perejil"]
// recetas.get(0).minutosPreparacion() → 25
```

### 9.3 Con Map para datos libres

```java
Map<String, Object> datos = chatClient.prompt()
    .user("Dame datos sobre España en formato clave-valor")
    .call()
    .entity(new ParameterizedTypeReference<Map<String, Object>>() {});

// datos = {"capital": "Madrid", "poblacion": 47000000, "idioma": "español", ...}
String capital = (String) datos.get("capital");  // "Madrid"
```

---

## 9.4 Cómo funciona internamente — BeanOutputConverter

Entender lo que ocurre internamente es fundamental para diagnosticar problemas.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  LO QUE HACE Spring AI cuando llamas a .entity(MiClase.class)               │
│                                                                              │
│  PASO 1 — Antes de enviar al modelo:                                         │
│  BeanOutputConverter inspecciona MiClase via reflexión.                      │
│  Genera automáticamente instrucciones JSON en el prompt:                     │
│                                                                              │
│  "Responde EXCLUSIVAMENTE con un objeto JSON válido con esta estructura:     │
│   {                                                                          │
│     \"sentimiento\": string,                                                 │
│     \"puntuacion\": number (entero),                                         │
│     \"puntosPositivos\": array de strings                                   │
│   }                                                                          │
│   No incluyas explicaciones, markdown ni texto adicional."                   │
│                                                                              │
│  PASO 2 — El modelo genera una respuesta:                                    │
│  {"sentimiento":"positivo","puntuacion":8,"puntosPositivos":["envío rápido"]}│
│                                                                              │
│  PASO 3 — Spring AI deserializa con Jackson:                                 │
│  ObjectMapper.readValue(respuesta, MiClase.class)                            │
│  → objeto Java tipado listo para usar                                        │
│                                                                              │
│  Si el modelo no devuelve JSON válido → lanza ShutdownOutputParserException  │
└──────────────────────────────────────────────────────────────────────────────┘
```

> **Implicación práctica:** el modelo necesita ver instrucciones JSON en el prompt
> para responder en formato estructurado. Esto consume tokens adicionales
> (aprox. 100-200 tokens extra por llamada con `.entity()`). Es un coste razonable
> frente a parsear JSON manualmente.

### Usar BeanOutputConverter directamente

A veces necesitas más control (separar la instrucción de formato del mensaje):

```java
@Service
public class AnalysisService {

    @Autowired ChatClient chatClient;

    public AnalisisProducto analizar(String review) {
        // Instanciar el converter directamente
        BeanOutputConverter<AnalisisProducto> converter =
            new BeanOutputConverter<>(AnalisisProducto.class);

        // Obtener las instrucciones JSON que genera el converter
        String instruccionesFormato = converter.getFormat();
        // → "Responde SOLO con JSON válido con esta estructura: {...}"

        String respuestaTexto = chatClient.prompt()
            .system("Eres un analista de reviews. " + instruccionesFormato)
            .user("Analiza: " + review)
            .call()
            .content();

        // Deserializar manualmente
        return converter.convert(respuestaTexto);
    }
}
```

---

## 9.5 Manejo de errores de parsing

El caso más frecuente: el modelo no devuelve JSON válido. Ocurre más con modelos
pequeños (llama3.2, phi-3) que con GPT-4o o Claude.

### Cuándo falla el parsing

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CAUSAS DE FALLO DE PARSING (de más a menos frecuente)                       │
├──────────────────────────────────────────────────────────────────────────────┤
│  1. Modelo pequeño / local → añade explicaciones antes o después del JSON   │
│     Respuesta: "Aquí tienes el análisis:\n{\"sentimiento\":\"positivo\"...}" │
│                ↑ el texto antes del { rompe el parsing                       │
├──────────────────────────────────────────────────────────────────────────────┤
│  2. Temperature alta → el modelo "varía" la estructura del JSON              │
│     Respuesta: {"Sentimiento": "positivo"} (mayúscula != minúscula en Java)  │
├──────────────────────────────────────────────────────────────────────────────┤
│  3. Respuesta cortada → maxTokens demasiado bajo → JSON incompleto           │
│     Respuesta: {"sentimiento":"positivo","puntuacion":8,"puntosPositivos":["│
│                                                                  ↑ cortado   │
├──────────────────────────────────────────────────────────────────────────────┤
│  4. Modelo nuevo / cambiado que no sigue bien las instrucciones JSON         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Estrategia de manejo con retry manual

```java
@Service
public class ResilientStructuredService {

    @Autowired ChatClient chatClient;

    public AnalisisProducto analizarConRetry(String review) {
        int intentos = 0;
        int maxIntentos = 3;
        Exception ultimoError = null;

        while (intentos < maxIntentos) {
            try {
                return chatClient.prompt()
                    .user("Analiza esta review: " + review)
                    .call()
                    .entity(AnalisisProducto.class);

            } catch (Exception e) {
                ultimoError = e;
                intentos++;
                log.warn("Intento {} fallido en parsing JSON: {}", intentos, e.getMessage());
                // Si falla el parsing, el siguiente intento puede obtener JSON válido
            }
        }

        // Si después de 3 intentos sigue fallando, devuelve un objeto por defecto
        log.error("No se pudo parsear la respuesta después de {} intentos", maxIntentos, ultimoError);
        return new AnalisisProducto("desconocido", List.of(), List.of(), 0);
    }
}
```

### Soluciones preventivas (mejor que el retry)

```java
// ✅ SOLUCIÓN 1: bajar la temperature para structured output
chatClient.prompt()
    .user(pregunta)
    .options(OpenAiChatOptions.builder().withTemperature(0.0f).build())
    .call()
    .entity(MiClase.class);

// ✅ SOLUCIÓN 2: ser explícito en el system prompt
chatClient.prompt()
    .system("""
        IMPORTANTE: Responde EXCLUSIVAMENTE con el objeto JSON solicitado.
        Sin texto adicional, sin markdown, sin explicaciones, sin bloques ```json```.
        Solo el objeto JSON puro.
        """)
    .user(pregunta)
    .call()
    .entity(MiClase.class);

// ✅ SOLUCIÓN 3: subir maxTokens si la respuesta puede ser grande
chatClient.prompt()
    .user(pregunta)
    .options(OpenAiChatOptions.builder().withMaxTokens(1000).build())
    .call()
    .entity(MiClase.class);
```

---

## 9.6 Validación con Jakarta Bean Validation

Spring AI no valida el contenido de los campos — solo verifica que el JSON es parseable.
Para validar el contenido usa las anotaciones estándar de Jakarta.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```java
// Con anotaciones de validación en el record
record AnalisisProducto(
    @NotBlank
    String sentimiento,           // no puede ser null ni vacío

    @Min(1) @Max(10)
    int puntuacion,               // debe estar entre 1 y 10

    @NotNull @Size(min = 0, max = 20)
    List<String> puntosPositivos, // lista no nula, máx 20 elementos

    @NotNull
    List<String> puntosNegativos
) {}

// Validar después de deserializar
@Service
public class ValidatedAnalysisService {

    @Autowired ChatClient chatClient;
    @Autowired Validator validator;  // jakarta.validation.Validator

    public AnalisisProducto analizarYValidar(String review) {
        AnalisisProducto resultado = chatClient.prompt()
            .user("Analiza: " + review)
            .options(OpenAiChatOptions.builder().withTemperature(0.0f).build())
            .call()
            .entity(AnalisisProducto.class);

        // Validar que los valores están dentro del rango esperado
        Set<ConstraintViolation<AnalisisProducto>> violaciones =
            validator.validate(resultado);

        if (!violaciones.isEmpty()) {
            String errores = violaciones.stream()
                .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                .collect(Collectors.joining(", "));
            throw new IllegalStateException("Respuesta de IA inválida: " + errores);
        }

        return resultado;
    }
}
```

---

## 9.7 Objetos anidados y grafos complejos

Spring AI maneja automáticamente objetos Java anidados, listas de objetos y
estructuras complejas.

```java
// Ejemplo: respuesta con objetos anidados
record Direccion(String calle, String ciudad, String codigoPostal) {}

record Empresa(
    String nombre,
    String sector,
    int empleados,
    Direccion sede,                    // objeto anidado
    List<String> productos,            // lista de strings
    List<Contacto> contactos           // lista de objetos
) {}

record Contacto(String nombre, String email, String cargo) {}

// Spring AI genera las instrucciones JSON para toda la estructura automáticamente
Empresa empresa = chatClient.prompt()
    .user("Extrae todos los datos de esta empresa del siguiente texto: " + textoEmpresa)
    .options(OpenAiChatOptions.builder().withTemperature(0.0f).build())
    .call()
    .entity(Empresa.class);

// La respuesta incluye toda la jerarquía tipada:
// empresa.sede().ciudad() → "Madrid"
// empresa.contactos().get(0).email() → "juan@empresa.com"
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  JSON generado por el modelo para el ejemplo anterior:                       │
│                                                                              │
│  {                                                                           │
│    "nombre": "TechCorp S.L.",                                                │
│    "sector": "software",                                                     │
│    "empleados": 150,                                                         │
│    "sede": {                                                                 │
│      "calle": "Paseo de la Castellana 100",                                  │
│      "ciudad": "Madrid",                                                     │
│      "codigoPostal": "28046"                                                 │
│    },                                                                        │
│    "productos": ["ERP", "CRM", "BI Dashboard"],                              │
│    "contactos": [                                                            │
│      {"nombre": "Ana García", "email": "ana@techcorp.es", "cargo": "CEO"},   │
│      {"nombre": "Luis Pérez", "email": "luis@techcorp.es", "cargo": "CTO"}   │
│    ]                                                                         │
│  }                                                                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

> **Límite práctico:** estructuras con más de 3-4 niveles de anidamiento o más
> de 10-15 campos empiezan a degradar la precisión, especialmente con modelos
> pequeños. Para extracción compleja con muchos campos, considera dividir en
> varias llamadas.

---

> **[← Prompts](04_prompts_y_templates.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Streaming →](06_streaming.md)**
