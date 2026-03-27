<!-- navegación -->
> **[← Embeddings](07_embeddings.md)** | **[← Inicio](../README.md)** | **[Siguiente: RAG →](09_rag.md)**

---

# Capítulo 08 — Vector Store

> Base de datos especializada en almacenar y buscar vectores (embeddings)
> por similitud semántica. Fundamento técnico del RAG.

## 8. Vector Store — base de datos para IA

Un **Vector Store** es una base de datos especializada en almacenar y buscar vectores
(embeddings). En lugar de buscar por texto exacto o índices, busca por **similitud
semántica** usando cosine similarity.

### ¿Por qué no sirve una base de datos normal (SQL)?

Esta es la primera pregunta que todo desarrollador Java hace. Vamos a responderla con claridad:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SQL NORMAL vs VECTOR STORE                                                  │
├────────────────────────────────────────┬─────────────────────────────────────┤
│  SQL (PostgreSQL, MySQL...)             │  Vector Store                       │
├────────────────────────────────────────┼─────────────────────────────────────┤
│  SELECT * WHERE texto LIKE '%perro%'   │  Busca por similitud semántica       │
│  → Solo encuentra "perro"              │  → Encuentra "perro", "can",        │
│  → No encuentra "can" ni "mascota"     │    "mascota", "animal doméstico"     │
├────────────────────────────────────────┼─────────────────────────────────────┤
│  Índices B-Tree → eficientes para      │  Índices HNSW/IVFFlat → eficientes  │
│  igualdad y rangos                     │  para distancias en 1536 dimensiones │
├────────────────────────────────────────┼─────────────────────────────────────┤
│  Buscar el vector más cercano en SQL:  │  VectorStore.similaritySearch():     │
│  SELECT ... ORDER BY cosine_sim(v, ?)  │  usa índice especializado            │
│  LIMIT 5;   ← escanea TODA la tabla   │  → 1000x más rápido con millones     │
│  → con 1M vectores: >10 segundos       │    de vectores                       │
└────────────────────────────────────────┴─────────────────────────────────────┘
```

**¿Pero pgvector no es PostgreSQL?** Sí, pero la extensión `vector` añade a PostgreSQL
tipos de datos (`vector`) e índices especializados (`ivfflat`, `hnsw`) que no existen
en SQL estándar. Es PostgreSQL + capacidades de vector store.

### Cómo funciona el índice vectorial por dentro

Un índice HNSW (Hierarchical Navigable Small World) organiza los vectores en un grafo
jerárquico para encontrar los vecinos más cercanos sin comparar todos:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  BÚSQUEDA SIN ÍNDICE (brute force):                                          │
│                                                                              │
│  Tienes 1.000.000 vectores almacenados.                                      │
│  Consulta: "¿Cómo configuro JWT?"                                            │
│  → Compara tu vector con los 1.000.000 vectores uno a uno                    │
│  → 1.000.000 multiplicaciones de 1536 números cada una                       │
│  → Tiempo: varios segundos  ← inaceptable en producción                      │
│                                                                              │
│  BÚSQUEDA CON ÍNDICE HNSW:                                                   │
│                                                                              │
│  El índice organiza los vectores en capas de grafos:                         │
│  Capa 0 (más alta):  pocos nodos, conexiones largas (orientación general)   │
│  Capa 1:             más nodos, conexiones medias                            │
│  Capa 2 (más baja):  todos los nodos, conexiones finas (búsqueda exacta)    │
│                                                                              │
│  Búsqueda:                                                                   │
│  1. Empieza en Capa 0 → encuentra el nodo más cercano (aproximado)          │
│  2. Baja a Capa 1 → refina la búsqueda alrededor de ese nodo                │
│  3. Baja a Capa 2 → búsqueda exacta solo en la zona relevante               │
│  4. Compara solo ~100 vectores (no 1.000.000) → resultado en milisegundos   │
│                                                                              │
│  → Con 1.000.000 vectores: ~5ms  (en vez de segundos)                       │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Proveedores soportados

```
┌────────────────────────────────────────────────────────────────────────┐
│  VECTOR STORES EN SPRING AI                                            │
├─────────────────────┬──────────────────────────────────────────────────┤
│  pgvector           │  PostgreSQL con extensión vector — muy usado     │
│  Chroma             │  Open source, ideal para desarrollo              │
│  Pinecone           │  Managed cloud, escala masiva                    │
│  Weaviate           │  Cloud y self-hosted                             │
│  Redis              │  RedisSearch con vectores                         │
│  Qdrant             │  Open source, alto rendimiento                   │
│  Milvus             │  Enterprise, millones de vectores                │
│  SimpleVectorStore  │  En memoria — SOLO para tests/demos              │
└─────────────────────┴──────────────────────────────────────────────────┘
```

### Uso básico con SimpleVectorStore (en memoria)

```java
@Configuration
public class VectorStoreConfig {

    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        // Solo para tests y demos — los datos se pierden al reiniciar
        return SimpleVectorStore.builder(embeddingModel).build();
    }
}

