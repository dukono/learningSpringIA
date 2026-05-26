PROMPT VISUAL — EVALUADOR DE BLOQUES Y GENERADOR DE DIAGRAMAS

Este prompt se ejecuta DESPUÉS de que el contenido de un bloque ha sido generado
completamente por prompt-section.md. Su única tarea es evaluar el campo
contenido.concepto del bloque e incorporar diagramas Mermaid donde mejoren
la comprensión del aprendiz.

En la aplicación (Spring Batch), corresponde a un ItemProcessor posterior al
BlockGenerator que actualiza el campo contenido.concepto en MongoDB.
En el pipeline manual, opera directamente sobre el texto del bloque.

No genera ni modifica texto, código, tablas ni ejemplos. Solo añade diagramas.

---

ENTRADA

Especifica uno de los dos modos:

──────────────────────────────────────────────
MODO BLOQUE  (evalúa un único bloque)
──────────────────────────────────────────────

BLOQUE_ID = [ID del bloque; ej: B012]
CONTENIDO = [pega aquí el valor del campo contenido.concepto del bloque]

──────────────────────────────────────────────
MODO MÓDULO  (evalúa todos los bloques de un módulo)
──────────────────────────────────────────────

MÓDULO = [nombre del módulo]

Al recibir MÓDULO, ejecuta este protocolo:

  PASO M1 — Obtener los bloques del módulo:
    Lee ruta.json y extrae todos los nodos con modulo == MÓDULO y tipo == "bloque".
    Descarta tareas integradoras — su contenido no tiene el campo contenido.concepto.

  PASO M2 — Procesar cada bloque de forma independiente:
    Por cada bloque, ejecuta el PROTOCOLO DE EVALUACIÓN completo sobre su campo
    contenido.concepto. El orden no importa — cada bloque es independiente.

  PASO M3 — Resumen al terminar:
    Lista los bloques procesados, cuántos diagramas se añadieron en cada uno
    y qué tipos se usaron.

---

PROTOCOLO DE EVALUACIÓN

PASO 1 — Leer el campo contenido.concepto del bloque completo.

PASO 2 — Recorrer el contenido concepto a concepto y aplicar este criterio:

  Para cada concepto, pregunta:
  "¿El texto, el código o la tabla que ya existen transmiten esto con claridad
   al lector objetivo, o hay algo que solo se entiende bien visualmente?"

  Si la respuesta es NO → genera un diagrama Mermaid justo debajo del concepto.
  Si la respuesta es SÍ → no se añade nada.

  Un bloque ASCII existente que represente el mismo concepto se SUSTITUYE
  por el Mermaid equivalente. No coexisten.

---

PASO 3 — SELECCIÓN DE TIPO

Elige el tipo que mejor expresa la naturaleza del concepto.
Evita usar siempre flowchart — la variedad de tipos es lo que hace la documentación
visualmente rica y no monótona.

  CONCEPTO                                      TIPO
  ────────────────────────────────────────────  ──────────────────────────────
  Árbol de propiedades YAML o jerarquía plana   mindmap
  Flujo con decisiones (sí/no, if/else)         flowchart TD  con nodos diamante
  Pipeline lineal de transformación             flowchart LR  con nodos variados
  Interacción entre actores o sistemas          sequenceDiagram
  Máquina de estados o ciclo de vida            stateDiagram-v2
  Fases o momentos temporales en orden          timeline
  Comparativa en dos ejes independientes        quadrantChart
  Métricas, series de datos, conteos            xychart-beta
  Relaciones entre entidades con atributos      erDiagram
  Agrupación arquitectónica de componentes      flowchart con subgraph

  NUNCA usar (no renderizan en GitHub):
  sankey, block, treemap, kanban, architecture, radar, venn, packet

---

PASO 4 — GUÍA DE USO MODERNO POR TIPO

El objetivo es variedad visual y claridad. Cada tipo tiene sus propias
convenciones modernas — seguirlas produce diagramas limpios y diferentes entre sí.

