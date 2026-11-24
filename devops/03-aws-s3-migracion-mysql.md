# 3. AWS S3 + mi migración de MySQL a S3

> **Preguntas típicas:**
> - "¿Qué es S3 y cuándo lo usarías en lugar de una base de datos?"
> - "Cuéntame un proyecto del que estés orgulloso / un problema técnico difícil que hayas resuelto."

> ⚠️ **Estado:** la parte técnica de S3 está completa. La sección **"Mi historia"** es una **plantilla**: todo lo que dice `[COMPLETAR]` tiene que salir de **tu experiencia real**. No inventes números: si no los recuerdas exactos, usa aproximaciones honestas ("cerca de 200 GB", "de minutos a segundos").

---

## Parte A: S3 (la pregunta técnica)

### Respuesta corta (para decir en entrevista)

> **Amazon S3** es un servicio de **almacenamiento de objetos**. No es un sistema de archivos ni una base de datos: guarda **objetos** (bytes + metadatos) dentro de **buckets** y cada objeto se identifica por una **key** (por ejemplo `invoices/2025/11/INV-1234.pdf`). Es prácticamente **ilimitado**, tiene una durabilidad de **11 nueves** (99,999999999 %) porque replica entre varias zonas de disponibilidad, se cobra por GB almacenado, por peticiones y por transferencia, y tiene **clases de almacenamiento** más baratas para datos que se acceden poco (Standard-IA, Glacier). Lo uso para **archivos y datos grandes o históricos**: documentos, imágenes, backups, logs, exportaciones. La base de datos relacional la dejo para **datos estructurados y transaccionales** que necesito consultar, filtrar y relacionar con SQL. El patrón típico es guardar el archivo en S3 y en la base de datos solo la **key y los metadatos**.

### Conceptos clave

