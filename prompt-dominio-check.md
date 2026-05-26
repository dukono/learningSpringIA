PROMPT DOMINIO-CHECK — REVISOR Y CORRECTOR DE dominio-*.json

Actúa como revisor de calidad de documentación de estudio. Tu única tarea es
auditar UN fichero dominio-*.json generado por prompt-dominio.md, identificar
todos los defectos y aplicar las correcciones necesarias.

---

ENTRADA

CONTEXTO

Pega aquí el contenido completo de meta.md. El revisor leerá:
  - Identificación → Versión de referencia         coherencia de versión en conceptos
  - Identificación → Prefijo de ficheros           validación de naming en nodos_hoja
  - Clasificación → Tipo primario                  criterios G1/G2 y ciclo de conocimiento
  - Clasificación → Cert flag                      pregunta de contexto operativo en T5
  - Perfil del aprendiz → Nivel actual             sujeto y profundidad en Q5 y T5
  - Fuera del alcance                              validación de Grupo 2

<< PEGA AQUÍ EL CONTENIDO COMPLETO DE meta.md >>

FICHERO A REVISAR

Pega aquí el contenido completo del fichero dominio-*.json a auditar.

<< PEGA AQUÍ EL CONTENIDO COMPLETO DEL dominio-*.json >>

---

FASE 1 — VERIFICACIÓN ESTRUCTURAL

Comprueba que el fichero contiene exactamente estas 9 claves de primer nivel,
en este orden:

  1. slug
  2. nombre
  3. modulo_analizado
  4. mapa_competencias
  5. grupo1
  6. grupo2
  7. analisis_cohesion
  8. nodos_hoja
  9. notas_integracion

Para cada clave ausente anota: FALTA [nombre_clave].
Para cada clave presente marca: OK [nombre_clave].

No continúes a Fase 2 hasta completar esta tabla.

---

FASE 2 — VERIFICACIÓN DE CALIDAD DEL CONTENIDO EXISTENTE

Aplica estas reglas sobre las secciones que SÍ existen en el fichero.

Q1 — GRANULARIDAD DEL MAPA DE COMPETENCIAS
  Cada concepto en mapa_competencias debe ser nombrable e inequívoco:
  una propiedad, fórmula, norma, mecanismo o comportamiento específico
  propio del módulo; un símbolo, término técnico o construcción concreta;
  un patrón de fallo o caso edge identificable.
  Fallo: concepto genérico que no designa nada específico, como
  "configuración básica", "uso del módulo" o "conceptos fundamentales".
  → Anota cada concepto genérico encontrado.

Q2 — CLASIFICACIÓN GRUPO 1 / GRUPO 2
  Criterios Grupo 1 (pertenece al módulo):
  - El concepto vive en el espacio de responsabilidad del módulo: su
    nomenclatura, sintaxis, reglas, mecanismos o comportamientos propios.
  - El símbolo, término, construcción o mecanismo lo provee el módulo
    directamente, no una dependencia externa.
  - El comportamiento es una decisión de diseño del módulo, no una
    convención general de la disciplina.

  Criterios Grupo 2 (externo al módulo):
  - Pertenece a otra disciplina, componente o herramienta del mismo
    ecosistema pero fuera del alcance del módulo.
  - Es una herramienta, sistema o infraestructura externa que el módulo
    utiliza pero no define.
  - Es un concepto general de la disciplina (principio, teorema, patrón,
    metodología) que aplica al módulo pero no le pertenece.
  - Está en "Fuera del alcance" de meta.md.

  REGLA DE FRONTERA: si el módulo expone un mecanismo propio que
  configura o invoca algo externo, ese mecanismo es Grupo 1 aunque
  el concepto subyacente sea Grupo 2.
    Ejemplo abstracto: [parámetro del módulo que activa algo externo] → Grupo 1
                       [API de la herramienta externa en sí]          → Grupo 2

  → Anota cualquier concepto mal clasificado con la corrección propuesta.

