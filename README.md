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
│   │   ├── WorkflowCheckpoint.cs   # Snapshot de instancia suspendida
│   │   ├── WorkflowDefinition.cs   # Grafo compilado e inmutable
│   │   └── WorkflowExecutionResult.cs
│   ├── Persistence/
│   │   ├── IWorkflowCheckpointStore.cs       # Contrato de persistencia
│   │   └── InMemoryWorkflowCheckpointStore.cs # Implementación en memoria
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

**Stack de testing:** xUnit · Shouldly · Moq · NetArchTest · coverlet

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

---

### 🔜 Próximos pasos

La siguiente fase evoluciona FlowForge hacia una plataforma de gestión de procesos de negocio (BPM) completa, con capacidades equivalentes a sistemas como PEGA. Las entregas están ordenadas por dependencia: cada paso habilita los siguientes.

---

#### Paso 1 — Persistent Store (EF Core) · *prerequisito de todo lo siguiente*

Reemplaza `InMemoryWorkflowCheckpointStore` con una implementación SQL real como nuevo paquete `FlowForge.Persistence.EntityFramework`. El store debe exponer queries indexadas por `EventName`, `TimeoutAt` y `Status` para soportar el event dispatcher y el SLA monitor de forma eficiente.

```
FlowForge.Persistence.EntityFramework/
├── FlowForgeDbContext.cs
├── EfWorkflowCheckpointStore.cs     # IWorkflowCheckpointStore sobre EF Core
├── Migrations/
└── FlowForgeEfServiceCollectionExtensions.cs   # AddFlowForgeEfStore(connectionString)
```

Los métodos nuevos que el store debe implementar son:
- `FindByEventNameAsync(eventName)` — para el event dispatcher
- `FindExpiredAsync(asOf)` — para el SLA monitor y timeouts

---

#### Paso 2 — Case Management · *diferenciador principal*

Introduce la entidad `WorkflowCase`: una entidad de negocio con ciclo de vida propio que puede contener múltiples ejecuciones de workflow, sub-casos, historial de auditoría y documentos adjuntos. Es el salto conceptual clave: FlowForge deja de gestionar "ejecuciones" para gestionar **casos**.

```
FlowForge.CaseManagement/
├── Models/
│   ├── WorkflowCase.cs          # entidad raíz — CaseId, CaseType, Status, Data, ParentCaseId
│   ├── CaseHistoryEntry.cs      # registro inmutable de cada transición
│   └── CaseStatus.cs            # Open / InProgress / Suspended / Closed
├── ICaseRepository.cs           # separado de IWorkflowCheckpointStore
├── ICaseService.cs              # OpenAsync / TransitionAsync / CloseAsync / AttachDocumentAsync
└── CaseServiceCollectionExtensions.cs
```

Un caso sobrevive a múltiples ejecuciones de workflow — por ejemplo, una solicitud de crédito que pasa por evaluación, aprobación y desembolso son tres workflows distintos sobre el mismo `WorkflowCase`.

---

#### Paso 3 — SLA Engine · *sobre `WaitForEventActivity`*

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
│  Case Management    │  Rules Engine             │  paso 2 & 5
├─────────────────────────────────────────────────┤
│  SLA Engine         │  Event Bus               │  paso 3 & 4
├─────────────────────────────────────────────────┤
│  FlowForge.Core  ✅  (motor, builder, DI)       │  hoy
├─────────────────────────────────────────────────┤
│  Persistent Store — EF Core / SQL               │  paso 1
└─────────────────────────────────────────────────┘
```

> FlowForge parte con una ventaja estructural sobre sistemas BPM maduros: `WorkflowDefinition` inmutable, indexación O(1), `ExecuteAsync` que nunca lanza excepciones y un middleware pipeline componible. Cada capa nueva se construye *sobre* esa base sin modificarla.

---

*FlowForge · .NET 10 · MIT License*
