<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 9. Structured Output — respuestas tipadas

En lugar de recibir texto y parsear JSON manualmente, Spring AI puede convertir
automáticamente la respuesta en un objeto Java.

### 9.1 Con Records (recomendado en Java 21+)

```java
// Define el record que quieres recibir
record AnalisisProducto(
    String sentimiento,           // "positivo", "negativo", "neutro"
    List<String> puntosPositivos,
    List<String> puntosNegativos,
    int puntuacion                // del 1 al 10
) {}

// Uso con .entity()
@Service
public class ReviewService {

    public AnalisisProducto analizar(String review) {
        return chatClient.prompt()
            .user("Analiza esta review: " + review)
            .call()
            .entity(AnalisisProducto.class);  // conversión automática
    }
}
```

```
┌───────────────────────────────────────────────────────────────────┐
│  LO QUE HACE Spring AI internamente con .entity():                │
│                                                                   │
│  1. Lee los campos de AnalisisProducto.class via reflexión        │
│  2. Genera instrucciones JSON automáticamente en el prompt:       │
│     "Responde SOLO con JSON con estos campos:                     │
│      sentimiento (String), puntosPositivos (array)..."            │
│  3. El modelo responde en JSON                                    │
│  4. Spring AI deserializa con Jackson a AnalisisProducto          │
│  5. Tú recibes un objeto Java tipado listo para usar              │
│                                                                   │
│  Si el modelo no devuelve JSON válido → lanza excepción           │
│  (poco frecuente con GPT-4o, más frecuente con modelos pequeños)  │
└───────────────────────────────────────────────────────────────────┘
```

### 9.2 Con listas

```java
record Receta(String nombre, List<String> ingredientes, int minutosPreparacion) {}

List<Receta> recetas = chatClient.prompt()
    .user("Dame 3 recetas fáciles con pollo")
    .call()
    .entity(new ParameterizedTypeReference<List<Receta>>() {});

// recetas.get(0).nombre()       → "Pollo al ajillo"
// recetas.get(0).ingredientes() → ["pollo", "ajo", "aceite", "perejil"]
// recetas.get(0).minutosPreparacion() → 25
```

### 9.3 Con Map para datos libres

```java
Map<String, Object> datos = chatClient.prompt()
    .user("Dame datos sobre España en formato clave-valor")
    .call()
    .entity(new ParameterizedTypeReference<Map<String, Object>>() {});

// datos = {"capital": "Madrid", "poblacion": 47000000, "idioma": "español", ...}
String capital = (String) datos.get("capital");  // "Madrid"
```

---


---

> **[← Volver al índice](00_indice.md)**
