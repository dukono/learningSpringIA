# Arquitectura — Sistema de Aprendizaje Guiado

## Stack

| Capa | Tecnología |
|---|---|
| Frontend | React + React Query + react-flow |
| Backend | Spring Boot 3 + Spring AI 1.1.0 + Spring Batch |
| Base de datos | MongoDB |
| Tiempo real | SSE (Server-Sent Events) |
| LLM | Spring AI ChatClient (Ollama dev / proveedor externo prod) |

---

## Flujo completo

```
USUARIO                    FRONTEND              BACKEND                    LLM
  │                           │                     │                        │
  │  rellena formulario        │                     │                        │
  │ ─────────────────────────► │                     │                        │
  │                           │  POST /paths        │                        │
  │                           │ ──────────────────► │                        │
  │                           │  { pathId }         │ inicia Job             │
  │                           │ ◄────────────────── │ ──────────────────────►│
  │                           │                     │                        │
  │                           │  SSE /paths/{id}/progress                    │
  │                           │ ◄══════════════════ │◄═══════ actualizaciones│
  │  ve progreso en tiempo real│                     │                        │
  │ ◄───────────────────────── │                     │                        │
  │                           │                     │ generación completa    │
  │                           │  GET /paths/{id}    │                        │
  │                           │ ──────────────────► │                        │
  │  navega su ruta            │  ruta completa      │                        │
  │ ◄───────────────────────── │ ◄────────────────── │                        │
```

---

## Modelo de datos — MongoDB

### Colección `learning_paths`

```javascript
{
  _id: ObjectId,
  userId: String,
  status: "pending | generating | ready | error",
  createdAt: Date,

  input: {
    tema:         "Java 25",
    nivel_actual: "nunca programé",
    objetivo:     "trabajo como desarrollador junior"
    // sin campo 'tiempo' — la profundidad la determina la competencia, no el tiempo
  },

  meta: {
    dominio_tipo:       "lenguaje",
    version_referencia: "Java 25",
    prefijo:            "java-",
    patrones_contenido: { ... }
  },

  progress: {
    phase:            "meta | domain | route | blocks | integrations | diagrams",
    blocks_total:     42,
    blocks_done:      17,
    integrations_total: 9,
    integrations_done:  4,
    diagrams_done:    17
  }
}
```

### Colección `nodes`

```javascript
{
  _id: ObjectId,
  pathId:              ObjectId,
  nodeId:              "B012",
  tipo:                "bloque | integracion",
  orden:               12,
  nivel_integracion:   null,        // "micro | meso | macro" si tipo=integracion

  // comunes
  prerequisitos: ["B010", "B011"],
  desbloquea:    ["B013", "T03"],

  // ── si tipo = bloque ──────────────────────────────────────
  objetivo:      "El aprendiz puede recorrer una lista con for en Java 21",
  skip_question: "¿Puedes escribir un bucle for sin ayuda?",
  contenido: {
    concepto:  "...",   // markdown + diagramas Mermaid (enriquecido por DiagramEnricher)
    ejemplo: {
      codigo:      "...",
      explicacion: "..."
    }
  },
  practica: {
    enunciado: "...",
    criterios: ["...", "..."],
    pista:     "...",   // oculta por defecto
    solucion:  "..."    // oculta hasta que el usuario entrega
  },
  qa: [
    { pregunta: "...", respuesta: "..." }
  ],

  // ── si tipo = integracion ─────────────────────────────────
  nivel_integracion: "micro",        // micro | meso | macro
  combina:           ["B010", "B011", "B012"],
  enunciado:         "...",
  criterios:         ["...", "..."],
  pista:             "...",
  solucion:          "..."
}
```

### Colección `user_progress`

```javascript
{
  _id: ObjectId,
  userId: String,
  pathId: ObjectId,
  nodes: [
    {
      nodeId:          "B001",
      estado:          "no_iniciado | saltado | en_progreso | completado",
      intentos:        2,
      fecha_completado: ISODate("2026-05-26")
    }
  ],
  completados: 8,
  porcentaje:  19
}
```

---

## Backend — capas

