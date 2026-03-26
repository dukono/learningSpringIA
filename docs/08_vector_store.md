<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 12. Vector Store — base de datos para IA

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
            SearchRequest.query(consulta)
                .withTopK(3)                        // los 3 más similares
                .withSimilarityThreshold(0.7)       // mínimo 70% similitud
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
    SearchRequest.query("¿Cómo devuelvo un producto?")
        .withTopK(5)
        .withSimilarityThreshold(0.7)
        .withFilterExpression("filename == 'politica-devoluciones.pdf'")
        // otros filtros posibles:
        // "contentType == 'pdf' && uploadDate >= '2025-01-01'"
        // "department == 'RRHH'"
);
```

> **¿Cuándo usar filtros?** Si tienes documentos de distintos clientes (multi-tenant),
> distintos departamentos o distintos proyectos, los filtros evitan que la IA mezcle
> información de unos con otros. Esencial para aplicaciones B2B.

---


---

> **[← Volver al índice](00_indice.md)**
