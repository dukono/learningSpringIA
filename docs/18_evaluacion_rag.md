<!-- navegación -->
> **[← Testing](17_testing.md)** | **[← Inicio](../README.md)** | **[Siguiente: Proyecto completo →](15_proyecto_completo.md)**

---

# Capítulo 18 — Evaluación de respuestas RAG

> Cómo medir la calidad de un sistema RAG: relevancia de los documentos
> recuperados y fidelidad de la respuesta generada. Framework de evaluación
> de Spring AI con `RelevancyEvaluator` y `FaithfulnessEvaluator`.

## 18.1 El problema — ¿cómo sabes si tu RAG funciona bien?

Construir un RAG es relativamente sencillo. **Saber si responde bien** es el problema real.
Sin un framework de evaluación, solo puedes hacer pruebas manuales, que no escalan y
no detectan regresiones cuando cambias el modelo, el tamaño del chunk o el threshold.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  DOS TIPOS DE FALLOS EN UN SISTEMA RAG                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  FALLO 1 — RETRIEVAL INCORRECTO (problema de búsqueda)                       │
│  El sistema recupera documentos que NO son relevantes para la pregunta.      │
│  La respuesta es mala porque el modelo no tiene los datos correctos.         │
│                                                                              │
│  Ejemplo:                                                                    │
│  Pregunta: "¿Cuáles son los días de vacaciones?"                             │
│  Chunks recuperados: (sobre política de gastos — irrelevantes)               │
│  Respuesta: inventa o dice "no tengo esa información"                        │
│  → Evalúa con: RelevancyEvaluator                                            │
│                                                                              │
│  FALLO 2 — RESPUESTA INFIEL (problema de generación)                         │
│  El sistema recupera los documentos CORRECTOS, pero el modelo genera         │
│  una respuesta que no está basada en ellos (alucina usando los chunks        │
│  como excusa pero añadiendo información que no está en ellos).               │
│                                                                              │
│  Ejemplo:                                                                    │
│  Pregunta: "¿Cuáles son los días de vacaciones?"                             │
│  Chunks recuperados: "Los empleados tienen 22 días de vacaciones al año."    │
│  Respuesta del modelo: "Tienes 22 días más 5 de asuntos propios y 3 de      │
│  libre disposición..." (el modelo añadió info que no estaba en el chunk)     │
│  → Evalúa con: FaithfulnessEvaluator                                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 18.2 El framework de evaluación de Spring AI

Spring AI 1.1.0 incluye dos evaluadores basados en LLM que analizan pares
pregunta/respuesta y devuelven una puntuación de calidad:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CÓMO FUNCIONAN LOS EVALUADORES                                              │
│                                                                              │
│  1. Tu RAG genera una respuesta para una pregunta de prueba                  │
│                                                                              │
│  2. Spring AI envía al LLM un meta-prompt de evaluación:                     │
│     "Dada esta pregunta, estos documentos recuperados y esta respuesta,      │
│      ¿es la respuesta relevante/fiel? Responde solo con true o false."       │
│                                                                              │
│  3. El LLM actúa como juez: devuelve true (pasa) o false (falla)             │
│                                                                              │
│  Ventaja: no requiere ground truth manual para cada pregunta                 │
│  Coste:   hace llamadas al LLM por cada evaluación — usar en tests, no prod  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 18.3 RelevancyEvaluator — ¿los documentos recuperados son relevantes?

`RelevancyEvaluator` evalúa si los **documentos que el RAG recuperó del vector store**
son relevantes para la pregunta formulada.

```java
@SpringBootTest
class RelevancyTest {

    @Autowired ChatClient chatClient;
    @Autowired VectorStore vectorStore;
    @Autowired ChatModel  chatModel;    // necesario para el evaluador

    @Test
    void documentosRecuperadosSonRelevantes() {

        String pregunta = "¿Cuáles son los días de vacaciones para empleados fijos?";

        // 1. Obtener la respuesta completa (con los documentos usados)
        ChatResponse respuesta = chatClient.prompt()
            .user(pregunta)
            .advisors(new QuestionAnswerAdvisor(vectorStore,
                SearchRequest.builder().topK(5).build()))
            .call()
            .chatResponse();

        // 2. Construir la request de evaluación
        //    EvaluationRequest encapsula: pregunta + ChatResponse completa
        EvaluationRequest evalRequest = new EvaluationRequest(pregunta, respuesta);

        // 3. Evaluar relevancia — usa el LLM para juzgar
        RelevancyEvaluator evaluador = new RelevancyEvaluator(
            ChatClient.builder(chatModel).build()  // cliente dedicado para evaluación
        );

        EvaluationResponse resultado = evaluador.evaluate(evalRequest);

        // 4. Verificar en el test
        assertThat(resultado.isPass())
            .as("Los documentos recuperados deben ser relevantes para la pregunta")
            .isTrue();
    }
}
```