| Concepto | Qué es |
|---|---|
| **Bucket** | Contenedor de objetos. Nombre único global, vive en una región. |
| **Objeto / key** | El archivo y su "ruta". Las carpetas no existen de verdad: son **prefijos** de la key. |
| **Clases de almacenamiento** | Standard, Intelligent-Tiering, Standard-IA, Glacier Instant/Flexible/Deep Archive: más baratas para guardar, más caras o lentas para leer. |
| **Lifecycle rules** | Mover objetos a clases más baratas o borrarlos automáticamente según su antigüedad. |
| **Versioning** | Guarda versiones anteriores de un objeto. Protege contra borrados y sobrescrituras accidentales. |
| **Seguridad** | IAM policies + bucket policies, **Block Public Access**, cifrado (SSE-S3 por defecto, SSE-KMS). |
| **Presigned URL** | URL temporal firmada para subir o descargar un objeto **sin pasar el archivo por tu servidor** y sin hacer público el bucket. |
| **Consistencia** | Desde 2020 es **consistencia fuerte** de lectura después de escritura. |
| **Multipart upload** | Subir archivos grandes en partes, en paralelo y con reintentos. |
| **Consultas** | **Athena** permite consultar con SQL archivos CSV/JSON/**Parquet** guardados en S3. |

### S3 vs base de datos relacional (MySQL)

| | MySQL | S3 |
|---|---|---|
| Modelo | Tablas, filas, relaciones | Objetos key → bytes |
| Consultas | SQL, índices, joins, transacciones | GET/PUT por key, listar por prefijo (o Athena para análisis) |
| Bueno para | Datos transaccionales, pequeños y consultables | Archivos, blobs, histórico, backups, data lake |
| Escala de almacenamiento | Limitada por el disco/instancia; crecer es caro | Prácticamente ilimitada |
| Costo por GB | Alto (disco de BD + réplicas + backups) | Muy bajo, y aún menor con IA/Glacier |
| Archivos grandes (BLOB) | Inflan la BD, los backups y la replicación | Diseñado para eso |

### Ejemplo de código: subida con presigned URL (.NET)

```csharp
// El backend NO recibe el archivo: genera una URL firmada y el frontend sube directo a S3
app.MapPost("/documents/upload-url", (UploadRequest req, IAmazonS3 s3, ICurrentUser user) =>
{
    var key = $"documents/{user.Id}/{Guid.NewGuid()}/{req.FileName}";
    var url = s3.GetPreSignedURL(new GetPreSignedUrlRequest
    {
        BucketName  = "mi-app-documents",
        Key         = key,
        Verb        = HttpVerb.PUT,
        Expires     = DateTime.UtcNow.AddMinutes(10),
        ContentType = req.ContentType,
    });
    return Results.Ok(new { url, key }); // después el frontend confirma y guardo la key en la BD
});
```

---

## Parte B: Mi historia (la pregunta de comportamiento)

### Cómo contarla: formato STAR

| Parte | Qué cuentas | Tiempo |
|---|---|---|
| **S**ituación | Contexto: empresa/sistema, qué guardaba MySQL, cuál era el dolor | 20–30 s |
| **T**area | Cuál era **tu** responsabilidad y el objetivo | 10–15 s |
| **A**cción | Qué decidiste, **por qué**, qué alternativas descartaste, cómo migraste sin romper nada | 60–90 s |
| **R**esultado | Impacto medible + qué aprendiste | 20–30 s |

> Total: 2–3 minutos. Habla en **primera persona** ("yo decidí", "yo implementé"), aunque haya sido trabajo en equipo; aclara qué parte fue tuya.

### Plantilla para completar

**Situación**
- [COMPLETAR] ¿Qué sistema/producto era y para qué tipo de empresa o cliente?
- [COMPLETAR] ¿Qué se guardaba en MySQL que terminó en S3? (¿archivos como BLOB: PDFs, imágenes, adjuntos? ¿tablas históricas o logs que crecían sin parar? ¿otra cosa?)
- [COMPLETAR] ¿Cuál era el problema concreto? (tamaño de la BD, backups lentos, costo, consultas lentas, réplica atrasada, disco lleno, restauraciones que tardaban horas...)
- [COMPLETAR] Números del "antes", aunque sean aproximados: tamaño de la BD/tabla, cantidad de registros/archivos, tiempo de backup, costo mensual, latencia.

**Tarea**
- [COMPLETAR] ¿Qué te pidieron o qué propusiste tú? ¿Cuál era el criterio de éxito? (sin caída, sin perder datos, bajar costo, etc.)

**Acción** (aquí se gana la entrevista: las **decisiones** y el **porqué**)
- [COMPLETAR] Diseño: ¿cómo quedó? (por ejemplo: archivo en S3 + key y metadatos en MySQL; o datos históricos exportados a S3 en CSV/Parquet y consultados con Athena)
- [COMPLETAR] Alternativas que consideraste y por qué las descartaste (¿EFS? ¿agrandar la instancia RDS? ¿particionar la tabla? ¿otra base de datos?)
- [COMPLETAR] Estructura de keys / prefijos que elegiste y por qué.
- [COMPLETAR] Cómo migraste los datos existentes: ¿script/batch? ¿en lotes? ¿cuántos a la vez? ¿cómo reintentabas los fallos?
- [COMPLETAR] Cómo evitaste la caída: ¿doble escritura (MySQL + S3) durante la transición? ¿feature flag para leer de S3? ¿lectura con fallback a MySQL?
- [COMPLETAR] Cómo verificaste que no se perdió nada (conteos, checksums/MD5/ETag, muestreo, comparación de tamaños).
- [COMPLETAR] Seguridad: ¿bucket privado? ¿presigned URLs? ¿cifrado? ¿permisos IAM mínimos?
- [COMPLETAR] Costos: ¿usaste lifecycle rules o clases más baratas para lo antiguo?
- [COMPLETAR] Cuándo y cómo limpiaste los datos viejos de MySQL (después de cuánto tiempo de validación).

**Resultado**
- [COMPLETAR] Números del "después": tamaño de la BD, tiempo de backup, costo, rendimiento.
- [COMPLETAR] Impacto en el negocio/equipo (por ejemplo, restauraciones más rápidas, se evitó comprar una instancia más grande...).

**Lo que aprendí** (siempre lo preguntan; ten 2 o 3 preparados)
- [COMPLETAR] Algo técnico (por ejemplo, sobre migrar en lotes, idempotencia o validar antes de borrar).
- [COMPLETAR] Algo que harías diferente hoy.
- [COMPLETAR] Algo de proceso o comunicación (con el equipo, QA o negocio).

### Preguntas que te van a hacer después (prepara cada una)

1. ¿Por qué S3 y no otra opción? ¿Qué alternativas evaluaste?
2. ¿Cómo garantizaste que no se perdieron datos durante la migración?
3. ¿Hubo caída del servicio? ¿Cómo la evitaste?
4. ¿Qué pasa si se guarda el archivo en S3 pero falla el insert en MySQL (o al revés)? ¿Cómo manejas esa **inconsistencia**? (pista: orden de escritura, reintentos, proceso que limpia objetos huérfanos)
5. ¿Cómo controlas quién puede ver o descargar los archivos?
6. ¿Cuánto se ahorró? ¿Cómo lo mediste?
7. ¿Qué salió mal o qué no salió como esperabas?
8. Si tuvieras que hacerlo con 100 veces más datos, ¿qué cambiarías?

### Errores a evitar al contarla

- Contarla solo como "moví datos de un lado a otro". Lo valioso son el **problema**, las **decisiones** y los **trade-offs**.
- No tener números. Aunque sean aproximados, dan credibilidad.
- Decir "nosotros" todo el tiempo y que no se entienda qué hiciste tú.
- Inventar o exagerar: te van a pedir detalles y se nota.
- Olvidar el "qué aprendí".
