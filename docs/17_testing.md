<!-- navegación -->
> **[← Proyecto completo](15_proyecto_completo.md)** | **[← Inicio](../README.md)** | **[Siguiente: Comparativa →](16_comparativa_y_cheatsheet.md)**

---

# Capítulo 19 — Testing con Spring AI

> Cómo escribir tests unitarios e integración para servicios que usan ChatClient,
> VectorStore, Advisors y Tools, sin llamar a APIs reales en los tests.

## 19.1 El problema — ¿por qué es diferente testear con IA?

Testear servicios que usan IA tiene retos que no existen con una base de datos o un
REST client convencional:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  RETOS ESPECÍFICOS DEL TESTING CON IA                                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. NO DETERMINISMO                                                          │
│     El mismo prompt puede dar respuestas distintas en cada llamada.         │
│     Un test que verifica "respuesta exacta" fallará aleatoriamente.         │
│                                                                              │
│  2. COSTE                                                                    │
│     Cada llamada a OpenAI/Anthropic cuesta dinero.                          │
│     Un suite de 200 tests llamando a la API real = facturas inesperadas.    │
│                                                                              │
│  3. VELOCIDAD                                                                │
│     Una llamada a GPT-4o tarda 2-15 segundos.                               │
│     200 tests × 5s = 16 minutos de CI. Inviable.                            │
│                                                                              │
│  4. DISPONIBILIDAD                                                           │
│     Los tests de CI deben funcionar sin internet ni API keys.               │
│     En pipelines de CI las API keys a veces no están disponibles.           │
│                                                                              │
│  SOLUCIÓN: mockear ChatModel (no ChatClient) y usar TestContainers          │
│  para los componentes que sí necesitan infraestructura real (pgvector).     │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 19.2 Cómo funciona internamente — qué mockear y por qué

La clave es entender la jerarquía de Spring AI para saber **en qué capa mockear**:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  JERARQUÍA DE CAPAS EN SPRING AI                                             │
│                                                                              │
│  Tu Servicio                                                                 │
│       │  usa                                                                 │
│       ▼                                                                      │
│  ChatClient   ← NO mockear esto — es el objeto que quieres testear          │
│       │  delega en                                                           │
│       ▼                                                                      │
│  Advisors (Memory, RAG, Logging...)  ← pueden testearse unitariamente       │
│       │  delegan en                                                          │
│       ▼                                                                      │
│  ChatModel   ← ✅ AQUÍ SE MOCKEA — es la interfaz del proveedor             │
│       │  llama a                                                             │
│       ▼                                                                      │
│  API externa (OpenAI, Ollama...)  ← NUNCA en tests                          │
└──────────────────────────────────────────────────────────────────────────────┘

Regla: mockear ChatModel, no ChatClient.
ChatClient se construye desde el mock de ChatModel → comportamiento real del cliente.
```

### Dependencias de test

```xml
<dependencies>
    <!-- Spring AI test utilities — MockChatModel, MockEmbeddingModel -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers — para tests de integración con pgvector -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 19.3 Ejemplo mínimo — test unitario con MockChatModel

`MockChatModel` del artefacto `spring-ai-test` devuelve respuestas predefinidas
sin llamar a ninguna API.

```java
// Servicio a testear
@Service
public class ResumenService {

    private final ChatClient chatClient;

    public ResumenService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String resumir(String texto) {
        return chatClient.prompt()
            .user("Resume en una frase: " + texto)
            .call()
            .content();
    }
}
```

