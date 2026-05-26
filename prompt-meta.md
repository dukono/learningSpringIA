PROMPT 0 — GENERADOR DE META.MD

Actúas como arquitecto de aprendizaje. Tu única tarea es analizar el TEMA y el
perfil del aprendiz para generar el fichero meta.md completo, que sirve como
fuente de verdad para todo el pipeline de generación de la ruta de aprendizaje.

meta.md es leído por el PROMPT D (genera el análisis de dominio por módulo),
por el PROMPT 1 (genera la ruta con el grafo de prerequisitos) y por el PROMPT 2
(genera cada bloque). Debe contener toda la información que esos prompts necesitan
para trabajar sin tener que volver a razonar sobre el dominio ni sobre el perfil.

---

ENTRADA

Pega aquí el contenido de premeta.md generado por PROMPT 00.
Los campos TEMA, VERSION, NIVEL_ACTUAL, OBJETIVO y TIEMPO se toman de él.

<< PEGA AQUÍ premeta.md >>

---

FASE 1 — CLASIFICACIÓN DEL DOMINIO

Identifica a qué categoría pertenece el TEMA.

DOMINIOS TECNOLÓGICOS

  Tipo                         Ejemplos
  ───────────────────────────  ─────────────────────────────────────────────────────
  Lenguaje de programación     Python, Java, Go, Kotlin, Rust, C++
  Framework / Librería         Spring Boot, FastAPI, React, Quarkus, Spring AI
  Plataforma cloud             Azure, AWS, GCP, Oracle Cloud
  Certificación técnica        CKA, CKAD, AZ-900, AZ-104, AWS-SAA, RHCSA
  Herramienta / CLI            Docker, kubectl, Terraform, Ansible, Helm, Git
  Orquestador / Middleware     Kubernetes, Kafka, Elasticsearch, Redis, RabbitMQ

DOMINIOS ACADÉMICOS

  Tipo                         Ejemplos
  ───────────────────────────  ─────────────────────────────────────────────────────
  Ciencia formal               Matemáticas, Álgebra, Lógica, Estadística
  Ciencia experimental         Física, Química, Biología, Ciencias de la Tierra
  Humanidades                  Literatura, Filosofía, Historia, Lingüística
  Ciencias sociales            Economía, Psicología, Sociología, Derecho
  Artes                        Teoría musical, Artes visuales, Arquitectura

Si el TEMA es híbrido (ej: "Machine Learning con Python" = ciencia formal + lenguaje),
declara tipo primario y secundario. El tipo primario domina la estructura; el secundario
informa la profundidad y los ejemplos.

DETECCIÓN DEL CERT FLAG

Determina si el alcance está definido por un organismo externo oficial:

  ¿El scope de este proyecto está delimitado por una certificación privada,
  un título universitario regulado o un temario de oposición oficial?

  Cert flag activo si:
  - El TEMA incluye el nombre de un examen o certificación (AZ-104, OCP, CKA...)
  - El TEMA incluye expresiones como "según temario", "para oposición", "título oficial"
  - El OBJETIVO hace referencia a superar un examen de organismo certificador

  Si cert flag = activo: identifica el organismo y el nombre exacto del examen o título.
  Si cert flag = inactivo: el alcance lo define el OBJETIVO del aprendiz, no un organismo.

---

FASE 2 — INFERENCIA DE PARÁMETROS

Si VERSION no se proporcionó, infiere la versión más reciente y estable del TEMA
buscándola activamente. No uses conocimiento estático para datos que cambian.

Si el OBJETIVO no especifica el nivel de profundidad requerido, infiere el alcance
mínimo necesario para que el aprendiz pueda cumplirlo con el TIEMPO disponible.

Valores de referencia cuando no se especifican:

  Tema                    Versión por defecto         Alcance por defecto
  ──────────────────────  ──────────────────────────  ─────────────────────────────────────
  Java                    Java 21 (LTS)               lenguaje + Spring Boot básico
  Spring Boot             Spring Boot 3.x + Java 21   web + data + testing
  Spring AI               Spring AI 1.1.0             chat, RAG, tools, embeddings
  Python                  Python 3.12                 lenguaje generalista
  Kubernetes              1.30                        operador + desarrollador
  Azure                   actual                      AZ-104 Administrator
  AWS                     actual                      SAA-C03 Solutions Architect
  Álgebra lineal          n/a                         primer curso universitario
  Cálculo                 n/a                         diferencial e integral (una variable)
  Historia                n/a                         según período indicado en TEMA
  Física                  n/a                         mecánica clásica y electromagnetismo

---

FASE 3 — DERIVACIÓN DE PATRONES DE CONTENIDO

Los patrones de contenido describen qué significa un bloque de aprendizaje
completo en este dominio. Son usados por el PROMPT 2 para generar cada bloque.

La completitud de un bloque no se mide por número de secciones ni de líneas,
sino por si el aprendiz puede cumplir la competencia del bloque después de leerlo.

