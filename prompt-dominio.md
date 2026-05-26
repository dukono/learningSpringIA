PROMPT D — ANALIZADOR DE DOMINIO POR MÓDULO

EJECUCIONES REQUERIDAS

Este prompt debe ejecutarse una vez por cada módulo del campo "Mapa de competencias"
de meta.md. El número total de ejecuciones es igual al número de módulos de ese campo.

Antes de ejecutar, construye la tabla de seguimiento leyendo el CONTEXTO:

  Módulo (sección de "Mapa de competencias")  →  dominio-[prefijo]-[slug].json

No ejecutes el siguiente prompt de la cadena (prompt-indice.md) hasta tener
un fichero dominio-*.json por cada módulo del campo "Dentro del alcance".

---

Actúa como experto en el dominio indicado. Tu única tarea es producir el mapa
completo de conocimiento profesional para UN módulo del proyecto de documentación
y derivar de él la lista exacta de nodos hoja que ese módulo requiere en el índice.

Su salida (dominio-[módulo].json) es la entrada que consume el prompt-indice.md
para construir la rama correcta del árbol, en lugar de razonar desde cero
sobre el contenido del módulo.

Sin este paso, el índice se genera desde los nombres de los módulos (qué existe)
en lugar de desde los requisitos de competencia (qué necesita saber un profesional).
El resultado de omitirlo es siempre el mismo: nodos colapsados, temas omitidos
y granularidad incorrecta.

---

ENTRADA

CONTEXTO

Pega aquí el contenido completo de meta.md generado por PROMPT 0.
El modelo leerá los siguientes campos:

  Campo en meta.md                               Usado en
  ─────────────────────────────────────────────  ────────────────────────────────────────
  Identificación → Versión de referencia         coherencia de APIs en los ejemplos
  Identificación → Prefijo de ficheros           naming de los nodos hoja
  Clasificación → Tipo primario                  criterios de Grupo 1/Grupo 2 en Fase 2
                                                 y orden didáctico en Fase 4
  Clasificación → Cert flag                      pregunta de contexto operativo en Fase 1
                                                 y sección de evaluación en Fase 4
  Perfil del aprendiz → Nivel actual             profundidad y vocabulario de los conceptos
  Perfil del aprendiz → Prerequisitos implícitos qué puede clasificarse directamente como Grupo 2
  Mapa de competencias                           verifica que el MÓDULO está en el alcance
                                                 y que las competencias del módulo están cubiertas
  Fuera del alcance                              descarta conceptos excluidos en Fase 1

<< PEGA AQUÍ EL CONTENIDO COMPLETO DE meta.md >>

MÓDULO

Nombre exacto del módulo tal como aparece en "Dentro del alcance" de meta.md.

MÓDULO = "[copia el nombre exacto del módulo del campo Mapa de competencias]"

Ejemplo:
  MÓDULO = "ChatClient y conversación"

---

FASE 1 — MAPA DE COMPETENCIAS

Enumera TODOS los conceptos que el destinatario necesita dominar para operar
este módulo en su contexto profesional real, con el nivel y versión de referencia
del CONTEXTO.

Para cada área del módulo, formula la pregunta de contexto operativo antes de
listar. La pregunta exacta depende del Tipo primario y Cert flag del CONTEXTO:

  Cert flag = sí (cualquier tipo de dominio):
    "¿Qué pregunta del examen o evaluación oficial respondería mal el candidato
     si no conociera este concepto?"

  Cert flag = no — Tipo primario tech
  (Lenguaje, Framework, Plataforma, Herramienta, Orquestador):
    "¿Qué fallaría o quedaría mal configurado en el sistema si el profesional
     no conociera este concepto?"

  Cert flag = no — Tipo primario ciencias formales o experimentales:
    "¿Qué resultado no podría demostrar, derivar o calcular correctamente
     si el estudiante no conociera este concepto?"

  Cert flag = no — Tipo primario humanidades, sociales o artes:
    "¿Qué análisis o argumento formularía incorrectamente si el estudiante
     no conociera este concepto?"

