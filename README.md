# Task Manager API — GraphQL + NestJS

API GraphQL para la gestión de tareas de proyectos de desarrollo de software.

**Temas aplicados:** Programación Orientada a Aspectos (AOP), GitFlow y Clean Code.

---

## Tabla de contenidos

- [Requisitos](#requisitos)
- [Instalación y ejecución](#instalación-y-ejecución)
- [Modelo de datos](#modelo-de-datos)
- [Programación Orientada a Aspectos](#programación-orientada-a-aspectos)
- [Logging](#logging)
- [Documentación JSDoc](#documentación-jsdoc)
- [Arquitectura y Clean Code](#arquitectura-y-clean-code)
- [Ejemplos de consultas](#ejemplos-de-consultas)
- [GitFlow](#gitflow)

---

## Requisitos

- Node.js 18 o superior
- npm 9 o superior

## Instalación y ejecución

```bash
npm install
cp .env.example .env
npm run start:dev
```

El servidor queda disponible en `http://localhost:3000/graphql`.

| Script | Descripción |
|---|---|
| `npm run start:dev` | Modo desarrollo con recarga automática |
| `npm run build` | Compila a `dist/` |
| `npm run start:prod` | Ejecuta la versión compilada |
| `npm run lint` | Analiza el código |
| `npm run format` | Formatea con Prettier |
| `npm run docs` | Genera documentación HTML a partir del JSDoc |

El esquema GraphQL se genera automáticamente en `src/schema.gql` (enfoque *code-first*).

## Modelo de datos

La entidad `Task` cumple con todos los campos exigidos en el enunciado:

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | `ID` | Identificador único (UUID v4) |
| `title` | `String` | Título de la tarea |
| `description` | `String` | Descripción detallada |
| `status` | `TaskStatus` | `BACKLOG`, `TODO`, `IN_PROGRESS`, `DONE` |
| `tags` | `[String]` | Arreglo dinámico de etiquetas |
| `createdAt` | `Date` | Fecha de creación |
| `updatedAt` | `Date` | Fecha de última modificación |
| `assignedUser` | `User` | Usuario responsable |
| `project` | `Project` | Proyecto al que pertenece |

Los datos se almacenan **en memoria** detrás de la clase `TasksRepository`. Al estar
aislada la persistencia, migrar a PostgreSQL con TypeORM solo implica reemplazar esa
clase, sin tocar servicios ni resolvers.

## Programación Orientada a Aspectos

El objetivo de la AOP es separar los *cross-cutting concerns* (logging, auditoría,
medición de rendimiento, manejo de errores) de la lógica de negocio. En este proyecto
se aplican **dos mecanismos complementarios**:

### 1. Aspectos a nivel de método — decoradores propios

Ubicados en `src/common/aop/decorators/`. Implementan un *around advice* manual:
envuelven el método original (el *join point*) y le añaden comportamiento antes,
después y ante un error.

| Aspecto | Archivo | Qué hace |
|---|---|---|
| `@LogExecution()` | `log-execution.decorator.ts` | Registra entrada, salida, duración y errores |
| `@MeasurePerformance(ms)` | `measure-performance.decorator.ts` | Advierte si la operación supera un umbral |
| `@Audit(action, entity)` | `audit.decorator.ts` | Deja traza de quién modificó qué y cuándo |

Ejemplo real tomado de `TasksService`:

```ts
@LogExecution()
@Audit(AuditAction.DELETE, 'Task')
async remove(id: string): Promise<boolean> {
  await this.findById(id);
  return this.tasksRepository.delete(id);
}
```

El método de negocio **no contiene una sola línea de log**. Todo el comportamiento
transversal se inyecta desde fuera: esa es exactamente la idea de la AOP.

### 2. Aspectos a nivel de petición — mecanismos nativos de NestJS

Ubicados en `src/common/aop/`, registrados globalmente en `AopModule`.

| Aspecto | Archivo | Qué hace |
|---|---|---|
| `GraphqlLoggingInterceptor` | `interceptors/` | Traza toda consulta o mutación con un ID de correlación |
| `GraphqlExceptionFilter` | `filters/` | Traduce cualquier excepción a un `GraphQLError` uniforme |

El interceptor asigna un `requestId` (UUID) a cada operación, lo que permite seguir
una petición completa en los logs aunque haya varias en paralelo.

### Por qué dos niveles

El interceptor ve la petición *completa* pero no sabe qué métodos internos se
ejecutaron. Los decoradores ven cada método pero no el contexto GraphQL. Juntos
permiten pasar del síntoma (una mutación tardó 300 ms) a la causa (el método
`findAll` tardó 280 ms) sin instrumentar el código a mano.

## Logging

Implementado con **Winston** (`src/common/logger/`), configurado con tres transportes:

| Transporte | Destino | Contenido |
|---|---|---|
| Consola | `stdout` | Formato legible y coloreado por nivel |
| Archivo | `logs/application.log` | Historial completo en JSON, rotación a 5 MB |
| Archivo | `logs/error.log` | Solo errores, para revisión rápida de incidentes |

El nivel se controla con la variable `LOG_LEVEL` del `.env`. Los logs en archivo van
en JSON para que herramientas externas (ELK, Datadog) puedan indexarlos.

La utilidad `safeStringify` protege el logging de tres problemas: referencias
circulares, fuga de datos sensibles (enmascara `password`, `token`, `secret`) y
mensajes excesivamente largos.

Ejemplo de salida real:

```
2026-07-23 03:16:18.642 INFO    [GraphqlLoggingInterceptor] [4d806843-...] Query.tasks iniciada variables={}
2026-07-23 03:16:18.643 INFO    [TasksService] -> findAll() args=[]
2026-07-23 03:16:18.646 DEBUG   [TasksService:performance] findAll() tardo 0.15ms
2026-07-23 03:16:18.646 INFO    [TasksService] <- findAll() OK en 3ms
2026-07-23 03:16:18.646 INFO    [GraphqlLoggingInterceptor] [4d806843-...] Query.tasks completada en 4ms
```

## Documentación JSDoc

Todo el código está documentado con JSDoc: `@fileoverview` en cada archivo, y
`@param`, `@returns`, `@throws`, `@example` en clases y métodos públicos y privados.

Para generar la documentación navegable:

```bash
npm run docs   # genera la carpeta docs/
```

## Arquitectura y Clean Code

```
src/
├── common/                    # Código transversal
│   ├── aop/                   # Aspectos: decoradores, interceptores, filtros
│   ├── logger/                # Configuración de Winston
│   ├── exceptions/            # Excepciones de dominio
│   └── utils/                 # Utilidades puras
├── tasks/                     # Dominio principal
│   ├── dto/                   # Entradas validadas con class-validator
│   ├── entities/              # Entidad Task
│   ├── enums/                 # TaskStatus
│   └── repositories/          # Persistencia aislada
├── users/
└── projects/
```

Decisiones tomadas y su motivo:

- **Resolvers delgados.** Los resolvers solo traducen GraphQL a casos de uso; no
  contienen reglas de negocio.
- **Validación en los DTO, no en el servicio.** El servicio se ocupa de reglas de
  negocio; los formatos los verifica `class-validator`.
- **Excepciones de dominio propias.** `EntityNotFoundException` y
  `BusinessRuleException` evitan repetir mensajes de error por todo el código.
- **Persistencia detrás de un repositorio.** Inversión de dependencias: el servicio
  no sabe si los datos están en memoria o en una base de datos.
- **Transiciones de estado explícitas.** La constante `ALLOWED_TRANSITIONS` declara
  qué movimientos son válidos en el tablero, en vez de esconderlos en condicionales.
- **Sin números mágicos.** Umbrales y límites viven en constantes con nombre.

## Ejemplos de consultas

**Listar tareas con sus relaciones**

```graphql
query {
  tasks {
    id
    title
    status
    tags
    createdAt
    assignedUser { fullName email }
    project { name }
  }
}
```

**Filtrar tareas**

```graphql
query {
  tasks(filter: { status: IN_PROGRESS, tags: ["aop"] }) {
    id
    title
  }
}
```

**Crear una tarea**

```graphql
mutation {
  createTask(input: {
    title: "Documentar la API"
    description: "Escribir el README y los ejemplos de uso."
    status: TODO
    tags: ["docs"]
    assignedUserId: "<uuid-de-un-usuario>"
    projectId: "<uuid-de-un-proyecto>"
  }) {
    id
    title
    status
  }
}
```

> Los identificadores se obtienen con las consultas `users` y `projects`.

**Cambiar el estado**

```graphql
mutation {
  changeTaskStatus(input: { id: "<uuid>", status: IN_PROGRESS }) {
    id
    status
  }
}
```

**Agregar etiquetas y eliminar**

```graphql
mutation {
  addTaskTags(id: "<uuid>", tags: ["urgente", "backend"]) { id tags }
}

mutation {
  deleteTask(id: "<uuid>")
}
```

**Manejo de errores**

Una transición inválida devuelve un error controlado por el filtro de excepciones:

```json
{
  "errors": [{
    "message": "No se permite mover una tarea de DONE a BACKLOG.",
    "extensions": { "code": "CONFLICT", "status": 409 }
  }]
}
```

## GitFlow

El repositorio sigue el modelo GitFlow:

- `main` — código estable y evaluable
- `develop` — rama de integración
- `feature/*` — una rama por funcionalidad, integrada a `develop` con `--no-ff`

El detalle de las ramas y commits está en [`GITFLOW.md`](./GITFLOW.md).

---

**Autor:** Sebastián Moreno — Ingeniería Informática