```java
// Test unitario — sin Spring context, sin API, en milisegundos
class ResumenServiceTest {

    @Test
    void resumir_devuelve_respuesta_del_modelo() {
        // 1. Crear un ChatModel que devuelve siempre la misma respuesta
        ChatModel mockModel = new MockChatModel("Resumen generado por mock.");

        // 2. Construir ChatClient desde el mock (igual que en producción)
        ChatClient.Builder builder = ChatClient.builder(mockModel);
        ResumenService service = new ResumenService(builder);

        // 3. Ejecutar y verificar
        String resultado = service.resumir("Texto largo de prueba...");

        assertThat(resultado).isEqualTo("Resumen generado por mock.");
    }

    @Test
    void resumir_propaga_excepcion_del_modelo() {
        // MockChatModel que lanza excepción (simula error 429 / timeout)
        ChatModel modelConError = mock(ChatModel.class);
        when(modelConError.call(any(Prompt.class)))
            .thenThrow(new TransientAiException("Rate limit exceeded"));

        ChatClient.Builder builder = ChatClient.builder(modelConError);
        ResumenService service = new ResumenService(builder);

        assertThatThrownBy(() -> service.resumir("texto"))
            .isInstanceOf(TransientAiException.class);
    }
}
```

> 💡 `MockChatModel` acepta también una lista de respuestas para que devuelva
> respuestas distintas en llamadas sucesivas:
> `new MockChatModel(List.of("primera", "segunda", "tercera"))`

## 19.4 @TestConfiguration — inyectar ChatClient de test en el contexto de Spring

Para tests con `@SpringBootTest` que cargan el contexto completo, usa
`@TestConfiguration` para sustituir el `ChatModel` real por uno mock:

```java
// src/test/java/com/empresa/config/TestAiConfig.java
@TestConfiguration
public class TestAiConfig {

    // Reemplaza el ChatModel real (OpenAI/Ollama) con un mock
    @Bean
    @Primary  // tiene prioridad sobre el bean de producción
    public ChatModel mockChatModel() {
        return new MockChatModel("Respuesta de test predefinida.");
    }

    @Bean
    @Primary
    public EmbeddingModel mockEmbeddingModel() {
        // MockEmbeddingModel genera vectores aleatorios de la dimensión indicada
        return new MockEmbeddingModel(1536);
    }
}
```

```java
// Test de integración con contexto de Spring completo
@SpringBootTest
@Import(TestAiConfig.class)   // ← carga la config de test
class ChatServiceIntegrationTest {

    @Autowired ChatService chatService;  // el servicio real, con el mock inyectado

    @Test
    void chat_devuelve_respuesta_del_mock() {
        String respuesta = chatService.chat("user-1", "Hola");
        assertThat(respuesta).isEqualTo("Respuesta de test predefinida.");
    }
}
```

### Testear Structured Output (`.entity()`)

El test verifica que el servicio parsea correctamente la respuesta del modelo
a un objeto Java. El mock devuelve el JSON que el converter esperaría:

```java
record ProductoInfo(String nombre, double precio, int stock) {}

@Service
public class CatalogoService {
    private final ChatClient chatClient;

    public CatalogoService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public ProductoInfo extraerProducto(String descripcion) {
        return chatClient.prompt()
            .user("Extrae: " + descripcion)
            .call()
            .entity(ProductoInfo.class);
    }
}
```

```java
class CatalogoServiceTest {

    @Test
    void extraerProducto_parsea_json_correctamente() {
        // El mock devuelve el JSON que el BeanOutputConverter espera
        String jsonRespuesta = """
            {"nombre": "Teclado mecánico", "precio": 89.99, "stock": 15}
            """;
        ChatModel mockModel = new MockChatModel(jsonRespuesta.trim());
        CatalogoService service = new CatalogoService(ChatClient.builder(mockModel));

        ProductoInfo resultado = service.extraerProducto("Teclado 89.99€ stock 15");

        assertThat(resultado.nombre()).isEqualTo("Teclado mecánico");
        assertThat(resultado.precio()).isEqualTo(89.99);
        assertThat(resultado.stock()).isEqualTo(15);
    }
}
```

### Testear Streaming

```java
@Service
public class StreamingChatService {
    private final ChatClient chatClient;

    public StreamingChatService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public Flux<String> chatStream(String mensaje) {
        return chatClient.prompt().user(mensaje).stream().content();
    }
}
```

```java
class StreamingChatServiceTest {

    @Test
    void chatStream_emite_tokens_correctamente() {
        // MockChatModel emite la respuesta como un stream de tokens
        ChatModel mockModel = new MockChatModel("Hola mundo desde streaming");
        StreamingChatService service =
            new StreamingChatService(ChatClient.builder(mockModel));

        // StepVerifier de Reactor para testear Flux
        StepVerifier.create(service.chatStream("saluda"))
            .expectNextMatches(token -> !token.isBlank())  // recibe al menos un token
            .thenConsumeWhile(token -> true)               // consume el resto
            .verifyComplete();
    }
}
```