Si la respuesta es "ningún impacto concreto" → el concepto no pertenece al mapa.
Si la respuesta identifica un impacto concreto → el concepto pertenece al mapa.

Organiza los conceptos en áreas temáticas. Un área es un conjunto de conceptos
que comparten el mismo objeto de operación (ej: "ciclo de vida de un proceso",
"leyes del movimiento", "instrumentos de política fiscal", "fuentes primarias del período").

Regla de granularidad en esta fase: no colapsar. Un concepto es la unidad mínima
que tiene nombre propio, parámetros o configuración propios, o un comportamiento
diferenciado respecto al resto. No hay límite de conceptos; el número correcto es
el que cubre la realidad del módulo en producción.

Regla de familia: cuando varios ítems comparten un patrón nominal o pertenecen
a la misma familia conceptual, cada uno con parámetros, condiciones de contorno
o comportamiento propios es un concepto separado, no una entrada colapsada.

Regla de exhaustividad: recorre mentalmente el ciclo de vida completo del módulo
en un entorno de producción real — desde la primera dependencia en el pom.xml
hasta la operación en un clúster con múltiples instancias. No omitas el día 2
(actualización de configuración, rotación de credenciales, degradación y recuperación).

Checklist de sub-áreas de omisión frecuente — responde estas dos preguntas antes
de cerrar el mapa de competencias:

  a) ¿Hay sub-áreas del módulo que no emergen de una descripción general del tema
     y requieren enumeración explícita? El tipo de sub-área depende del dominio:
     - Frameworks / herramientas tech: puntos de extensión, variantes del componente,
       adaptadores y factories personalizables
     - Certificaciones: servicios o características de nicho, casos edge, escenarios
       de troubleshooting que el temario oficial lista pero rara vez se explican
     - Humanidades / ciencias sociales: corrientes interpretativas minoritarias,
       fuentes primarias, debates historiográficos o metodológicos sin consenso
     - Ciencias formales: demostraciones, contraejemplos, condiciones de contorno,
       casos degenerados de teoremas
     Si existen sub-áreas de este tipo, inclúyelas en el mapa antes de continuar.

  b) ¿El CONTEXTO especifica una edición, versión, corte temporal o temario oficial
     que pueda ser más reciente que tu conocimiento base? Si es así, advierte
     explícitamente qué conceptos o secciones podrías estar omitiendo antes de
     continuar a Fase 2.

---

FASE 2 — SEPARACIÓN GRUPO 1 / GRUPO 2

Clasifica cada concepto del mapa en uno de estos dos grupos:

GRUPO 1 — Propio del módulo
  El concepto vive en el espacio de responsabilidad del módulo.
  La documentación lo explica en profundidad.

  Criterios según Tipo primario del CONTEXTO (basta con que se cumpla uno):

  TECH (Lenguaje, Framework, Plataforma, Herramienta, Orquestador):
  - La propiedad o parámetro pertenece al namespace o API del módulo
  - La clase, anotación o comando lo provee el paquete del módulo
  - El endpoint o salida lo registra o produce el módulo
  - El comportamiento es una decisión de diseño interna del módulo

  CIENCIAS FORMALES Y EXPERIMENTALES:
  - El concepto se define o demuestra dentro del sistema formal del módulo
  - El teorema, propiedad o resultado es derivable en el módulo sin importarlo
  - La técnica o método es específica del área estudiada en el módulo

  HUMANIDADES, SOCIALES Y ARTES:
  - El concepto, evento, obra o autor pertenece al período, corriente o
    disciplina que el módulo estudia directamente
  - La fuente es primaria para el objeto de estudio del módulo
  - El debate o interpretación es central e interno al módulo

