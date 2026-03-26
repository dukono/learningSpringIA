<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 11. Embeddings — convertir texto en vectores

Un **embedding** es la representación numérica de un texto. Permite medir la
**similitud semántica** entre textos: dos frases con el mismo significado tendrán
vectores similares aunque usen palabras distintas.

### ¿Por qué existen los embeddings? El problema que resuelven

Las computadoras no entienden texto. Entienden números. Antes de los embeddings, el
enfoque era **bag of words** (contar palabras) o búsqueda exacta de texto. El problema:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  EL PROBLEMA DE BUSCAR POR TEXTO EXACTO                                      │
│                                                                              │
│  Usuario busca: "perro"                                                      │
│  Documento contiene: "can", "mascota", "animal doméstico"                   │
│  → Búsqueda exacta: NO ENCUENTRA el documento   ← error                     │
│                                                                              │
│  Usuario busca: "cómo configuro spring security"                             │
│  Documento dice: "implementar autenticación JWT en spring boot"             │
│  → Búsqueda exacta: NO ENCUENTRA el documento   ← error                     │
│                                                                              │
│  Los embeddings resuelven esto:                                              │
│  "perro" y "can" → vectores casi idénticos → SÍ los encuentra               │
│  "spring security JWT" y "autenticación spring boot" → vectores             │
│  muy cercanos → SÍ los encuentra                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Qué pasa internamente cuando generas un embedding

Un modelo de embeddings es una **red neuronal** entrenada con millones de textos para
que aprenda a colocar textos de significado similar cerca en un espacio de alta dimensión.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CÓMO FUNCIONA INTERNAMENTE                                                  │
│                                                                              │
│  Texto: "El perro ladra"                                                     │
│         │                                                                    │
│         ▼                                                                    │
│  [Tokenización]  →  ["El", "perro", "ladra"]  →  [tokens IDs: 123, 456, 789]│
│         │                                                                    │
│         ▼                                                                    │
│  [Red neuronal Transformer]  (aprendió de millones de textos)               │
│         │                                                                    │
│         ▼                                                                    │
│  Vector de 1536 números: [0.23, -0.85, 0.12, 0.67, 0.03, -0.44, ...]       │
│         │                                                                    │
│  Cada número = una "dimensión semántica" aprendida automáticamente           │
│  (no hay una regla; la red neuronal aprendió qué dimensiones capturan       │
│   mejor el significado del texto)                                            │
│                                                                              │
│  "El gato maúlla" → [0.22, -0.83, 0.13, 0.65, 0.04, -0.42, ...]            │
│                       ↑ muy cercano al vector anterior = mismo concepto      │
│                                                                              │
│  "La bolsa sube hoy" → [0.91, 0.34, -0.45, 0.02, 0.78, 0.33, ...]          │
│                          ↑ muy lejos = concepto completamente distinto       │
└──────────────────────────────────────────────────────────────────────────────┘
```

> **Importante:** no hay ningún número mágico que diga "dimensión 5 = concepto animal".
> La red neuronal aprendió sola cómo distribuir el significado en esas 1536 dimensiones.
> Lo que sí es real y medible: textos similares producen vectores cercanos.

### Concepto visual

```
"El perro corre"     → [0.23, -0.85, 0.12, 0.67, ...] (vector de 1536 números)
"El can corre"       → [0.24, -0.84, 0.11, 0.68, ...] muy similar (sinónimos)
"La pizza está rica" → [0.91,  0.34, -0.45, 0.02, ...] muy diferente
```

La **distancia** entre vectores = similitud semántica.

### Cómo se mide la similitud — Cosine Similarity

```
┌──────────────────────────────────────────────────────────────────────────┐
│  COSINE SIMILARITY — cómo se comparan vectores                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Valor 1.0  → textos idénticos o sinónimos exactos                       │
│  Valor 0.9  → textos muy relacionados ("perro" vs "can")                 │
│  Valor 0.7  → textos del mismo tema ("Java" vs "programación")           │
│  Valor 0.5  → poca relación                                              │
│  Valor 0.0  → textos sin ninguna relación semántica                      │
│  Valor -1.0 → textos de significado opuesto                              │
│                                                                          │
│  Ejemplo real:                                                           │
│  "¿Cómo configuro JWT?"  vs  "Spring Security JWT configuración"         │
│   → similitud: 0.89  ← muy alto = este documento es relevante            │
│                                                                          │
│  "¿Cómo configuro JWT?"  vs  "Recetas de cocina italiana"                │
│   → similitud: 0.12  ← muy bajo = este documento no es relevante         │
└──────────────────────────────────────────────────────────────────────────┘
```

### Modelos de embeddings disponibles

No todos los modelos producen vectores del mismo tamaño ni calidad:

```
┌────────────────────────────────────────────────────────────────────────────┐
│  MODELOS DE EMBEDDINGS EN SPRING AI                                        │
├─────────────────────────────────┬──────────┬──────────┬────────────────────┤
│  Modelo                         │  Dims    │  Coste   │  Cuándo usarlo     │
├─────────────────────────────────┼──────────┼──────────┼────────────────────┤
│  text-embedding-3-small (OpenAI)│  1536    │  $0.02   │  Producción        │
│                                 │          │  /1M tok │  equilibrio precio │
├─────────────────────────────────┼──────────┼──────────┼────────────────────┤
│  text-embedding-3-large (OpenAI)│  3072    │  $0.13   │  Máxima calidad    │
│                                 │          │  /1M tok │                    │
├─────────────────────────────────┼──────────┼──────────┼────────────────────┤
│  nomic-embed-text (Ollama)      │  768     │  GRATIS  │  Dev local         │
│                                 │          │  local   │  sin coste         │
├─────────────────────────────────┼──────────┼──────────┼────────────────────┤
│  mxbai-embed-large (Ollama)     │  1024    │  GRATIS  │  Dev local         │
│                                 │          │  local   │  más preciso       │
└─────────────────────────────────┴──────────┴──────────┴────────────────────┘

