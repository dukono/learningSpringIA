<!-- navegación -->
> **[← Advisors](12_advisors.md)** | **[← Inicio](../README.md)** | **[Siguiente: Observabilidad →](14_observabilidad.md)**

---

# Capítulo 13 — Multimodalidad

> Cómo enviar imágenes y audio al modelo, generar imágenes con DALL-E
> y sintetizar voz con Text-to-Speech. Requiere modelos multimodales (GPT-4o, Claude 3, Gemini).

## 13. Multimodalidad — imágenes y audio

### El problema que resuelve

Los LLMs de texto son ciegos y sordos: solo procesan palabras. Pero muchos casos de
uso empresariales requieren trabajar con otros tipos de datos:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SIN MULTIMODALIDAD — solo texto                                              │
│                                                                              │
│  Usuario: [sube factura.jpg] "¿Cuánto es el total?"                          │
│  Modelo:  ❌ No puede leer la imagen — solo ve texto                          │
│                                                                              │
│  CON MULTIMODALIDAD — varios tipos de entrada                                │
│                                                                              │
│  Usuario: [sube factura.jpg] "¿Cuánto es el total?"                          │
│  GPT-4o:  ✅ "El total de la factura es 1.247,50 € con IVA incluido."        │
└──────────────────────────────────────────────────────────────────────────────┘
```

Spring AI unifica cuatro capacidades multimodales bajo una API común:

| Capacidad | Dirección | API de Spring AI |
|---|---|---|
| **Visión** | Imagen → texto (análisis, descripción, extracción) | `ChatClient` + `Media` |
| **Transcripción** | Audio → texto (Whisper) | `AudioTranscriptionModel` |
| **Generación de imágenes** | Texto → imagen (DALL-E, Stable Diffusion) | `ImageModel` |
| **Text-to-Speech** | Texto → audio | `SpeechModel` |

---

### Cómo funciona internamente

Antes de enviar al modelo, Spring AI **codifica la imagen en Base64** y la incluye
en el cuerpo JSON de la petición junto al texto:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  LO QUE REALMENTE SE ENVÍA A LA API                                          │
│                                                                              │
│  {                                                                           │
│    "model": "gpt-4o",                                                        │
│    "messages": [{                                                            │
│      "role": "user",                                                         │
│      "content": [                                                            │
│        { "type": "text",  "text": "¿Cuál es el total de esta factura?" },   │
│        { "type": "image_url",                                                │
│          "image_url": { "url": "data:image/jpeg;base64,/9j/4AAQ..." } }     │
│      ]                                                                       │
│    }]                                                                        │
│  }                                                                           │
│                                                                              │
│  Spring AI hace esta conversión automáticamente desde byte[]/Resource/URI.  │
└──────────────────────────────────────────────────────────────────────────────┘
```

El modelo procesa texto e imagen juntos como una sola unidad de contexto.
**No hay una API separada para imágenes** — es el mismo endpoint de chat.

---

### ¿Qué modelos soportan qué capacidades?

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SOPORTE DE CAPACIDADES MULTIMODALES POR MODELO                              │
├────────────────────┬──────────┬───────────────┬────────────┬─────────────────┤
│  Modelo            │  Visión  │  Transcripción│  Img. Gen. │  TTS            │
├────────────────────┼──────────┼───────────────┼────────────┼─────────────────┤
│  GPT-4o            │  ✅      │  —            │  —         │  —              │
│  GPT-4-turbo       │  ✅      │  —            │  —         │  —              │
│  GPT-4o-mini       │  ✅      │  —            │  —         │  —              │
│  whisper-1         │  —       │  ✅           │  —         │  —              │
│  dall-e-3          │  —       │  —            │  ✅        │  —              │
│  dall-e-2          │  —       │  —            │  ✅        │  —              │
│  tts-1 / tts-1-hd  │  —       │  —            │  —         │  ✅             │
│  Claude 3 Opus/S/H │  ✅      │  —            │  —         │  —              │
│  Gemini 1.5 Pro    │  ✅      │  ✅           │  —         │  —              │
│  Ollama llava      │  ✅      │  —            │  —         │  —              │
│  Ollama bakllava   │  ✅      │  —            │  —         │  —              │
│  GPT-3.5-turbo     │  ❌      │  —            │  —         │  —              │
│  llama3.2 (texto)  │  ❌      │  —            │  —         │  —              │
└────────────────────┴──────────┴───────────────┴────────────┴─────────────────┘
```

> 🔑 **Regla**: para visión siempre usa un modelo con `-vision` o multimodal explícito.
> `gpt-3.5-turbo` y los modelos Llama de texto puro **no procesan imágenes**.

---

### Límites importantes de las APIs (referencia para producción)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  LÍMITES A CONOCER ANTES DE DESPLEGAR                                        │
├──────────────────────────────────────────────────┬───────────────────────────┤
│  VISIÓN (OpenAI GPT-4o)                          │                           │
│  Tamaño máximo por imagen                        │  20 MB                    │
│  Formatos soportados                             │  JPEG, PNG, GIF, WebP     │
│  Imágenes por petición                           │  Hasta ~10 (recomendado 1-3)│
│  Coste extra por imagen (modo "high")            │  ~510 tokens adicionales   │
├──────────────────────────────────────────────────┼───────────────────────────┤
│  TRANSCRIPCIÓN (Whisper)                         │                           │
│  Tamaño máximo por archivo                       │  25 MB                    │
│  Duración máxima                                 │  Sin límite oficial (25MB) │
│  Formatos soportados                             │  mp3, mp4, wav, m4a, webm │
│  Para archivos > 25MB                            │  Dividir en segmentos     │
├──────────────────────────────────────────────────┼───────────────────────────┤
│  GENERACIÓN DE IMÁGENES (DALL-E 3)               │                           │
│  Tamaños disponibles                             │  1024×1024, 1792×1024,    │
│                                                  │  1024×1792                │
│  Imágenes por petición                           │  Máx. 1 (DALL-E 3)        │
│  Restricciones de contenido                      │  Sin violencia, adulto    │
│                                                  │  ni personajes reales     │
├──────────────────────────────────────────────────┼───────────────────────────┤
│  TEXT-TO-SPEECH (OpenAI)                         │                           │
│  Caracteres máximos por petición                 │  4.096 caracteres         │
│  Para textos más largos                          │  Dividir y concatenar     │
│  Formatos de salida                              │  mp3, opus, aac, flac     │
└──────────────────────────────────────────────────┴───────────────────────────┘
```

