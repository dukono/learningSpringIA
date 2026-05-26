PROMPT 1 — GENERADOR DE RUTA DE APRENDIZAJE

Actúas como arquitecto de aprendizaje. Tu única tarea es consolidar los nodos hoja
de todos los módulos analizados, construir el grafo de prerequisitos completo e
insertar las tareas integradoras en los puntos de cohesión correctos.

El output (ruta.json) es la estructura que el pipeline de generación (Spring Batch)
usa para procesar los bloques en el orden correcto y que el frontend usa para
renderizar el mapa visual de la ruta del aprendiz.

---

CONTEXTO

Pega aquí meta.md seguido de todos los ficheros dominio-[módulo].json
validados por prompt-dominio-check.md, uno por módulo.

Campos que se leen de meta.md:

  Campo                                Usado en
  ─────────────────────────────────────────────────────────────────────────
  Identificación → Tema/Versión        identificación de la ruta
  Perfil del aprendiz → Nivel actual   criterio para inferir prerequisitos implícitos
  Mapa de competencias                 módulos y competencias esperadas — verifica cobertura
  Estrategia de progresión             orden entre módulos y puntos de síntesis para macros

De cada dominio-[módulo].json se leen únicamente:

  nodos_hoja           bloques con prerequisitos[] y desbloquea[] provisionales
  analisis_cohesion    cohesión semántica entre candidatos (detecta puntos de integración micro)
  notas_integracion    dependencias de orden no obvias entre nodos

<< PEGA AQUÍ meta.md >>
<< PEGA AQUÍ dominio-[módulo-1].json >>
<< PEGA AQUÍ dominio-[módulo-2].json >>
... (uno por módulo)

PROTOCOLO DE LECTURA (obligatorio antes de Fase 1)

Para cada fichero dominio-[módulo].json:
  - Lee ÚNICAMENTE nodos_hoja, analisis_cohesion y notas_integracion.
  - No leas mapa_competencias, grupo1 ni grupo2 — son grandes y no son necesarias aquí.
  - Si un módulo del Mapa de competencias no tiene su dominio-*.json, márcalo como
    PENDIENTE: sus bloques no aparecen en ruta.json hasta que se genere y valide.

---

FASE 1 — CONSOLIDACIÓN Y ASIGNACIÓN DE IDs

Lee los nodos_hoja de todos los módulos en el orden de la Estrategia de progresión
de meta.md. Asigna IDs definitivos:

  Bloques:              B001, B002, B003... (secuencia global continua)
  Tareas integradoras:  T01, T02...        (se determinan en Fase 3)

Para cada bloque, registra:
  id           → "B00N" asignado aquí
  titulo       → título del nodo hoja
  modulo       → módulo al que pertenece
  prerequisitos → lista de IDs del dominio-*.json (números provisionales "X.N")
  desbloquea   → lista de IDs del dominio-*.json

Construye la tabla de traducción provisional → definitivo antes de avanzar:

  provisional "X.N"  →  definitivo "B00N"

Esta tabla es necesaria para la Fase 2. Sin ella los prerequisitos estarán rotos.

Verificación de completitud:
  Cada nodo hoja de cada dominio-*.json tiene exactamente un ID asignado.
  Si algún nodo_hoja queda sin ID, asignarlo antes de continuar.

---

FASE 2 — VALIDACIÓN Y COMPLETADO DEL GRAFO

Los prerequisitos vienen de dos fuentes:
  a) Intra-módulo: prerequisitos[] declarados en dominio-*.json
  b) Inter-módulo: inferidos de la Estrategia de progresión de meta.md

Para cada bloque, en este orden:

  PASO 2A — Traducir IDs: reemplaza los números provisionales "X.N" por los IDs
    definitivos "B00N" usando la tabla de la Fase 1.

  PASO 2B — Completar prerequisitos intra-módulo:
    Si el bloque N asume conocimiento del bloque N-1 del mismo módulo y ese
    prerequisito no está declarado, añadirlo.

  PASO 2C — Añadir prerequisitos inter-módulo:
    Si el primer bloque de un módulo secundario asume conceptos del módulo que
    lo precede según la Estrategia de progresión, el último bloque del módulo
    previo es prerequisito. Añadirlo si no está declarado.

  PASO 2D — Verificar inversas:
    Para cada par (A → prerequisito de B), confirmar que B está en desbloquea de A.
    Corregir cualquier asimetría.

  PASO 2E — Verificar ausencia de ciclos:
    Recorre el grafo en orden topológico. Si detectas un ciclo (A requiere B que
    requiere A), es un error de diseño en el dominio-*.json — reportarlo explícitamente
    con los IDs involucrados antes de continuar.

  PASO 2F — Verificar nodos raíz:
    Al menos un bloque debe tener prerequisitos vacíos (es el punto de entrada).
    Si ningún bloque tiene prerequisitos vacíos, hay un ciclo no detectado.

---

FASE 3 — INSERCIÓN DE TAREAS INTEGRADORAS

Las tareas integradoras no enseñan conceptos nuevos. Se insertan donde el aprendiz
tiene bloques suficientes para construir algo funcional y con sentido por sí solo.

NIVEL MICRO — cada 2-3 bloques con cohesión semántica real

  Criterio de inserción:
    Cuando 2-3 bloques consecutivos pertenecen al mismo candidato en analisis_cohesion
    o comparten área temática en grupo1 del mismo módulo.
    El aprendiz puede combinarlos en un ejercicio que produce algo real.

  Cuándo NO insertar:
    Si los bloques no tienen cohesión suficiente para un ejercicio combinado,
    no insertar. La inserción mecánica cada N bloques sin cohesión no aporta.

  Posición: inmediatamente después del último bloque del grupo.
  Prerequisitos: todos los bloques del grupo que combina.