GRUPO 2 — Externo al módulo
  El concepto pertenece a otro módulo, disciplina o dominio de conocimiento.
  La documentación lo menciona con una referencia concreta. No se explica.

  Criterios según Tipo primario del CONTEXTO (basta con que se cumpla uno):

  TECH:
  - Pertenece a otra librería, plataforma o herramienta que el módulo invoca
    pero no implementa
  - Es un concepto general de arquitectura o ingeniería de sistemas
  - Está en la lista "Fuera del alcance" del CONTEXTO

  CIENCIAS FORMALES Y EXPERIMENTALES:
  - El resultado se importa de otra rama sin demostrarse en este módulo
  - La herramienta de cálculo o software es externa al módulo
  - Es un prerequisito ya cubierto en módulos anteriores del CONTEXTO

  HUMANIDADES, SOCIALES Y ARTES:
  - El contexto pertenece a otro período, corriente o disciplina
  - La fuente es secundaria o pertenece a otra rama del conocimiento
  - Es background general que el módulo asume conocido

  En todos los tipos: está en la lista "Fuera del alcance" del CONTEXTO → Grupo 2.

Regla de frontera: si el módulo EXPONE un elemento que CONFIGURA o APLICA un
concepto externo, ese elemento es Grupo 1 aunque el concepto subyacente sea Grupo 2.
  Ejemplo tech:        parámetro del módulo que activa un comportamiento externo → G1
                       la API interna del componente externo                      → G2
  Ejemplo humanidades: interpretación propia del período sobre un evento externo  → G1
                       el evento externo en sí, perteneciente a otro módulo       → G2

VERIFICACIÓN F1 → F2 (obligatoria antes de continuar a Fase 3)

  Recorre cada área del `mapa_competencias` generado en Fase 1.
  Para cada área, confirma que al menos uno de sus conceptos aparece en `grupo1`
  o en `grupo2`.
  Si algún área no está representada en ninguno de los dos grupos, asígnala ahora.
  No continúes a Fase 3 hasta que cada área del mapa tenga asignación de grupo.

---

FASE 3 — ANÁLISIS DE COHESIÓN

Aplica únicamente sobre los conceptos del Grupo 1.

Agrupa los conceptos en candidatos a fichero. Para cada candidato, ejecuta
estos dos tests en orden. No asignes un nodo hoja sin completar ambos tests.

TEST 1 — Test de competencia (obligatorio primero)

  Formula: "El aprendiz puede [VERBO] [OBJETO] bajo [CONDICIÓN O CONTEXTO]"

  — Si no puedes formularlo con un único verbo y un único objeto concreto:
    la agrupación no tiene competencia definida → fusionar con el hermano de mayor
    afinidad semántica (el que comparte objeto de operación, no el adyacente en lista)
    o eliminar si ningún hermano la absorbe con coherencia.

  — Si la formulación requiere dos verbos con objetos no relacionados:
    la agrupación cubre dos competencias distintas → dividir en dos candidatos separados.

TEST 2 — Test de sostenibilidad

  ¿Puede este candidato sostener de forma natural un bloque de aprendizaje completo?
  Un bloque completo contiene: objetivo observable, explicación del concepto con sus
  caras ocultas, ejemplo funcional, práctica verificable y preguntas QA.

  Sí a los dos tests → nodo hoja válido.
  No al test 1     → fusionar o eliminar antes de continuar.
  No al test 2     → fusionar con el hermano de mayor afinidad semántica
                     (criterio: comparte objeto de operación o el concepto
                     absorbido tiene una interacción directa con el hermano receptor).

Reglas de tamaño (aplicar después de los tests):

  Candidato que al desarrollarse superaría 500 líneas → dividir en hijos.
  Candidato que al desarrollarse no alcanzaría 100 líneas → fusionar con hermano.
  Dos conceptos con TAREA no relacionada en el mismo candidato → fichero separado.

Antes de estimar: si alguna entrada de grupo1 en el candidato agrupa múltiples
variantes bajo una sola cadena (indicado por barras, comas o conjunciones del
tipo "X / Y / Z" o "X, Y y Z"), expande cada variante en ítem individual antes
de contar. Una entrada colapsada cuenta como tantos ítems como variantes
representa, no como uno.