```
┌─────────────────────────────────────────────────────────────┐
│  API Layer  (Controllers REST + SSE)                        │
│  LearningPathController  BlockController  ProgressController│
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Service Layer                                              │
│  LearningPathService   ProgressService   NodeService        │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Batch Layer  (Spring Batch)                                │
│  LearningPathJob                                            │
│    Step 1: MetaStep          (secuencial)                   │
│    Step 2: DomainStep        (secuencial, 1 item/módulo)    │
│    Step 3: RouteStep         (secuencial)                   │
│    ── parallelFlow ──────────────────────────────────────── │
│    Step 4: BlocksStep        (paralelo — TaskExecutor)      │
│    Step 5: IntegrationsStep  (paralelo — TaskExecutor)      │
│    Step 6: DiagramStep       (paralelo — TaskExecutor)      │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  AI Layer  (Spring AI)                                      │
│  MetaGenerator    DomainAnalyzer    DomainValidator         │
│  RouteGenerator   BlockGenerator    TaskGenerator           │
│  DiagramEnricher                                            │
│  — todos usan ChatClient + Structured Output —              │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Repository Layer  (Spring Data MongoDB)                    │
│  LearningPathRepo   NodeRepo   UserProgressRepo             │
└─────────────────────────────────────────────────────────────┘
```

---

## Pipeline de prompts — mapeo a componentes

| Prompt | Componente Spring AI | Spring Batch Step | Output |
|---|---|---|---|
| `prompt-init.md` | — (UI) | — | `premeta.md` |
| `prompt-meta.md` | `MetaGenerator` | `MetaStep` | meta document en MongoDB |
| `prompt-dominio.md` | `DomainAnalyzer` | `DomainStep` (processor 1) | dominio doc parcial |
| `prompt-dominio-check.md` | `DomainValidator` | `DomainStep` (processor 2) | dominio doc validado |
| `prompt-ruta.md` | `RouteGenerator` | `RouteStep` | nodos en MongoDB (`ruta.json`) |
| `prompt-section.md` MODO BLOQUE | `BlockGenerator` | `BlocksStep` | nodo tipo=bloque completo |
| `prompt-section.md` MODO INTEGRADORA | `TaskGenerator` | `IntegrationsStep` | nodo tipo=integracion completo |
| `prompt-section-visual.md` | `DiagramEnricher` | `DiagramStep` | actualiza `contenido.concepto` |

---

## Spring Batch — estructura del Job

```java
@Bean
public Job learningPathJob() {
    return jobBuilder.get("learningPathJob")
        .start(metaStep())          // prompt-meta.md → meta document en MongoDB
        .next(domainStep())         // prompt-dominio.md + check → dominio docs en MongoDB
        .next(routeStep())          // prompt-ruta.md → nodos en MongoDB (ruta.json)
        .next(parallelFlow())
        .end()
        .build();
}

@Bean
public Flow parallelFlow() {
    return flowBuilder.get("parallelFlow")
        .split(taskExecutor())
        .add(blocksFlow(), integrationsFlow(), diagramFlow())
        .build();
}

// BlocksStep: prompt-section.md MODO BLOQUE
@Bean
public Step blocksStep() {
    return stepBuilder.get("blocksStep")
        .<BlockNode, Block>chunk(3)
        .reader(blockNodeReader())        // lee nodos tipo=bloque de MongoDB
        .processor(blockGenerator())      // llama al LLM → Block (objetivo+contenido+practica+qa)
        .writer(blockWriter())            // guarda en MongoDB + emite evento SSE
        .taskExecutor(taskExecutor())
        .throttleLimit(3)                 // máx 3 llamadas LLM simultáneas
        .build();
}

// IntegrationsStep: prompt-section.md MODO TAREA INTEGRADORA
@Bean
public Step integrationsStep() {
    return stepBuilder.get("integrationsStep")
        .<IntegrationNode, IntegrationTask>chunk(2)
        .reader(integrationNodeReader())  // lee nodos tipo=integracion de MongoDB
        .processor(taskGenerator())       // llama al LLM → enunciado+criterios+pista+solucion
        .writer(integrationWriter())
        .taskExecutor(taskExecutor())
        .throttleLimit(2)
        .build();
}

// DiagramStep: prompt-section-visual.md — enriquece contenido.concepto con Mermaid
@Bean
public Step diagramStep() {
    return stepBuilder.get("diagramStep")
        .<Block, Block>chunk(5)
        .reader(generatedBlockReader())   // lee bloques ya generados de MongoDB
        .processor(diagramEnricher())     // llama al LLM → actualiza contenido.concepto
        .writer(blockConceptWriter())     // sobreescribe solo el campo contenido.concepto
        .taskExecutor(taskExecutor())
        .throttleLimit(3)
        .build();
}

// DomainStep: prompt-dominio.md + prompt-dominio-check.md como processors encadenados
@Bean
public Step domainStep() {
    return stepBuilder.get("domainStep")
        .<ModuleSpec, DomainDoc>chunk(1)
        .reader(moduleReader())
        .processor(CompositeItemProcessor.<ModuleSpec, DomainDoc>builder()
            .delegates(domainAnalyzer(), domainValidator())  // analiza → valida/corrige
            .build())
        .writer(domainDocWriter())
        .build();
}
```

