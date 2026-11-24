# 3. AWS S3 + mi migración de logs de auditoría de MySQL a S3

> **Preguntas típicas:**
> - "¿Qué es S3 y cuándo lo usarías en lugar de una base de datos?"
> - "Cuéntame un proyecto del que estés orgulloso / algo que hayas aprendido en un proyecto."

---

## Parte A: S3 (la pregunta técnica)

### Respuesta corta (para decir en entrevista)

> **Amazon S3** es un servicio de **almacenamiento de objetos**. No es un sistema de archivos ni una base de datos: guarda **objetos** (bytes + metadatos) dentro de **buckets** y cada objeto se identifica por una **key** (por ejemplo `audit-logs/2025/11/24/part-0001.json`). Es prácticamente **ilimitado**, tiene una durabilidad de **11 nueves** (99,999999999 %) porque replica entre varias zonas de disponibilidad, se cobra por GB almacenado, por peticiones y por transferencia, y tiene **clases de almacenamiento** más baratas para datos que se acceden poco (Standard-IA, Glacier). Lo uso para **archivos y datos grandes o históricos**: documentos, imágenes, backups, **logs**, exportaciones. La base de datos relacional la dejo para **datos estructurados y transaccionales** que necesito consultar, filtrar y relacionar con SQL.

### Conceptos clave

