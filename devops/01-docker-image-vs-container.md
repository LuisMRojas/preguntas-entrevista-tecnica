# 1. Docker: image vs container, y para qué sirve un `Dockerfile`

> **Pregunta típica:** "¿Cuál es la diferencia entre una imagen y un contenedor en Docker? ¿Qué es un Dockerfile?"

---

## Respuesta corta (para decir en entrevista)

> Una **imagen** es una **plantilla inmutable, de solo lectura**: contiene el sistema de archivos con todo lo que la app necesita para correr (runtime, dependencias, código compilado, configuración por defecto) y metadatos como el comando de arranque. Está formada por **capas** que se cachean y se comparten entre imágenes. Un **contenedor** es una **instancia en ejecución de una imagen**: un proceso aislado (con namespaces y cgroups del kernel de Linux) al que Docker le agrega una **capa de escritura** propia. La relación es como **clase e instancia**: de una imagen puedo levantar muchos contenedores. Si borro un contenedor, se pierde lo que escribió, salvo que use **volúmenes**.
>
> El **`Dockerfile`** es la **receta** para construir la imagen: un archivo de texto con instrucciones (`FROM`, `COPY`, `RUN`, `ENTRYPOINT`...). Cada instrucción genera una capa. Sirve para que el build sea **reproducible** y versionado junto al código, y así se elimina el "en mi máquina funciona".

---

## Explicación a fondo

### Analogía

| Docker | Programación | Otra analogía |
|---|---|---|
| `Dockerfile` | Código fuente de una clase | Receta |
| Imagen | Clase compilada | Pastel congelado (siempre igual) |
| Contenedor | Instancia (`new`) | Porción servida (cada una se puede modificar) |
| Registry (Docker Hub, ECR, GHCR) | Repositorio de paquetes (NuGet, npm) | Tienda |

### Imagen: capas inmutables

```
┌───────────────────────────────┐  ← capa de escritura (solo del contenedor)
├───────────────────────────────┤
│ COPY ./publish /app           │  capa 4
│ RUN apt-get install ...       │  capa 3
│ ENV ASPNETCORE_URLS=...       │  capa 2 (metadatos)
│ FROM mcr.microsoft.com/...    │  capas base
└───────────────────────────────┘
```

- Cada capa guarda **solo las diferencias** con la anterior (union filesystem / OverlayFS).
- Si 10 contenedores usan la misma imagen, las capas de solo lectura **se comparten** en disco.
- Una imagen se identifica por un **digest** (`sha256:...`) y uno o más **tags** (`miapp:1.4.2`, `miapp:latest`).

### Contenedor: proceso aislado

- **No es una máquina virtual**: comparte el kernel del host, por eso arranca en milisegundos y pesa poco.
- Aislamiento con **namespaces** (procesos, red, filesystem, usuarios) y límites de recursos con **cgroups** (CPU, memoria).
- Ciclo de vida: `created → running → paused/stopped → removed`.
- Debe ser **efímero** y *stateless*: el estado va en una base de datos, en volúmenes o en S3.

### Comandos clave

```bash
docker build -t catalog-api:1.0 .           # Dockerfile → imagen
docker images                               # listar imágenes
docker run -d -p 8080:8080 --name api1 catalog-api:1.0   # imagen → contenedor
docker run -d -p 8081:8080 --name api2 catalog-api:1.0   # otro contenedor, misma imagen
docker ps                                   # contenedores corriendo
docker logs -f api1
docker exec -it api1 sh                     # entrar al contenedor
docker stop api1 && docker rm api1          # se pierde su capa de escritura
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/catalog-api:1.0
```

### Instrucciones más usadas del Dockerfile

| Instrucción | Para qué |
|---|---|
| `FROM` | Imagen base (y en multi-stage, cada etapa) |
| `WORKDIR` | Directorio de trabajo |
| `COPY` / `ADD` | Copiar archivos al build (`ADD` además descomprime y acepta URLs; preferir `COPY`) |
| `RUN` | Ejecutar un comando **en tiempo de build** (crea una capa) |
| `ENV` / `ARG` | Variables de entorno (runtime) / argumentos de build |
| `EXPOSE` | Documenta el puerto (no lo publica; eso lo hace `-p`) |
| `USER` | Correr como usuario no-root |
| `ENTRYPOINT` / `CMD` | Qué se ejecuta **al arrancar el contenedor** (`CMD` da argumentos por defecto) |
| `HEALTHCHECK` | Comando para verificar que la app está sana |