**¿Qué evalúa internamente?** Comprueba si los chunks recuperados del vector store
contienen información que responde a la pregunta. Si los chunks son sobre un tema
distinto, devuelve `false`.

---

## 18.4 FaithfulnessEvaluator — ¿la respuesta está basada en los documentos?

`FaithfulnessEvaluator` evalúa si la **respuesta generada por el LLM** se deriva
únicamente de los documentos recuperados, sin añadir información inventada.

```java
@SpringBootTest
class FaithfulnessTest {

    @Autowired ChatClient chatClient;
    @Autowired VectorStore vectorStore;
    @Autowired ChatModel  chatModel;

    @Test
    void respuestaEsFielALosDocumentos() {

        String pregunta = "¿Cuántos días de vacaciones tiene un empleado de contrato fijo?";

        ChatResponse respuesta = chatClient.prompt()
            .user(pregunta)
            .advisors(new QuestionAnswerAdvisor(vectorStore,
                SearchRequest.builder().topK(5).build()))
            .call()
            .chatResponse();

        EvaluationRequest evalRequest = new EvaluationRequest(pregunta, respuesta);

        FaithfulnessEvaluator evaluador = new FaithfulnessEvaluator(
            ChatClient.builder(chatModel).build()
        );

        EvaluationResponse resultado = evaluador.evaluate(evalRequest);

        assertThat(resultado.isPass())
            .as("La respuesta no debe contener información que no esté en los documentos")
            .isTrue();
    }
}
```

**¿Qué evalúa internamente?** Comprueba que cada afirmación en la respuesta generada
pueda rastrearse hasta al menos uno de los chunks recuperados. Si el modelo añade
información propia que no está en los documentos, devuelve `false`.

---

## 18.5 EvaluationRequest y EvaluationResponse — el contrato

```java
// EvaluationRequest — lo que le das al evaluador
EvaluationRequest request = new EvaluationRequest(
    pregunta,      // String — la pregunta del usuario
    chatResponse   // ChatResponse — respuesta completa (incluye chunks usados)
);

// EvaluationResponse — lo que devuelve el evaluador
EvaluationResponse resultado = evaluador.evaluate(request);

boolean pasa     = resultado.isPass();      // true = calidad correcta
float   score    = resultado.getScore();    // 0.0 a 1.0 (si el evaluador lo soporta)
String  feedback = resultado.getFeedback(); // explicación del evaluador (opcional)
```

---

## 18.6 Suite de evaluación completa

En la práctica, lo más útil es un test que evalúe un conjunto de pares
pregunta/respuesta esperada sobre tu documentación real:

```java
@SpringBootTest
@Import(TestAiConfig.class)
class RagQualityTest {

    @Autowired ChatClient chatClient;
    @Autowired VectorStore vectorStore;
    @Autowired ChatModel   chatModel;

    // Pares de evaluación: preguntas que debería responder bien el sistema
    private static final List<String> PREGUNTAS_EVALUACION = List.of(
        "¿Cuáles son los días de vacaciones para empleados fijos?",
        "¿Cuál es el proceso para solicitar una baja médica?",
        "¿Qué documentación necesito para incorporarme al equipo?"
    );

    @Test
    void sistemRagSuperaEvaluacionDeRelevancia() {

        RelevancyEvaluator   relevancyEval   = new RelevancyEvaluator(
            ChatClient.builder(chatModel).build());
        FaithfulnessEvaluator faithfulnessEval = new FaithfulnessEvaluator(
            ChatClient.builder(chatModel).build());

        int totalPreguntas = PREGUNTAS_EVALUACION.size();
        int pasanRelevancia    = 0;
        int pasanFidelidad     = 0;

        for (String pregunta : PREGUNTAS_EVALUACION) {

            ChatResponse resp = chatClient.prompt()
                .user(pregunta)
                .advisors(new QuestionAnswerAdvisor(vectorStore,
                    SearchRequest.builder().topK(5).build()))
                .call()
                .chatResponse();

            EvaluationRequest er = new EvaluationRequest(pregunta, resp);

            if (relevancyEval.evaluate(er).isPass())    pasanRelevancia++;
            if (faithfulnessEval.evaluate(er).isPass()) pasanFidelidad++;
        }

        double ratioRelevancia = (double) pasanRelevancia / totalPreguntas;
        double ratioFidelidad  = (double) pasanFidelidad  / totalPreguntas;

        System.out.printf("Relevancia:  %.0f%% (%d/%d)%n",
            ratioRelevancia * 100, pasanRelevancia, totalPreguntas);
        System.out.printf("Fidelidad:   %.0f%% (%d/%d)%n",
            ratioFidelidad  * 100, pasanFidelidad,  totalPreguntas);

        // Umbral mínimo de calidad: 80%
        assertThat(ratioRelevancia).isGreaterThanOrEqualTo(0.8);
        assertThat(ratioFidelidad).isGreaterThanOrEqualTo(0.8);
    }
}
```

