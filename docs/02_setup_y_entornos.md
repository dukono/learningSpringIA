<!-- navegación -->
> **[← Introducción](01_introduccion.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: ChatClient →](03_chatclient.md)**

---

# Capítulo 02 — Setup y entornos

> Cómo crear el proyecto, añadir las dependencias necesarias y configurar los
> entornos de desarrollo (Ollama local) y producción (proveedor externo).

## Contenido

- [2.1 Dependencias (pom.xml)](#21-dependencias-pomxml)
- [2.2 Configuración (application.yml)](#22-configuración-applicationyml)
- [2.3 Obtener las dependencias](#23-obtener-las-dependencias)
- [2.4 Configuración por entornos — dev vs prod](#24-configuración-por-entornos--dev-vs-prod)
- [2.5 Estructura de ficheros de configuración](#25-estructura-de-ficheros-de-configuración)
- [2.6 Los ficheros de configuración](#26-los-ficheros-de-configuración)
- [2.7 Las dependencias — el truco clave](#27-las-dependencias--el-truco-clave)
- [2.8 Alternativa más limpia — @Profile en @Configuration](#28-alternativa-más-limpia--profile-en-configuration)
- [2.9 Cómo activar el perfil](#29-cómo-activar-el-perfil)
- [2.10 Ejemplo completo funcional](#210-ejemplo-completo-funcional)
- [2.11 Flujo visual completo](#211-flujo-visual-completo)
- [2.12 Añadir un tercer entorno — staging](#212-añadir-un-tercer-entorno--staging)
- [2.13 Resumen visual de decisión](#213-resumen-visual-de-decisión)

---

## 2. Setup del proyecto

### 2.1 Dependencias (pom.xml)

Spring AI usa su propio BOM (Bill of Materials), igual que Spring Boot:

```xml
<!-- pom.xml -->
<properties>
    <java.version>21</java.version>
    <spring-ai.version>1.1.0</spring-ai.version>
</properties>

<!-- BOM de Spring AI — controla todas las versiones -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Elige UNO de estos según tu proveedor: -->

<!-- OpenAI (ChatGPT, DALL-E, Whisper, TTS) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

<!-- Anthropic (Claude) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
</dependency>

<!-- Ollama (modelos locales GRATIS) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>

<!-- Google Gemini -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-vertex-ai-gemini-spring-boot-starter</artifactId>
</dependency>

<!-- Para Streaming necesitas WebFlux -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### 2.2 Configuración (application.yml)

```yaml
# Para OpenAI
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}          # variable de entorno — NUNCA hardcodear
      chat:
        options:
          model: gpt-4o-mini              # modelo por defecto
          temperature: 0.7                # creatividad (0.0=exacto, 1.0=creativo)
          max-tokens: 2000                # tokens máximos en la respuesta
```

```yaml
# Para Ollama (LOCAL, sin API key, sin coste)
spring:
  ai:
    ollama:
      base-url: http://localhost:11434    # URL donde corre Ollama
      chat:
        options:
          model: llama3.2                 # modelo descargado localmente
```

```yaml
# Para Anthropic (Claude)
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-3-5-sonnet-20241022
          max-tokens: 4096
```

### 2.3 Obtener las dependencias

Spring AI 1.0.0 está disponible en **Maven Central**, el repositorio estándar.
No necesitas añadir repositorios adicionales.

```bash
# Verificar que todo está bien
mvn dependency:resolve
mvn spring-boot:run
```

Si usas una versión **snapshot o milestone** (versiones de desarrollo), entonces sí necesitas:

```xml
<!-- SOLO para snapshots/milestones, NO para 1.0.0 estable -->
<repositories>
    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>
```

---


---

## 2.4 Configuración por entornos — dev (Ollama local) vs prod (OpenAI)

### El problema real

En el mundo profesional necesitas **dos configuraciones distintas**:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     EL PROBLEMA DE LOS ENTORNOS                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   DESARROLLO (dev)                    PRODUCCIÓN (prod)                      │
│   ┌──────────────────┐                ┌──────────────────┐                  │
│   │  Ollama LOCAL    │                │  OpenAI / Claude │                  │
│   │  - Gratis        │                │  - De pago       │                  │
│   │  - Sin internet  │                │  - Alta calidad  │                  │
│   │  - Sin API key   │                │  - Escalable     │                  │
│   │  - Lento a veces │                │  - Rápido        │                  │
│   └──────────────────┘                └──────────────────┘                  │
│                                                                              │
│   → No quieres pagar por cada prueba que haces en local                     │
│   → No quieres que prod use un modelo local en un servidor                  │
│   → La solución: Perfiles de Spring Boot                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.5 Estructura de ficheros de configuración

La solución usa los **perfiles de Spring Boot** — exactamente igual que con bases de datos:

```
src/
└── main/
    └── resources/
        ├── application.yml              ← configuración base (común a todos)
        ├── application-dev.yml          ← solo se carga con perfil "dev"
        └── application-prod.yml         ← solo se carga con perfil "prod"
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    CÓMO FUNCIONAN LOS PERFILES                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   application.yml                                                            │
│   ┌──────────────────────────────────┐                                       │
│   │  spring.application.name: myapp  │  ← siempre se carga                  │
│   │  server.port: 8080               │                                       │
│   └──────────────────────────────────┘                                       │
│              +                                                               │
│   Si perfil = "dev"          Si perfil = "prod"                              │
│   ┌──────────────────┐        ┌──────────────────┐                           │
│   │ application-dev  │   OR   │ application-prod │                           │
│   │ Ollama config    │        │ OpenAI config    │                           │
│   └──────────────────┘        └──────────────────┘                           │
│                                                                              │
│   El perfil activo lo decides tú al arrancar la app                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.6 Los ficheros de configuración

**`application.yml`** — configuración común, sin nada de IA:

```yaml
# application.yml — base común
spring:
  application:
    name: mi-chatbot

server:
  port: 8080

logging:
  level:
    org.springframework.ai: DEBUG    # útil para ver qué manda a la IA
```

---

**`application-dev.yml`** — Ollama local, GRATIS:

```yaml
# application-dev.yml — desarrollo local con Ollama
spring:
  ai:
    ollama:
      base-url: http://localhost:11434   # Ollama corriendo en tu máquina
      chat:
        options:
          model: llama3.2               # modelo descargado en local
          temperature: 0.7
```

---

**`application-prod.yml`** — OpenAI en producción:

```yaml
# application-prod.yml — producción con OpenAI
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}         # variable de entorno del servidor
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
          max-tokens: 2000
```

### 2.7 Las dependencias — el truco clave

Aquí está el **problema**: si tienes ambos starters en el `pom.xml`, Spring Boot auto-configura
los dos y puede haber conflictos. La solución más limpia es **incluir ambos** pero controlar
qué bean se activa según el perfil.

```xml
<!-- pom.xml — incluyes AMBAS dependencias -->
<dependencies>

    <!-- OpenAI — para prod -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>

    <!-- Ollama — para dev -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
    </dependency>

</dependencies>
```

Para evitar conflictos, **desactiva la auto-configuración** de ambos en `application.yml`
y actívalos manualmente por perfil:

```yaml
# application.yml — desactiva auto-configuración de ambos
spring:
  autoconfigure:
    exclude:
      - org.springframework.ai.autoconfigure.openai.OpenAiAutoConfiguration
      - org.springframework.ai.autoconfigure.ollama.OllamaAutoConfiguration
```

```yaml
# application-dev.yml — activa solo Ollama
spring:
  autoconfigure:
    exclude:
      - org.springframework.ai.autoconfigure.openai.OpenAiAutoConfiguration
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
```

```yaml
# application-prod.yml — activa solo OpenAI
spring:
  autoconfigure:
    exclude:
      - org.springframework.ai.autoconfigure.ollama.OllamaAutoConfiguration
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
```

### 2.8 Alternativa más limpia — @Profile en @Configuration

Si prefieres controlar todo desde Java (más explícito y testeable):

```java
// config/AiConfig.java

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
public class AiConfig {

    // ─── PERFIL DEV ───────────────────────────────────────────────────────────
    // Este bean SOLO existe cuando el perfil activo es "dev"
    // Ollama corre en tu máquina local, sin coste, sin internet
    @Bean
    @Profile("dev")
    public ChatClient devChatClient(OllamaChatModel ollamaModel) {
        return ChatClient.builder(ollamaModel)
                .defaultSystem("Eres un asistente de desarrollo. Responde de forma concisa.")
                .build();
    }

    // ─── PERFIL PROD ──────────────────────────────────────────────────────────
    // Este bean SOLO existe cuando el perfil activo es "prod"
    // OpenAI en el servidor de producción, con API key real
    @Bean
    @Profile("prod")
    public ChatClient prodChatClient(OpenAiChatModel openAiModel) {
        return ChatClient.builder(openAiModel)
                .defaultSystem("Eres un asistente profesional. Responde de forma clara y precisa.")
                .build();
    }
}
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│               CÓMO FUNCIONA @Profile EN TIEMPO DE ARRANQUE                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  App arranca con perfil "dev"                                                │
│       │                                                                      │
│       ▼                                                                      │
│  Spring escanea todos los @Bean                                              │
│       │                                                                      │
│       ├── @Profile("dev")  → SÍ se registra  ✓                              │
│       └── @Profile("prod") → NO se registra  ✗                              │
│                                                                              │
│  El @Service que inyecta ChatClient recibe el bean de Ollama                 │
│                                                                              │
│  → Si cambias a "prod", automáticamente recibe el bean de OpenAI            │
│  → El código del @Service NO cambia nunca                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

Tu servicio NO necesita saber qué proveedor está activo:

```java
// service/ChatService.java — igual en dev y en prod
@Service
public class ChatService {

    private final ChatClient chatClient;

    // Spring inyecta el bean correcto según el perfil activo
    // En dev → inyecta el ChatClient con Ollama
    // En prod → inyecta el ChatClient con OpenAI
    public ChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String chat(String mensaje) {
        return chatClient
                .prompt()
                .user(mensaje)
                .call()
                .content();
    }
}
```

### 2.9 Cómo activar el perfil

**Opción 1 — Variable de entorno (recomendado para CI/CD y producción):**

```bash
# Arrancar en modo dev
SPRING_PROFILES_ACTIVE=dev mvn spring-boot:run

# Arrancar en modo prod
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

**Opción 2 — Flag en la JVM:**

```bash
# Dev
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Prod
java -jar app.jar --spring.profiles.active=prod
```

**Opción 3 — En `application.yml` (solo para desarrollo, nunca en prod):**

```yaml
# application.yml — activa dev por defecto en local
spring:
  profiles:
    active: dev    # ← solo para local, en prod se sobreescribe con variable de entorno
```

**Opción 4 — En IntelliJ IDEA:**

```
Run → Edit Configurations → Spring Boot → Active profiles: dev
```

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     RESUMEN: ¿CÓMO ACTIVAR EL PERFIL?                       │
├────────────────────┬─────────────────────────────────────────────────────────┤
│  Dónde             │  Cómo                                                   │
├────────────────────┼─────────────────────────────────────────────────────────┤
│  Local / Terminal  │  SPRING_PROFILES_ACTIVE=dev mvn spring-boot:run         │
│  IntelliJ IDEA     │  Run Config → Active profiles → dev                     │
│  Docker            │  ENV SPRING_PROFILES_ACTIVE=prod                        │
│  Kubernetes        │  env: - name: SPRING_PROFILES_ACTIVE value: prod        │
│  GitHub Actions    │  env: SPRING_PROFILES_ACTIVE: prod                      │
│  application.yml   │  spring.profiles.active: dev  (solo para local)        │
└────────────────────┴─────────────────────────────────────────────────────────┘
```

### 2.10 Ejemplo completo funcional

Escenario: chatbot que usa **Ollama en dev** y **OpenAI en prod**. El código Java es
idéntico en los dos entornos.

**Estructura del proyecto:**

```
src/
├── main/
│   ├── java/com/ejemplo/chatbot/
│   │   ├── ChatbotApplication.java
│   │   ├── config/
│   │   │   └── AiConfig.java           ← beans con @Profile
│   │   ├── controller/
│   │   │   └── ChatController.java
│   │   └── service/
│   │       └── ChatService.java        ← no sabe qué proveedor hay
│   └── resources/
│       ├── application.yml             ← config base
│       ├── application-dev.yml         ← Ollama
│       └── application-prod.yml        ← OpenAI
```

**`application.yml`:**

```yaml
spring:
  application:
    name: chatbot
  profiles:
    active: dev    # por defecto, local usa dev

server:
  port: 8080
```

**`application-dev.yml`:**

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
          temperature: 0.7

logging:
  level:
    com.ejemplo: DEBUG
```

**`application-prod.yml`:**

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
          max-tokens: 2000

logging:
  level:
    com.ejemplo: INFO
```

**`AiConfig.java`:**

```java
@Configuration
public class AiConfig {

    @Bean
    @Profile("dev")
    public ChatClient chatClientDev(OllamaChatModel model) {
        System.out.println(">>> Usando Ollama LOCAL (dev)");
        return ChatClient.create(model);
    }

    @Bean
    @Profile("prod")
    public ChatClient chatClientProd(OpenAiChatModel model) {
        System.out.println(">>> Usando OpenAI (prod)");
        return ChatClient.create(model);
    }
}
```

**`ChatService.java`:**

```java
@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String preguntar(String pregunta) {
        return chatClient
                .prompt()
                .user(pregunta)
                .call()
                .content();
    }
}
```

**`ChatController.java`:**

```java
@RestController
@RequestMapping("/chat")
public class ChatController {

    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    @GetMapping
    public String chat(@RequestParam String mensaje) {
        return chatService.preguntar(mensaje);
    }
}
```

**Lo que ves en los logs al arrancar:**

```
# Con perfil dev:
>>> Usando Ollama LOCAL (dev)
Started ChatbotApplication — perfil activo: [dev]

# Con perfil prod:
>>> Usando OpenAI (prod)
Started ChatbotApplication — perfil activo: [prod]
```

### 2.11 Flujo visual completo

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    FLUJO COMPLETO POR ENTORNO                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PERFIL: dev                           PERFIL: prod                          │
│                                                                              │
│  Tu máquina local                      Servidor / Cloud                      │
│  ┌──────────────────────┐              ┌──────────────────────┐              │
│  │  GET /chat?msg=Hola  │              │  GET /chat?msg=Hola  │              │
│  └──────────┬───────────┘              └──────────┬───────────┘              │
│             │                                     │                          │
│             ▼                                     ▼                          │
│  ┌──────────────────────┐              ┌──────────────────────┐              │
│  │   ChatController     │              │   ChatController     │              │
│  └──────────┬───────────┘              └──────────┬───────────┘              │
│             │                                     │                          │
│             ▼                                     ▼                          │
│  ┌──────────────────────┐              ┌──────────────────────┐              │
│  │    ChatService       │              │    ChatService       │              │
│  │  (código idéntico)   │              │  (código idéntico)   │              │
│  └──────────┬───────────┘              └──────────┬───────────┘              │
│             │                                     │                          │
│             ▼                                     ▼                          │
│  ┌──────────────────────┐              ┌──────────────────────┐              │
│  │  ChatClient (Ollama) │              │ ChatClient (OpenAI)  │              │
│  │  @Profile("dev")     │              │  @Profile("prod")    │              │
│  └──────────┬───────────┘              └──────────┬───────────┘              │
│             │                                     │                          │
│             ▼                                     ▼                          │
│  ┌──────────────────────┐              ┌──────────────────────┐              │
│  │  Ollama localhost    │              │  api.openai.com      │              │
│  │  :11434  (gratis)    │              │  (de pago, rápido)   │              │
│  └──────────────────────┘              └──────────────────────┘              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.12 Añadir un tercer entorno — staging

En equipos grandes se suele tener también un entorno **staging** (pre-producción):

```
application-staging.yml  → OpenAI con modelo más barato (gpt-4o-mini)
application-prod.yml     → OpenAI con modelo más potente (gpt-4o)
```

```yaml
# application-staging.yml — igual que prod pero modelo más barato
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY_STAGING}
      chat:
        options:
          model: gpt-4o-mini          # más barato para probar
          max-tokens: 1000

# application-prod.yml — modelo de máxima calidad
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY_PROD}
      chat:
        options:
          model: gpt-4o               # más caro pero mejor calidad
          max-tokens: 4000
```

```java
@Configuration
public class AiConfig {

    @Bean
    @Profile("dev")
    public ChatClient chatClientDev(OllamaChatModel model) {
        return ChatClient.create(model);          // Ollama local — gratis
    }

    @Bean
    @Profile({"staging", "prod"})                 // funciona para ambos
    public ChatClient chatClientCloud(OpenAiChatModel model) {
        return ChatClient.create(model);          // OpenAI — diferencia solo en yml
    }
}
```

### 2.13 Resumen visual de decisión

```
┌──────────────────────────────────────────────────────────────────────────────┐
│              ¿QUÉ ESTRATEGIA USAR SEGÚN TU CASO?                             │
├─────────────────────────┬────────────────────────────────────────────────────┤
│  Situación              │  Solución recomendada                              │
├─────────────────────────┼────────────────────────────────────────────────────┤
│  Proyecto pequeño       │  Solo application-dev.yml / application-prod.yml   │
│  personal o de equipo   │  + spring.profiles.active en variable de entorno   │
├─────────────────────────┼────────────────────────────────────────────────────┤
│  Proyecto mediano con   │  @Profile("dev") + @Profile("prod") en @Config     │
│  lógica específica      │  + application-{perfil}.yml para propiedades       │
│  por entorno            │                                                    │
├─────────────────────────┼────────────────────────────────────────────────────┤
│  Proyecto grande con    │  dev → Ollama                                      │
│  staging y prod         │  staging → OpenAI gpt-4o-mini                     │
│                         │  prod → OpenAI gpt-4o                             │
│                         │  + @Profile({"staging","prod"}) para cloud         │
├─────────────────────────┼────────────────────────────────────────────────────┤
│  Tests unitarios        │  @Profile("test") con un ChatModel mock            │
│                         │  → sin coste, sin dependencias externas            │
└─────────────────────────┴────────────────────────────────────────────────────┘
```



---

> **[← Introducción](01_introduccion.md)** | **[← Inicio](00_indice.md)** | **[Siguiente: ChatClient →](03_chatclient.md)**