> `throttleLimit` controla el rate limit del proveedor LLM. Configurable por entorno.

---

## Spring AI — generación de bloques

Los prompts `.md` del pipeline se usan como system prompt templates con variables. `Structured Output` mapea la respuesta directamente al POJO sin parsing manual.

```java
@Component
public class BlockGenerator implements ItemProcessor<BlockNode, Block> {

    private final ChatClient chatClient;

    @Override
    public Block process(BlockNode node) {
        return chatClient.prompt()
            .system(promptLoader.load("prompt-section.md")   // carga el prompt del fichero
                .render(Map.of(
                    "dominio",      node.modulo(),
                    "perfil",       node.pathMeta().nivelActual(),
                    "competencia",  node.titulo()
                )))
            .user("MODO BLOQUE\nBLOQUE_ID = " + node.id() + "\nTÍTULO = " + node.titulo())
            .call()
            .entity(Block.class);    // Structured Output → POJO directo
    }
}

// POJOs — mapean exactamente la estructura de prompt-section.md
public record Block(
    String objetivo,
    String skipQuestion,
    Contenido contenido,
    Practica practica,
    List<QA> qa
) {}

public record Contenido(String concepto, Ejemplo ejemplo) {}
public record Ejemplo(String codigo, String explicacion) {}
public record Practica(String enunciado, List<String> criterios, String pista, String solucion) {}
public record QA(String pregunta, String respuesta) {}

// DiagramEnricher — actualiza solo contenido.concepto
@Component
public class DiagramEnricher implements ItemProcessor<Block, Block> {

    @Override
    public Block process(Block block) {
        String conceptoEnriquecido = chatClient.prompt()
            .system(promptLoader.load("prompt-section-visual.md"))
            .user("MODO BLOQUE\nBLOQUE_ID = " + block.id() + "\nCONTENIDO =\n" + block.contenido().concepto())
            .call()
            .content();
        return block.withConcepto(conceptoEnriquecido);
    }
}
```

---

## SSE — progreso en tiempo real

```java
@GetMapping(
    value    = "/paths/{id}/progress",
    produces = MediaType.TEXT_EVENT_STREAM_VALUE
)
public Flux<ServerSentEvent<ProgressEvent>> streamProgress(@PathVariable String id) {
    return Flux.interval(Duration.ofSeconds(2))
        .map(_ -> progressService.getProgress(id))
        .takeUntil(p -> p.status().equals("ready") || p.status().equals("error"))
        .map(p -> ServerSentEvent.builder(p).build());
}

// Evento recibido en el frontend:
// { phase: "blocks", done: 17, total: 42, percent: 40 }
```

---

## Frontend React — estructura

```
src/
├── pages/
│   ├── HomePage.tsx           ← formulario: tema + nivel + objetivo
│   ├── GeneratingPage.tsx     ← barra de progreso SSE en tiempo real
│   ├── PathOverviewPage.tsx   ← mapa visual de la ruta (react-flow)
│   ├── BlockPage.tsx          ← lector de bloque + ejercicio + check
│   ├── IntegrationPage.tsx    ← tarea integradora + entrega + solución
│   └── DashboardPage.tsx      ← progreso del usuario
│
├── components/
│   ├── PathGraph.tsx          ← grafo de prerequisitos con react-flow
│   ├── BlockReader.tsx        ← contenido + navegación anterior/siguiente
│   ├── ExercisePanel.tsx      ← enunciado + editor + criterios de éxito
│   ├── ProgressBar.tsx        ← progreso SSE durante generación
│   └── SkipGate.tsx           ← "¿ya sabes esto?" antes de cada bloque
│
└── hooks/
    ├── useGenerationProgress.ts   ← consume SSE
    ├── useLearningPath.ts         ← React Query para la ruta
    └── useProgress.ts             ← progreso del usuario
```

