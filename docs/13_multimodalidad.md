<!-- navegación -->
> **[← Advisors](12_advisors.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Observabilidad →](14_observabilidad.md)**

---

# Capítulo 13 — Multimodalidad

> Cómo enviar imágenes y audio al modelo, generar imágenes con DALL-E
> y sintetizar voz con Text-to-Speech. Requiere modelos multimodales (GPT-4o, Claude 3, Gemini).

## 13. Multimodalidad — imágenes y audio

Spring AI soporta el envío de imágenes y audio al modelo (modelos multimodales como GPT-4o,
Claude 3, Gemini).

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

### Transcripción de audio (Speech-to-Text)

```java
@Autowired
AudioTranscriptionModel transcriptionModel;

@PostMapping("/transcribir")
public String transcribir(@RequestParam MultipartFile audio) throws IOException {
    AudioTranscriptionPrompt prompt = new AudioTranscriptionPrompt(
        new ByteArrayResource(audio.getBytes()),
        OpenAiAudioTranscriptionOptions.builder()
            .withLanguage("es")
            .withModel("whisper-1")
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
            .withModel("dall-e-3")
            .withQuality("hd")
            .withN(1)                    // número de imágenes
            .withHeight(1024)
            .withWidth(1024)
            .withStyle("vivid")          // "vivid" o "natural"
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
                .withModel("tts-1")         // tts-1 = rápido, tts-1-hd = calidad
                .withVoice(OpenAiAudioApi.SpeechRequest.Voice.NOVA)
                .withResponseFormat(
                    OpenAiAudioApi.SpeechRequest.AudioResponseFormat.MP3)
                .withSpeed(1.0f)            // 0.25 (muy lento) a 4.0 (muy rápido)
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
            .withModel("tts-1-hd")
            .withVoice(OpenAiAudioApi.SpeechRequest.Voice.NOVA)
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

> **[← Advisors](12_advisors.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: Observabilidad →](14_observabilidad.md)**
