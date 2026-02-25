# FlowForge

Motor de ejecución de workflows para .NET 10, diseñado con énfasis en corrección estructural, rendimiento en runtime y extensibilidad progresiva.

---

## Tabla de contenidos

- [Estructura del proyecto](#estructura-del-proyecto)
- [Conceptos clave](#conceptos-clave)
- [Uso rápido](#uso-rápido)
- [Definir actividades](#definir-actividades)
- [Construir un workflow](#construir-un-workflow)
- [Validación](#validación)
- [Ejecución](#ejecución)
- [Protección contra ciclos](#protección-contra-ciclos)
- [Arquitectura interna](#arquitectura-interna)
- [Integración con Dependency Injection](#integración-con-dependency-injection)
- [Persistent Store — EF Core](#persistent-store--ef-core)
- [Case Management](#case-management)
- [SLA Engine](#sla-engine)
- [Sandbox — Demo interactivo](#sandbox--demo-interactivo)
- [Tests](#tests)
- [Roadmap](#roadmap)

---

## Estructura del proyecto

```
FlowForge/
├── FlowForge.Core/                 # Biblioteca principal — sin dependencias externas de negocio
│   ├── Abstractions/
│   │   ├── IActivity.cs            # Contrato de actividad ejecutable
│   │   └── IWorkflowEngine.cs      # Contrato público del motor de ejecución
│   ├── Builders/
│   │   └── WorkflowBuilder.cs      # Fluent API para definir workflows
│   ├── Exceptions/
│   │   └── WorkflowDefinitionException.cs
│   ├── Execution/
│   │   ├── WorkflowEngine.cs       # Implementación del motor
│   │   ├── WorkflowExecutionContext.cs
│   │   └── WorkflowExecutionOptions.cs
│   ├── Models/
│   │   ├── ActivityConnection.cs
│   │   ├── ActivityExecutionResult.cs
│   │   ├── ActivityOutcomes.cs     # Constantes Done / Failed
│   │   ├── ParallelGroup.cs        # Metadata de un grupo fork/join
│   │   ├── TerminationReason.cs
│   │   ├── WaitForEventKeys.cs     # Claves reservadas de contexto para IWaitActivity
│   │   ├── WorkflowCheckpoint.cs   # Snapshot de instancia suspendida
│   │   ├── WorkflowDefinition.cs   # Grafo compilado e inmutable
│   │   └── WorkflowExecutionResult.cs
│   ├── Persistence/
│   │   ├── IWorkflowCheckpointStore.cs       # Contrato de persistencia
│   │   └── InMemoryWorkflowCheckpointStore.cs # Implementación en memoria
│   ├── Serialization/
│   │   └── WorkflowCheckpointSerializer.cs   # Serializa/deserializa checkpoints con type tags
│   └── Validation/
│       ├── ValidationErrorCode.cs
│       ├── ValidationSeverity.cs
│       ├── WorkflowValidationError.cs
│       ├── WorkflowValidationResult.cs
│       └── WorkflowValidator.cs    # internal — detalle del builder
│
├── FlowForge.Extensions.DependencyInjection/   # Integración con el ecosistema .NET DI
│   └── FlowForgeServiceCollectionExtensions.cs # AddFlowForge() / AddFlowForge(options =>)
│
├── FlowForge.Persistence.EntityFramework/      # Store persistente — SQL Server, PostgreSQL, SQLite
│   ├── CheckpointEntity.cs                     # Fila de BD con columnas de proyección desnormalizadas
│   ├── FlowForgeDbContext.cs                   # DbContext con índices optimizados
│   ├── EfWorkflowCheckpointStore.cs            # Implementación de IWorkflowCheckpointStoreExtended
│   ├── IWorkflowCheckpointStore.Extended.cs    # Extiende el contrato con FindByEventName / FindExpired
│   ├── FlowForgeEfServiceCollectionExtensions.cs  # AddFlowForgeSqlServerStore / PostgreSql / Sqlite
│   └── Migrations/
│       └── MIGRATIONS_GUIDE.cs                 # Comandos dotnet-ef por proveedor
│
├── FlowForge.Persistence.EntityFramework.Tests/ # Tests de integración del store EF Core
│   └── EfWorkflowCheckpointStoreTests.cs
│
├── FlowForge.CaseManagement/                   # Case Management — ciclo de vida de casos de negocio
│   ├── Models/
│   │   ├── CaseStatus.cs                       # Open / InProgress / Suspended / Closed / Cancelled
│   │   ├── WorkflowCase.cs                     # Entidad raíz del caso — payload tipado con GetData<T>/SetData<T>
│   │   ├── CaseHistoryEntry.cs                 # Registro inmutable de auditoría + CaseEventType enum
│   │   ├── CaseAttachment.cs                   # Metadatos de documentos adjuntos (sin binarios)
│   │   ├── CaseComment.cs                      # Comentarios de usuarios y sistema
│   │   └── CaseWorkflowExecution.cs            # Registro de una ejecución de workflow dentro del caso
│   ├── Abstractions/
│   │   ├── ICaseRepository.cs                  # Contrato de persistencia — separado de IWorkflowCheckpointStore
│   │   └── ICaseService.cs                     # Contrato del servicio de aplicación
│   ├── Services/
│   │   └── CaseService.cs                      # Orquesta ciclo de vida y auditoría automática
│   └── CaseManagementServiceCollectionExtensions.cs  # AddFlowForgeCaseManagement()
│
├── FlowForge.CaseManagement.EntityFramework/   # Persistencia EF Core para Case Management
│   ├── Entities/
│   │   └── CaseEntities.cs                     # Entidades de BD: Case, History, Attachment, Comment, Execution
│   ├── FlowForgeCaseDbContext.cs               # DbContext con índices optimizados para las queries del servicio
│   ├── EfCaseRepository.cs                     # Implementación de ICaseRepository sobre EF Core
│   └── FlowForgeCaseEfServiceCollectionExtensions.cs  # AddFlowForgeCaseEfRepository()
│
├── FlowForge.CaseManagement.Tests/             # Tests de integración del Case Management
│   └── CaseServiceTests.cs
│
├── FlowForge.Sla/                              # SLA Engine — monitoreo de plazos y escalación
│   ├── Models/
│   │   ├── SlaDefinition.cs                    # Regla de SLA: Goal, Deadline, EscalateTo, OnBreachOutcome
│   │   ├── SlaOutcomes.cs                      # Constantes SlaBreached / SlaGoalMissed + SlaViolationType enum
│   │   └── SlaViolation.cs                     # Registro inmutable de una violación detectada
│   ├── Abstractions/
│   │   ├── ISlaRepository.cs                   # Contrato de persistencia para definiciones y violaciones
│   │   └── ISlaMonitor.cs                      # ISlaMonitor + ISlaEventPublisher
│   ├── Services/
│   │   ├── SlaDefinitionRegistry.cs            # Registro en memoria de SLAs definidos en código (defaults)
│   │   ├── SlaMonitor.cs                       # Motor principal: evalúa checkpoints, fuerza outcomes
│   │   ├── SlaMonitorBackgroundService.cs      # BackgroundService que hace tick al monitor
│   │   └── NullSlaEventPublisher.cs            # No-op por defecto — reemplaza con tu implementación
│   └── SlaServiceCollectionExtensions.cs       # AddFlowForgeSla(options, registry)
│
├── FlowForge.Sla.EntityFramework/              # Persistencia EF Core para SLA
│   ├── Entities/
│   │   └── SlaEntities.cs                      # SlaDefinitionEntity + SlaViolationEntity
│   ├── FlowForgeSlaDbContext.cs                # DbContext con índices para resolución y dashboards
│   ├── EfSlaRepository.cs                      # Implementación de ISlaRepository — SQLite-safe
│   └── FlowForgeSlaEfServiceCollectionExtensions.cs  # AddFlowForgeSlaEfRepository()
│
├── FlowForge.Sla.Tests/                        # Tests del SLA Engine
│   └── SlaTests.cs
│
├── FlowForge.Sandbox/              # Demo interactivo — aprobación de préstamo
│   ├── Program.cs                  # Flujo completo: ejecutar → suspender → decisión humana → reanudar
│   ├── Activities/
│   │   └── WriteLineActivity.cs
│   └── Usings/
│       └── GlobalUsings.cs
│
└── FlowForge.Core.Tests/           # Proyecto de tests
    ├── Architecture/               # Reglas estructurales con NetArchTest
    ├── Builders/                   # Tests del builder y validador
    ├── DependencyInjection/        # Tests de AddFlowForge
    ├── Executions/                 # Tests del engine, middleware y ejecución paralela
    ├── Helpers/                    # Dobles de prueba reutilizables
    └── Models/                    # Tests de WorkflowDefinition y resultados
```

---

## Conceptos clave

| Concepto | Descripción |
|---|---|
| `IActivity` | Unidad de trabajo. Implementa `ExecuteAsync` y retorna un `ActivityExecutionResult` con un *outcome* string. |
| `WorkflowBuilder` | Fluent API que construye el grafo. Llama a `Build()` para obtener la definición compilada. |
| `WorkflowDefinition` | Grafo inmutable con índices O(1) preconstruidos. Solo el builder puede instanciarla. |
| `IWorkflowEngine` | Contrato del motor. La implementación concreta es `WorkflowEngine`. |
| *Outcome* | String que retorna una actividad para determinar qué conexión seguir. Las constantes predefinidas son `ActivityOutcomes.Done` y `ActivityOutcomes.Failed`. |

---

## Uso rápido

```csharp
// 1. Definir actividades
sealed class EnviarEmailActivity : IActivity
{
    public string Id   { get; init; } = Guid.NewGuid().ToString();
    public string Name { get; init; } = "Enviar Email";

    public async Task<ActivityExecutionResult> ExecuteAsync(
        WorkflowExecutionContext context,
        CancellationToken cancellationToken = default)
    {
        var destinatario = context.GetVariable<string>("destinatario");
        await emailService.SendAsync(destinatario!, cancellationToken);
        return ActivityExecutionResult.Success();
    }
}

// 2. Construir el workflow
var workflow = new WorkflowBuilder()
    .WithName("Proceso de Bienvenida")
    .StartWith(new ValidarUsuarioActivity())
    .Then(new EnviarEmailActivity())
    .Then(new RegistrarAuditoriaActivity())
    .Build();

// 3. Ejecutar
IWorkflowEngine engine = new WorkflowEngine(logger);

var context = new WorkflowExecutionContext();
context.SetVariable("destinatario", "usuario@ejemplo.com");

var result = await engine.ExecuteAsync(workflow, context);

if (result.IsSuccess)
    Console.WriteLine($"Completado en {result.Duration.TotalMilliseconds:F0}ms");
else
    Console.WriteLine($"Falló: {result.ErrorMessage} ({result.TerminationReason})");
```

---

## Definir actividades

Implementa `IActivity`. Los campos `Id` y `Name` son **init-only** — se asignan en la construcción y no pueden mutarse después, garantizando que los índices del grafo nunca queden desincronizados.

```csharp
public sealed class ProcesarPagoActivity : IActivity
{
    public string Id   { get; init; } = Guid.NewGuid().ToString();
    public string Name { get; init; } = "Procesar Pago";

    public async Task<ActivityExecutionResult> ExecuteAsync(
        WorkflowExecutionContext context,
        CancellationToken cancellationToken = default)
    {
        var monto = context.GetVariable<decimal>("monto");

        var aprobado = await pagoService.ProcesarAsync(monto, cancellationToken);

        // El outcome determina qué rama seguirá el engine
        return aprobado
            ? ActivityExecutionResult.Success("Approved")
            : ActivityExecutionResult.Success("Rejected");
    }
}
```

---

## Construir un workflow

### Flujo lineal

```csharp
var workflow = new WorkflowBuilder()
    .WithName("Flujo Lineal")
    .StartWith(new PasoA())
    .Then(new PasoB())
    .Then(new PasoC())
    .Build();
```

### Flujo condicional (múltiples outcomes)

```csharp
var validar  = new ValidarPedidoActivity();
var aprobar  = new AprobarPedidoActivity();
var rechazar = new RechazarPedidoActivity();
var notificar = new NotificarActivity();

var workflow = new WorkflowBuilder()
    .WithName("Aprobación de Pedido")
    .StartWith(validar)
    .Then(aprobar,  "Approved")   // validar → aprobar  [Approved]
    .Then(rechazar, "Rejected")   // validar → rechazar  [Rejected]
    .Then(notificar)              // aprobar → notificar [Done]
    .Connect(rechazar.Id, notificar.Id)  // rechazar → notificar [Done]
    .Build();
```

### `Connect` — conexiones manuales

Cuando la API fluida no es suficiente para expresar el grafo, usa `Connect` para declarar conexiones arbitrarias:

```csharp
builder.Connect(sourceId, targetId, outcome: "Retry");
```

---

## Validación

`Build()` valida el grafo antes de compilar los índices. Los **errores** bloquean la compilación; las **advertencias** no.

```csharp
// Opción A — validar antes de Build para obtener feedback detallado
var validation = builder.Validate();

foreach (var error in validation.Errors)
    Console.WriteLine($"[{error.Severity}] {error.Code}: {error.Message}");

// Opción B — Build lanza WorkflowDefinitionException si hay errores
try
{
    var workflow = builder.Build();
}
catch (WorkflowDefinitionException ex)
{
    foreach (var error in ex.ValidationResult.Errors)
        logger.LogError("{Code}: {Message}", error.Code, error.Message);
}
```

### Códigos de validación

| Código | Severidad | Descripción |
|---|---|---|
| `NoActivities` | Error | El workflow no tiene actividades. |
| `NoStartActivity` | Error | No se llamó a `StartWith()`. |
| `StartActivityNotFound` | Error | El ID de inicio no existe en el grafo. |
| `DuplicateActivityId` | Error | Dos actividades comparten el mismo `Id`. |
| `DuplicateConnection` | Error | Mismo `(source, outcome)` con dos destinos. |
| `ConnectionSourceNotFound` | Error | `sourceId` de una conexión no existe. |
| `ConnectionTargetNotFound` | Error | `targetId` de una conexión no existe. |
| `MissingWorkflowName` | Warning | No se llamó a `WithName()`. |
| `UnreachableActivity` | Warning | Actividad registrada pero nunca alcanzable. |
| `CycleDetected` | Warning | Ciclo dirigido detectado. Válido, pero requiere condición de salida. |

---

## Ejecución

### `TerminationReason` — por qué terminó el workflow

```csharp
var result = await engine.ExecuteAsync(workflow, context, options, cancellationToken);

switch (result.TerminationReason)
{
    case TerminationReason.Completed:
        // Fin normal: el grafo llegó a un nodo sin salidas
        break;
    case TerminationReason.ActivityFailed:
        // Una actividad retornó IsSuccess = false
        logger.LogError("Actividad falló: {Error}", result.ErrorMessage);
        break;
    case TerminationReason.MaxStepsExceeded:
        // Posible ciclo sin condición de salida
        logger.LogCritical("Workflow detenido por seguridad tras {Steps} pasos", options.MaxSteps);
        break;
    case TerminationReason.Cancelled:
        // CancellationToken fue cancelado
        break;
    case TerminationReason.UnhandledException:
        // Excepción no controlada dentro de una actividad
        logger.LogCritical("Excepción inesperada: {Error}", result.ErrorMessage);
        break;
}
```

> `ExecuteAsync` **nunca lanza excepciones**. Todos los errores — incluyendo excepciones no controladas dentro de actividades — se encapsulan en el resultado.

### Ejecución paralela — Fork / Join

Bifurca el flujo en ramas independientes que se ejecutan al mismo tiempo con `Task.WhenAll`:

```csharp
var workflow = new WorkflowBuilder()
    .WithName("Procesar pedido")
    .StartWith(validarPedido)
    .Fork(
        rama => rama.Then(verificarCredito),
        rama => rama.Then(verificarInventario))
    .Join(procesarPago)
    .Then(enviarConfirmacion)
    .Build();
```

Las ramas reciben un **snapshot del contexto** al momento del fork — las escrituras de una rama no son visibles en las otras. Al join, las variables de todas las ramas se fusionan de vuelta al contexto principal:

```csharp
// En cada rama puedes escribir variables con SetVariable(...)
// Al llegar al Join, todas esas variables están disponibles:
var credito     = context.GetVariable<bool>("creditoAprobado");
var inventario  = context.GetVariable<bool>("inventarioDisponible");
```

Si una o más ramas fallan, el engine espera a que **todas** terminen y retorna `BranchFailed`. El nodo join no se ejecuta:

```csharp
if (result.TerminationReason == TerminationReason.BranchFailed)
    logger.LogError("Fallo en ramas paralelas: {Error}", result.ErrorMessage);
```

### Suspensión y reanudación — Persistencia de instancias

Para workflows de larga duración (aprobaciones humanas, procesos multi-día), implementa `IWaitActivity` en cualquier actividad que deba pausar el flujo:

```csharp
public sealed class EsperarAprobacionActivity : IWaitActivity
{
    public string Id   { get; init; } = Guid.NewGuid().ToString();
    public string Name { get; init; } = "Esperar aprobación";

    public Task<ActivityExecutionResult> ExecuteAsync(
        WorkflowExecutionContext context,
        CancellationToken cancellationToken = default)
    {
        // Código de setup: enviar email, registrar en BD, etc.
        context.SetVariable("esperandoDesde", DateTimeOffset.UtcNow);
        return Task.FromResult(ActivityExecutionResult.Success());
    }
}
```

Cuando el engine encuentra una `IWaitActivity`, la ejecuta y luego suspende devolviendo un `WorkflowCheckpoint`:

```csharp
var result = await engine.ExecuteAsync(workflow);

if (result.TerminationReason == TerminationReason.Suspended)
{
    // Persistir el checkpoint para reanudarlo más tarde
    await store.SaveAsync(result.Checkpoint!);
}
```

Para reanudar, carga el checkpoint y llama a `ResumeAsync` con el outcome de la decisión:

```csharp
// Días o semanas después...
var checkpoint = await store.LoadAsync(instanceId);
var resumed    = await engine.ResumeAsync(checkpoint!, workflow, resumeOutcome: "Approved");
```

El grafo puede tener múltiples salidas desde una `IWaitActivity` — el outcome de reanudación decide el camino:

```csharp
var wait = new EsperarAprobacionActivity();

var workflow = new WorkflowBuilder()
    .WithName("Aprobación de pedido")
    .StartWith(crearPedido)
    .Then(wait)
    .Register(procesarPago)
    .Register(notificarRechazo)
    .Connect(wait.Id, procesarPago.Id,    outcome: "Approved")
    .Connect(wait.Id, notificarRechazo.Id, outcome: "Rejected")
    .Build();
```

Para registrar el store en memoria (desarrollo y tests):

```csharp
builder.Services
    .AddFlowForge()
    .AddFlowForgeInMemoryCheckpointStore();
```

### Contexto compartido entre actividades

```csharp
var context = new WorkflowExecutionContext();
context.SetVariable("pedidoId", 12345);
context.SetVariable("monto", 299.99m);

var result = await engine.ExecuteAsync(workflow, context);

// Las actividades pueden escribir datos en el contexto
var factura = result.Context?.GetVariable<string>("facturaUrl");
```

---

## Protección contra ciclos

Los ciclos son válidos en FlowForge (reintentos, aprobaciones iterativas). El sistema los maneja en dos capas:

**Compile-time** — `Build()` detecta ciclos dirigidos con DFS tricolor y emite un warning con el camino exacto:

```
[Warning] CycleDetected: Validar Pedido —[Retry]→ Procesar Pago —[Done]→ Validar Pedido
```

**Runtime** — el engine cuenta los pasos ejecutados y detiene la ejecución si se supera `MaxSteps`:

```csharp
var result = await engine.ExecuteAsync(
    workflow,
    options: new WorkflowExecutionOptions { MaxSteps = 50 });

// Si el ciclo no tiene condición de salida:
// result.TerminationReason == TerminationReason.MaxStepsExceeded
```

El valor por defecto de `MaxSteps` es **1 000**, suficiente para cualquier workflow lineal y para ciclos de reintento razonables.

---

## Arquitectura interna

### Indexación O(1) en `Build()`

Cuando se llama a `Build()`, el builder compila dos índices:

```
activitiesById:   Dictionary<string, IActivity>
connectionIndex:  Dictionary<(sourceId, outcome), targetId>
```

El loop de ejecución del engine realiza exactamente dos lookups por paso, ambos O(1), independientemente del tamaño del grafo.

### Inmutabilidad de `WorkflowDefinition`

- Constructor `internal` — solo `WorkflowBuilder` puede instanciarla.
- `Id` y `Name` de `IActivity` son `init`-only — los índices nunca quedan desincronizados.
- Los índices internos son `IReadOnlyDictionary` — el grafo no puede mutarse después de `Build()`.

### `IWorkflowEngine` como contrato público

`WorkflowEngine` implementa `IWorkflowEngine`. Programar contra la interfaz habilita:

```csharp
// Registro en DI
services.AddScoped<IWorkflowEngine, WorkflowEngine>();

// Decoradores sin modificar el Core
public sealed class TelemetryEngine(IWorkflowEngine inner) : IWorkflowEngine
{
    public async Task<WorkflowExecutionResult> ExecuteAsync(...)
    {
        using var span = tracer.StartActiveSpan("workflow.execute");
        var result = await inner.ExecuteAsync(...);
        span.SetAttribute("termination", result.TerminationReason.ToString());
        return result;
    }
}
```

### Fallback de outcome

Si una actividad retorna un outcome para el que no existe conexión, el engine intenta automáticamente la conexión `Done` del mismo origen antes de terminar:

```
outcome "CustomOutcome" → no hay conexión → intenta [Done] → sigue si existe
```

---

## Integración con Dependency Injection

Añade el paquete `FlowForge.Extensions.DependencyInjection` y registra los servicios en `Program.cs`:

```csharp
// Registro básico — usa WorkflowExecutionOptions.Default
builder.Services.AddFlowForge();

// Registro con configuración personalizada
builder.Services.AddFlowForge(options =>
{
    options.MaxSteps = 200; // límite más conservador para este servicio
});
```

Luego inyecta `IWorkflowEngine` en cualquier clase:

```csharp
public class PedidoService(IWorkflowEngine engine)
{
    public async Task ProcesarAsync(Pedido pedido)
    {
        var context = new WorkflowExecutionContext();
        context.SetVariable("pedido", pedido);

        var result = await engine.ExecuteAsync(_workflowPedidos, context);

        if (!result.IsSuccess)
            throw new PedidoException(result.ErrorMessage);
    }
}
```

> `AddFlowForge` usa `TryAdd` internamente: si registraste un decorador o un doble de test antes de llamarlo, ese registro no se sobreescribe.

---

## Persistent Store — EF Core

El paquete `FlowForge.Persistence.EntityFramework` reemplaza `InMemoryWorkflowCheckpointStore` con una implementación SQL real, compatible con SQL Server, PostgreSQL y SQLite.

### Serialización de checkpoints — `WorkflowCheckpointSerializer`

La serialización vive en `FlowForge.Core.Serialization.WorkflowCheckpointSerializer`. El store EF Core la delega completamente — no contiene lógica de serialización propia.

El deserializador resuelve dos problemas que `System.Text.Json` no puede manejar por sí solo sobre `WorkflowCheckpoint`:

**Diccionario privado** — `WorkflowExecutionContext` expone `_variables` como `private` para encapsular el acceso tipado. El serializer usa los métodos `internal` `GetVariables()` y `LoadVariables()` diseñados explícitamente para este fin, accesibles desde el mismo assembly.

**Tipos CLR en variables** — sin metadatos de tipo, `System.Text.Json` devuelve `JsonElement` al deserializar un `Dictionary<string, object?>`, lo que hace que `GetVariable<int>()` retorne `default`. El serializer resuelve esto con type tags cortos:

| Tag | Tipo CLR | Ejemplo serializado |
|---|---|---|
| `s` | `string` | `"hola mundo"` |
| `i` | `int` | `"42"` |
| `l` | `long` | `"9876543210"` |
| `f` | `double` | `"3.14"` |
| `d` | `decimal` | `"299.99"` |
| `b` | `bool` | `"1"` / `"0"` |
| `dt` | `DateTime` | `"2026-02-24T10:00:00.0000000"` |
| `dto` | `DateTimeOffset` | `"2026-02-24T10:00:00.0000000+00:00"` |
| `g` | `Guid` | `"550e8400-e29b-41d4-a716-446655440000"` |
| `ts` | `TimeSpan` | `"01:30:00"` |
| `j` | cualquier objeto | JSON anidado (fallback) |

Los tags son portables — no dependen de `AssemblyQualifiedName` y no se rompen al renombrar namespaces.

### Estrategia de almacenamiento

Cada checkpoint se almacena como una única fila en la tabla `FlowForge_Checkpoints`. El payload completo (`WorkflowCheckpoint`) se serializa a JSON mediante `WorkflowCheckpointSerializer` en la columna `CheckpointJson`. Las columnas `EventName`, `TimeoutAt` y `Status` se desnormalizan para soportar las queries del event dispatcher y el SLA monitor sin necesidad de deserializar el JSON en la consulta.

```
FlowForge_Checkpoints
─────────────────────────────────────────────────────────
InstanceId      PK  varchar(128)
EventName           varchar(256)   — proyectado desde contexto
TimeoutAt           datetimeoffset — proyectado desde contexto
Status              varchar(32)    — Suspended | Resumed | Expired
CheckpointJson      text           — payload JSON completo
CreatedAt           datetimeoffset
UpdatedAt           datetimeoffset
```

Los índices `IX_Checkpoints_EventName_Status` y `IX_Checkpoints_TimeoutAt_Status` garantizan que `FindByEventNameAsync` y `FindExpiredAsync` sean O(log n) sobre el store SQL.

### Registro por proveedor

```csharp
// SQL Server
builder.Services.AddFlowForgeSqlServerStore(
    connectionString: builder.Configuration.GetConnectionString("FlowForge")!);

// PostgreSQL
builder.Services.AddFlowForgePostgreSqlStore(
    connectionString: builder.Configuration.GetConnectionString("FlowForge")!);

// SQLite — desarrollo y tests de integración
builder.Services.AddFlowForgeSqliteStore("Data Source=flowforge.db");
```

Todos los métodos usan `TryAdd` internamente: si registraste un decorador antes de llamarlos, ese registro no se sobreescribe.

### Extensión del contrato de persistencia

El store EF Core implementa `IWorkflowCheckpointStoreExtended`, que amplía `IWorkflowCheckpointStore` con dos métodos nuevos:

```csharp
// Devuelve todos los checkpoints suspendidos que esperan un evento concreto
Task<IReadOnlyList<WorkflowCheckpoint>> FindByEventNameAsync(string eventName, CancellationToken ct);

// Devuelve todos los checkpoints suspendidos cuyo TimeoutAt <= asOf y los marca como Expired
Task<IReadOnlyList<WorkflowCheckpoint>> FindExpiredAsync(DateTimeOffset asOf, CancellationToken ct);
```

`FindExpiredAsync` marca atómicamente los registros como `Expired` en la misma operación, evitando que el SLA Monitor los procese dos veces en ejecuciones consecutivas.

### Migraciones

Aplica las migraciones pendientes al arrancar la aplicación:

```csharp
await using var scope = app.Services.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<FlowForgeDbContext>();
await db.Database.MigrateAsync();
```

Para crear la migración inicial con `dotnet-ef`:

```bash
# SQL Server
dotnet ef migrations add InitialCreate \
  --project FlowForge.Persistence.EntityFramework \
  --startup-project <tu-proyecto-de-startup>

# SQLite (desarrollo / tests)
# En tests usa EnsureCreated() en lugar de migraciones
dbContext.Database.EnsureCreated();
```

### Configuración avanzada

El segundo parámetro de cada método acepta un `Action<DbContextOptionsBuilder>` para configuración adicional:

```csharp
builder.Services.AddFlowForgeSqlServerStore(connectionString, options =>
{
    options.EnableSensitiveDataLogging(); // solo en desarrollo
    options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
});
```

---

## Case Management

El paquete `FlowForge.CaseManagement` introduce la entidad `WorkflowCase` — una unidad de negocio con ciclo de vida propio que sobrevive a múltiples ejecuciones de workflow. Es el salto conceptual clave respecto al motor puro: FlowForge pasa de gestionar *ejecuciones* a gestionar *casos*.

### Modelo de datos

Un caso tiene cinco entidades relacionadas, todas cargadas por el repositorio:

```
WorkflowCase
├── CaseHistoryEntry[]    — auditoría inmutable de cada transición y evento
├── CaseAttachment[]      — metadatos de documentos adjuntos (sin binarios)
├── CaseComment[]         — comentarios de usuarios y del sistema
├── CaseWorkflowExecution[] — registro de cada ejecución de workflow dentro del caso
└── SubCases (WorkflowCase[]) — casos hijos con su propio ciclo de vida
```

### Ciclo de vida

```
Open → InProgress → Suspended → InProgress → Closed
  ↘                                         ↗
   ────────────── Cancelled ───────────────
```

Las transiciones desde `Closed` o `Cancelled` lanzan `InvalidOperationException` — son estados terminales.

### Payload de negocio

El campo `DataJson` almacena el payload del caso serializado. Accede a él con los helpers tipados:

```csharp
// Escritura — serializa automáticamente a JSON
@case.SetData(new SolicitudCredito { Monto = 250_000m, Solicitante = "Carlos Mendoza" });

// Lectura — deserializa de vuelta al tipo original
var solicitud = @case.GetData<SolicitudCredito>();
Console.WriteLine(solicitud?.Monto); // 250000
```

### Registrar servicios

```csharp
// Registra ICaseService + CaseService
builder.Services.AddFlowForgeCaseManagement();

// Registra ICaseRepository con EF Core (proveedor genérico)
builder.Services.AddFlowForgeCaseEfRepository(options =>
    options.UseSqlServer(connectionString));

// Aplica migraciones al arrancar
await using var scope = app.Services.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<FlowForgeCaseDbContext>();
await db.Database.MigrateAsync();
```

### Ciclo de vida completo — ejemplo

```csharp
// 1. Abrir el caso
var @case = await caseService.OpenAsync(
    caseType:    "LoanApplication",
    title:       "Préstamo Carlos Mendoza",
    description: "Solicitud de crédito hipotecario",
    createdBy:   "oficina-norte");

@case.SetData(new SolicitudCredito { Monto = 250_000m });

// 2. Iniciar el primer workflow — transiciona a InProgress automáticamente
await caseService.RecordWorkflowStartedAsync(@case.CaseId, "EvaluacionRiesgo", workflowInstanceId);

// El workflow se suspende esperando decisión del comité
await caseService.RecordWorkflowSuspendedAsync(@case.CaseId, workflowInstanceId);

// Días después — el comité aprueba
await caseService.RecordWorkflowResumedAsync(@case.CaseId, workflowInstanceId);
await caseService.RecordWorkflowCompletedAsync(@case.CaseId, workflowInstanceId);

// 3. Segundo workflow — desembolso
await caseService.RecordWorkflowStartedAsync(@case.CaseId, "Desembolso", instanceId2);
await caseService.RecordWorkflowCompletedAsync(@case.CaseId, instanceId2);

// 4. Adjuntar documentos
await caseService.AddAttachmentAsync(@case.CaseId, "contrato.pdf", "application/pdf",
    sizeBytes: 102_400, storageUri: "blob://contratos/contrato.pdf");

// 5. Cerrar el caso
await caseService.CloseAsync(@case.CaseId, actorId: "sistema");
```

Cada operación registra automáticamente su entrada en el historial de auditoría — sin código adicional del caller.

### Auditoría — historial completo

```csharp
var history = await caseService.GetAsync(@case.CaseId);
// Cargado con historial completo por el repositorio

foreach (var entry in @case.History)
    Console.WriteLine($"[{entry.OccurredAt:HH:mm}] {entry.EventType}: {entry.Description}");

// [09:01] Created: Caso 'Préstamo Carlos Mendoza' creado.
// [09:01] WorkflowStarted: Workflow 'EvaluacionRiesgo' iniciado.
// [09:02] WorkflowSuspended: Workflow suspendido. InstanceId: ...
// [09:15] WorkflowResumed: Workflow reanudado.
// [09:15] WorkflowCompleted: Workflow completado.
// [09:16] WorkflowStarted: Workflow 'Desembolso' iniciado.
// [09:16] WorkflowCompleted: Workflow completado.
// [09:16] AttachmentAdded: Adjunto 'contrato.pdf' añadido.
// [09:16] StatusChanged: Estado cambiado de InProgress a Closed.
```

### Tablas en BD

```
FlowForge_Cases                 — entidad raíz del caso
FlowForge_CaseHistory           — auditoría inmutable (append-only)
FlowForge_CaseAttachments       — metadatos de adjuntos
FlowForge_CaseComments          — comentarios
FlowForge_CaseWorkflowExecutions — ejecuciones de workflow por caso
```

---

## SLA Engine

El paquete `FlowForge.Sla` monitorea los workflows suspendidos y actúa cuando se superan los plazos configurados. Tiene dos umbrales independientes por actividad: **Goal** (soft — notifica y escala) y **Deadline** (hard — fuerza un outcome en el workflow).

### Prioridad BD sobre código

Los SLAs pueden definirse en dos capas con prioridad clara:

```
BD (ISlaRepository)      ← mayor prioridad — configurable en runtime sin redesplegar
    ↓ fallback si null
Código (SlaDefinitionRegistry)  ← defaults declarados en Program.cs
```

El monitor resuelve la definición más específica disponible para cada par `(workflowName, activityId)` con esta prioridad adicional dentro de cada capa:

```
workflowName + activityId  →  coincidencia exacta (mayor prioridad)
workflowName solo          →  aplica a todas las actividades del workflow
wildcard (ambos null)      →  aplica a cualquier workflow y actividad
```

### Configurar SLAs

```csharp
// En código — defaults
builder.Services.AddFlowForgeSla(
    configureOptions: options =>
    {
        options.InitialDelay = TimeSpan.FromSeconds(10);
        options.Interval     = TimeSpan.FromSeconds(60);
    },
    configureRegistry: registry =>
    {
        // SLA específico: LoanWorkflow + wait-committee
        registry.Define(new SlaDefinition
        {
            WorkflowName    = "LoanWorkflow",
            ActivityId      = "wait-committee",
            Goal            = TimeSpan.FromHours(8),    // soft: notifica al supervisor
            Deadline        = TimeSpan.FromHours(24),   // hard: fuerza SlaBreached
            OnBreachOutcome = SlaOutcomes.Breached,
            EscalateTo      = "loan-supervisor-role",
        });

        // Wildcard: aplica a cualquier workflow sin SLA más específico
        registry.Define(new SlaDefinition
        {
            Deadline = TimeSpan.FromDays(7),
        });
    });

// Repositorio EF (BD tiene prioridad sobre el registry en código)
builder.Services.AddFlowForgeSlaEfRepository(options =>
    options.UseSqlServer(connectionString));
```

### Acciones al vencer un plazo

**Goal superado** — sin forzar el workflow:
- Registra `SlaViolation` con `ViolationType = GoalMissed`
- Escala al usuario/rol configurado en `EscalateTo`
- Publica `ISlaEventPublisher.PublishGoalMissedAsync`

**Deadline superado** — intervención sobre el workflow:
- Registra `SlaViolation` con `ViolationType = DeadlineBreached`
- Marca el checkpoint con `_slaBreachOutcome` y `_slaBreachAt` en el contexto
- Publica `ISlaEventPublisher.PublishDeadlineBreachedAsync`
- El Event Bus (Paso 4) reanuda el workflow con el outcome forzado

> El monitor es **idempotente** — si ya existe una `SlaViolation` del mismo tipo para la instancia, no la duplica.

### Conectar tu sistema de notificaciones

`ISlaEventPublisher` está registrado como `NullSlaEventPublisher` (no-op) por defecto. Reemplázalo con tu implementación:

```csharp
public sealed class EmailSlaEventPublisher(IEmailService email) : ISlaEventPublisher
{
    public async Task PublishGoalMissedAsync(SlaViolation violation, CancellationToken cancellationToken = default)
    {
        await email.SendAsync(
            to:      violation.EscalatedTo!,
            subject: $"SLA Goal superado — {violation.WorkflowName}",
            body:    $"El workflow {violation.WorkflowInstanceId} lleva " +
                     $"{(violation.DetectedAt - violation.SuspendedAt).TotalHours:F0}h suspendido.");
    }

    public Task PublishDeadlineBreachedAsync(SlaViolation violation, CancellationToken cancellationToken = default)
    {
        // Notificar breach crítico — Slack, PagerDuty, etc.
        return Task.CompletedTask;
    }
}

// Registrar antes de AddFlowForgeSla para que TryAdd no lo sobreescriba
builder.Services.AddSingleton<ISlaEventPublisher, EmailSlaEventPublisher>();
builder.Services.AddFlowForgeSla(...);
```

### Tablas en BD

```
FlowForge_SlaDefinitions   — reglas configurables en runtime (BD override)
FlowForge_SlaViolations    — registro inmutable de cada violación detectada
```

---

## Sandbox — Demo interactivo

`FlowForge.Sandbox` contiene un flujo completo de aprobación de préstamo que demuestra la integración de todos los features en un escenario real con interacción humana en consola.

**Flujo del demo:**

```
Recibir solicitud → Analizar riesgo → [SUSPENDER] → Aprobar préstamo
                                      IWaitActivity  ↘
                                                      Rechazar préstamo
```

**Ejecución:**

```
══════════════════════════════════════════════
  FlowForge — Aprobación de préstamo
══════════════════════════════════════════════

▶ Iniciando workflow...

  👤 Solicitante : Carlos Mendoza
  💰 Monto       : $250,000.00
  🆔 ID          : SOL-20260220143022
  📊 Score crediticio: 741
  📋 Resultado: Riesgo BAJO — apto para comité
  📧 Expediente enviado al comité de crédito.
  ⏸️  Workflow suspendido — pendiente de decisión humana.

══════════════════════════════════════════════
  👔 Decisión del comité [aprobar / rechazar]: aprobar
══════════════════════════════════════════════

▶ Reanudando con decisión: Approved

  🎉 Préstamo APROBADO
  📄 Número de crédito: CRED-482910
  👤 Titular: Carlos Mendoza | Monto: $250,000.00

══════════════════════════════════════════════
  Estado final  : ✅ Completado
  Actividades   : 5
  Duración total: 12ms
══════════════════════════════════════════════
```

El demo acepta `aprobar` / `a` / `si` para aprobar y cualquier otra entrada para rechazar. El checkpoint se persiste en `InMemoryWorkflowCheckpointStore` entre la suspensión y la reanudación, simulando el ciclo completo execute → save → load → resume.

---

## Tests

El proyecto `FlowForge.Core.Tests` cubre:

| Suite | Qué verifica |
|---|---|
| `WorkflowBuilderTests` | Todos los códigos de `ValidationErrorCode`, warnings vs errores, idempotencia de `Validate()`, conexiones multi-outcome. |
| `WorkflowEngineTests` | Todos los valores de `TerminationReason`, orden de ejecución, contexto compartido, fallback de outcome, logging con Moq + source generators. |
| `MiddlewarePipelineTests` | Orden de ejecución del pipeline, cortocircuito, modificación de outcome, timing middleware, integración con DI. |
| `ParallelExecutionTests` | Fork/Join con 2 y 3 ramas, aislamiento de contexto entre ramas, fusión de variables al join, fallo de rama (`BranchFailed`), ramas multi-paso, orden global inicio-fork-join-final. |
| `PersistenceTests` | Suspensión en `IWaitActivity`, checkpoint con contexto y historial, `ResumeAsync` con outcomes condicionales, múltiples puntos de espera, flujo completo execute→save→load→resume, operaciones del store. |
| `WorkflowDefinitionTests` | `GetActivity` O(1), `GetNextActivityId` con outcome exacto, fallback a Done, retorno null en nodo terminal. |
| `ActivityExecutionResultTests` | Factory methods, `GetOutput<T>` con tipo correcto/incompatible/null, integración con engine. |
| `WorkflowExecutionContextTests` | API tipada completa, encapsulación de `Variables`, validación de argumentos, variables compartidas. |
| `FlowForgeServiceCollectionExtensionsTests` | Registro, lifetime Scoped, opciones inyectadas, comportamiento `TryAdd`, encadenamiento. |
| `ArchitectureTests` | Reglas estructurales con NetArchTest: dependencias entre capas, visibilidad de interfaces públicas, `WorkflowValidator` internal. |
| `EfWorkflowCheckpointStoreTests` | `SaveAsync` / `LoadAsync` / `DeleteAsync` / `ListAsync`, proyección de columnas `EventName` y `TimeoutAt`, `FindByEventNameAsync`, `FindExpiredAsync` con marcado atómico, registro DI genérico, comportamiento `TryAdd`. Corre sobre SQLite in-memory sin infraestructura. |
| `CaseServiceTests` | `OpenAsync` con historial, transiciones válidas e inválidas desde terminales, ciclo completo multi-workflow, integración suspend/resume, comentarios, adjuntos, sub-casos, queries del repositorio (`FindByCaseType`, `FindByStatus`, `FindByAssignee`), registro DI. |
| `SlaTests` | `SlaDefinitionRegistry`: prioridad exacto→workflow→wildcard, reemplazo, definiciones inactivas. `EfSlaRepository`: persist/load con TimeSpan como ticks, `FindDefinitionAsync` por prioridad, idempotencia de `HasViolationAsync`, rango temporal. `SlaMonitor`: sin checkpoints, sin definición, Deadline forzado, Goal sin forzar, idempotencia, fallback al registry de código. |

**Stack de testing:** xUnit · Shouldly · Moq · NetArchTest · coverlet · Microsoft.Data.Sqlite

---

## Roadmap

### ✅ Completado

- [x] Fluent builder API (`StartWith` / `Then` / `Connect` / `Register` / `Build`)
- [x] Indexación O(1) en `Build()` — sin búsquedas lineales en ejecución
- [x] `WorkflowDefinition` inmutable con constructor `internal`
- [x] `IActivity` con propiedades `init`-only — índices siempre consistentes
- [x] Framework de validación con errores bloqueantes y advertencias
- [x] Detección de ciclos dirigidos con DFS tricolor y camino exacto en el mensaje
- [x] Guardia de runtime contra ciclos infinitos (`MaxSteps`)
- [x] `TerminationReason` tipado — sin strings mágicos en el resultado
- [x] `IWorkflowEngine` — contrato público para DI y decoradores
- [x] Logging estructurado con `[LoggerMessage]` source generators
- [x] Suite de tests: unitarios, integración y arquitectura
- [x] Eliminación de strings mágicos (`ActivityOutcomes.Done / Failed`)
- [x] `WorkflowExecutionContext.Variables` encapsulado — API tipada completa (`GetVariable<T>` / `SetVariable` / `ContainsVariable` / `RemoveVariable`)
- [x] `GetOutput<T>()` en `ActivityExecutionResult` — acceso tipado y seguro al output de una actividad sin riesgo de `InvalidCastException` (propiedad raw renombrada a `RawOutput` para claridad)
- [x] `FlowForge.Extensions.DependencyInjection` — `AddFlowForge()` con overload de configuración, `TryAdd` para respetar registros previos, `WorkflowEngine` con opciones inyectables por constructor
- [x] Middleware pipeline — `IActivityMiddleware` / `ActivityMiddlewareDelegate`, composición O(1) por ejecución, `AddFlowForgeMiddleware<T>()` para registro desde DI con orden preservado
- [x] Ejecución paralela de actividades — `Fork` / `Join` con `Task.WhenAll`, contexto hijo por rama, fusión al join con last-write-wins, `TerminationReason.BranchFailed` y `ForkNode` sintético para compatibilidad con el validador
- [x] Persistencia de instancias — `IWaitActivity` como punto de suspensión, `WorkflowCheckpoint` con contexto y historial, `ResumeAsync` con outcome configurable, `IWorkflowCheckpointStore` / `InMemoryWorkflowCheckpointStore`, `AddFlowForgeInMemoryCheckpointStore()` para DI
- [x] Persistent Store EF Core — `EfWorkflowCheckpointStore` sobre SQL Server / PostgreSQL / SQLite, serialización JSON en columna, índices O(log n) para `FindByEventNameAsync` y `FindExpiredAsync`, `IWorkflowCheckpointStoreExtended`, migraciones con `dotnet-ef`, `AddFlowForgeSqlServerStore` / `AddFlowForgePostgreSqlStore` / `AddFlowForgeSqliteStore`
- [x] `WorkflowCheckpointSerializer` — serialización robusta del checkpoint con type tags (`s` / `i` / `d` / `dto` / `j` …), resuelve el diccionario privado de `WorkflowExecutionContext` vía `GetVariables()` / `LoadVariables()` internos, compatible con todos los proveedores EF Core
- [x] Case Management — `WorkflowCase` con ciclo de vida (`Open→InProgress→Suspended→Closed/Cancelled`), `CaseHistoryEntry` inmutable, `CaseAttachment`, `CaseComment`, `CaseWorkflowExecution`, `ICaseRepository` / `ICaseService` / `CaseService`, persistencia EF Core (`FlowForgeCaseDbContext` + `EfCaseRepository`), `AddFlowForgeCaseManagement()` / `AddFlowForgeCaseEfRepository()`
- [x] SLA Engine — `SlaDefinition` con Goal (soft) y Deadline (hard), `SlaDefinitionRegistry` en código con prioridad BD→código, `SlaMonitor` idempotente con `BackgroundService` configurable, `ISlaEventPublisher` extensible, `SlaViolation` inmutable, `EfSlaRepository` con `TimeSpan` como ticks, `AddFlowForgeSla()` / `AddFlowForgeSlaEfRepository()`

---

### 🔜 Próximos pasos

La siguiente fase evoluciona FlowForge hacia una plataforma de gestión de procesos de negocio (BPM) completa, con capacidades equivalentes a sistemas como PEGA. Las entregas están ordenadas por dependencia: cada paso habilita los siguientes.

---

#### ~~Paso 1 — Persistent Store (EF Core)~~ · ✅ *Completado*

Store SQL real sobre EF Core con soporte para SQL Server, PostgreSQL y SQLite. Serialización JSON en columna, índices O(log n) para el event dispatcher y SLA monitor, y extensión del contrato base con `IWorkflowCheckpointStoreExtended`. Ver sección [Persistent Store — EF Core](#persistent-store--ef-core).

---

#### ~~Paso 2 — Case Management~~ · ✅ *Completado*

Entidad `WorkflowCase` con ciclo de vida propio, historial de auditoría inmutable, adjuntos, comentarios y sub-casos. Un caso puede contener múltiples ejecuciones de workflow secuenciales. Ver sección [Case Management](#case-management).

---

#### ~~Paso 3 — SLA Engine~~ · ✅ *Completado*

`SlaDefinition` con Goal y Deadline, `SlaDefinitionRegistry` con prioridad BD→código, `SlaMonitor` idempotente con `BackgroundService`, `ISlaEventPublisher` extensible (no-op por defecto). Ver sección [SLA Engine](#sla-engine).

---

#### Paso 4 — Event Bus

Expande el soporte de `Timeout` en `WaitForEventActivity` a un motor de SLAs completo con objetivos (goal), deadlines hard y escalación automática. Un `BackgroundService` dedicado — el **SLA Monitor** — evalúa periódicamente los checkpoints activos y fuerza outcomes de escalación cuando se vencen los plazos.

```
FlowForge.Sla/
├── Models/
│   └── SlaDefinition.cs         # ActivityId, Goal, Deadline, EscalateTo, OnBreachOutcome
├── ISlaRepository.cs
├── SlaMonitor.cs                # BackgroundService — tick cada minuto
└── SlaServiceCollectionExtensions.cs
```

El SLA Monitor reutiliza `FindExpiredAsync` del store (Paso 1) y el `WorkflowEventDispatcher` (Paso 4) para reanudar instancias con el outcome `SlaBreached` sin modificar el Core.

---

#### Paso 4 — Event Bus integrado · *sobre `WorkflowEventDispatcher`*

Abstrae el transporte de eventos detrás de `IWorkflowEventBus` e introduce adaptadores intercambiables como paquetes independientes. El dispatcher del Core no cambia — solo varía el adaptador registrado en DI.

```
FlowForge.Messaging/
├── IWorkflowEventBus.cs
└── InMemoryWorkflowEventBus.cs   # para desarrollo y tests

FlowForge.Messaging.MassTransit/
FlowForge.Messaging.AzureServiceBus/
FlowForge.Messaging.Kafka/
```

---

#### Paso 5 — Rules Engine · *sobre `IActivity`*

Permite definir reglas de negocio como tablas de decisión serializables (JSON / base de datos), evaluables en runtime sin recompilación. Cada regla se envuelve en una `RuleActivity` estándar — el motor no distingue entre una actividad codificada y una basada en reglas.

```
FlowForge.Rules/
├── IBusinessRule.cs
├── DecisionTableRule.cs          # filas condición → outcome, cargadas desde BD
├── RuleActivity.cs               # IActivity que delega en IBusinessRule
├── IRuleRepository.cs            # carga y versiona reglas desde BD
└── RulesServiceCollectionExtensions.cs
```

Las reglas se versionan: una nueva versión de una regla no afecta a instancias en vuelo que ya cargaron la versión anterior.

---

### 🔭 Futuro (post fase 2)

- **API REST** — endpoints para iniciar casos, consultar estado, disparar eventos y gestionar tareas humanas. Habilita integraciones externas sin acceso directo al Core.
- **UI de tareas humanas** — formularios Blazor generados a partir de metadatos del caso. Equivalente al portal de trabajo de PEGA.
- **Multi-tenancy** — aislamiento de datos por tenant en el store y el repositorio de casos. Prerequisito para oferta SaaS.
- **Reporting y auditoría** — dashboards de throughput, SLA compliance y cuello de botella por actividad, construidos sobre `CaseHistoryEntry`.
- **Designer visual** — editor de workflows drag-and-drop que genera definiciones compatibles con `WorkflowBuilder`. La definición compilada e inmutable del Core garantiza que cualquier grafo válido del designer sea ejecutable sin modificaciones.

---

### Visión de la arquitectura objetivo

```
┌─────────────────────────────────────────────────┐
│  UI — Blazor Portal / API REST                  │  futuro
├─────────────────────────────────────────────────┤
│  Case Management  ✅ │  Rules Engine             │  completado & paso 5
├─────────────────────────────────────────────────┤
│  SLA Engine  ✅      │  Event Bus               │  completado & paso 4
├─────────────────────────────────────────────────┤
│  FlowForge.Core  ✅  (motor, builder, DI)       │  hoy
├─────────────────────────────────────────────────┤
│  Persistent Store — EF Core  ✅                 │  completado
└─────────────────────────────────────────────────┘
```

> FlowForge parte con una ventaja estructural sobre sistemas BPM maduros: `WorkflowDefinition` inmutable, indexación O(1), `ExecuteAsync` que nunca lanza excepciones y un middleware pipeline componible. Cada capa nueva se construye *sobre* esa base sin modificarla.

---

*FlowForge · .NET 10 · MIT License*
