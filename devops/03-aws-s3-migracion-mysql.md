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
| **Bucket** | Contenedor de objetos. Nombre único global, vive en una región. |
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
| Bueno para | Datos transaccionales, pequeños y consultables | Archivos, blobs, **histórico**, **logs**, backups, data lake |
| Escala de almacenamiento | Limitada por el disco/instancia; crecer es caro | Prácticamente ilimitada |
| Costo por GB | Alto (disco de BD + réplicas + backups) | Muy bajo, y aún menor con IA/Glacier |

### ¿Por qué los logs de auditoría encajan tan bien en S3?

| Característica de un log de auditoría | Por qué encaja con S3 |
|---|---|
| **Solo se agregan registros**, nunca se modifican | S3 es ideal para escribir objetos y no tocarlos más (y con Object Lock se puede garantizar). |
| **Crece sin parar** (cada movimiento genera un registro) | Almacenamiento prácticamente ilimitado y barato. |
| **Se consulta poco**, y casi siempre **por rango de fechas** | Si las keys llevan la fecha (`año/mes/día`), se lee solo el prefijo necesario. |
| No necesita joins ni transacciones | No hace falta pagar una base relacional para guardarlo. |
| Lo viejo casi no se lee | Lifecycle rules → Standard-IA / Glacier. |

### Ejemplo de código: escribir un lote de eventos de auditoría (.NET)

```csharp
public class S3AuditLogWriter(IAmazonS3 s3, TimeProvider clock) : IAuditLogWriter
{
    private const string Bucket = "mi-app-audit-logs";

    public async Task WriteBatchAsync(IReadOnlyList<AuditEvent> events, CancellationToken ct)
    {
        var now = clock.GetUtcNow();
        // Prefijo por fecha → las búsquedas por fecha solo leen lo necesario
        var key = $"audit-logs/{now:yyyy/MM/dd}/{now:HHmmss}-{Guid.NewGuid()}.jsonl";

        var body = string.Join('\n', events.Select(e => JsonSerializer.Serialize(e))); // JSON Lines

        await s3.PutObjectAsync(new PutObjectRequest
        {
            BucketName  = Bucket,
            Key         = key,
            ContentBody = body,
            ContentType = "application/x-ndjson",
        }, ct);
    }
}
```

> En pruebas, el `IAmazonS3` apunta a un **mock local de S3** (por ejemplo LocalStack o MinIO) cambiando el `ServiceURL`. Así se prueba la integración sin tocar AWS real ni generar costos.

---

## Parte B: Mi historia (la pregunta de comportamiento)

### Los hechos (lo que realmente pasó)

| | |
|---|---|
| **Qué se migró** | Los **logs de auditoría**: registrábamos **cada movimiento** de los usuarios en MySQL. |
| **El problema** | La base de datos se volvió **muy grande**, empezó a ser **costosa** por el volumen y las **búsquedas por fechas antiguas eran lentas**. |
| **El diseño** | Lo definió el **arquitecto** del equipo. Yo no conocía S3 y fue un gran aprendizaje. |
| **Cómo se validó** | Todo se probó en **ambientes de prueba con un mock de S3**. |
| **Resultado** | La base de datos **se redujo muchísimo** y los **tiempos de carga de datos mejoraron**. No hay cifras exactas. |
| **Lo que aprendí** | A elegir el almacenamiento según cómo se usa el dato, y que **hacen falta más pruebas en ambientes de prueba**, porque **no todo salió a la primera**. |

### Versión para decir en voz alta (~2 minutos)

> **Situación.** En [COMPLETAR: empresa o tipo de sistema] teníamos un sistema que guardaba en MySQL un **log de auditoría de cada movimiento** que hacían los usuarios. Como se registraba absolutamente todo, esa información creció muchísimo: la base de datos se volvió muy grande, el costo empezó a subir por el volumen, y las búsquedas de registros de fechas antiguas eran lentas.
>
> **Tarea.** El arquitecto del equipo propuso sacar esos logs de la base relacional y moverlos a **Amazon S3**, y yo participé en [COMPLETAR: tu parte, por ejemplo "implementar la integración con S3", "la migración de los datos existentes" o "las pruebas"].
>
> **Acción.** Yo no conocía S3, así que lo primero fue entender el porqué del diseño: un log de auditoría **solo se escribe y casi nunca se modifica**, **crece sin parar** y se consulta poco, normalmente **por fechas**. Eso encaja muy bien con un almacenamiento de objetos barato y prácticamente ilimitado como S3, y deja MySQL para los datos transaccionales. Para no arriesgar producción, **validamos todo en ambientes de prueba usando un mock de S3**, así podíamos probar la escritura y la lectura de los logs sin depender de AWS real.
>
> **Resultado.** La base de datos **se redujo de forma muy significativa** y los **tiempos de carga de datos mejoraron** de forma notoria. No tengo las cifras exactas, pero la diferencia se notó claramente.
>
> **Aprendizaje.** Me quedo con dos cosas. Primero, a **elegir el almacenamiento según cómo se usa el dato**, en lugar de meter todo en la base relacional por costumbre. Y segundo, que **no todo salió a la primera**: [COMPLETAR: un ejemplo concreto]. Por eso hoy invertiría todavía más en pruebas, en un ambiente lo más parecido posible a producción, antes de migrar.