Q3 — COBERTURA GRUPO 1 → GRUPO 2
  Verifica que cada área de mapa_competencias tiene al menos un concepto
  en grupo1 o en grupo2.
  → Anota cualquier área sin asignación.

Q3b — COBERTURA grupo1 → analisis_cohesion (si analisis_cohesion existe)
  Verifica que cada área de grupo1 tiene al menos un concepto cubierto
  por algún candidato en analisis_cohesion.
  Un área de grupo1 sin candidato asignado significa que sus conceptos no
  tendrán nodo hoja y quedarán fuera de la ruta final.
  Método: para cada área de grupo1, comprueba que al menos uno de sus
  conceptos aparece nombrado en el campo competencia o razon de algún candidato,
  o que el candidato agrupa explícitamente esa área por nombre.
  Fallo: área de grupo1 sin ningún candidato que la cubra.
  → Anota las áreas huérfanas con el candidato más afín al que deberían asignarse.

Q4 — ESTRUCTURA DE grupo2
  Cada entrada de grupo2 debe tener los 4 campos:
  concepto, aparece_en, tratamiento, referencia.
  El campo tratamiento debe contener un ejemplo concreto y ejecutable
  del dominio (fórmula, definición exacta, comando, fragmento de código,
  tabla de una fila), de 2–5 líneas como máximo, más un enlace a la
  fuente autoritativa.
  Fallo: tratamiento vago ("ver documentación oficial") sin snippet.
  → Anota entradas incompletas.

Q5 — TEST DE COMPETENCIA en analisis_cohesion (si existe)
  Cada candidato debe tener el campo competencia formulado como:
  "El aprendiz puede [VERBO] [OBJETO] bajo [CONDICIÓN O CONTEXTO]"
  con un único verbo y un único objeto concreto.
  Fallo: competencia con dos verbos no relacionados → candidato debe dividirse.
  Fallo: competencia que no puede formularse en forma observable → candidato debe fusionarse.
  Fallo: competencia vaga ("puede usar X", "puede entender Y") sin objeto ni condición.
  → Anota cada candidato con formulación incorrecta o ausente.

Q6 — TEST DE SOSTENIBILIDAD en analisis_cohesion (si existe)
  Cada nodo hoja válido debe poder sostener naturalmente un bloque de aprendizaje
  completo: objetivo observable, explicación del concepto con sus caras ocultas,
  ejemplo funcional, práctica verificable y preguntas QA.

  Antes de estimar: si alguna entrada de grupo1 en el candidato agrupa
  múltiples variantes bajo una sola cadena ("X / Y / Z" o "X, Y y Z"),
  expándelas en ítems individuales antes de contar.

  Estimación de líneas por ítem expandido:
  - concepto simple: dato, definición o propiedad con un ejemplo y tabla
    ≈ 40–60 líneas
  - concepto intermedio: requiere varios ejemplos, casos edge, o interacción
    con conceptos adyacentes del mismo candidato ≈ 60–80 líneas
  - concepto complejo: comportamiento con desarrollo extendido, diagrama
    o múltiples modos de operación ≈ 80–120 líneas
  Suma las densidades individuales. Ante la duda, aplica la categoría superior.

  Rango aceptable por nodo: 100–500 líneas.
  Fallo < 100 líneas → fusionar con el hermano de mayor afinidad semántica
                       (el que comparte objeto de operación, no el adyacente).
  Fallo > 500 líneas → dividir en hijos. En cada candidato hijo resultante
                       añadir el campo:
                         "concepto_padre": "[nombre exacto del candidato original]"
                       El mismo campo debe añadirse en cada entrada de nodos_hoja
                       correspondiente. prompt-ruta.md lo usa para construir
                       el nodo intermedio de agrupación sin fichero.
                       El mecanismo es recursivo: si un hijo sigue superando
                       500 líneas se divide a su vez, y sus hijos apuntan a él
                       como concepto_padre.
  NOTA DE ESQUEMA: "concepto_padre" es un campo que introduce este check
  al aplicar splits; no lo produce prompt-dominio.md en su salida original.
  Solo debe añadirse en los candidatos e ítems de nodos_hoja que resulten
  de una división detectada aquí. Los candidatos que ya eran nodo_valido
  sin split NO deben recibir este campo.
  prompt-ruta.md debe leer "concepto_padre" para construir el nodo
  intermedio de agrupación; si no está presente, asume que el nodo es
  de primer nivel dentro del módulo.
  → Anota cada candidato fuera de rango.

