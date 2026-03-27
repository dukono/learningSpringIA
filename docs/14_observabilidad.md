<!-- navegación -->
> **[← Multimodalidad](13_multimodalidad.md)** | **[← Inicio](../README.md)** | **[Siguiente: Proyecto Completo →](15_proyecto_completo.md)**

---

# Capítulo 14 — Observabilidad y métricas

> Métricas, trazas distribuidas y logs de las operaciones de IA.
> Cómo usar Micrometer, Actuator, Zipkin y Prometheus/Grafana con Spring AI.

## 14. Observabilidad y métricas

Spring AI incluye soporte nativo para **Micrometer** (la misma librería que usa Spring Boot
para métricas y trazas). Tiene **tres artefactos de observabilidad separados**, uno por
tipo de modelo, porque cada uno tiene sus propias métricas y spans:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ARTEFACTOS DE OBSERVABILIDAD EN SPRING AI 1.1.0                            │
├────────────────────────────────────────────────────────┬─────────────────────┤
│  Artefacto                                             │  Qué observa        │
├────────────────────────────────────────────────────────┼─────────────────────┤
│  spring-ai-autoconfigure-model-chat-observation        │  Chat / LLM         │
│  spring-ai-autoconfigure-model-embedding-observation   │  Embeddings         │
│  spring-ai-autoconfigure-model-image-observation       │  Generación imagen  │
└────────────────────────────────────────────────────────┴─────────────────────┘

Todos se activan automáticamente con los starters correspondientes.
No necesitas dependencias adicionales — vienen incluidas.
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
  metrics:
    tags:
      application: mi-app-ia
```

### 14.1 Observabilidad de Chat (ChatModel)

```yaml
# Activar/desactivar observabilidad de chat
spring:
  ai:
    chat:
      observations:
        include-prompt: false        # false por defecto — el prompt puede tener datos sensibles
        include-completion: false    # false por defecto — la respuesta puede tener datos sensibles
        log-prompt: false            # loguear el prompt en DEBUG
        log-completion: false        # loguear la respuesta en DEBUG
```

**Métricas automáticas de Chat:**

```
gen_ai.client.token.usage{
  gen_ai.operation.name="chat",
  gen_ai.system="openai",
  gen_ai.request.model="gpt-4o-mini",
  gen_ai.token.type="input"/"output"
}
→ tokens consumidos por entrada y salida

gen_ai.client.operation.duration{
  gen_ai.operation.name="chat",
  gen_ai.system="openai",
  gen_ai.request.model="gpt-4o-mini"
}
→ tiempo total de cada llamada al modelo (ms)
```

### 14.2 Observabilidad de Embeddings (EmbeddingModel)

```yaml
spring:
  ai:
    embedding:
      observations:
        include-input: false   # false por defecto — los textos pueden ser sensibles
```

**Métricas automáticas de Embeddings:**

```
gen_ai.client.token.usage{
  gen_ai.operation.name="embeddings",
  gen_ai.system="openai",
  gen_ai.request.model="text-embedding-3-small"
}
→ tokens consumidos al generar embeddings

gen_ai.client.operation.duration{
  gen_ai.operation.name="embeddings"
}
→ tiempo de generación de embeddings
```

### 14.3 Observabilidad de Image Generation (ImageModel)

```yaml
spring:
  ai:
    image:
      observations:
        include-prompt: false   # false por defecto
```

**Métricas automáticas de Image Generation:**

```
gen_ai.client.operation.duration{
  gen_ai.operation.name="image_generation",
  gen_ai.system="openai",
  gen_ai.request.model="dall-e-3"
}
→ tiempo de generación de imágenes
```

### 14.4 Ver métricas en Actuator

```bash
# Tokens de chat usados
GET /actuator/metrics/gen_ai.client.token.usage

# Latencia de chat
GET /actuator/metrics/gen_ai.client.operation.duration

# Ejemplo de respuesta con tags:
{
  "name": "gen_ai.client.token.usage",
  "measurements": [{"statistic": "COUNT", "value": 47}, {"statistic": "TOTAL", "value": 12453}],
  "availableTags": [
    {"tag": "gen_ai.system",         "values": ["openai"]},
    {"tag": "gen_ai.request.model",  "values": ["gpt-4o-mini"]},
    {"tag": "gen_ai.token.type",     "values": ["input", "output"]}
  ]
}
```

### 14.5 Trazas distribuidas con Zipkin (incluyendo Tools)

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # 100% de las trazas
spring:
  zipkin:
    base-url: http://localhost:9411
```