| Concepto | Qué es |
|---|---|
| **Bucket** | Contenedor de objetos. Nombre único global, vive en **una región**. |
| **Región** | Cada bucket existe en una región (por ejemplo `us-east-1`). El cliente/SDK tiene que estar configurado con la región correcta o las peticiones fallan o se redirigen. |
| **Objeto / key** | El archivo y su "ruta". Las carpetas no existen de verdad: son **prefijos** de la key. |
| **Clases de almacenamiento** | Standard, Intelligent-Tiering, Standard-IA, Glacier Instant/Flexible/Deep Archive: más baratas para guardar, más caras o lentas para leer. |
| **Lifecycle rules** | Mover objetos a clases más baratas o borrarlos automáticamente según su antigüedad. |
| **Versioning** | Guarda versiones anteriores de un objeto. Protege contra borrados y sobrescrituras accidentales. |
| **Object Lock** | Modo WORM (*write once, read many*): nadie puede modificar ni borrar un objeto durante un período. Muy usado en **auditoría** y cumplimiento normativo. |
| **Seguridad** | IAM policies + bucket policies, **Block Public Access**, cifrado (SSE-S3 por defecto, SSE-KMS). |
| **Presigned URL** | URL temporal firmada para subir o descargar un objeto sin pasar el archivo por tu servidor y sin hacer público el bucket. |
| **Consistencia** | Desde 2020 es **consistencia fuerte** de lectura después de escritura. |
| **Consultas** | **Athena** permite consultar con SQL archivos CSV/JSON/**Parquet** guardados en S3. |

### S3 vs base de datos relacional (MySQL)

| | MySQL | S3 |
|---|---|---|
| Modelo | Tablas, filas, relaciones | Objetos key → bytes |
| Consultas | SQL, índices, joins, transacciones | GET/PUT por key, listar por prefijo (o Athena para análisis) |
| Bueno para | Datos transaccionales y **datos recientes** que se consultan seguido | Archivos, blobs, **histórico**, **logs**, backups, data lake |
| Escala de almacenamiento | Limitada por el disco/instancia; crecer es caro | Prácticamente ilimitada |
| Costo por GB | Alto (disco de BD + réplicas + backups) | Muy bajo, y aún menor con IA/Glacier |

### ¿Por qué los logs de auditoría encajan tan bien en S3?

| Característica de un log de auditoría | Por qué encaja con S3 |
|---|---|
| **Solo se agregan registros**, nunca se modifican | S3 es ideal para escribir objetos y no tocarlos más (y con Object Lock se puede garantizar). |
| **Crece sin parar** (cada movimiento genera un registro) | Almacenamiento prácticamente ilimitado y barato. |
| **Se consulta poco**, y casi siempre **por rango de fechas** | Si las keys llevan la fecha (`año/mes/día`), se lee solo el prefijo necesario. |
| **Lo reciente se consulta mucho más que lo antiguo** | Patrón **caliente / frío**: lo reciente en la BD (rápido), lo antiguo en S3 (barato). |
| No necesita joins ni transacciones | No hace falta pagar una base relacional para guardar el histórico. |

### Patrón "datos calientes / datos fríos" (*hot / cold storage*)

```
                         consulta por rango de fechas
                                    │
                 ┌──────────────────┴──────────────────┐
                 ▼                                     ▼
     últimos 45 días → MySQL                 más de 45 días → S3
     (caliente: rápido, indexado)            (frío: barato, ilimitado)
                 ▲                                     ▲
                 └──── job de fin de semana ───────────┘
                       mueve lo que pasó los 45 días
```

### Ejemplo de código: job que mueve los logs antiguos (.NET)

> Ejemplo **ilustrativo** de cómo se ve un job así, útil para explicar la idea en una entrevista técnica. No es el código del proyecto.

```csharp
public class AuditLogArchiver(AuditDbContext db, IAmazonS3 s3, TimeProvider clock, ILogger<AuditLogArchiver> log)
{
    private const string Bucket = "mi-app-audit-logs";
    private const int BatchSize = 5_000;

    public async Task RunAsync(CancellationToken ct)
    {
        var cutoff = clock.GetUtcNow().AddDays(-45);

        while (!ct.IsCancellationRequested)
        {
            // 1) Leer un lote de registros antiguos (en lotes para no cargar millones en memoria)
            var batch = await db.AuditLogs
                .AsNoTracking()
                .Where(l => l.CreatedAt < cutoff)
                .OrderBy(l => l.Id)
                .Take(BatchSize)
                .ToListAsync(ct);

            if (batch.Count == 0) break;

            long firstId = batch[0].Id, lastId = batch[^1].Id;

            // 2) Key determinística: si el job se cae y se vuelve a ejecutar, sobrescribe el mismo objeto
            //    en lugar de duplicar los datos → el proceso es idempotente
            var key = $"audit-logs/{batch[0].CreatedAt:yyyy/MM/dd}/{firstId}-{lastId}.jsonl";
            var body = string.Join('\n', batch.Select(l => JsonSerializer.Serialize(l)));

            await s3.PutObjectAsync(new PutObjectRequest { BucketName = Bucket, Key = key, ContentBody = body }, ct);

            // 3) Solo DESPUÉS de confirmar la escritura en S3 se borra de MySQL
            var deleted = await db.AuditLogs
                .Where(l => l.Id >= firstId && l.Id <= lastId && l.CreatedAt < cutoff)
                .ExecuteDeleteAsync(ct);

            log.LogInformation("Archivados {Count} logs en {Key}", deleted, key);
        }
    }
}
```

Para programarlo los fines de semana en horario de poco uso se puede usar, por ejemplo, Hangfire/Quartz o un **CronJob de Kubernetes** (`schedule: "0 2 * * 6,0"` = sábado y domingo a las 2:00).

---

## Parte B: Mi historia (la pregunta de comportamiento)

### Los hechos (lo que realmente pasó)

| | |
|---|---|
| **Qué se migró** | Los **logs de auditoría**: registrábamos **cada movimiento** de los usuarios en MySQL. |
| **El problema** | La base de datos se volvió **muy grande**, empezó a ser **costosa** por el volumen y las **búsquedas por fechas antiguas eran lentas**. |
| **El diseño** | Lo definió el **arquitecto** del equipo. Yo no conocía S3. Diseño por **rango de tiempo**: los **últimos 45 días en MySQL** y lo **más antiguo en S3**. |
| **Mi parte** | **Toda la implementación**, incluido el **job de migración**, que corría los **fines de semana en horarios en que nadie consultaba** los datos. |
| **Cómo se validó** | En **ambientes de prueba con un mock de S3**. |
| **Lo que no salió a la primera** | En las primeras pruebas **no llegaban datos** a S3: el cliente de S3 **no tenía bien configurada la región**. |
| **Resultado** | La base de datos **se redujo muchísimo** y los **tiempos de carga de datos mejoraron**. No hay cifras exactas. |
| **Lo que aprendí** | Qué es S3 y cuándo usarlo; a elegir el almacenamiento según cómo se usa el dato; y que hacen falta **más pruebas en ambientes de prueba** antes de migrar. |

### Versión para decir en voz alta (~2 minutos)

> **Situación.** En [COMPLETAR: empresa o tipo de sistema] guardábamos en MySQL un **log de auditoría de cada movimiento** que hacían los usuarios. Como se registraba absolutamente todo, esa información creció muchísimo: la base de datos se volvió muy grande, el costo empezó a subir por el volumen, y las búsquedas de registros de fechas antiguas eran lentas.
>
> **Tarea.** El arquitecto del equipo diseñó una solución con Amazon S3, y **yo me encargué de toda la implementación**. Yo no conocía S3, así que fue un reto aprender una tecnología nueva y entregarla.
>
> **Acción.** La idea era separar los datos por antigüedad: los **últimos 45 días** se quedaban en **MySQL**, porque son los que más se consultan y ahí la respuesta es rápida, y todo lo **más antiguo** se movía a **S3**, que es mucho más barato y prácticamente ilimitado. Para mover los datos **implementé un job** que corría **los fines de semana, en horarios en los que nadie consultaba** la información, así la migración no afectaba a los usuarios. Todo lo validé primero en **ambientes de prueba usando un mock de S3**.
>
> Y no todo salió a la primera: en las primeras pruebas **los datos no llegaban a S3**. Después de investigar, encontré que **el cliente de S3 no tenía bien configurada la región**. Lo corregí y a partir de ahí la escritura funcionó.
>
> **Resultado.** La base de datos **se redujo de forma muy significativa** y los **tiempos de carga de datos mejoraron** de forma notoria. No tengo las cifras exactas, pero la diferencia se notó claramente.
>
> **Aprendizaje.** Me llevo tres cosas. Primero, **elegir el almacenamiento según cómo se usa el dato**: lo reciente y consultado en la base relacional, y el histórico en almacenamiento barato. Segundo, que **un detalle de configuración, como la región, puede romper toda la integración**, así que hoy valido la configuración de cada ambiente desde el inicio. Y tercero, que vale la pena **invertir más en pruebas** en un ambiente lo más parecido posible a producción antes de migrar.

### Lo que todavía falta (opcional, pero suma)

- [ ] **Empresa o tipo de sistema**, aunque sea genérico ("un sistema de gestión para...", "una plataforma de...").
- [ ] **Cómo se resolvía una búsqueda que cruzaba los 45 días** (por ejemplo, de hace 60 días hasta hoy): ¿se consultaban MySQL y S3 y se unían los resultados? ¿o la búsqueda era de un solo rango?
- [ ] **Cómo verificaba el job que no se perdía nada** antes de borrar de MySQL (¿se comparaban conteos? ¿se borraba solo después de escribir en S3?).
- [ ] **Qué pasaba si el job fallaba a la mitad**: ¿se volvía a ejecutar? ¿había alertas?

### Sobre no tener números

No inventes cifras. Puedes decir:

> "No tengo el número exacto, pero la tabla de auditoría era la más grande de la base y, al sacar todo lo anterior a 45 días, el tamaño bajó de forma muy notoria."

Solo dilo si es cierto. Si puedes, estima un orden de magnitud ("cientos de GB", "millones de registros").

> 💡 Evita decir "decreció **exponencialmente**". Técnicamente significa otra cosa y un entrevistador técnico lo puede notar. Di "**se redujo muchísimo**" o "**de forma muy significativa**".

### Sobre "el diseño lo hizo el arquitecto"

**No es algo malo**: tú hiciste **toda la implementación**, y eso es mucho. Frase útil:

> "La decisión de arquitectura la tomó el arquitecto, pero toda la implementación fue mía: la integración con S3 y el job de migración. Para hacerlo bien tuve que entender a fondo por qué S3 y el corte de 45 días eran la mejor opción para este tipo de datos."

---

## Preguntas que te van a hacer después (con respuestas)

**1. ¿Por qué S3 y no otra opción?**
> "El diseño lo propuso el arquitecto, pero entendí el razonamiento: los logs de auditoría solo se agregan, crecen sin parar y lo antiguo casi no se consulta. Mantener todo en MySQL era pagar almacenamiento caro e índices enormes para datos que casi nadie lee. Por eso dejamos lo reciente en MySQL, donde se consulta rápido, y el histórico en S3, que es mucho más barato."

**2. ¿Por qué 45 días?**
> Si sabes el motivo, dilo (por ejemplo, que la mayoría de las consultas eran del último mes y medio). Si no: "Fue el corte que definió el arquitecto según el uso. Lo importante es que la mayoría de las consultas caían en ese rango, y lo más antiguo se consultaba muy poco."

**3. ¿Por qué el job corría los fines de semana?**
> "Para no afectar a los usuarios. Mover y borrar muchos registros genera carga en la base de datos, así que lo programé en horarios en los que nadie consultaba la información."

**4. ¿Cómo te aseguraste de que no se perdieran datos?**
> Cuenta lo que realmente hacía el job. Si no lo recuerdas, no lo inventes. Puedes explicar cómo lo haces hoy: "Primero escribir en S3, confirmar que se guardó, y recién ahí borrar de MySQL. Si el job se cae a la mitad, se vuelve a ejecutar sin duplicar ni perder nada porque es idempotente."

**5. Cuéntame qué salió mal.**
> "En las primeras pruebas los datos no llegaban a S3. Revisando, encontré que el cliente de S3 tenía mal configurada la región. Lo corregí, pero me dejó una lección: hoy valido la configuración de cada ambiente desde el inicio, y me aseguro de que si una escritura falla se registre un error claro, en lugar de enterarme porque los datos no aparecen."

**6. ¿Qué pasa si el job falla a la mitad?**
> Cuenta lo que pasaba en tu caso. Si no estaba resuelto, explica cómo lo harías hoy: "Procesar en lotes, borrar de MySQL solo lo que ya se confirmó en S3, usar keys determinísticas para que reintentar no duplique datos, y una alerta si el job falla."

**7. ¿Cómo se consultan los logs antiguos?**
> "Por rango de tiempo: lo de los últimos 45 días se consulta en MySQL y lo anterior está en S3." Agrega cómo se leía de S3 en tu caso. Ojo con las búsquedas que cruzan los dos rangos (ver la lista de lo que falta).

**8. Los logs de auditoría son sensibles, ¿cómo los proteges?**
> Lo que hayan hecho, más lo que sumarías hoy: "Bucket privado con Block Public Access, cifrado, permisos IAM mínimos y, para auditoría, **Object Lock** para que nadie pueda modificar ni borrar los registros."

**9. Si fueran 100 veces más datos, ¿qué cambiarías?**
> "Organizaría las keys por fecha (`año/mes/día`), usaría un formato columnar como **Parquet** y consultaría el histórico con **Athena**. Además, lifecycle rules para mover lo muy antiguo a Glacier."

**10. ¿Por qué no lo diseñaste tú?**
> "En ese momento no conocía S3 y el arquitecto tenía la experiencia. Yo me encargué de toda la implementación y lo aproveché para aprender: hoy entiendo cuándo conviene almacenamiento de objetos y cuándo una base relacional, y podría proponer un diseño así."

---

## Errores a evitar al contarla

- Contarla solo como "moví datos de un lado a otro". Lo valioso es el **problema**, el **porqué** (lo reciente en la BD, el histórico en S3) y lo que **aprendiste**.
- Quitarle mérito a tu parte: **toda la implementación fue tuya**, dilo claro.
- Inventar números o detalles: te van a pedir profundidad y se nota.
- Olvidar el ejemplo de la región: es un "qué salió mal" concreto y creíble.