Para estimar las líneas: para cada ítem expandido de grupo1 en el candidato,
asigna su densidad individual y suma los resultados:

  - concepto simple: dato, definición o propiedad con un ejemplo y tabla
    ≈ 40–60 líneas
  - concepto intermedio: requiere varios ejemplos, casos edge, o explicar
    su interacción con conceptos adyacentes del mismo candidato ≈ 60–80 líneas
  - concepto complejo: comportamiento con desarrollo extendido, diagrama
    o múltiples modos de operación ≈ 80–120 líneas

No apliques una densidad media uniforme a todos los ítems del candidato.
Suma las densidades individuales. Ante la duda entre categorías, aplica
la superior — el riesgo de subestimar es mayor que el de sobreestimar.

VERIFICACIÓN F2 → F3 (obligatoria antes de continuar a Fase 4)

  Recorre cada área de `grupo1` generada en Fase 2.
  Para cada área, confirma que al menos uno de sus conceptos aparece en algún
  candidato del `analisis_cohesion`.
  Si algún área de `grupo1` no tiene candidato, procésala ahora aplicando los
  dos tests de Fase 3 (test de tarea + test de sostenibilidad).
  No continúes a Fase 4 hasta que cada área de grupo1 tenga candidato asignado.

---

FASE 4 — GENERACIÓN DE NODOS

Produce la lista final de nodos para este módulo en el formato de indice.md.

Convenciones de naming (idénticas a las de prompt-indice.md):
- Usa X como marcador del número de tema; se sustituye al integrar en indice.md.
- Nodos intermedios (con hijos): sin fichero → indicar entre paréntesis cuántos
  hijos agrupa y por qué forman un sub-dominio distinto.
- Nodos hoja (sin hijos): con fichero [prefijo]-nombre-en-kebab-case.md.
  El prefijo se toma del campo "Prefijo de ficheros" del CONTEXTO.
- El último nodo hoja de cada módulo es siempre:
  X.N  Testing / Verificación de [Nombre del módulo]  → [prefijo]-[slug]-testing.md
- Orden didáctico interno dentro del módulo.
  Aplica la secuencia según Tipo primario y Cert flag del CONTEXTO:

  TECH — Cert flag = no:
    Arquitectura y concepto → Setup y primera configuración →
    Configuración avanzada → Uso operativo → Variantes y casos edge →
    Integración con otros módulos del CONTEXTO → Testing

  TECH — Cert flag = sí:
    Concepto y alcance en el examen → Configuración base →
    Casos de uso evaluados → Integración con otros servicios →
    Escenarios tipo examen → Testing

  CIENCIAS FORMALES/EXPERIMENTALES — Cert flag = no:
    Motivación y contexto → Definición formal → Teoremas y propiedades →
    Demostraciones → Casos degenerados y contraejemplos → Aplicaciones

  CIENCIAS FORMALES/EXPERIMENTALES — Cert flag = sí:
    Motivación y contexto → Definición formal → Teoremas y propiedades →
    Demostraciones → Casos degenerados → Ejercicios tipo examen

  HUMANIDADES/SOCIALES/ARTES — Cert flag = no:
    Contexto y período → Causas y antecedentes → Desarrollo →
    Consecuencias → Fuentes e interpretaciones → Debate académico

  HUMANIDADES/SOCIALES/ARTES — Cert flag = sí:
    Contexto y período → Causas y antecedentes → Desarrollo →
    Consecuencias → Fuentes e interpretaciones → Preguntas tipo examen

Para cada nodo hoja, rellena `prerequisitos` y `desbloquea`:
  - `prerequisitos`: IDs de nodos que el aprendiz debe haber completado antes de
    abordar este. Se infieren del orden didáctico: si el concepto del nodo usa
    o asume el concepto de otro nodo, ese nodo es prerequisito.
  - `desbloquea`: IDs de nodos que solo pueden abordarse después de completar este.
    Es la relación inversa: si este nodo es prerequisito de otro, aparece en `desbloquea`.
  Los IDs usan el formato "X.N" provisional; se consolidarán en prompt-ruta.md.

Para cada concepto de Grupo 2 identificado en Fase 2, añade al final de la lista
una sección "Tratamiento de Grupo 2" con el formato:

  [G2] [nombre del concepto]
       Aparece en: [nombre del fichero donde se menciona]
       Tratamiento: [snippet de N líneas / comando exacto / tabla de una fila]
                    + enlace a [fuente autoritativa]