@Service
public class DocumentService {

    @Autowired VectorStore vectorStore;

    // Guardar documentos (genera embeddings automáticamente)
    public void guardarDocumentos() {
        List<Document> docs = List.of(
            new Document("Spring Boot es un framework de Java para microservicios"),
            new Document("Docker permite contenerizar aplicaciones"),
            new Document("Kubernetes orquesta contenedores a escala")
        );

        vectorStore.add(docs);  // calcula embeddings y los guarda
    }

    // Buscar documentos similares a una consulta
    public List<Document> buscar(String consulta) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)
                .topK(3)                        // los 3 más similares
                .similarityThreshold(0.7)       // mínimo 70% similitud
                .build()
        );
    }
}
```

Búsqueda visual:

```
Consulta: "¿Cómo escalar microservicios?"

Vector de la consulta: [0.34, -0.12, 0.89, ...]

Comparación con vectores almacenados:
  doc1 "Spring Boot..."    → similitud: 0.65  (< 0.7, no pasa el umbral)
  doc2 "Docker..."         → similitud: 0.72  (pasa el umbral)
  doc3 "Kubernetes..."     → similitud: 0.91  (el más relevante)

Resultado: [doc3, doc2]  (ordenados de mayor a menor similitud)
```

### Uso con pgvector (PostgreSQL — producción)

**1. Levantar PostgreSQL con pgvector (Docker):**

```bash
docker run -d \
  --name pgvector \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=miapp \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

**2. Dependencia:**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
```

**3. Configuración:**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/miapp
    username: postgres
    password: secret
  ai:
    vectorstore:
      pgvector:
        initialize-schema: true    # crea la tabla vector automáticamente
        dimensions: 1536           # debe coincidir con el modelo de embeddings
                                   # OpenAI text-embedding-3-small → 1536
                                   # nomic-embed-text (Ollama) → 768
        index-type: HNSW           # tipo de índice: HNSW (más rápido) o IVFFLAT
        distance-type: COSINE_DISTANCE  # función de distancia
```

**4. ¿Qué tabla crea Spring AI automáticamente?**

```sql
-- Spring AI crea esto en tu PostgreSQL:
CREATE TABLE IF NOT EXISTS vector_store (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    content     TEXT,                        -- el texto original del chunk
    metadata    JSON,                        -- tus metadatos (filename, date, etc.)
    embedding   VECTOR(1536)                 -- el vector de 1536 dimensiones
);

-- Índice HNSW para búsqueda rápida por similitud
CREATE INDEX ON vector_store
USING hnsw (embedding vector_cosine_ops);
```

**5. Filtrado por metadata — buscar solo en documentos específicos:**

```java
// Buscar solo en documentos de un tipo o fecha específica
List<Document> resultados = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("¿Cómo devuelvo un producto?")
        .topK(5)
        .similarityThreshold(0.7)
        .filterExpression("filename == 'politica-devoluciones.pdf'")
        // otros filtros posibles:
        // "contentType == 'pdf' && uploadDate >= '2025-01-01'"
        // "department == 'RRHH'"
        .build()
);
```

> **¿Cuándo usar filtros?** Si tienes documentos de distintos clientes (multi-tenant),
> distintos departamentos o distintos proyectos, los filtros evitan que la IA mezcle
> información de unos con otros. Esencial para aplicaciones B2B.

## 8.3 Gestión del ciclo de vida de documentos

Más allá de añadir documentos, en producción necesitas **actualizar y eliminar** los documentos
ya indexados:

```java
@Service
public class DocumentLifecycleService {

    @Autowired VectorStore vectorStore;

    // ELIMINAR documentos por ID
    public void eliminar(List<String> ids) {
        vectorStore.delete(ids);
    }

    // ACTUALIZAR un documento: eliminar el viejo y añadir el nuevo
    // VectorStore no tiene update() — la actualización es delete + add
    public void actualizar(String idViejo, Document docNuevo) {
        vectorStore.delete(List.of(idViejo));
        vectorStore.add(List.of(docNuevo));
    }

    // ELIMINAR todos los documentos de un archivo (usando metadata)
    // Recupera primero los IDs con similaritySearch y luego elimina
    public void eliminarPorArchivo(String filename) {
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query("")          // query vacía para buscar por filtro solo
                .topK(1000)         // máximo documentos del archivo
                .filterExpression("filename == '" + filename + "'")
                .build()
        );
        List<String> ids = docs.stream().map(Document::getId).toList();
        if (!ids.isEmpty()) vectorStore.delete(ids);
    }
}
```

