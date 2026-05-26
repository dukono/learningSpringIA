# LLMs de referencia — Sistema de Aprendizaje Guiado

> Última revisión: mayo 2026  
> ⚠️ Los modelos del mercado asiático (DeepSeek V4, Qwen 3.6 Plus, MiniMax M2.7, Kimi K2.6) están listados pero sus capacidades específicas deben verificarse contra benchmarks actualizados.

---

## Capacidades que importan en este pipeline

| Capacidad | Prompts que la necesitan |
|---|---|
| **Seguimiento de instrucciones complejas** | Todos |
| **JSON estructurado fiel** | `prompt-dominio`, `prompt-ruta`, `prompt-section` |
| **Razonamiento explícito paso a paso** | `prompt-ruta` (DAG + prerequisitos), dominios matemáticos |
| **Generación de código ejecutable** | `prompt-section` — dominios tech |
| **Auditoría / detección de errores** | `prompt-dominio-check` |
| **Contexto largo (>200K tokens)** | `prompt-section` en rutas largas (>40 bloques) |
| **Diagramas Mermaid** | `prompt-section-visual` |

---

## Tabla comparativa

```
                  Claude    GPT-4.1  Gemini    o3        DeepSeek  Qwen3     Kimi      MiniMax
                  Sonnet4            2.5Pro              R1/V4     72B+      K2.x      M2.x
──────────────────────────────────────────────────────────────────────────────────────────────
Instrucciones     ★★★★★    ★★★★☆   ★★★★☆    ★★★★☆    ★★★★☆    ★★★★☆   ★★★☆☆   ★★★☆☆
JSON fiel         ★★★★★    ★★★★☆   ★★★★★    ★★★☆☆    ★★★★★    ★★★★★   ★★★★☆   ★★★★☆
Razonamiento      ★★★★☆    ★★★★☆   ★★★★☆    ★★★★★    ★★★★★    ★★★★☆   ★★★☆☆   ★★★☆☆
Código            ★★★★★    ★★★★★   ★★★★☆    ★★★★☆    ★★★★★    ★★★★★   ★★★☆☆   ★★★☆☆
Mermaid           ★★★★☆    ★★★☆☆   ★★★★★    ★★★☆☆    ★★★★☆    ★★★★☆   ★★★☆☆   ★★★☆☆
No-complaciente   ★★★★★    ★★★☆☆   ★★★☆☆    ★★★★★    ★★★★☆    ★★★★☆   ★★★☆☆   ★★★☆☆
Contexto          200K     1M       1M        200K      128K+     128K+    1M+      1M+
Coste API         Medio    Medio    Medio     Alto      Muy bajo  Muy bajo  Bajo    Bajo
Local (Ollama)    ❌        ❌        ❌        ❌        ✅         ✅        ❌        ❌
```

---

## Recomendación por paso del pipeline

### Por tipo de dominio

#### 🖥️ Dominio tecnológico (Spring AI, Kubernetes, React, Docker...)

| Paso del pipeline | Modelo recomendado | Alternativa | Motivo |
|---|---|---|---|
| `prompt-meta` | **Claude Sonnet 4** | DeepSeek V4 | Clasificación de dominio precisa, no mezcla tipos |
| `prompt-dominio` | **DeepSeek V4** | Qwen 3.6 Plus | Calidad comparable a Claude, coste ~80% menor |
| `prompt-dominio-check` | **Claude Sonnet 4** | — | Auditoría rigurosa, no complaciente, no pasa errores |
| `prompt-ruta` | **DeepSeek R1** | o3 | Razonamiento explícito para DAG + ciclos + prerequisitos |
| `prompt-section` | **DeepSeek V4** / **Qwen 3.6 Plus** | Claude Sonnet 4 | Código completo y ejecutable, JSON fiel, barato |
| `prompt-section-visual` | **Gemini 2.5 Pro** | Claude Sonnet 4 | Mejor selección de tipo de diagrama y sintaxis Mermaid |

#### 📐 Dominio formal (Álgebra Lineal, Estadística, Algoritmia...)

| Paso del pipeline | Modelo recomendado | Alternativa | Motivo |
|---|---|---|---|
| `prompt-meta` | **o3** | Claude Opus 4 | Taxonomía de competencias matemáticas exacta |
| `prompt-dominio` | **o3** | DeepSeek R1 | Prerequisitos matemáticos sutiles — un error rompe la cadena |
| `prompt-dominio-check` | **Claude Sonnet 4** | o3 | Auditoría formal rigurosa |
| `prompt-ruta` | **DeepSeek R1** | o3 | Razonamiento explícito para el grafo |
| `prompt-section` | **o3** | DeepSeek R1 | Demostraciones correctas, contraejemplos válidos, condiciones de contorno exactas |
| `prompt-section-visual` | **Gemini 2.5 Pro** | — | Diagramas de grafos, matrices, visualizaciones formales |

#### 📚 Dominio humanístico (Historia, Filosofía, Literatura, Economía...)