---

FORMATO DE SALIDA

Genera el fichero dominio-[slug-del-módulo].json con la siguiente estructura JSON:

───────────────────────────────────────────────────────────────────────────────
{
  "slug": "[slug-del-módulo]",
  "nombre": "[Nombre del módulo]",
  "modulo_analizado": "[Nombre exacto del módulo copiado de MÓDULO]",

  "mapa_competencias": [
    {
      "area": "[nombre del área temática]",
      "conceptos": [
        "[concepto 1]",
        "[concepto 2]"
      ]
    }
  ],

  "grupo1": [
    {
      "area": "[nombre del área]",
      "conceptos": ["[concepto A]", "[concepto B]"]
    }
  ],

  "grupo2": [
    {
      "concepto": "[nombre del concepto externo]",
      "aparece_en": "[nombre del fichero donde se menciona]",
      "tratamiento": "[snippet de N líneas / comando exacto / tabla de una fila]",
      "referencia": "[URL o fuente autoritativa]"
    }
  ],

  "analisis_cohesion": [
    {
      "candidato": "[nombre del candidato a fichero]",
      "competencia": "El aprendiz puede [VERBO] [OBJETO] bajo [CONDICIÓN O CONTEXTO]",
      "test_sostenibilidad": true,
      "estimacion_lineas": 0,
      "decision": "[nodo_valido | fusionar_con:[hermano] | dividir_en:[lista]]",
      "razon": "[explicación de la decisión si aplica]"
    }
  ],

  "nodos_hoja": [
    {
      "numero": "X.N",
      "titulo": "[Título del nodo]",
      "fichero": "[prefijo]-nombre-en-kebab-case.json",
      "tipo": "[hoja | intermedio]",
      "prerequisitos": [],
      "desbloquea": [],
      "hijos": []
    }
  ],

  "notas_integracion": [
    "[advertencia o decisión de orden no obvia desde el nombre del nodo]"
  ]
}
───────────────────────────────────────────────────────────────────────────────

---

PROTOCOLO DE ESCRITURA AUTÓNOMA

El JSON es extenso por diseño. Para evitar errores de límite de output,
cada sección se genera y escribe de forma independiente usando Edit.
Nunca se genera más de una sección en una sola operación.

Ejecuta los pasos de forma secuencial y autónoma, sin solicitar
confirmación entre ellos. No uses herramientas de gestión de tareas ni
de planificación — interrumpen la secuencia antes de la escritura.

REGLA DE ESCRITURA EN DISCO

Estrategia: Bash una sola vez (skeleton vacío) + Edit por sección.
No produzcas contenido JSON como texto en la respuesta.
La primera acción al iniciar el protocolo es crear el skeleton (PASO A0).
Cada sección se rellena con su propio Edit independiente.

VERIFICACIÓN PRE-EDIT (obligatoria antes de cada llamada Edit):
  Confirma que el contenido a insertar tiene: todos los arrays y objetos
  cerrados, sin trailing commas, sin comillas sin escapar.
  Si detectas un error, corrígelo antes de llamar a Edit.

VERIFICACIÓN POST-EDIT (obligatoria después de cada llamada Edit):
  Lee dominio-[slug].json desde disco y confirma que la sección recién
  escrita existe, no está vacía y es JSON válido en su conjunto.
  Si la sección está truncada o ausente, repite el Edit.

BARRERA DE EJECUCIÓN (obligatoria entre fases)

Al completar Fase 2: DETÉN la generación de contenido analítico.
  Ejecuta PASO A1, A2 y A3 (Edits) antes de continuar a Fase 3.
  No generes ningún contenido de Fase 3 hasta que los tres Edits
  hayan completado. Tras el último Edit, ejecuta RESTAURACIÓN DE
  CONTEXTO y continúa a Fase 3.