Q7 — COHERENCIA nodos_hoja ↔ analisis_cohesion (si ambos existen)
  Cada nodo en nodos_hoja debe tener su candidato en analisis_cohesion
  con decisión nodo_valido.
  Ningún candidato con decisión fusionar_con o dividir_en debe aparecer
  como nodo hoja independiente.
  → Anota inconsistencias.

Q8 — CONVENCIONES DE NAMING en nodos_hoja (si existe)
  - Prefijo de fichero = prefijo de meta.md (campo "Prefijo de ficheros")
  - Kebab-case para el nombre del fichero
  - El último nodo del módulo debe ser:
    X.N  Testing / Verificación de [Nombre del módulo]  → [prefijo]-[slug]-testing.md
  - Los números usan X como marcador de tema (se sustituye al integrar)
  - Cada nodo hoja debe tener los campos prerequisitos[] y desbloquea[].
    Si están ausentes o vacíos cuando debería haber dependencias — anotarlo.
  - Los nodos producto de un split (Q6) deben incluir el campo
    "concepto_padre": "[nombre del candidato original]"
    tanto en analisis_cohesion como en nodos_hoja.
  → Anota ficheros con prefijo incorrecto, snake_case u otras violaciones.
  → Anota nodos de split sin el campo "concepto_padre".
  → Anota nodos sin prerequisitos/desbloquea cuando las dependencias son evidentes.

Q9 — CALIDAD DE notas_integracion (si existe)
  Cada nota en notas_integracion debe:
  - Advertir sobre una decisión de orden, dependencia o interacción no obvia
    desde el nombre del nodo (información que no se deduce leyendo solo los
    títulos de nodos_hoja).
  - Mencionar al menos un nombre de fichero concreto de nodos_hoja al que
    aplica la advertencia.
  - No duplicar información ya contenida en el campo razon de analisis_cohesion
    ni en el campo tratamiento de grupo2.

  Fallo A — nota vaga: no menciona ningún fichero concreto o usa solo
    referencias genéricas del tipo "ver módulo X" sin nombre de fichero.
  Fallo B — nota duplicada: su contenido ya está cubierto por el campo
    razon de algún candidato en analisis_cohesion o por tratamiento en grupo2.
  Fallo C — nota ausente obligatoria: existe alguna de estas situaciones
    sin nota correspondiente:
      · Un nodo es producto de un split (concepto_padre presente) y no hay
        nota que explique la agrupación en el índice.
      · Dos nodos tienen dependencia de lectura no evidente por su posición
        (el segundo asume conceptos del primero sin ser consecutivos).
      · Un nodo de grupo2 requiere configuración previa en otro módulo del
        CONTEXTO para funcionar y esa dependencia no está advertida.
  → Anota notas que fallen A o B, y situaciones que activen Fallo C sin nota.

---

FASE 2B — REVISIÓN TÉCNICA Y LÓGICA DEL CONTENIDO

Aplica estas verificaciones sobre el fondo del material, no solo su forma.
Lee el campo "Fuente autoritativa → Principal" y "Versión de referencia"
de meta.md antes de comenzar.