---

## 18.7 Dependencias necesarias

```xml
<!-- Los evaluadores forman parte del módulo de testing de Spring AI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- El BOM gestiona la versión — no añadir versión explícita -->
```

> 💡 `RelevancyEvaluator` y `FaithfulnessEvaluator` hacen llamadas reales al LLM.
> Para usarlos en CI sin coste en cada PR, considera ejecutarlos solo en un
> pipeline de evaluación semanal o en ramas de release.

---

## 18.8 Cuándo usar evaluación automática

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CUÁNDO EVALUAR AUTOMÁTICAMENTE                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│  ✅ Al cambiar el modelo de embeddings (EmbeddingModel)                       │
│     → riesgo: los vectores son incomparables si cambia la dimensión          │
│                                                                              │
│  ✅ Al cambiar el tamaño de chunk o el overlap en TokenTextSplitter           │
│     → riesgo: chunks más grandes o pequeños pueden empeorar la recuperación  │
│                                                                              │
│  ✅ Al cambiar el threshold de similitud en SearchRequest                     │
│     → riesgo: demasiado alto = no encuentra nada; demasiado bajo = ruido     │
│                                                                              │
│  ✅ Al cambiar el modelo de chat (LLM)                                        │
│     → riesgo: un modelo más pequeño puede alucinar más con los mismos chunks │
│                                                                              │
│  ✅ Al añadir nuevos documentos o reindexar                                   │
│     → verificar que no se rompió la calidad del RAG existente                │
│                                                                              │
│  ❌ En cada PR de código que no toca la configuración de IA                   │
│     → innecesario y costoso (hace llamadas al LLM)                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 18.9 Errores comunes

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES EN EVALUACIÓN RAG                                        │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El evaluador siempre devuelve │  Se está usando MockChatModel en los tests. │
│  true aunque la respuesta es   │  Los evaluadores necesitan un ChatModel     │
│  mala                          │  real (OpenAI, Anthropic, Ollama) para      │
│                                │  analizar la respuesta con un LLM de verdad.│
│                                │  ✅ En la suite de evaluación, inyectar     │
│                                │  un ChatModel real, no el mock.             │
│                                │  Separar tests unitarios (mock) de tests    │
│                                │  de evaluación (modelo real).               │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  EvaluationRequest lanza       │  ChatResponse no contiene los documentos    │
│  NullPointerException          │  de contexto. Esto ocurre si el advisor     │
│                                │  QuestionAnswerAdvisor no está registrado,  │
│                                │  o si se usa .content() en lugar de         │
│                                │  .chatResponse() — el primero pierde los    │
│                                │  metadatos de la respuesta.                 │
│                                │  ✅ Usar siempre .call().chatResponse()     │
│                                │  para pasar la ChatResponse completa al     │
│                                │  EvaluationRequest.                         │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El test de evaluación pasa    │  El VectorStore está vacío — no hay         │
│  aunque no hay documentos      │  documentos indexados. Sin documentos,      │
│  indexados                     │  QuestionAnswerAdvisor devuelve contexto    │
│                                │  vacío, el modelo responde con su           │
│                                │  conocimiento general y el evaluador        │
│                                │  puede aceptarlo como "fiel" (no hay        │
│                                │  chunks que contradigan).                   │
│                                │  ✅ Verificar antes del test que el         │
│                                │  VectorStore tiene documentos con           │
│                                │  vectorStore.similaritySearch(...)          │
│                                │  .size() > 0.                               │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Los resultados de evaluación  │  Los LLMs son no deterministas — incluso    │
│  varían entre ejecuciones      │  el evaluador puede dar resultados          │
│                                │  ligeramente distintos para la misma        │
│                                │  pregunta. Un único pass/fail no es         │
│                                │  representativo.                            │
│                                │  ✅ Evaluar con al menos 10-20 preguntas    │
│                                │  y usar un umbral de porcentaje (≥80%)     │
│                                │  en lugar de exigir 100% de pass.           │
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Testing](17_testing.md)** | **[← Inicio](../README.md)** | **[Siguiente: Proyecto completo →](15_proyecto_completo.md)**