## 19.5 Testear Advisors custom

Un advisor custom se puede testear directamente sin levantar el contexto de Spring,
instanciando la cadena manualmente:

```java
// Advisor a testear — loguea y audita cada llamada
@Component
public class AuditAdvisor implements CallAroundAdvisor {

    @Autowired AuditRepository auditRepo;

    @Override
    public AdvisedResponse aroundCall(AdvisedRequest req, CallAroundAdvisorChain chain) {
        AdvisedResponse response = chain.nextAroundCall(req);
        auditRepo.save(new AuditLog(req.userText(),
            response.response().getResult().getOutput().getContent()));
        return response;
    }

    @Override public String getName()  { return "AuditAdvisor"; }
    @Override public int    getOrder() { return 0; }
}
```

```java
class AuditAdvisorTest {

    @Test
    void aroundCall_guarda_auditoria() {
        // Mocks de dependencias
        AuditRepository mockRepo = mock(AuditRepository.class);
        AuditAdvisor advisor = new AuditAdvisor();
        ReflectionTestUtils.setField(advisor, "auditRepo", mockRepo);

        // Simular la cadena de advisors con un modelo mock
        ChatModel mockModel = new MockChatModel("respuesta de test");
        AdvisedRequest request = AdvisedRequest.builder()
            .chatClient(ChatClient.builder(mockModel).build())
            .userText("pregunta de test")
            .build();

        // Cadena real con el advisor bajo test
        DefaultCallAroundAdvisorChain chain =
            DefaultCallAroundAdvisorChain.builder(mockModel).build();

        advisor.aroundCall(request, chain);

        // Verificar que guardó la auditoría
        verify(mockRepo, times(1)).save(any(AuditLog.class));
    }
}
```

## 19.6 Testear Tools (@Tool)

Los métodos anotados con `@Tool` son métodos Java normales — se testean como
cualquier método de negocio, sin necesidad de Spring AI en el test:

```java
@Component
public class WeatherTools {

    @Autowired WeatherApiClient weatherApi;

    @Tool(description = "Obtiene la temperatura actual de una ciudad")
    public WeatherInfo getWeather(String ciudad) {
        return weatherApi.getCurrentWeather(ciudad);
    }
}
```

```java
class WeatherToolsTest {

    WeatherApiClient mockApi = mock(WeatherApiClient.class);
    WeatherTools tools;

    @BeforeEach
    void setUp() {
        tools = new WeatherTools();
        ReflectionTestUtils.setField(tools, "weatherApi", mockApi);
    }

    @Test
    void getWeather_retorna_info_de_la_api() {
        when(mockApi.getCurrentWeather("Madrid"))
            .thenReturn(new WeatherInfo(22.0, "soleado", "Madrid"));

        WeatherInfo resultado = tools.getWeather("Madrid");

        assertThat(resultado.temperatura()).isEqualTo(22.0);
        assertThat(resultado.condicion()).isEqualTo("soleado");
    }

    @Test
    void getWeather_propaga_excepcion_de_api() {
        when(mockApi.getCurrentWeather(anyString()))
            .thenThrow(new RuntimeException("API caída"));

        assertThatThrownBy(() -> tools.getWeather("Madrid"))
            .isInstanceOf(RuntimeException.class)
            .hasMessage("API caída");
    }
}
```

> ✅ **Buena práctica**: testear los tools de forma aislada garantiza que la lógica
> de negocio es correcta independientemente de cuándo el modelo decida llamarlos.

## 19.7 Tests de integración con TestContainers — VectorStore real

Para testear el `VectorStore` con pgvector necesitas una base de datos real.
TestContainers levanta un PostgreSQL+pgvector en Docker durante el test:

```java
@SpringBootTest
@Testcontainers
@Import(TestAiConfig.class)   // MockEmbeddingModel — no llamar a la API de embeddings
class VectorStoreIntegrationTest {

    // Levanta pgvector en Docker antes de los tests y lo destruye al terminar
    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("pgvector/pgvector:pg16")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    // Conecta Spring Boot a este contenedor
    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.ai.vectorstore.pgvector.initialize-schema", () -> "true");
    }

    @Autowired VectorStore vectorStore;

    @Test
    void add_y_buscar_devuelve_documento_correcto() {
        // Indexar
        Document doc = new Document("Spring AI es un framework para integrar IA en Java.");
        doc.getMetadata().put("categoria", "frameworks");
        vectorStore.add(List.of(doc));

        // Buscar
        List<Document> resultados = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query("framework Java inteligencia artificial")
                .topK(3)
                .build()
        );

        assertThat(resultados).isNotEmpty();
        assertThat(resultados.get(0).getContent()).contains("Spring AI");
    }

    @Test
    void busqueda_con_filtro_de_metadata() {
        vectorStore.add(List.of(
            documentoConMetadata("Spring Data", "spring"),
            documentoConMetadata("LangChain Python", "python")
        ));

        List<Document> resultados = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query("framework")
                .filterExpression("categoria == 'spring'")
                .topK(5)
                .build()
        );

        assertThat(resultados).allMatch(d ->
            "spring".equals(d.getMetadata().get("categoria")));
    }

    private Document documentoConMetadata(String texto, String categoria) {
        Document doc = new Document(texto);
        doc.getMetadata().put("categoria", categoria);
        return doc;
    }
}
```

```yaml
# src/test/resources/application-test.yml
spring:
  ai:
    vectorstore:
      pgvector:
        initialize-schema: true   # crea la tabla automáticamente en tests
```

---

## 19.8 Errores comunes en testing

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES EN TESTS DE SPRING AI                                    │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  "No bean of type ChatModel"   │  El contexto de test no tiene ningún        │
│  al levantar @SpringBootTest   │  ChatModel configurado y no hay API key.    │
│                                │  ✅ Añadir @Import(TestAiConfig.class) con  │
│                                │  un MockChatModel como @Bean @Primary.      │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Los tests pasan en local      │  En local hay Ollama corriendo; en CI no.   │
│  pero fallan en CI             │  El test llama a la API real sin darse      │
│                                │  cuenta porque no se mockeó ChatModel.      │
│                                │  ✅ Verificar que @Import(TestAiConfig)     │
│                                │  está en TODOS los tests de integración.    │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  StepVerifier.create().        │  El Flux del streaming no emite ningún      │
│  verifyComplete() cuelga       │  elemento o no completa. Suele ser porque   │
│  sin terminar                  │  el MockChatModel no soporta streaming y    │
│                                │  devuelve un Flux vacío o infinito.         │
│                                │  ✅ Usar .expectNextCount(1) + .thenCancel()│
│                                │  o poner un timeout: .expectComplete()      │
│                                │  .verify(Duration.ofSeconds(5))             │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  TestContainers falla con      │  Docker no está corriendo en la máquina     │
│  "Could not find a valid       │  de CI, o el usuario no tiene permisos      │
│  Docker environment"           │  para lanzar contenedores.                  │
│                                │  ✅ Separar tests con @Tag("integration")   │
│                                │  y excluirlos del perfil de CI sin Docker:  │
│                                │  mvn test -DexcludedGroups=integration      │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  El test de Structured Output  │  El BeanOutputConverter añade instrucciones │
│  falla con error de parseo     │  JSON al final del prompt. Si el mock       │
│  aunque el JSON parece válido  │  devuelve JSON puro pero el converter        │
│                                │  espera JSON dentro de un bloque ```json``` │
│                                │  el parseo falla.                           │
│                                │  ✅ El mock debe devolver exactamente el    │
│                                │  formato que devolvería el modelo real:     │
│                                │  JSON plano sin bloques de código.          │
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Proyecto completo](15_proyecto_completo.md)** | **[← Inicio](../README.md)** | **[Siguiente: Comparativa →](16_comparativa_y_cheatsheet.md)**