──────────────────────────────────────────────────────────────────────────────
FLOWCHART — usa formas de nodo con semántica, no solo rectángulos
──────────────────────────────────────────────────────────────────────────────

  Formas disponibles y su uso semántico:
    [texto]       rectángulo         — proceso, acción, componente
    (texto)       rectángulo redondeado — estado, resultado
    {texto}       diamante           — decisión, condición if/else
    ((texto))     círculo            — inicio / fin / evento
    [(texto)]     cilindro           — almacenamiento, fichero, base de datos
    {{texto}}     hexágono           — configuración, trigger, dispatcher
    [/texto/]     paralelogramo      — entrada / salida de datos
    >texto]       flecha asimétrica  — anotación, advertencia lateral

  Usa subgraph para agrupar componentes relacionados:
    subgraph "Nombre del grupo"
      direction LR
      nodo1 --> nodo2
    end

  Paleta de colores (classDef al final del bloque):
    classDef root      fill:#1f2328,color:#fff,stroke:#444,font-weight:bold
    classDef primary   fill:#0969da,color:#fff,stroke:#0550ae
    classDef secondary fill:#2da44e,color:#fff,stroke:#1a7f37
    classDef danger    fill:#cf222e,color:#fff,stroke:#a40e26
    classDef neutral   fill:#e6edf3,color:#1f2328,stroke:#d0d7de
    classDef warning   fill:#9a6700,color:#fff,stroke:#7d4e00
    classDef storage   fill:#6e40c9,color:#fff,stroke:#5a32a3

  Semántica de colores:
    root      — nodo central o raíz
    primary   — flujo principal, elementos obligatorios
    secondary — resultado exitoso, disponible, activo
    danger    — error, fallo, riesgo de seguridad, cancelado
    neutral   — sub-propiedades, hojas, opcionales
    warning   — advertencia, excepción, comportamiento especial
    storage   — ficheros, almacenamiento, artefactos

  Ejemplo moderno — árbol de decisión con formas mixtas:
    flowchart TD
        EVT(("evento push"))
        CHK{{"¿coincide branch?"}}
        CHK2{{"¿coincide path?"}}
        RUN["ejecuta workflow"]
        SKIP["no se activa"]

        EVT --> CHK
        CHK -->|sí| CHK2
        CHK -->|no| SKIP
        CHK2 -->|sí| RUN
        CHK2 -->|no| SKIP

        classDef root fill:#1f2328,color:#fff,stroke:#444,font-weight:bold
        ...
        class EVT root
        class CHK,CHK2 warning
        class RUN secondary
        class SKIP danger