---

### 13.1 Visión — análisis de imágenes

### Enviar imagen para análisis

```java
@GetMapping("/analizar-imagen")
public String analizarImagen() throws IOException {
    // Cargar imagen desde archivo
    byte[] imagenBytes = Files.readAllBytes(Path.of("factura.jpg"));

    // Construir mensaje con imagen
    UserMessage mensaje = new UserMessage(
        "¿Cuál es el total de esta factura?",
        List.of(new Media(MimeTypeUtils.IMAGE_JPEG, imagenBytes))
    );

    return chatClient.prompt()
        .messages(mensaje)
        .call()
        .content();
}

@GetMapping("/analizar-url")
public String analizarURL() {
    // También funciona con URL
    return chatClient.prompt()
        .user(u -> u.text("Describe lo que ves en esta imagen")
                    .media(MimeTypeUtils.IMAGE_JPEG,
                           URI.create("https://ejemplo.com/foto.jpg")))
        .call()
        .content();
}
```

### Visión con Ollama (desarrollo local, sin coste)

Para desarrollo local sin coste ni API key, Ollama ofrece modelos de visión:

```bash
# Descargar modelos de visión para Ollama
ollama pull llava          # LLaVA 7B — bueno para uso general (~4GB)
ollama pull llava:13b      # LLaVA 13B — más preciso (~8GB)
ollama pull bakllava       # BakLLaVA — alternativa más ligera (~4GB)
```

```yaml
# application-dev.yml — configuración para visión con Ollama
spring:
  ai:
    ollama:
      chat:
        model: llava        # ← modelo con visión, no llama3.2
```

```java
// El código Java es idéntico al de OpenAI — solo cambia el modelo en la config
public String analizarImagenLocal(byte[] imagenBytes) {
    return chatClient.prompt()
        .user(u -> u.text("Describe lo que ves en esta imagen en detalle.")
                    .media(MimeTypeUtils.IMAGE_JPEG, imagenBytes))
        .call()
        .content();
}
```

> ⚠️ Los modelos de visión de Ollama (llava, bakllava) son menos precisos que
> GPT-4o en tareas complejas (análisis de facturas, lectura de texto, diagramas).
> Úsalos para desarrollo y pruebas; en producción evalúa GPT-4o o Claude 3.

---

### Enviar múltiples imágenes en un prompt

```java
// GPT-4o admite hasta ~10 imágenes por petición
public String compararImagenes(byte[] imagen1, byte[] imagen2) {
    return chatClient.prompt()
        .user(u -> u
            .text("Compara estas dos imágenes y describe las diferencias principales.")
            .media(MimeTypeUtils.IMAGE_JPEG, imagen1)
            .media(MimeTypeUtils.IMAGE_JPEG, imagen2))
        .call()
        .content();
}
```

---

### Transcripción de audio (Speech-to-Text)