Al completar Fase 3: DETÉN la generación de contenido analítico.
  Ejecuta PASO B (Edit) antes de continuar a Fase 4.
  No generes ningún contenido de Fase 4 hasta que el Edit haya
  completado. Tras el Edit, ejecuta RESTAURACIÓN DE CONTEXTO
  y continúa a Fase 4.

Al completar Fase 4: DETÉN la generación de contenido analítico.
  Ejecuta PASO C1 y C2 (Edits) antes de terminar.

RESTAURACIÓN DE CONTEXTO (obligatoria tras cada barrera, antes de
la siguiente Fase)

  1. Lee dominio-[slug].json desde disco.
  2. Extrae y mantén activos en memoria de trabajo:
       — todas las áreas y conceptos de mapa_competencias
       — la clasificación G1/G2 de cada concepto
       — los candidatos y decisiones de analisis_cohesion (si ya existen)
  3. La siguiente Fase opera sobre estos datos como fuente de verdad.
     No regeneres ni reformules lo ya escrito — refiérelo literalmente.
  4. Si algún dato necesario para la siguiente Fase no está en el
     fichero, detente y reporta el problema antes de continuar.

PASO A0 — Antes de comenzar Fase 1 (skeleton):
  Usa SIEMPRE Bash para crear el fichero. Write requiere haber leído
  el fichero previamente; si el fichero no existe aún, Write falla.
  Bash no tiene esa restricción.

  Comando exacto (reemplaza [slug] por el slug real del módulo):

    cat > dominio-[slug].json << 'ENDJSON'
    {
      "slug": "[slug-del-módulo]",
      "nombre": "[Nombre del módulo]",
      "modulo_analizado": "[Nombre exacto del módulo copiado de MÓDULO]",
      "mapa_competencias": [],
      "grupo1": [],
      "grupo2": [],
      "analisis_cohesion": [],
      "nodos_hoja": [],
      "notas_integracion": []
    }
    ENDJSON

  Este Bash es pequeño por diseño — solo establece la estructura.
  A partir de aquí, cada sección se rellena con Edit (que sí requiere
  leer primero; usa Read sobre dominio-[slug].json antes de cada Edit).

PASO A1 — Al terminar Fase 1:
  1. Ejecuta VERIFICACIÓN PRE-EDIT sobre el contenido de mapa_competencias.
  2. Edit: reemplaza "mapa_competencias": [] por el resultado de Fase 1.
  3. Ejecuta VERIFICACIÓN POST-EDIT.

PASO A2 — Al terminar Fase 2 (grupo1):
  1. Ejecuta VERIFICACIÓN PRE-EDIT sobre el contenido de grupo1.
  2. Edit: reemplaza "grupo1": [] por el resultado de Fase 2.
  3. Ejecuta VERIFICACIÓN POST-EDIT.

PASO A3 — Al terminar Fase 2 (grupo2):
  1. Ejecuta VERIFICACIÓN PRE-EDIT sobre el contenido de grupo2.
  2. Edit: reemplaza "grupo2": [] por el resultado de Fase 2.
  3. Ejecuta VERIFICACIÓN POST-EDIT.
  → Ejecuta BARRERA: RESTAURACIÓN DE CONTEXTO y continúa a Fase 3.

PASO B — Al terminar Fase 3:
  1. Ejecuta VERIFICACIÓN PRE-EDIT sobre el contenido de analisis_cohesion.
  2. Edit: reemplaza "analisis_cohesion": [] por el resultado de Fase 3.
     El contenido debe referenciar literalmente los slugs y nombres
     guardados en mapa_competencias y grupo1.
  3. Ejecuta VERIFICACIÓN POST-EDIT.
  → Ejecuta BARRERA: RESTAURACIÓN DE CONTEXTO y continúa a Fase 4.

PASO C1 — Al terminar Fase 4 (nodos_hoja):
  1. Ejecuta VERIFICACIÓN PRE-EDIT sobre el contenido de nodos_hoja.
  2. Edit: reemplaza "nodos_hoja": [] por el resultado de Fase 4.
     Los ficheros referenciados deben coincidir con los candidatos
     aprobados en analisis_cohesion.
  3. Ejecuta VERIFICACIÓN POST-EDIT.