| Paso del pipeline | Modelo recomendado | Alternativa | Motivo |
|---|---|---|---|
| `prompt-meta` | **Claude Opus 4** | Claude Sonnet 4 | Preserva tensión entre interpretaciones, no la resuelve |
| `prompt-dominio` | **Claude Opus 4** | Claude Sonnet 4 | Equilibra fuentes primarias vs secundarias |
| `prompt-dominio-check` | **Claude Sonnet 4** | — | Detecta historiografía inventada o anacronismos |
| `prompt-ruta` | **Claude Sonnet 4** | DeepSeek R1 | Los prerequisitos humanísticos son cronológicos y conceptuales |
| `prompt-section` | **Claude Opus 4** | Claude Sonnet 4 | Manejo de ambigüedad interpretativa — GPT tiende a resolverla, Claude la preserva |
| `prompt-section-visual` | **Claude Sonnet 4** | Gemini 2.5 Pro | Timelines y mindmaps históricos más coherentes |

---

## Configuración recomendada en Spring AI

### Dev local — sin coste, sin internet

```yaml
# application-dev.yml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        model: deepseek-r1:32b     # razonamiento (prompt-ruta)
      embedding:
        model: nomic-embed-text
```

```bash
# Modelos a descargar para dev
ollama pull deepseek-r1:32b        # razonamiento — prompt-ruta
ollama pull qwen3:32b              # código y JSON — prompt-section tech
ollama pull llama3.3:70b           # fallback general — prompt-dominio
ollama pull nomic-embed-text       # embeddings
```

### Producción — dominio tech (balance calidad/coste)

```yaml
# application-prod.yml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        model: claude-sonnet-4-20250514
    openai:
      api-key: ${OPENAI_API_KEY}
```

```java
// Selección de modelo por tipo de paso en Spring Batch
@Component
public class ModelSelector {

    // prompt-dominio-check y prompt-meta → Claude (no complaciente)
    private final ChatClient claudeClient;

    // prompt-section tech → DeepSeek V4 vía OpenAI-compatible API
    private final ChatClient deepseekClient;

    // prompt-ruta → DeepSeek R1 (razonamiento explícito)
    private final ChatClient deepseekR1Client;

    // prompt-section-visual → Gemini
    private final ChatClient geminiClient;
}
```

---

## Costes estimados por ruta completa (~50 bloques)

| Configuración | Coste estimado | Calidad |
|---|---|---|
| Todo Claude Sonnet 4 | ~$4–8 | ★★★★★ |
| Mix óptimo (tabla anterior) | ~$0.80–1.50 | ★★★★★ |
| Todo DeepSeek V4 + R1 | ~$0.20–0.50 | ★★★★☆ |
| Todo local (Ollama 32B) | $0 | ★★★☆☆ |
| Todo local (Ollama 70B) | $0 | ★★★★☆ |

> 💡 El coste de `prompt-dominio-check` con Claude Sonnet 4 es el que **no debe recortarse**: un error que pasa el check y llega a `prompt-ruta` cuesta más en regeneración que usar el modelo más caro.

---

## Notas sobre modelos del mercado asiático

### DeepSeek V4
- Heredero de V3 — el mejor de la familia en código y JSON estructurado
- Código abierto: disponible en Ollama
- API pública con precios menores que OpenAI/Anthropic
- ⚠️ Verificar: contexto máximo, mejoras en seguimiento de instrucciones largas vs V3

### Qwen 3.6 Plus (Alibaba)
- La familia Qwen3 fue un salto significativo en código y multilingual
- Disponible en Ollama (`qwen3:72b`)
- ⚠️ Verificar: qué mejora "Plus" sobre 72B base, si el contexto supera 128K

### MiniMax M2.7
- La familia MiniMax se ha especializado en contextos muy largos (1M+ tokens)
- Relevante para `prompt-section` en rutas largas (>40 bloques) donde el contexto acumulado supera 200K tokens
- ⚠️ Verificar: capacidades de JSON estructurado y seguimiento de instrucciones vs Gemini 2.5 Pro

### Kimi K2.6 (Moonshot AI)
- Heredero de k2 — especialista en 1M de tokens de contexto
- Ventaja principal: cuando el contexto de bloques previos crece mucho
- ⚠️ Verificar: mejoras en calidad de contenido pedagógico vs k2 base

---

## Criterio de selección rápida

```
¿El paso requiere auditoría o detección de errores?
  → Claude Sonnet 4 (no complaciente)

¿El paso construye un grafo, valida prerequisitos o razona paso a paso?
  → DeepSeek R1 (razonamiento explícito)

¿El paso genera código ejecutable o JSON complejo?
  → DeepSeek V4 / Qwen 3.6 Plus (calidad + coste bajo)

¿El paso añade diagramas Mermaid?
  → Gemini 2.5 Pro

¿El dominio es humanístico con tensión interpretativa?
  → Claude Opus 4 (preserva ambigüedad, no la resuelve)

¿El contexto acumulado supera 300K tokens?
  → Gemini 2.5 Pro / Kimi K2.6 / MiniMax M2.7 (1M context)

¿Es dev local sin coste?
  → DeepSeek R1:32b o Qwen3:32b vía Ollama
```