⚠️  REGLA CRÍTICA: las dimensiones deben coincidir en TODO el sistema.
    Si indexas con text-embedding-3-small (1536 dims) →
    DEBES buscar también con text-embedding-3-small (1536 dims).
    Mezclar modelos de distinto tamaño da resultados incorrectos.
```

### Uso en Spring AI

```java
@Service
public class EmbeddingService {

    @Autowired
    EmbeddingModel embeddingModel;

    // Obtener embedding de un texto
    public float[] obtenerVector(String texto) {
        float[] vector = embeddingModel.embed(texto);
        System.out.println("Dimensiones: " + vector.length);  // 1536 para OpenAI
        return vector;
    }

    // Comparar similitud entre dos textos
    public double calcularSimilitud(String texto1, String texto2) {
        EmbeddingResponse response = embeddingModel.embedForResponse(
            List.of(texto1, texto2)
        );
        float[] v1 = response.getResults().get(0).getOutput();
        float[] v2 = response.getResults().get(1).getOutput();

        // Calcular cosine similarity manualmente
        double dotProduct = 0, norm1 = 0, norm2 = 0;
        for (int i = 0; i < v1.length; i++) {
            dotProduct += v1[i] * v2[i];
            norm1 += v1[i] * v1[i];
            norm2 += v2[i] * v2[i];
        }
        return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
    }
}

// Uso:
double sim = service.calcularSimilitud("perro", "can");    // ~0.92
double sim2 = service.calcularSimilitud("perro", "pizza"); // ~0.15
```

### Ejemplo real end-to-end: buscador semántico

```java
// Caso de uso: buscar en un FAQ por significado, no por palabras exactas
@Service
public class FaqSemanticSearch {

    @Autowired EmbeddingModel embeddingModel;

    // Preguntas del FAQ con sus respuestas
    private final Map<String, String> faq = Map.of(
        "¿Cómo devuelvo un producto?",   "Tienes 30 días para devoluciones...",
        "¿Cuál es el plazo de envío?",   "Los pedidos llegan en 2-3 días laborables...",
        "¿Puedo cambiar mi pedido?",     "Puedes modificar hasta 1h después del pago...",
        "¿Dónde está mi paquete?",       "Consulta el seguimiento con tu número de pedido..."
    );

    // Al arrancar: genera embeddings de todas las preguntas del FAQ
    private Map<String, float[]> faqVectors;

    @PostConstruct
    public void indexarFaq() {
        faqVectors = new HashMap<>();
        for (String pregunta : faq.keySet()) {
            faqVectors.put(pregunta, embeddingModel.embed(pregunta));
        }
        System.out.println("FAQ indexado: " + faqVectors.size() + " entradas");
    }

    // Busca la pregunta del FAQ más similar a la consulta del usuario
    public String buscar(String consultaUsuario) {
        float[] vectorConsulta = embeddingModel.embed(consultaUsuario);

        String mejorPregunta = null;
        double mejorSimilitud = 0;

        for (Map.Entry<String, float[]> entry : faqVectors.entrySet()) {
            double similitud = cosineSimilarity(vectorConsulta, entry.getValue());
            if (similitud > mejorSimilitud) {
                mejorSimilitud = similitud;
                mejorPregunta = entry.getKey();
            }
        }

        if (mejorSimilitud < 0.6) {
            return "No encontré una respuesta relevante para tu pregunta.";
        }

        return faq.get(mejorPregunta);
    }
}
```

```
Consulta: "¿En cuánto tiempo me llega el pedido?"
                    │
                    ▼ (embedding de la consulta)
          [0.12, -0.45, 0.78, ...]
                    │
          Comparación con FAQ indexado:
          "¿Cómo devuelvo un producto?" → 0.31 (baja similitud)
          "¿Cuál es el plazo de envío?" → 0.89 (¡alta similitud!)
          "¿Puedo cambiar mi pedido?"   → 0.42
          "¿Dónde está mi paquete?"     → 0.51
                    │
                    ▼
          Respuesta: "Los pedidos llegan en 2-3 días laborables..."

→ Aunque el usuario usó "tiempo" y "llegar", encontró "plazo de envío"
→ Búsqueda por texto exacto no lo hubiera encontrado
```

### ¿Para qué sirven los embeddings?

```
┌─────────────────────────────────────────────────────────────────────────┐
│  USOS DE EMBEDDINGS                                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  Búsqueda semántica  → buscar por significado, no por palabras exactas  │
│  RAG                 → encontrar documentos relevantes para un prompt   │
│  Clasificación       → categorizar textos automáticamente               │
│  Deduplicación       → detectar contenido duplicado aunque sea distinto │
│  Recomendaciones     → "productos similares", "artículos relacionados"  │
│  Detección spam      → un email raro vs emails legítimos → vectores     │
│                        muy alejados → alerta automática                 │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Resumen conceptual:** los embeddings son el puente entre el texto humano y las
> matemáticas. Sin embeddings no existe RAG, no existe búsqueda semántica, no existe
> ninguna de las funcionalidades avanzadas de IA en tu app. Son la base de todo.

---


---

> **[← Volver al índice](00_indice.md)**