PASO C2 — Al terminar Fase 4 (notas_integracion):
  1. Ejecuta VERIFICACIÓN PRE-EDIT sobre el contenido de notas_integracion.
  2. Edit: reemplaza "notas_integracion": [] por el resultado de Fase 4.
  3. Ejecuta VERIFICACIÓN POST-EDIT.

---

REGLAS DE CALIDAD

Q1. El mapa de competencias no contiene conceptos genéricos ("configuración básica",
    "uso del módulo"). Cada concepto es nombrable: una propiedad, una anotación,
    un endpoint, un comportamiento diferenciado o un patrón de fallo específico.

Q2. La clasificación Grupo 1 / Grupo 2 es binaria y no negociable. No existe
    "Grupo 1 con mención breve". Si es Grupo 1 se documenta; si es Grupo 2
    se menciona con snippet y referencia.

Q3. El test de competencia de Fase 3 es obligatorio para cada nodo candidato.
    Ningún nodo hoja aparece en la lista sin su COMPETENCIA formulada explícitamente
    en forma observable: "El aprendiz puede [VERBO] [OBJETO] bajo [CONDICIÓN]".

Q4. Un módulo con nombre compuesto (ej: "Config Server y Config Client") puede
    y debe generar sub-ramas independientes si los componentes tienen ciclos de vida
    y TAREA distintos. El nombre compuesto no obliga a un único agrupador.

Q5. Los nodos hoja deben ser auto-consistentes: ningún fichero cubre conceptos
    de Grupo 2 en profundidad, y ningún concepto de Grupo 1 queda sin fichero.

Q6. El slug del fichero dominio-[slug].json se deriva del nombre del módulo en
    kebab-case con el prefijo del CONTEXTO. Ejemplos:
    "Config Server y Config Client" → dominio-sc-config.json
    "Spring Cloud Gateway"          → dominio-sc-gateway.json
    "Spring Cloud Stream"           → dominio-sc-stream.json

---

POSICIÓN EN LA CADENA DE PROMPTS

La cadena correcta es:

  premeta.md
      ↓
  prompt-meta.md    →  meta.md
      ↓
  prompt-dominio.md →  dominio-[módulo].json ← ejecutar UNA VEZ POR MÓDULO
      ↓                                         antes de construir la ruta
  prompt-ruta.md    →  ruta.json             ← consume los dominio-*.json
      ↓                                         en lugar de razonar desde cero
  prompt-section.md →  bloque generado       ← un bloque por nodo hoja

El prompt-ruta.md recibe en su CONTEXTO: meta.md + todos los ficheros dominio-*.json
del proyecto. Consolida los nodos hoja de todos los módulos en un grafo de prerequisitos
único y determina dónde insertar las tareas integradoras.

NOTA DE EJECUCIÓN — PARALELISMO

Todos los módulos del campo "Dentro del alcance" de meta.md son independientes
entre sí. Sus ficheros dominio-*.json pueden generarse en paralelo — no existe
dependencia de orden entre módulos distintos.

La única dependencia de orden es interna a cada módulo:
  PASO A → PASO B → PASO C  (secuencial dentro del mismo módulo)

LÍMITE DE CONCURRENCIA

Lanzar como máximo 2–3 módulos en paralelo por tanda.
Lanzar N módulos a la vez genera un burst de llamadas simultáneas que
puede activar el rate limiting del proveedor de API (AWS Bedrock,
Azure OpenAI, etc.) y terminar los agentes antes de que arranquen.

Con más de 3 módulos, usar tandas:
  Tanda 1: módulos 1, 2, 3 → esperar confirmación de completado
  Tanda 2: módulos 4, 5, …

El tiempo total sigue siendo el del módulo más complejo de cada tanda,
no la suma de todos los módulos.

---

Cómo usarlo: pega meta.md en CONTEXTO, escribe el nombre exacto del módulo
en MÓDULO y ejecuta. El resultado es dominio-[slug].json, listo para alimentar
al prompt-ruta.md.
