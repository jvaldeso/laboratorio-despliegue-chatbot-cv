# Laboratorio — Despliegue de Chatbot CV en Railway

API REST de un chatbot inteligente construido con **Spring Boot 3**, **Java 21**, **Gemini AI** y **Pinecone** como base de datos vectorial. Este repositorio es una guía práctica para aprender a desplegar una aplicación Java en Railway usando Docker.

---

## Stack tecnológico

| Tecnología | Versión | Rol |
|---|---|---|
| Java | 21 | Lenguaje |
| Spring Boot | 3.3.5 | Framework web (WebFlux reactivo) |
| Gradle | 8.x | Build tool |
| Gemini AI | API | Modelo de lenguaje |
| Pinecone | API | Base de datos vectorial |
| Docker | - | Contenedorización |
| Railway | - | Plataforma de despliegue |

---

## Variables de entorno requeridas

Antes de desplegar necesitas tener estas 3 claves:

```
GEMINI_API_KEY=tu_clave_de_google_ai_studio
PINECONE_API_KEY=tu_clave_de_pinecone
PINECONE_URL=https://tu-indice.svc.turegion.pinecone.io
```

- **GEMINI_API_KEY** → obtenerla en [aistudio.google.com](https://aistudio.google.com)
- **PINECONE_API_KEY** → obtenerla en [app.pinecone.io](https://app.pinecone.io)
- **PINECONE_URL** → la URL de tu índice en Pinecone (sección "Indexes")

> 💡 **Contexto del laboratorio**
>
> Este laboratorio está diseñado para explorar y aprender sobre RAG, modelos de lenguaje y despliegue con Docker desde cero. Para sacarle el máximo provecho:
>
> - **Usa tu dispositivo personal.** Las APIs de Gemini y Pinecone son herramientas externas pensadas para experimentación, y es desde tu equipo personal donde vas a poder jugar con ellas libremente. Ten en cuenta que la capa gratuita de estas plataformas suele incluir políticas de uso de datos donde la información enviada puede ser utilizada para mejorar sus modelos — por eso es importante no mezclarlas con información corporativa.
> - **¿Te generó ideas para aplicarlo en tu trabajo?** Genial, ese es el objetivo. Si quieres explorar algo similar en un contexto interno, el equipo de IA es el mejor punto de partida — ya tienen el camino recorrido y pueden orientarte sobre cómo construirlo bien dentro del ecosistema de la empresa.

---

## Paso 1 — Clonar el repositorio

```bash
git clone https://github.com/JevDev2304/laboratorio-despliegue-chatbot-cv.git
cd laboratorio-despliegue-chatbot-cv
```

---

## Paso 2 — Crear el Dockerfile

En la raíz del proyecto crea un archivo llamado `Dockerfile` con este contenido:

```dockerfile
# Etapa 1: compilar el proyecto
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY settings.gradle .
COPY src src
RUN chmod +x gradlew && ./gradlew bootJar -x test --no-daemon

# Etapa 2: imagen final liviana solo con el JRE
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8085
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### ¿Por qué dos etapas (multi-stage)?

- La **etapa `build`** usa el JDK completo para compilar. Es pesada (~300MB) pero solo existe durante el build.
- La **etapa final** usa solo el JRE (Java Runtime), sin herramientas de compilación. La imagen resultante pesa ~80MB en vez de ~300MB.

---

## Paso 3 — Crear el .dockerignore

En la raíz del proyecto crea un archivo `.dockerignore` para evitar copiar archivos innecesarios a la imagen:

```
build/
.gradle/
.git/
.idea/
*.iml
```

---

## Paso 4 — Probar Docker localmente

```bash
# Construir la imagen
docker build -t chatbot-cv .

# Correr el contenedor con las variables de entorno
docker run -p 8085:8085 \
  -e GEMINI_API_KEY=tu_clave \
  -e PINECONE_API_KEY=tu_clave \
  -e PINECONE_URL=tu_url \
  chatbot-cv
```

La API estará disponible en `http://localhost:8085`

Documentación Swagger: `http://localhost:8085/swagger-ui.html`

---

## Paso 5 — Desplegar en Railway

1. Ve a [railway.app](https://railway.app) y crea una cuenta
2. Click en **New Project** → **Deploy from GitHub repo**
3. Selecciona este repositorio
4. Railway detecta el `Dockerfile` automáticamente y construye la imagen
5. Ve a **Variables** y agrega las 3 variables de entorno:
   - `GEMINI_API_KEY`
   - `PINECONE_API_KEY`
   - `PINECONE_URL`
6. Ve a **Settings → Networking → Generate Domain** para obtener la URL pública

Railway construye la imagen Docker y despliega. El primer deploy tarda ~5 minutos.

---

## Estructura del proyecto

```
laboratorio-despliegue-chatbot-cv/
├── src/
│   └── main/
│       ├── java/com/chatbot/
│       │   ├── aplicacion/      # Casos de uso
│       │   ├── dominio/         # Entidades y lógica de negocio
│       │   └── infraestructura/ # Controllers, adapters, config
│       └── resources/
│           └── application.properties
├── build.gradle                 # Dependencias del proyecto
├── settings.gradle
├── gradlew                      # Wrapper de Gradle
│
│   ── Archivos que debes crear tú ──
├── Dockerfile                   # (ver Paso 2)
└── .dockerignore                # (ver Paso 3)
```

---

## Endpoints principales

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/api/chat` | Enviar mensaje al chatbot |
| `GET` | `/swagger-ui.html` | Documentación interactiva |

---

## Errores comunes

**Error: `./gradlew: Permission denied`**
```bash
chmod +x gradlew
```

**Error: `No main manifest attribute`**
Asegúrate de que `build.gradle` tiene el plugin `org.springframework.boot`.

**El contenedor arranca pero falla con 500**
Verifica que las 3 variables de entorno estén configuradas correctamente en Railway.