T1 — EXACTITUD RESPECTO A LA FUENTE AUTORITATIVA
  Comprueba que los conceptos del mapa_competencias son coherentes con la
  fuente autoritativa del CONTEXTO y con la versión o edición de referencia.
  ¿Hay conceptos que corresponden a versiones, ediciones o períodos
  distintos al indicado en meta.md?
  ¿Hay conceptos que contradicen o no aparecen en la fuente autoritativa?
  → Anota conceptos potencialmente obsoletos, incorrectos o fuera de versión.

T2 — COMPLETITUD DEL CICLO DE CONOCIMIENTO DEL MÓDULO
  El mapa debe cubrir el ciclo completo según el Tipo primario del CONTEXTO.
  Verifica que ninguna fase queda sin representación:

  TECH: desde la primera exposición y configuración básica hasta el uso
  avanzado en situaciones reales (mantenimiento, degradación, recuperación,
  troubleshooting, casos edge).

  CIENCIAS FORMALES/EXPERIMENTALES: desde la motivación y definición formal
  hasta las demostraciones, casos degenerados y aplicaciones prácticas.

  HUMANIDADES/SOCIALES/ARTES: desde el contexto y antecedentes hasta las
  consecuencias, fuentes primarias e interpretaciones académicas.

  ¿Hay fases del ciclo sin representación en el mapa?
  → Anota las fases ausentes con ejemplos concretos de lo que falta.

T3 — COBERTURA DE LÍMITES Y CASOS NO NOMINALES
  El destinatario necesita conocer los límites del módulo y los casos no
  nominales, no solo el uso o interpretación estándar. Según Tipo primario:

  TECH: fallos frecuentes, límites de configuración, escenarios de degradación.
  CIENCIAS FORMALES/EXPERIMENTALES: casos degenerados, contraejemplos,
  condiciones de contorno en las que los resultados no se aplican.
  HUMANIDADES/SOCIALES/ARTES: interpretaciones divergentes, debate académico,
  revisiones o corrientes minoritarias significativas.

  ¿El mapa incluye al menos un área dedicada a estos casos no nominales?
  → Anota los casos relevantes que faltan.

T4 — PROGRESIÓN PEDAGÓGICA DE LOS NODOS
  Dentro del módulo, cada nodo debe poder comprenderse con los conocimientos
  que aportan los nodos anteriores del mismo módulo (más los prerrequisitos
  globales del CONTEXTO).
  ¿El orden de nodos_hoja permite aprendizaje progresivo sin saltos?
  ¿Algún nodo asume conocimiento no introducido aún en el módulo?
  → Anota nodos mal ordenados con la posición correcta propuesta.

T5 — CONCEPTOS CRÍTICOS AUSENTES
  Aplica sobre cada área del mapa la pregunta de contexto operativo.
  Lee el Tipo primario y Cert flag de meta.md y selecciona la pregunta:

  Cert flag = sí (cualquier tipo de dominio):
    "¿Qué pregunta del examen o evaluación oficial respondería mal el
     [perfil objetivo] si no conociera este concepto?"

  Cert flag = no — Tipo primario tech:
    "¿Qué competencia del aprendiz quedaría incompleta si no conociera este concepto?"

  Cert flag = no — Tipo primario ciencias formales o experimentales:
    "¿Qué resultado no podría demostrar, derivar o calcular correctamente
     el aprendiz si no conociera este concepto?"

  Cert flag = no — Tipo primario humanidades, sociales o artes:
    "¿Qué análisis o argumento formularía incorrectamente el aprendiz
     si no conociera este concepto?"

  Si la respuesta es "ningún impacto concreto" → el concepto sobra; anotarlo.
  Si hay áreas donde la respuesta identifica un impacto pero no hay concepto
  que lo cubra → el concepto falta; anotarlo.
  → Lista de conceptos a eliminar y conceptos ausentes a añadir.

---

FASE 3 — PLAN DE CORRECCIÓN