NIVEL MESO — al finalizar cada módulo

  Criterio de inserción:
    Al final de cada módulo con 4 o más bloques.
    El aprendiz integra todo lo aprendido en el módulo en un mini-proyecto.

  Posición: último nodo del módulo, antes del primer bloque del siguiente módulo.
  Prerequisitos: todos los bloques del módulo.

NIVEL MACRO — en puntos de síntesis entre módulos

  Criterio de inserción:
    Cuando la Estrategia de progresión de meta.md identifica un punto de síntesis
    entre módulos relacionados (ej: "al terminar módulos A y B el aprendiz puede
    construir X completo"). El aprendiz combina competencias de ambos módulos.

  Posición: después del último bloque del último módulo que combina.
  Prerequisitos: tareas MESO de los módulos que combina (no todos los bloques individuales).

---

FASE 4 — GENERACIÓN DE ruta.json

Produce el fichero ruta.json con la siguiente estructura:

───────────────────────────────────────────────────────────────────────────────
{
  "tema":         "[campo Tema de meta.md]",
  "version":      "[campo Versión de referencia de meta.md]",
  "nivel_actual": "[campo Nivel actual de meta.md]",
  "objetivo":     "[campo Objetivo de meta.md]",

  "nodos": [
    {
      "id":              "B001",
      "tipo":            "bloque",
      "titulo":          "[Título del bloque]",
      "modulo":          "[Nombre del módulo]",
      "orden_sugerido":  1,
      "prerequisitos":   [],
      "desbloquea":      ["B002"]
    },
    {
      "id":              "T01",
      "tipo":            "integracion",
      "nivel":           "micro",
      "titulo":          "[Título descriptivo — qué construye el aprendiz]",
      "combina":         ["B001", "B002", "B003"],
      "orden_sugerido":  4,
      "prerequisitos":   ["B001", "B002", "B003"],
      "desbloquea":      ["B004"]
    }
  ],

  "estadisticas": {
    "total_bloques":               0,
    "total_integraciones_micro":   0,
    "total_integraciones_meso":    0,
    "total_integraciones_macro":   0,
    "modulos": [
      { "nombre": "[Módulo]", "bloques": 0, "integraciones": 0 }
    ]
  }
}
───────────────────────────────────────────────────────────────────────────────

NOTA sobre orden_sugerido:
  Es la secuencia lineal recomendada para un aprendiz que sigue la ruta en orden.
  No es obligatorio — el aprendiz puede seguir cualquier camino que respete los
  prerequisitos. Sirve para el primer render del mapa visual y para que Spring Batch
  priorice el orden de generación de bloques.

---

PROTOCOLO DE ESCRITURA AUTÓNOMA

El JSON puede ser extenso. Escríbelo por secciones usando Bash + Edit.

PASO R0 — Bash: crea el skeleton vacío de ruta.json:

  cat > ruta.json << 'ENDJSON'
  {
    "tema": "",
    "version": "",
    "nivel_actual": "",
    "objetivo": "",
    "nodos": [],
    "estadisticas": {}
  }
  ENDJSON

PASO R1 — Edit: escribe la cabecera (tema, version, nivel_actual, objetivo).

PASO R2..RN — Edit por módulo: añade los bloques e integraciones de cada módulo
  al array "nodos". Máximo 80 líneas de contenido nuevo por Edit.
  Si un módulo supera ese límite, divide en dos Edits consecutivos.

PASO RF — Edit: escribe las estadísticas finales.

VERIFICACIÓN POST-EDIT (obligatoria tras cada Edit):
  Lee ruta.json y confirma que la sección recién escrita existe y es JSON válido.
  Confirma que prerequisitos y desbloquea son simétricos en los nodos escritos.
  Si hay error, corrígelo antes de continuar con el siguiente módulo.

---

REGLAS DE CALIDAD

Q1. Todo nodo hoja de todos los dominio-*.json aparece en ruta.json con ID asignado.
    Ningún bloque puede quedar sin representación.

Q2. Los arrays prerequisitos y desbloquea son la inversa exacta entre sí en todo el grafo.
    Si B002.prerequisitos contiene "B001", entonces B001.desbloquea contiene "B002".
    Sin excepciones.

Q3. No hay ciclos. Verificar con recorrido topológico en Fase 2 antes de emitir.

Q4. Las tareas integradoras micro se insertan donde hay cohesión semántica real,
    no mecánicamente cada N bloques. Si los bloques del tramo no tienen cohesión
    suficiente para un ejercicio combinado, no se inserta.

Q5. orden_sugerido es una secuencia continua de 1 a N sin huecos ni repeticiones.

Q6. Los prerequisitos de una tarea integradora micro son los bloques que combina.
    Los prerequisitos de una tarea meso son todos los bloques del módulo.
    Los prerequisitos de una tarea macro son las tareas meso de los módulos que combina.

---

POSICIÓN EN LA CADENA

  premeta.md
      ↓
  prompt-meta.md          →  meta.md
      ↓
  prompt-dominio.md       →  dominio-[módulo].json  (uno por módulo)
      ↓
  prompt-dominio-check.md →  dominio-[módulo].json  (validado)
      ↓
  prompt-ruta.md          →  ruta.json              ← ESTE PROMPT
      ↓
  prompt-section.md       →  bloque generado        (uno por nodo de ruta.json)

---

Cómo usarlo: pega meta.md seguido de todos los dominio-*.json validados en CONTEXTO
y ejecuta. El modelo consolida los nodos, construye y valida el grafo, inserta las
tareas integradoras y escribe ruta.json en disco.
