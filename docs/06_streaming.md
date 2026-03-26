<!-- navegación -->
> **[← Inicio](00_indice.md)**

---

## 10. Streaming — respuesta en tiempo real

El **streaming** es como funciona ChatGPT: la respuesta llega token a token en lugar
de esperar a que termine toda la generación.

### Dependencia necesaria

El streaming en Spring AI usa **Project Reactor** (WebFlux). Asegúrate de tenerlo:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### Implementación

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String pregunta) {
    return chatClient.prompt()
        .user(pregunta)
        .stream()                    // en lugar de .call()
        .content();                  // Flux<String> — cada elemento = un token
}
```

### Visualización del streaming

```
Sin streaming (bloqueante):
  Petición → [esperas 3-10 segundos en silencio] → "Aquí tienes la respuesta completa..."

Con streaming (reactivo):
  Petición → "Aquí" → " tienes" → " la" → " respuesta" → " completa..."
              ↑ cada token llega en milisegundos → el usuario ve texto aparecer
```

### Manejo de errores en streaming

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String pregunta) {
    return chatClient.prompt()
        .user(pregunta)
        .stream()
        .content()
        .onErrorResume(e -> {
            // Si la API falla a mitad del stream
            log.error("Error en streaming: {}", e.getMessage());
            return Flux.just("[Error: no se pudo completar la respuesta]");
        })
        .doOnComplete(() -> log.info("Stream completado"));
}
```

### Cliente JavaScript para consumir el stream (SSE)

```javascript
// Server-Sent Events — el navegador recibe los tokens en tiempo real
const eventSource = new EventSource('/chat/stream?pregunta=Explicame%20Java');
const div = document.getElementById('respuesta');

eventSource.onmessage = (event) => {
    div.textContent += event.data;  // cada evento = un token
};

eventSource.onerror = () => {
    console.log('Stream terminado o error');
    eventSource.close();
};
```

---


---

> **[← Volver al índice](00_indice.md)**