IMPORTANTE: antes de editar cualquier fichero, razona y escribe aquí
el contenido definitivo de TODAS las secciones que necesitan corrección
o creación. No hagas ninguna edición hasta tener el plan completo.

Para cada sección con problemas produce:

  SECCIÓN: [nombre_sección]
  ACCIÓN: [crear | corregir | ampliar]
  CONTENIDO COMPLETO:
  [bloque JSON exacto listo para insertar, sin truncar]

Orden obligatorio de razonamiento:
  1. mapa_competencias (si tiene conceptos genéricos: reformular)
  2. grupo1 / grupo2 (si hay conceptos mal clasificados: mover)
  3. analisis_cohesion (derivar de grupo1: test de tarea + sostenibilidad)
  4. nodos_hoja (derivar de analisis_cohesion: solo candidatos nodo_valido)
  5. notas_integracion (derivar de nodos_hoja: advertencias de orden y dependencias)

Dependencias entre secciones que debes respetar:
  analisis_cohesion depende de grupo1
  nodos_hoja        depende de los resultados de analisis_cohesion
  notas_integracion depende de los nombres de fichero en nodos_hoja

---

FASE 4 — APLICACIÓN DE CORRECCIONES

Aplica las correcciones del plan una sección a la vez, en este orden:

  Paso 1 → corregir mapa_competencias (si procede)
  Paso 2 → corregir grupo1 y/o grupo2 (si procede)
  Paso 3 → insertar o corregir analisis_cohesion
  Paso 4 → insertar o corregir nodos_hoja
  Paso 5 → insertar o corregir notas_integracion

REGLAS DE EDICIÓN PARA EVITAR ERRORES DE API:

  R1. Usa la herramienta Edit para cada paso por separado.
      Nunca combines dos secciones en una sola operación Edit.

  R2. Ninguna operación Edit debe superar 80 líneas de contenido nuevo.
      Si una sección (ej: analisis_cohesion con 10+ candidatos) supera
      ese límite, divídela en dos Edits consecutivos:
        - Edit A: candidatos 1 a 5 (apertura del array incluida)
        - Edit B: candidatos 6 en adelante (cierre del array incluido)

  R3. Confirma que cada Edit se aplicó correctamente antes de continuar
      con el siguiente paso.

  R4. Si el fichero requiere añadir secciones al final, la edición debe
      reemplazar el cierre del último array existente `\n}` por el nuevo
      contenido + cierre `\n}` para mantener el JSON válido.

---

FORMATO DEL INFORME FINAL

Al terminar todas las correcciones, emite este resumen:

  Módulo revisado : [nombre del módulo]
  Fichero         : [nombre del fichero dominio-*.json]

  Secciones encontradas antes de la revisión:
    [lista con OK / FALTA por sección]

  Defectos de calidad encontrados:
    [lista numerada; "Ninguno" si el fichero era correcto]

  Correcciones aplicadas:
    [lista numerada de cambios realizados; "Ninguna" si no hubo cambios]

  Estado final: [CORRECTO | CORREGIDO | REQUIERE REVISIÓN MANUAL]

---

POSICIÓN EN LA CADENA DE PROMPTS

  prompt-dominio.md       →  dominio-[módulo].json  (generación)
  prompt-dominio-check.md →  dominio-[módulo].json  (auditoría y corrección)
  prompt-ruta.md          →  ruta.json              (consume los dominio-*.json)

Ejecutar prompt-dominio-check.md sobre cada dominio-*.json antes de
alimentar prompt-ruta.md garantiza que la ruta se construye sobre
análisis de dominio completo y coherente.

---

Cómo usarlo: pega meta.md en CONTEXTO, pega el fichero dominio-*.json a revisar
en FICHERO A REVISAR y ejecuta. El modelo audita, planifica todas las correcciones
en Fase 3 y las aplica en Fase 4 sección a sección. Ejecutar antes de prompt-ruta.md.