---

## Flujo del usuario aprendiendo

```
PathOverviewPage  →  clic en bloque desbloqueado
        ↓
    SkipGate       →  "¿ya sabes escribir un bucle for?"
        │                       │
      no sabe                 ya sabe
        ↓                       ↓
    BlockPage              saltado → siguiente bloque
        │
        ├── lee concepto (markdown renderizado)
        ├── ve ejemplo de código
        ├── hace ejercicio (editor de código o texto libre)
        ├── ve criterios de éxito
        ├── solicita pista  (opcional — oculta por defecto)
        ├── entrega intento
        └── marca completado → desbloquea siguientes nodos
                │
                └── si N bloques completados
                        ↓
                IntegrationPage  desbloqueada
                        │
                        ├── lee enunciado (sin pistas de herramientas)
                        ├── construye el mini programa
                        ├── verifica contra criterios
                        └── ve solución completa
```

---

## Unidades de aprendizaje

### Bloque (tipo: `bloque`)

| Campo | Descripción |
|---|---|
| `objetivo` | Acción concreta y verificable al terminar |
| `skip_question` | Pregunta de 2 min — si la responde, puede saltar |
| `contenido.concepto` | Explicación completa del concepto — sin otros temas |
| `contenido.ejemplo` | Código o caso mínimo que demuestra el objetivo |
| `practica.enunciado` | Ejercicio que el usuario hace, no lee |
| `practica.criterios` | Verificables: "con input X produce output Y" |
| `practica.pista` | Oculta por defecto — el usuario la solicita |
| `practica.solucion` | Oculta hasta que el usuario entrega su intento |
| `qa` | 3-5 preguntas de repaso con respuesta modelo |

### Tarea integradora (tipo: `integracion`)

| Campo | Descripción |
|---|---|
| `nivel_integracion` | `micro` (2-3 bloques) / `meso` (módulo) / `macro` (2-3 módulos) |
| `combina` | IDs de los bloques que el usuario necesita haber completado |
| `enunciado` | Qué construir — sin indicar qué herramientas usar |
| `criterios` | Verificables: "el programa compila", "con entrada X produce Y" |
| `pista` | Oculta — mínima, no la solución |
| `solucion` | Oculta hasta que el usuario entrega |

---

## Decisiones de diseño

| Decisión | Elección | Razón |
|---|---|---|
| MongoDB vs PostgreSQL | MongoDB | Contenido documental, schema flexible, sin joins complejos |
| Spring Batch vs @Async | Spring Batch | Restart/resume si falla a mitad de generación |
| SSE vs WebSocket | SSE | Unidireccional server→client, más simple para progreso |
| Structured Output vs parsing | Structured Output | Spring AI garantiza JSON válido mapeado a POJO |
| react-flow para el grafo | react-flow | Visualización de grafos con prerequisitos |
| `throttleLimit` configurable | Por entorno | Evita rate limiting del proveedor LLM |
| Sin campo `tiempo` | Eliminado | Asignarlo fuerza al LLM a recortar contenido — profundidad la determina la competencia |
| DAG vs índice lineal | ruta.json (DAG) | Permite navegación no lineal y prerequisitos reales; Spring Batch usa `orden_sugerido` como fallback |
| `domainValidator` como segundo processor | CompositeItemProcessor | prompt-dominio-check reutiliza el mismo Step sin añadir complejidad al Job |
| `DiagramEnricher` en Step separado | `DiagramStep` paralelo | Depende de que los bloques existan; corre en paralelo con integraciones para reducir tiempo total |

---

## Próximos pasos de implementación

1. Modelo de datos y repositorios MongoDB
2. Pipeline Spring Batch — fases secuenciales (Meta → Domain → Route)
3. Generadores Spring AI — BlockGenerator, TaskGenerator, DiagramEnricher
4. SSE endpoint de progreso
5. Frontend — HomePage + GeneratingPage + PathOverviewPage
6. Frontend — BlockPage + IntegrationPage + lógica de desbloqueo