```java
@Autowired
AudioTranscriptionModel transcriptionModel;

@PostMapping("/transcribir")
public String transcribir(@RequestParam MultipartFile audio) throws IOException {
    AudioTranscriptionPrompt prompt = new AudioTranscriptionPrompt(
        new ByteArrayResource(audio.getBytes()),
        OpenAiAudioTranscriptionOptions.builder()
            .language("es")
            .model("whisper-1")
            .build()
    );

    return transcriptionModel.call(prompt).getResult().getOutput();
}
```

---

## 13.2 Image Generation — generar imágenes

```java
@Autowired
ImageModel imageModel;

@GetMapping("/generar-imagen")
public String generarImagen(@RequestParam String descripcion) {
    ImagePrompt prompt = new ImagePrompt(
        descripcion,
        OpenAiImageOptions.builder()
            .model("dall-e-3")
            .quality("hd")
            .n(1)                        // número de imágenes
            .height(1024)
            .width(1024)
            .style("vivid")              // "vivid" o "natural"
            .build()
    );

    ImageResponse response = imageModel.call(prompt);
    String imageUrl = response.getResult().getOutput().getUrl();

    return imageUrl;  // URL de la imagen generada
}
```

Ejemplo visual del resultado:

```
Prompt: "Un astronauta montando un caballo en Marte, estilo fotorrealista"
         │
         ▼
   [DALL-E 3 API]
         │
         ▼
   URL: "https://oaidalleapiprodscus.blob.core.windows.net/private/...imagen.png"
```

---




## 13.3 Text-to-Speech (TTS) — texto a voz

Spring AI soporta la generación de audio a partir de texto (**TTS**) mediante `SpeechModel`.
Disponible con OpenAI (`tts-1` y `tts-1-hd`).

### ¿Qué resuelve TTS?

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CASOS DE USO DE TEXT-TO-SPEECH                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│  Asistentes de voz      → respuesta hablada al usuario                       │
│  Accesibilidad          → leer contenido para usuarios con discapacidad      │
│  Bots telefónicos (IVR) → respuestas dinámicas en llamadas                   │
│  Narración de vídeos    → generar voz en off desde guiones                   │
│  Aprendizaje de idiomas → pronunciar vocabulario en la voz nativa            │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Modelos y voces (OpenAI)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MODELOS TTS                                                                 │
├────────────────┬────────────────────────────────────────────────────────────┤
│  tts-1         │  Baja latencia — ideal para tiempo real                    │
│  tts-1-hd      │  Alta calidad — para producción y grabaciones              │
├────────────────┴────────────────────────────────────────────────────────────┤
│  VOCES DISPONIBLES                                                           │
├────────────────┬────────────────────────────────────────────────────────────┤
│  alloy         │  Neutra, equilibrada. Buena para uso general               │
│  echo          │  Masculina, reflexiva. Formal                              │
│  fable         │  Expresiva, cálida. Para narración y storytelling          │
│  onyx          │  Profunda, autoritaria. Para presentaciones                │
│  nova           │  Amigable, energética. Para asistentes                    │
│  shimmer       │  Cálida, suave. Para contenido de bienestar                │
└────────────────┴────────────────────────────────────────────────────────────┘
```

### Dependencia

No necesitas dependencia adicional — viene incluida en el starter de OpenAI:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

### Implementación básica

```java
@Service
public class VoiceService {

    @Autowired
    SpeechModel speechModel;

    public byte[] textToSpeech(String texto) {
        SpeechPrompt prompt = new SpeechPrompt(
            texto,
            OpenAiAudioSpeechOptions.builder()
                .model("tts-1")             // tts-1 = rápido, tts-1-hd = calidad
                .voice(OpenAiAudioApi.SpeechRequest.Voice.NOVA)
                .responseFormat(
                    OpenAiAudioApi.SpeechRequest.AudioResponseFormat.MP3)
                .speed(1.0f)                // 0.25 (muy lento) a 4.0 (muy rápido)
                .build()
        );
        return speechModel.call(prompt).getResult().getOutput(); // byte[] con MP3
    }
}
```

### Endpoint REST que devuelve audio

```java
// Devuelve el MP3 directamente — el navegador/cliente lo reproduce
@GetMapping(value = "/audio/hablar", produces = "audio/mpeg")
public ResponseEntity<byte[]> hablar(@RequestParam String texto) {
    byte[] audio = voiceService.textToSpeech(texto);
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION,
                "inline; filename=\"respuesta.mp3\"")
        .body(audio);
}
```

### Asistente con respuesta hablada (chat + TTS combinados)

```java
@PostMapping(value = "/asistente/voz", produces = "audio/mpeg")
public ResponseEntity<byte[]> responderConVoz(@RequestBody String pregunta) {

    // Paso 1: LLM genera la respuesta en texto
    // IMPORTANTE: pedir respuesta concisa — TTS cobra por carácter
    String respuestaTexto = chatClient.prompt()
        .system("Eres un asistente de voz. Responde en 2-3 frases cortas. " +
                "Sin markdown ni asteriscos — solo texto plano.")
        .user(pregunta)
        .call()
        .content();

    // Paso 2: Convertir respuesta a audio
    byte[] audio = speechModel.call(new SpeechPrompt(
        respuestaTexto,
        OpenAiAudioSpeechOptions.builder()
            .model("tts-1-hd")
            .voice(OpenAiAudioApi.SpeechRequest.Voice.NOVA)
            .build()
    )).getResult().getOutput();

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"respuesta.mp3\"")
        .body(audio);
}
```

```
Flujo completo:
  POST /asistente/voz  {"pregunta de texto"}
         │
         ▼ Paso 1
  [ChatClient → LLM → respuesta en texto]  (ej: "Madrid está soleado hoy.")
         │
         ▼ Paso 2
  [SpeechModel → OpenAI TTS → tts-1-hd → voz NOVA]
         │
         ▼
  byte[] MP3 → HTTP 200 audio/mpeg → cliente reproduce el audio