> 💡 `Document.getId()` devuelve el UUID asignado al insertar. Si quieres recuperar ese
> ID para usarlo después, guárdalo en tu propia base de datos al hacer `vectorStore.add()`.

## 8.4 FilterExpression — búsqueda filtrada por metadata

`FilterExpression` es el lenguaje de filtros de Spring AI para buscar solo en documentos
que cumplan ciertas condiciones de metadata. Es clave en aplicaciones multi-tenant o con
documentos de distintas categorías.

### ¿Qué metadata puede filtrarse?

Solo la metadata que **tú añadiste explícitamente** al crear el `Document`:

```java
// Añadir metadata al crear documentos
Document doc = new Document(
    "Contenido del documento...",
    Map.of(
        "filename",    "politica-devoluciones.pdf",
        "department",  "RRHH",
        "year",        2025,
        "confidential", false,
        "tags",        List.of("política", "devoluciones")
    )
);
```

### Sintaxis completa de FilterExpression

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  OPERADORES DE COMPARACIÓN                                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│  ==   igual                  filename == 'informe.pdf'                       │
│  !=   distinto               department != 'RRHH'                            │
│  >    mayor que              year > 2023                                     │
│  >=   mayor o igual          year >= 2024                                    │
│  <    menor que              year < 2026                                     │
│  <=   menor o igual          priority <= 3                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│  OPERADORES LÓGICOS                                                          │
├──────────────────────────────────────────────────────────────────────────────┤
│  &&   AND                    department == 'RRHH' && year >= 2024            │
│  ||   OR                     department == 'RRHH' || department == 'Legal'   │
│  !    NOT                    !(confidential == true)                         │
├──────────────────────────────────────────────────────────────────────────────┤
│  OPERADORES DE COLECCIÓN                                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│  in     valor en lista       department in ['RRHH', 'Legal', 'Finanzas']     │
│  nin    valor NO en lista    status nin ['borrador', 'obsoleto']             │
└──────────────────────────────────────────────────────────────────────────────┘

Notas de sintaxis:
  - Strings entre comillas simples: 'valor'
  - Números sin comillas: year >= 2024
  - Listas entre corchetes: ['a', 'b', 'c']
  - Booleanos sin comillas: confidential == true
```

### Ejemplos de FilterExpression en código

```java
// Caso 1: filtro simple — solo documentos de un archivo
vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("¿Cuál es la política de vacaciones?")
        .topK(5)
        .filterExpression("filename == 'politica-vacaciones.pdf'")
        .build()
);

// Caso 2: filtro compuesto — departamento Y año reciente
vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("procedimientos de contratación")
        .topK(5)
        .filterExpression("department == 'RRHH' && year >= 2024")
        .build()
);

// Caso 3: filtro con lista — varios departamentos permitidos
vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("normativa de gastos")
        .topK(5)
        .filterExpression("department in ['Finanzas', 'Dirección']")
        .build()
);

// Caso 4: excluir documentos obsoletos
vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("proceso de incorporación")
        .topK(5)
        .filterExpression("status nin ['borrador', 'obsoleto'] && year >= 2023")
        .build()
);
```

### FilterExpressionBuilder — construcción programática

Cuando el filtro se construye dinámicamente (por ejemplo, basado en el rol del usuario),
usa `FilterExpressionBuilder` en lugar de concatenar strings:

```java
import org.springframework.ai.vectorstore.filter.FilterExpressionBuilder;
import org.springframework.ai.vectorstore.filter.Filter.Expression;

@Service
public class RagService {

    @Autowired VectorStore vectorStore;

    public List<Document> buscarConFiltrosDinamicos(
            String consulta, String userId, List<String> departamentosPermitidos) {

        FilterExpressionBuilder b = new FilterExpressionBuilder();

        // Construir filtro basado en el perfil del usuario
        Expression filtro = b.and(
            b.in("department", departamentosPermitidos),   // solo sus departamentos
            b.eq("tenantId", userId),                      // aislamiento multi-tenant
            b.gte("year", 2024)                            // solo documentos recientes
        );

        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)
                .topK(5)
                .similarityThreshold(0.65)
                .filterExpression(filtro)
                .build()
        );
    }
}
```

**Métodos de `FilterExpressionBuilder`:**

```java
FilterExpressionBuilder b = new FilterExpressionBuilder();