──────────────────────────────────────────────────────────────────────────────
MINDMAP — para jerarquías de propiedades, no flujos
──────────────────────────────────────────────────────────────────────────────

  Formas de nodo en mindmap:
    texto sin corchetes   — nodo hoja por defecto
    (texto)               — redondeado
    ((texto))             — círculo
    [texto]               — rectángulo
    ))texto((             — nube
    )texto(               — bang / explosión

  El nodo raíz (primer nivel, sin indentación) define la forma del centro.
  Los niveles de indentación crean la jerarquía automáticamente.

  No uses classDef — mindmap no lo soporta. La variedad visual viene
  de las formas de nodo y la estructura de indentación.

  Ejemplo moderno:
    mindmap
      root((workflow.yml))
        (on)
          push
          pull_request
          workflow_dispatch
        (jobs)
          job-1
          job-2
        [env]
        [defaults]
          run
            shell
            working-directory
        [concurrency]
          group
          cancel-in-progress

──────────────────────────────────────────────────────────────────────────────
SEQUENCEDIAGRAM — para interacciones entre actores
──────────────────────────────────────────────────────────────────────────────

  Usa los bloques de agrupación para añadir contexto:
    rect rgb(0, 80, 160)     ... end    — agrupa mensajes relacionados
    loop Cada 15s            ... end    — bucle periódico
    alt condición            ... end    — rama condicional
    else otra condición      ... end
    par Proceso A            ... end    — procesos en paralelo
    and Proceso B            ... end
    critical sección crítica ... end
    break condición de rotura... end

  Tipos de flecha:
    Actor->>Otro: mensaje          — flecha sólida (acción)
    Actor-->>Otro: respuesta       — flecha punteada (respuesta)
    Actor-xOtro: fallo             — flecha con X (error)
    Actor-)Otro: async             — flecha abierta (asíncrono)

  Note over Actor: texto           — anotación sobre un actor
  Note right of Actor: texto       — anotación lateral

──────────────────────────────────────────────────────────────────────────────
STATEDIAGRAM-V2 — para ciclos de vida y máquinas de estado
──────────────────────────────────────────────────────────────────────────────

  [*] es el estado inicial y final.
  Usa note para añadir condiciones o aclaraciones en un estado.
  Usa state "nombre" as alias para estados con texto largo.
  Agrupa estados relacionados con state "grupo" { ... }.

  Ejemplo con agrupación:
    stateDiagram-v2
      [*] --> Pendiente
      state "En ejecución" as running {
        [*] --> Iniciando
        Iniciando --> Ejecutando
      }
      Pendiente --> running
      running --> Exitoso
      running --> Fallido
      Exitoso --> [*]
      Fallido --> [*]

──────────────────────────────────────────────────────────────────────────────
TIMELINE — para fases temporales o momentos de disponibilidad
──────────────────────────────────────────────────────────────────────────────

  Usa section para agrupar eventos del mismo periodo.
  Los elementos dentro de cada section son los eventos de ese momento.
  No requiere fechas — cualquier texto sirve como etiqueta temporal.

  Ejemplo:
    timeline
      title Alcance de file commands
      section Step actual
        GITHUB_ENV    : escribe variable
        GITHUB_OUTPUT : escribe output
      section Steps siguientes
        GITHUB_ENV    : variable disponible
      section Jobs posteriores
        GITHUB_OUTPUT : accesible via needs
      section UI de GitHub
        GITHUB_STEP_SUMMARY : visible en Summary

──────────────────────────────────────────────────────────────────────────────
QUADRANTCHART — para comparativas en dos dimensiones
──────────────────────────────────────────────────────────────────────────────

  Define ejes con etiquetas descriptivas en los extremos.
  Los puntos van entre 0 y 1 en cada eje: [x, y].
  Nombra los cuadrantes para orientar la lectura.

  Ejemplo:
    quadrantChart
      title Triggers de código: origen vs acceso
      x-axis "Codigo fork" --> "Codigo base"
      y-axis "Sin secrets" --> "Secrets disponibles"
      quadrant-1 Seguro
      quadrant-2 Riesgo critico
      quadrant-3 Fork aislado
      quadrant-4 Base sin privilegios
      push: [0.85, 0.85]
      pull_request: [0.15, 0.15]
      pull_request_target: [0.85, 0.85]

──────────────────────────────────────────────────────────────────────────────
XYCHART-BETA — para métricas, conteos, comparativas numéricas
──────────────────────────────────────────────────────────────────────────────

  Usa bar para comparativas estáticas, line para tendencias.
  El título y los ejes son opcionales pero mejoran la lectura.

  Ejemplo:
    xychart-beta
      title "Jobs generados por matrix"
      x-axis ["ubuntu", "windows", "macos"]
      y-axis "jobs" 0 --> 9
      bar [3, 3, 3]

---

RESTRICCIONES

V1. El diagrama va siempre DESPUÉS de un párrafo de texto, nunca al inicio de sección.

V2. El diagrama representa una relación, flujo o estructura real del contenido.
    No es decoración — añade algo que el texto no transmite solo.

V3. Cada concepto distinto puede tener su propio diagrama.
    Un mismo concepto no tiene dos diagramas.

V4. No usar el mismo tipo de diagrama para todos los conceptos del fichero.
    La variedad de tipos es intencional y refleja la naturaleza de cada concepto.

V5. El estilo es consistente dentro del mismo tipo:
    todos los flowchart del fichero usan la misma paleta classDef.

V6. Después de cada bloque Mermaid añade una línea de caption en itálica
    cuando el diagrama está separado de su encabezado de sección o cuando
    la relación que muestra no es evidente desde el párrafo anterior:

        ```mermaid
        flowchart TD
          ...
        ```
        *Descripción breve de lo que muestra el diagrama.*

    Formato: itálica markdown (`*texto*`), una línea.
    No es un resumen del texto — describe la RELACIÓN o ESTRUCTURA visible
    en el diagrama, no lo que ya dice el párrafo.

    Correcto:
      *Flujo de evaluación de un step: de if: a conclusion.*
      *Precedencia de vars — organización → repositorio → environment.*
      *Ciclo de vida de una release desde borrador hasta publicación estable.*

    Incorrecto:
      *Los workflows pueden activarse de cuatro formas.*   ← ya lo dice el texto
      *Diagrama de flujo.*                                 ← no describe nada

---

PROTOCOLO DE ESCRITURA

El output de este prompt es el campo contenido.concepto actualizado con los
diagramas Mermaid insertados en los puntos correctos.

En el pipeline manual: el resultado se usa para actualizar el bloque en el
almacenamiento. En la aplicación (Spring Batch): el DiagramEnricher actualiza
el campo directamente en MongoDB.

REGLAS OBLIGATORIAS:

R1. Planifica TODOS los diagramas del bloque antes de insertar el primero.
    No intercales planificación y escritura.

R2. Inserta los diagramas de arriba a abajo, en orden de aparición en el contenido.

R3. Nunca más de un diagrama por concepto.
    Si un concepto podría beneficiarse de dos tipos distintos, elige el que
    mejor expresa la relación principal — no añadas ambos.

R4. En MODO MÓDULO: completa todos los diagramas de un bloque antes de pasar
    al siguiente. No intercales bloques.

---

Cómo usarlo: indica FICHERO con la ruta del .md a evaluar. El prompt lee el contenido,
identifica dónde un diagrama Mermaid mejora la comprensión, genera el tipo más adecuado
con uso moderno de sus características, y lo inserta sin modificar el contenido existente.