```

> **Consejo de coste:** TTS cobra por carácter. Pide siempre respuestas concisas
> en el system prompt cuando vayas a convertirlas a audio.

### Resumen de capacidades multimedia en Spring AI

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CAPACIDADES MULTIMEDIA                                                       │
├──────────────────────────────────────────────┬───────────────────────────────┤
│  TTS  (Texto → Audio)                        │  SpeechModel                  │
│  STT  (Audio → Texto con Whisper)            │  AudioTranscriptionModel      │
│  Visión (Imagen → Descripción/Análisis)      │  ChatClient multimodal        │
│  DALL-E (Descripción → Imagen generada)      │  ImageModel                   │
└──────────────────────────────────────────────┴───────────────────────────────┘
```

## 13.4 Errores comunes

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ERRORES FRECUENTES EN MULTIMODALIDAD                                        │
├────────────────────────────────┬─────────────────────────────────────────────┤
│  Error                         │  Causa y solución                           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  "Model does not support       │  Se está enviando imagen o audio a un       │
│  vision" o respuesta sin       │  modelo que no es multimodal. GPT-3.5,     │
│  interpretar la imagen         │  algunos modelos de Ollama, y versiones     │
│                                │  antiguas de Claude no procesan imágenes.   │
│                                │  ✅ Usar GPT-4o, GPT-4-turbo, Claude 3,    │
│                                │  Gemini 1.5 o un modelo Ollama con visión   │
│                                │  (llava, bakllava, moondream).              │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  TTS genera audio con          │  El texto de entrada contiene markdown:     │
│  pronunciación extraña:        │  asteriscos (**texto**), backticks,         │
│  "asterisco asterisco texto    │  corchetes de URLs, etc. TTS los lee        │
│  asterisco asterisco"          │  literalmente como caracteres.              │
│                                │  ✅ Añadir en el system prompt del LLM:     │
│                                │  "Responde solo con texto plano, sin        │
│                                │  markdown ni asteriscos."                   │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Error al enviar imagen:       │  La imagen supera el límite de tamaño de    │
│  "Image too large" o           │  la API (OpenAI: máx. 20MB, pero en la     │
│  respuesta lenta               │  práctica > 4MB es costoso y lento).        │
│                                │  ✅ Redimensionar/comprimir antes de enviar.│
│                                │  Para análisis de documentos, PDF → imagen  │
│                                │  a 1024px de ancho es suficiente.           │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  DALL-E genera imagen que      │  El prompt es demasiado ambiguo o corto.   │
│  no corresponde a lo pedido    │  DALL-E necesita prompts descriptivos y     │
│                                │  específicos: estilo, composición, colores, │
│                                │  iluminación, etc.                          │
│                                │  ✅ En lugar de "un coche rojo", usar:      │
│                                │  "Un Ferrari rojo brillante en la autopista │
│                                │  al atardecer, fotorrealista, 8K, cielo     │
│                                │  naranja de fondo."                         │
├────────────────────────────────┼─────────────────────────────────────────────┤
│  Transcripción de audio        │  Whisper puede transcribir varios idiomas,  │
│  (Whisper) devuelve texto      │  pero sin especificar el idioma puede       │
│  en el idioma incorrecto       │  detectar mal el idioma si hay mezcla o    │
│  o con errores                 │  el audio tiene ruido.                      │
│                                │  ✅ Especificar language("es") en las       │
│                                │  opciones para forzar el idioma destino.    │
└────────────────────────────────┴─────────────────────────────────────────────┘
```

---

> **[← Advisors](12_advisors.md)** | **[← Inicio](../README.md)** | **[Siguiente: Observabilidad →](14_observabilidad.md)**