Cada llamada a la IA se convierte en un span rastreable. Si usas Tools, **cada llamada
a un tool aparece como un span hijo**, permitiendo ver exactamente cuánto tardó cada parte:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  TRAZA: POST /api/chat  (tiempo total: 3240ms)                               │
│  └─ ChatClient.call()                         [3200ms]                       │
│      ├─ OpenAI API request (round 1)          [980ms]   ← primera llamada    │
│      │   (modelo decide: necesito tools)                                     │
│      ├─ Tool: getWeather("Madrid")            [45ms]    ← tu función Java    │
│      ├─ Tool: getProductPrice(42)             [12ms]    ← consulta a BD      │
│      └─ OpenAI API request (round 2)          [2160ms]  ← respuesta final    │
│          (con resultados de los tools)                                       │
└──────────────────────────────────────────────────────────────────────────────┘
```

Esto es muy útil para diagnosticar rendimiento: puedes ver si el tiempo lo consume
la API de IA o tus propios tools.

### 14.6 Prometheus + Grafana

Si usas Prometheus y Grafana, las métricas de Spring AI están disponibles en
`/actuator/prometheus`:

```
# HELP gen_ai_client_token_usage_tokens_total
# TYPE gen_ai_client_token_usage_tokens_total counter
gen_ai_client_token_usage_tokens_total{
  application="mi-app-ia",
  gen_ai_operation_name="chat",
  gen_ai_request_model="gpt-4o-mini",
  gen_ai_system="openai",
  gen_ai_token_type="input"
} 8723.0

gen_ai_client_token_usage_tokens_total{...gen_ai_token_type="output"} 3241.0

# HELP gen_ai_client_operation_duration_seconds
# TYPE gen_ai_client_operation_duration_seconds histogram
gen_ai_client_operation_duration_seconds_bucket{...,le="1.0"}  12.0
gen_ai_client_operation_duration_seconds_bucket{...,le="5.0"}  47.0
gen_ai_client_operation_duration_seconds_bucket{...,le="+Inf"} 52.0
```

**Ejemplos de queries PromQL para dashboard:**

```promql
# Tokens por minuto (input + output)
rate(gen_ai_client_token_usage_tokens_total[1m])

# Coste estimado por hora (GPT-4o-mini: $0.15/1M input, $0.60/1M output)
(
  rate(gen_ai_client_token_usage_tokens_total{gen_ai_token_type="input"}[1h])  * 0.00000015 +
  rate(gen_ai_client_token_usage_tokens_total{gen_ai_token_type="output"}[1h]) * 0.00000060
) * 3600

# Latencia P95 de las llamadas al modelo
histogram_quantile(0.95, rate(gen_ai_client_operation_duration_seconds_bucket[5m]))

# Número de llamadas al modelo por minuto
rate(gen_ai_client_operation_duration_seconds_count[1m])
```

## 14.7 Errores comunes

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES EN OBSERVABILIDAD                                        │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  /actuator/metrics no muestra  │  El endpoint no está expuesto. Por defecto  │
│  métricas de gen_ai            │  Spring Boot solo expone "health".          │
│                                │  ✅ Añadir en application.yml:             │
│                                │  management.endpoints.web.exposure          │
│                                │    .include: health, metrics, prometheus    │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  include-prompt/completion=    │  Los prompts y respuestas pueden contener   │
│  true en producción: datos     │  datos personales o información sensible    │
│  sensibles en los logs y       │  de los usuarios. Activarlos en producción  │
│  trazas                        │  viola GDPR y otras normativas.             │
│                                │  ✅ Mantener include-prompt: false e        │
│                                │  include-completion: false en producción.   │
│                                │  Activar solo en dev para debugging.        │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Las trazas de Zipkin no       │  tracing.sampling.probability está a 0.1   │
│  muestran todas las llamadas   │  (10% por defecto). Con poco tráfico, la   │
│  al modelo                     │  mayoría de llamadas no aparecen.           │
│                                │  ✅ Subir a 1.0 en dev/staging para ver     │
│                                │  todas las trazas. En prod, 0.1 es          │
│                                │  razonable para no saturar Zipkin.          │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Los contadores de tokens      │  Algunos proveedores (Ollama, por ejemplo)  │
│  siempre están a 0             │  no incluyen el conteo de tokens en la      │
│                                │  respuesta. Las métricas de token.usage     │
│                                │  solo funcionan con proveedores que         │
│                                │  devuelven usage metadata (OpenAI, Anthropic│
│                                │  Gemini).                                   │
│                                │  ✅ Verificar que el proveedor soporta      │
│                                │  usage en la respuesta. Con Ollama en dev,  │
│                                │  usar solo latencia y conteo de llamadas.   │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Dashboard de Grafana no       │  Las métricas en /actuator/prometheus usan  │
│  encuentra las métricas        │  guiones bajos (gen_ai_client_token_usage)  │
│  gen_ai.*                      │  pero en Micrometer/Actuator pueden variar  │
│                                │  según la versión. Verificar el nombre      │
│                                │  exacto en /actuator/metrics primero.       │
│                                │  ✅ Consultar GET /actuator/metrics para    │
│                                │  ver los nombres disponibles antes de       │
│                                │  configurar el dashboard.                   │
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Multimodalidad](13_multimodalidad.md)** | **[← Inicio](../README.md)** | **[Siguiente: Proyecto Completo →](15_proyecto_completo.md)**