Deriva los patrones según el tipo de dominio:

LENGUAJE / FRAMEWORK / HERRAMIENTA / ORQUESTADOR / PLATAFORMA / CERTIFICACIÓN
  competencia_observable  → "puede [verbo activo] [objeto específico] en [versión/contexto]"
  concepto_completo       → modelo mental + mecanismo de funcionamiento interno +
                            comportamiento en tiempo de ejecución + casos en que falla +
                            errores frecuentes y por qué ocurren
  ejemplo                 → código completo y ejecutable en situación real (no de laboratorio);
                            explicación línea a línea; señalar explícitamente los puntos de fallo
  práctica                → ejercicio que produce un resultado ejecutable y verificable;
                            diferente al ejemplo central; el aprendiz decide cómo implementarlo
  completitud_del_bloque  → cuando el aprendiz puede escribir código nuevo que use la
                            competencia sin consultar documentación ni el ejemplo del bloque

CIENCIA FORMAL (Matemáticas, Álgebra, Lógica, Estadística)
  competencia_observable  → "puede demostrar/calcular/probar [objeto] bajo [condición]"
  concepto_completo       → definición formal + intuición geométrica o física +
                            condiciones de validez + casos degenerados + contraejemplos
  ejemplo                 → demostración completa (hipótesis → desarrollo paso a paso → conclusión)
                            O resolución numérica con todos los pasos y unidades explícitas
  práctica                → ejercicio con resultado verificable por el aprendiz;
                            diferente al ejemplo; no "explica con tus palabras"
  completitud_del_bloque  → cuando el aprendiz puede resolver un ejercicio nuevo del mismo
                            tipo sin consultar el bloque ni sus notas

CIENCIA EXPERIMENTAL (Física, Química, Biología)
  competencia_observable  → "puede aplicar/calcular/explicar [fenómeno] dado [condición experimental]"
  concepto_completo       → descripción del fenómeno + mecanismo + modelo con sus límites +
                            condiciones en que el modelo falla + errores de medición frecuentes
  ejemplo                 → problema resuelto con datos reales (magnitudes, unidades, resultado);
                            señalar qué hipótesis se asumen y cuándo dejan de ser válidas
  práctica                → problema con datos concretos y resultado verificable;
                            el aprendiz aplica el modelo en situación diferente al ejemplo
  completitud_del_bloque  → cuando el aprendiz puede resolver problemas nuevos del mismo
                            tipo identificando correctamente las hipótesis del modelo

HUMANIDADES (Literatura, Filosofía, Historia, Lingüística)
  competencia_observable  → "puede analizar/identificar/argumentar [objeto] con [criterio metodológico]"
  concepto_completo       → definición del concepto + contexto que lo generó +
                            interpretaciones en tensión + matices que se suelen ignorar +
                            errores interpretativos frecuentes
  ejemplo                 → análisis concreto de un texto, argumento o caso con citas;
                            no descripciones genéricas; mostrar la aplicación del método
  práctica                → ejercicio activo y verificable: analizar un fragmento nuevo,
                            contrastar dos posiciones, identificar falacias en un argumento
  completitud_del_bloque  → cuando el aprendiz puede aplicar el concepto a material nuevo
                            y justificar su análisis con criterio metodológico propio

CIENCIAS SOCIALES (Economía, Psicología, Sociología, Derecho)
  competencia_observable  → "puede aplicar/explicar/evaluar [concepto] en [contexto concreto]"
  concepto_completo       → definición + evidencia empírica + supuestos del modelo +
                            limitaciones y cuándo no aplica + críticas recibidas
  ejemplo                 → caso de estudio con datos reales o situación bien definida;
                            aplicación explícita del concepto; señalar los límites del caso
  práctica                → análisis de un caso nuevo; el aprendiz aplica el modelo y
                            señala sus supuestos y limitaciones en ese contexto
  completitud_del_bloque  → cuando el aprendiz puede aplicar el concepto críticamente,
                            no solo describir su definición

ARTES (Teoría musical, Artes visuales, Arquitectura)
  competencia_observable  → "puede identificar/aplicar/analizar [técnica o principio] en [obra o trabajo]"
  concepto_completo       → descripción del principio + función expresiva + contexto histórico +
                            variaciones y cómo se reconocen + errores de interpretación frecuentes
  ejemplo                 → análisis de obra concreta con identificación de elementos y técnica;
                            referencias verificables; no descripciones genéricas
  práctica                → ejercicio de análisis o aplicación con resultado verificable;
                            diferente a la obra del ejemplo
  completitud_del_bloque  → cuando el aprendiz puede identificar o aplicar el principio
                            en obras o trabajos nuevos sin guía

---

FASE 4 — GENERACIÓN DE META.MD

Genera el fichero meta.md completo con exactamente estas secciones y este formato:

───────────────────────────────────────────────────────────────────────────────
# meta.md — [TEMA]

## Identificación

- **Tema**: [nombre completo del tema]
- **Versión de referencia**: [ej: Spring AI 1.1.0 / Java 21 / n/a si no aplica]
- **Idioma**: [español / inglés / otro]
- **Prefijo de ficheros**: [2-3 sílabas en kebab-case; ej: sai- / k8s- / alg- / fis-]

## Clasificación del dominio

- **Tipo primario**: [nombre del tipo]
- **Tipo secundario**: [nombre del tipo o "—" si no aplica]
- **Descripción**: [una frase que describe exactamente el alcance del dominio]
- **Cert flag**: [sí — [organismo]: [nombre exacto del examen] | no]

## Fuente autoritativa

- **Principal**: [fuente exacta; ej: docs.spring.io/spring-ai / temario oficial CKA]
- **Secundaria**: [fuente complementaria o "—" si no aplica]

## Perfil del aprendiz

- **Nivel actual**: [exactamente el campo NIVEL_ACTUAL de premeta.md]
- **Objetivo**: [exactamente el campo OBJETIVO de premeta.md]
- **Prerequisitos implícitos**:
  - [conocimiento que el aprendiz necesita antes de empezar el primer bloque]
  - [añadir los necesarios — solo lo que no está cubierto por la ruta]

## Mapa de competencias

Lo que el aprendiz DEBE poder hacer al terminar la ruta completa.
Organizado por módulos o fases, según la estructura natural del dominio.
Cada competencia en forma observable: "puede [verbo] [objeto] bajo [condición]".

No es una lista de temas — es una lista de capacidades adquiridas.

### [Módulo o fase 1]
- puede [competencia 1]
- puede [competencia 2]
- ...

### [Módulo o fase 2]
- puede [competencia 1]
- ...

[añadir los módulos necesarios]

## Fuera del alcance

Sub-dominios que el aprendiz podría asumir incluidos pero que esta ruta no cubre,
y la razón concreta por la que quedan fuera.

- [sub-dominio excluido] — razón: [por qué queda fuera dado el OBJETIVO del aprendiz]
- [añadir los necesarios; si no hay exclusiones evidentes, escribir "ninguna"]

## Estrategia de progresión

[Describe en prosa (3-6 frases) el orden didáctico de los módulos.
Responde implícitamente: ¿por qué este módulo va antes que el siguiente?
¿Dónde están los puntos de integración natural (tareas integradoras)?
¿Qué módulos pueden estudiarse en paralelo si el tiempo lo permite?]

## Patrones de contenido

Estos patrones son usados por el PROMPT 2 para generar cada bloque de aprendizaje.
Definen qué significa un bloque completo en este dominio concreto.

### Competencia observable
[copia aquí el patrón derivado en Fase 3 para este dominio]

### Concepto completo
[copia aquí el patrón derivado en Fase 3 para este dominio]

### Ejemplo
[copia aquí el patrón derivado en Fase 3 para este dominio]

### Práctica
[copia aquí el patrón derivado en Fase 3 para este dominio]

### Criterio de completitud del bloque
[copia aquí el patrón derivado en Fase 3 para este dominio]

## Notas

[Decisiones de alcance o inferencias que requieren justificación.
Por ejemplo: si el OBJETIVO implica un dominio muy amplio, justificar qué módulos
se incluyen y cuáles quedan fuera dado el perfil del aprendiz.
Si no hay nada que justificar, escribir "—".]
───────────────────────────────────────────────────────────────────────────────

---

REGLAS DE CALIDAD

1. Cada campo se rellena con información real derivada del TEMA y del perfil:
   no hay campos genéricos ni valores de plantilla sin completar.

2. El mapa de competencias describe capacidades observables, no una lista de temas.
   MAL: "conoce los advisors de Spring AI"
   BIEN: "puede implementar RAG automático usando QuestionAnswerAdvisor con cualquier VectorStore"

3. Los patrones de contenido deben ser concretos. El PROMPT 2 debe poder generar
   el bloque correcto sin volver a razonar sobre el dominio.
   MAL: "un ejemplo del concepto"
   BIEN: "código completo y ejecutable en situación real; explicación línea a línea;
          señalar explícitamente los puntos de fallo"

4. La estrategia de progresión describe el orden real de los módulos y por qué,
   no una lista genérica de categorías.

5. Los prerequisitos implícitos son lo que el aprendiz necesita ANTES de empezar
   el primer bloque — no los temas de la propia ruta.

6. El fichero meta.md generado debe poderse usar directamente como entrada del
   PROMPT D y del PROMPT 1 sin necesidad de edición manual.

---

Cómo usarlo: pega el contenido de premeta.md en ENTRADA y ejecuta. El modelo
analiza el dominio, clasifica el tipo, deriva los patrones de contenido y genera
meta.md completo listo para alimentar al PROMPT D y al PROMPT 1.