b.eq("campo", "valor")          // campo == 'valor'
b.ne("campo", "valor")          // campo != 'valor'
b.gt("campo", 100)              // campo > 100
b.gte("campo", 100)             // campo >= 100
b.lt("campo", 100)              // campo < 100
b.lte("campo", 100)             // campo <= 100
b.in("campo", List.of("a","b")) // campo in ['a', 'b']
b.nin("campo", List.of("a","b"))// campo nin ['a', 'b']
b.and(expr1, expr2)             // expr1 && expr2
b.or(expr1, expr2)              // expr1 || expr2
b.not(expr)                     // !(expr)
```

> ⚠️ **Usar `FilterExpressionBuilder` en lugar de strings**: concatenar el filtro como
> string desde input del usuario es vulnerable a injection. Úsalo siempre que el filtro
> incluya valores dinámicos no controlados.

### Soporte por vector store

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SOPORTE DE FILTEREXPRESSION POR PROVEEDOR                                   │
├───────────────────────┬──────────────────────────────────────────────────────┤
│  pgvector             │  ✅ Completo — traduce a WHERE sobre columna metadata │
│  SimpleVectorStore    │  ✅ Completo — filtra en memoria                      │
│  Chroma               │  ✅ Completo                                          │
│  Pinecone             │  ✅ Parcial — no soporta OR entre campos distintos    │
│  Redis                │  ✅ Completo                                          │
│  Weaviate             │  ✅ Completo                                          │
└───────────────────────┴──────────────────────────────────────────────────────┘

Nota: el filtro se traduce al lenguaje nativo del proveedor. El mismo código Java
funciona con todos los stores — Spring AI hace la traducción internamente.
```

## Errores comunes con Vector Store

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES CON VECTOR STORE                                         │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  SQLException al arrancar:     │  El número de dimensiones en el YAML no    │
│  "expected 1536 dimensions,    │  coincide con las que genera el modelo de   │
│  got 768"                      │  embeddings configurado. pgvector valida    │
│                                │  el tamaño del vector al insertar.          │
│                                │  ✅ Verificar que dimensions en application │
│                                │  .yml coincide con el modelo:               │
│                                │  OpenAI text-embedding-3-small → 1536       │
│                                │  nomic-embed-text (Ollama) → 768            │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Los datos desaparecen al      │  Se está usando SimpleVectorStore en        │
│  reiniciar la aplicación       │  producción o staging. Este store es en     │
│                                │  memoria — pierde todo al parar la JVM.     │
│                                │  ✅ SimpleVectorStore solo para tests.      │
│                                │  Producción: pgvector, Chroma o cualquier   │
│                                │  store persistente.                         │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  ERROR: relation               │  initialize-schema está a false (o no       │
│  "vector_store" does not exist │  configurado). Spring AI no crea la tabla   │
│                                │  automáticamente si no se le indica.        │
│                                │  ✅ Añadir en YAML:                         │
│                                │  spring.ai.vectorstore.pgvector             │
│                                │    .initialize-schema: true                 │
│                                │  (solo la primera vez; en prod es mejor     │
│                                │  usar Flyway o Liquibase para el schema).   │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  La búsqueda no devuelve nada  │  El umbral de similitud es demasiado alto.  │
│  aunque hay documentos         │  Con threshold(0.9) solo aparecen textos    │
│  relevantes                    │  casi idénticos. Los documentos reales      │
│                                │  rara vez superan 0.85 con consultas        │
│                                │  distintas aunque sean muy relevantes.      │
│                                │  ✅ Empezar con threshold(0.5) y ajustar    │
│                                │  subiendo hasta que los falsos positivos    │
│                                │  sean aceptables. 0.65-0.75 es habitual.    │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  La indexación de muchos       │  vectorStore.add() se llama con un documento│
│  documentos es muy lenta       │  a la vez en un bucle. Cada llamada genera  │
│                                │  una petición al modelo de embeddings y una │
│                                │  inserción en la BD.                        │
│                                │  ✅ Pasar la lista completa en una sola     │
│                                │  llamada: vectorStore.add(listaDeDocs)      │
│                                │  Spring AI hace el batch automáticamente.   │
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Embeddings](07_embeddings.md)** | **[← Inicio](../README.md)** | **[Siguiente: RAG →](09_rag.md)**
