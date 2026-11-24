# Preguntas de entrevista técnica

Repositorio de estudio personal. Cada tema tiene:

1. **La pregunta** tal como suelen hacerla en entrevista.
2. **Respuesta corta** (la que dirías en 30–60 segundos).
3. **Explicación a fondo** para entender el porqué.
4. **Ejemplo de aplicación real** con código.
5. **Errores comunes / trampas** que suelen preguntar después.
6. **Preguntas de seguimiento** para practicar.

## Índice

### Angular
| # | Tema | Archivo |
|---|------|---------|
| 1 | Change detection y cuándo usar OnPush | [angular/01-change-detection-onpush.md](angular/01-change-detection-onpush.md) |
| 2 | Observable, Subject y BehaviorSubject | [angular/02-observable-subject-behaviorsubject.md](angular/02-observable-subject-behaviorsubject.md) |
| 3 | Cómo evitar memory leaks con RxJS | [angular/03-memory-leaks-rxjs.md](angular/03-memory-leaks-rxjs.md) |
| 4 | Inyección de dependencias y alcance de los providers | [angular/04-inyeccion-dependencias-providers.md](angular/04-inyeccion-dependencias-providers.md) |

### .NET / C#
| # | Tema | Archivo |
|---|------|---------|
| 1 | async/await y Task: qué pasa al hacer `await`, `Task` vs `Thread` | [dotnet/01-async-await-task-vs-thread.md](dotnet/01-async-await-task-vs-thread.md) |
| 2 | Inyección de dependencias: Singleton, Scoped y Transient (comparado con Angular) | [dotnet/02-inyeccion-dependencias-lifetimes.md](dotnet/02-inyeccion-dependencias-lifetimes.md) |
| 3 | IEnumerable vs IQueryable (LINQ y EF Core) | [dotnet/03-ienumerable-vs-iqueryable.md](dotnet/03-ienumerable-vs-iqueryable.md) |

### DevOps / Cloud
| # | Tema | Archivo |
|---|------|---------|
| 1 | Docker: image vs container y `Dockerfile` | [devops/01-docker-image-vs-container.md](devops/01-docker-image-vs-container.md) |
| 2 | Kubernetes: Pod, Deployment y Service | [devops/02-kubernetes-pod-deployment-service.md](devops/02-kubernetes-pod-deployment-service.md) |
| 3 | AWS S3 + mi migración de MySQL a S3 (historia STAR, **pendiente de completar**) | [devops/03-aws-s3-migracion-mysql.md](devops/03-aws-s3-migracion-mysql.md) |

## Cómo estudiar

- Lee la **respuesta corta** y trata de decirla en voz alta sin mirar.
- Después intenta responder las **preguntas de seguimiento** antes de volver a leer.
- Marca aquí lo que ya dominas:

- [ ] Change detection / OnPush
- [ ] Observable / Subject / BehaviorSubject
- [ ] Memory leaks con RxJS
- [ ] DI y providers
- [ ] async/await y Task vs Thread
- [ ] DI en .NET: Singleton / Scoped / Transient
- [ ] IEnumerable vs IQueryable
- [ ] Docker: image vs container
- [ ] Kubernetes: Pod / Deployment / Service
- [ ] S3 (técnico)
- [ ] Historia de la migración MySQL → S3 (completar y practicar en voz alta)