---

## Ejemplo de aplicación real

**Escenario:** una API en **.NET** (el catálogo de productos del tema de IEnumerable/IQueryable) que antes se desplegaba copiando archivos por FTP a un servidor. Había diferencias entre las máquinas de los desarrolladores, QA y producción (versión del runtime, librerías nativas, variables de entorno).

### Dockerfile multi-stage

```dockerfile
# ---------- Etapa 1: build (SDK completo, ~800 MB) ----------
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# 1) Copio SOLO los .csproj y restauro → esta capa se cachea mientras no cambien las dependencias
COPY Catalog.Api/Catalog.Api.csproj Catalog.Api/
RUN dotnet restore Catalog.Api/Catalog.Api.csproj

# 2) Copio el resto del código (cambia seguido → va después)
COPY . .
RUN dotnet publish Catalog.Api/Catalog.Api.csproj -c Release -o /app/publish --no-restore

# ---------- Etapa 2: runtime (solo ASP.NET runtime, ~100 MB) ----------
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
COPY --from=build /app/publish .

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
# No correr como root (las imágenes oficiales de .NET 8+ ya traen el usuario "app")
# Ojo: en un Dockerfile los comentarios van en su propia línea, no al final de una instrucción
USER app

ENTRYPOINT ["dotnet", "Catalog.Api.dll"]
```

```gitignore
# .dockerignore: que no entren al contexto de build
bin/
obj/
.git/
**/appsettings.Development.json
```

**Por qué está hecho así:**
- **Multi-stage**: el SDK y el código fuente no llegan a producción. La imagen final es más chica y tiene menos superficie de ataque.
- **Orden de capas**: lo que cambia poco va primero (`restore`) y lo que cambia mucho va al final (`COPY . .`). Así un cambio de código no vuelve a descargar los paquetes NuGet.
- **Usuario no-root**: si alguien compromete la app, tiene menos privilegios.
- **Sin secretos en la imagen**: la cadena de conexión llega como variable de entorno o secret en runtime, nunca en el Dockerfile. Cualquiera con acceso a la imagen puede leer sus capas.

### Desarrollo local con `docker compose`

```yaml
# compose.yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      ConnectionStrings__Default: "Server=db;Database=catalog;User=root;Password=${DB_PASSWORD}"
    depends_on:
      db: { condition: service_healthy }
  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: catalog
    volumes: ["dbdata:/var/lib/mysql"]      # el volumen sobrevive aunque borre el contenedor
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
volumes:
  dbdata:
```

**Resultado:** un desarrollador nuevo clona el repo, hace `docker compose up` y tiene API + base de datos funcionando. La **misma imagen** (mismo digest) que pasó QA es la que se despliega en producción. Lo único que cambia entre entornos es la configuración.

---

## Errores comunes / trampas

- Confundir **imagen** y **contenedor**: "subí el contenedor a Docker Hub". Lo que se sube es la **imagen**.
- **Guardar estado dentro del contenedor** (uploads, logs, base de datos sin volumen) → se pierde al recrearlo.
- **Secretos en el Dockerfile** (`ENV PASSWORD=...`, `COPY .env`) → quedan en las capas aunque los borres en una capa posterior.
- **Usar `latest` en producción** → no sabes qué versión está corriendo y el rollback es difícil. Usa tags inmutables (versión o SHA del commit).
- **Mal orden de capas** → cada build reinstala todas las dependencias.
- **Imágenes gigantes** (SDK en producción, sin `.dockerignore`) → despliegues lentos y más vulnerabilidades.
- Creer que un contenedor es una VM: comparte el kernel, por eso un contenedor Linux no corre de forma nativa en un kernel Windows.
- **Correr como root**.

---

## Preguntas de seguimiento para practicar

1. ¿Qué diferencia hay entre `CMD` y `ENTRYPOINT`? ¿Y entre `RUN` y `CMD`?
2. ¿Qué es un build multi-stage y qué problema resuelve?
3. ¿Cómo aprovechas la caché de capas para builds más rápidos?
4. ¿Qué diferencia hay entre un **volumen** y un **bind mount**?
5. ¿En qué se diferencia un contenedor de una máquina virtual?
6. ¿Por qué no deberías usar `latest` en producción?
7. ¿Cómo manejas los secretos y la configuración por entorno?