### Lo que todavía falta completar (rápido, pero importante)

Te lo van a preguntar casi seguro, así que tenlo claro antes de la entrevista:

- [ ] **Empresa o tipo de sistema** (aunque sea genérico: "un sistema de gestión para un banco", "una plataforma de e-commerce"...).
- [ ] **Tu parte exacta**: ¿qué hiciste tú con tus manos?
- [ ] **Qué fue lo que no salió a la primera.** Es lo más importante que falta: un ejemplo concreto vale mucho más que "hubo problemas".
- [ ] **Qué mock de S3 usaron** (¿LocalStack? ¿MinIO? ¿otro?).
- [ ] **Cómo se consultan hoy los logs antiguos** (¿desde la app leyendo de S3? ¿con Athena? ¿se descargan cuando se necesitan?).
- [ ] **Qué pasó con los datos viejos en MySQL**: ¿se migraron a S3 y luego se borraron? ¿solo se movió lo nuevo?

### Sobre no tener números

No inventes cifras. Puedes decir:

> "No tengo el número exacto, pero la tabla de auditoría era la más grande de la base y, al sacarla, el tamaño bajó de forma muy notoria."

Solo dilo si es cierto. Si puedes, pregúntale a un excompañero o estima un orden de magnitud ("cientos de GB", "millones de registros").

> 💡 Evita decir "decreció **exponencialmente**". Técnicamente significa otra cosa y un entrevistador técnico lo puede notar. Di "**se redujo muchísimo**" o "**de forma muy significativa**".

### Sobre "el diseño lo hizo el arquitecto"

**No es algo malo**, siempre que lo cuentes bien. Lo que el entrevistador quiere ver es:
1. Que **entiendes el porqué** del diseño (por eso está la tabla de "por qué los logs encajan en S3").
2. Que **aprendes rápido** algo que no conocías.
3. Qué fue **tu aporte** concreto.

Frase útil:

> "La decisión de arquitectura la tomó el arquitecto, pero yo me encargué de [tu parte], y para hacerlo bien tuve que entender a fondo por qué S3 era la mejor opción para este tipo de datos."

---

## Preguntas que te van a hacer después (con respuestas honestas)

**1. ¿Por qué S3 y no otra opción?**
> "El diseño lo propuso el arquitecto, pero entendí el razonamiento: los logs de auditoría solo se agregan, crecen sin parar y se consultan poco y por fechas. Mantenerlos en MySQL era pagar almacenamiento caro y tener índices enormes para datos que casi no se leen. S3 es mucho más barato por GB, no tiene límite práctico y, si se organizan las keys por fecha, se puede leer solo el rango necesario."

**2. ¿Cómo te aseguraste de que no se perdieran datos?**
> "Validamos todo en ambientes de prueba con un mock de S3 antes de tocar producción." Agrega aquí lo que hayan hecho para comparar (conteos de registros antes y después, revisión de muestras). **Si no lo recuerdas, no lo inventes.** Puedes sumarlo como aprendizaje: "hoy agregaría una verificación automática de conteos antes de borrar nada de MySQL".

**3. ¿Hubo caída del servicio?**
> [COMPLETAR con lo que pasó realmente.]

**4. ¿Qué pasa si falla la escritura en S3?**
Si no lo manejaron explícitamente, responde cómo lo harías hoy:
> "Reintentos con backoff. Para no perder eventos de auditoría, los dejaría en una cola (por ejemplo SQS) o en una tabla temporal hasta confirmar que se guardaron en S3."

**5. Los logs de auditoría son sensibles, ¿cómo los proteges?**
Lo que corresponda a lo que hicieron, más lo que sumarías hoy:
> "Bucket privado con Block Public Access, cifrado, permisos IAM mínimos (la app solo puede escribir y leer ese bucket) y, para auditoría, **Object Lock** para que nadie pueda modificar ni borrar los registros."

**6. ¿Cuánto se ahorró?**
> Usa la frase de la sección "Sobre no tener números".

**7. ¿Qué salió mal?**
> Tu ejemplo concreto + qué hicieron para resolverlo + qué cambiarías.

**8. Si fueran 100 veces más datos, ¿qué cambiarías?**
Respuesta de "lo que aprendí después":
> "Organizaría las keys por fecha (`año/mes/día`), guardaría en lotes en lugar de un objeto por evento, usaría un formato columnar como **Parquet** y consultaría con **Athena**. Además agregaría lifecycle rules para mover lo antiguo a Glacier."

**9. ¿Por qué no lo diseñaste tú?**
> "En ese momento no conocía S3, y el arquitecto tenía la experiencia. Lo aproveché para aprender: hoy entiendo cuándo conviene un almacenamiento de objetos y cuándo una base relacional, y podría proponer un diseño así."

---

## Errores a evitar al contarla

- Contarla solo como "moví datos de un lado a otro". Lo valioso es el **problema**, el **porqué** y lo que **aprendiste**.
- Decir "nosotros" todo el tiempo sin que se entienda qué hiciste **tú**.
- Inventar números o detalles: te van a pedir profundidad y se nota.
- Decir solo "no todo salió a la primera" sin un ejemplo.
- Olvidar el "qué aprendí".
