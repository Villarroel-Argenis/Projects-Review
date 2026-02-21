# FlowForge — Case Management Roadmap

> Módulo que transforma el workflow engine del Core en una plataforma BPM con expedientes, ciclo de vida, SLA y auditoría completa. Equivalente al "Case Type" de PEGA.

---

## Estado actual del módulo

| Archivo | Capa | Estado |
|---|---|---|
| `Models/CaseStatus.cs` | Modelo | ✅ Completo |
| `Models/CaseAuditAction.cs` | Modelo | ✅ Completo |
| `Models/CaseAuditEntry.cs` | Modelo | ✅ Completo |
| `Models/Case.cs` | Entidad raíz | ✅ Completo |
| `Models/CaseStage.cs` | Modelo | ✅ Completo |
| `Models/CaseDefinition.cs` | Plantilla de tipo | ✅ Completo |
| `Models/StageDefinition.cs` | Modelo | ✅ Completo |
| `Models/CaseCreateOptions.cs` | DTO entrada | ✅ Completo |
| `Models/CaseAdvanceOptions.cs` | DTO entrada | ✅ Completo |
| `SLA/SlaUrgency.cs` | SLA | ✅ Completo |
| `SLA/SlaPolicy.cs` | SLA | ✅ Completo |
| `SLA/SlaMonitorBackgroundService.cs` | SLA | ✅ Completo |
| `Abstractions/ICaseService.cs` | Contrato | ✅ Completo |
| `Abstractions/ICaseDefinitionRegistry.cs` | Contrato | ✅ Completo |
| `Persistence/ICaseStore.cs` | Contrato persistencia | ✅ Completo |
| `Persistence/InMemoryCaseStore.cs` | Persistencia | ✅ Completo (dev/test) |
| `Persistence/InMemoryCaseDefinitionRegistry.cs` | Persistencia | ✅ Completo |
| `Services/CaseService.cs` | Servicio principal | ✅ Completo |
| `Activities/AdvanceStageActivity.cs` | Actividad | ✅ Completo |
| `Exceptions/CaseManagementExceptions.cs` | Excepciones | ✅ Completo |
| `Extensions/CaseManagementServiceCollectionExtensions.cs` | DI | ✅ Completo |

---

## Roadmap por fases

---

### Fase 1 — Persistencia real (EF Core)
**Objetivo:** Reemplazar `InMemoryCaseStore` por una implementación durable contra PostgreSQL o SQL Server.  
**Prioridad:** 🔴 Crítica — sin esto el módulo no es apto para producción.

#### Tareas

**1.1 — EfCoreCaseStore**
- Implementar `ICaseStore` con `DbContext` de EF Core
- Mapear `Case`, `CaseStage` y `CaseAuditEntry` como entidades con sus relaciones
- Configurar conversores para `Dictionary<string, object?>` (columna JSON)
- Implementar `GetActiveCasesWithSlaAsync` con índice por `Status` y `GlobalSlaDeadline`

```
Persistence/
├── EfCore/
│   ├── CaseManagementDbContext.cs
│   ├── EfCoreCaseStore.cs
│   ├── Configurations/
│   │   ├── CaseEntityConfiguration.cs
│   │   ├── CaseStageEntityConfiguration.cs
│   │   └── CaseAuditEntryEntityConfiguration.cs
│   └── Migrations/
│       └── (generadas con dotnet ef migrations add)
```

**1.2 — Migración de esquema**
```sql
-- Tablas mínimas
Cases          (Id, CaseTypeId, ReferenceNumber, Status, CurrentStageId,
                CreatedAt, UpdatedAt, ClosedAt, GlobalSlaDeadline,
                GlobalSlaUrgency, DataJson)

CaseStages     (Id, CaseId, StageId, StageName, StartedAt, CompletedAt,
                CompletionOutcome, ActiveCheckpointJson, SlaUrgency)

CaseAuditLog   (Id, CaseId, OccurredAt, Action, ActorId,
                StageId, Description, DataJson)
```

**1.3 — Índices recomendados**
- `Cases(Status, CaseTypeId)` — para queries de bandeja de trabajo
- `Cases(GlobalSlaDeadline)` WHERE Status IN ('Open','Pending') — para el SLA Monitor
- `CaseAuditLog(CaseId, OccurredAt DESC)` — para historial paginado
- `Cases(ReferenceNumber)` UNIQUE — para búsqueda por referencia

**1.4 — Unit of Work**
- Envolver `SaveAsync` en una transacción para garantizar que `Case` + `AuditEntry` se persistan atómicamente
- Agregar soporte de `ConcurrencyToken` (ETag/RowVersion) para detección de conflictos

---

### Fase 2 — API HTTP (ASP.NET Core)
**Objetivo:** Exponer el módulo como una API REST consumible desde frontends, integraciones y el portal BPM.  
**Prioridad:** 🔴 Crítica — sin endpoints no hay interfaz de operación.

#### Tareas

**2.1 — CasesController**
```
Controllers/
└── CasesController.cs
    POST   /cases                        → CreateAsync
    GET    /cases/{id}                   → GetAsync
    GET    /cases/ref/{referenceNumber}  → GetByReferenceAsync
    GET    /cases?caseTypeId=&status=    → QueryAsync
    POST   /cases/{id}/advance           → AdvanceAsync
    POST   /cases/{id}/resume            → ResumeAsync
    POST   /cases/{id}/cancel            → CancelAsync
    PATCH  /cases/{id}/data              → UpdateDataAsync
    GET    /cases/{id}/audit             → AuditLog paginado
```

**2.2 — DTOs de respuesta**
```
Dtos/
├── CaseSummaryDto.cs       // Id, ReferenceNumber, Status, CurrentStageName, SlaUrgency
├── CaseDetailDto.cs        // Todo lo anterior + Stages + últimas N entradas de auditoría
├── CaseStageDto.cs
└── AuditEntryDto.cs
```

**2.3 — Validación de entrada**
- Usar FluentValidation para `CaseCreateOptions` y `CaseAdvanceOptions`
- Retornar `ProblemDetails` (RFC 7807) en todos los errores

**2.4 — Autorización**
- Middleware que extrae `ActorId` del claim JWT y lo inyecta en las options
- Políticas por rol: `case:create`, `case:advance`, `case:cancel`

---

### Fase 3 — Sub-Cases y jerarquía
**Objetivo:** Un Case padre puede contener Cases hijo que se ejecutan en paralelo o en secuencia. Equivalente a los "Child Cases" de PEGA.  
**Prioridad:** 🟡 Alta — necesario para procesos complejos (ej. onboarding con sub-procesos de KYC, compliance y crédito en paralelo).

#### Tareas

**3.1 — Modelo de jerarquía**
```csharp
// Agregar a Case.cs
public string? ParentCaseId { get; private set; }
public IReadOnlyList<string> ChildCaseIds => _childCaseIds;
```

**3.2 — SpawnChildCaseActivity**
```
Activities/
└── SpawnChildCaseActivity.cs
    // Crea un Case hijo y registra su Id en el padre
    // Puede ejecutarse dentro de un Stage del padre para lanzar subprocesos
```

**3.3 — Awaitable child cases**
- `WaitForChildCasesActivity`: suspende el Case padre hasta que todos los hijos alcancen un status terminal
- El monitor de hijos comprueba periódicamente el estado y reanuda al padre cuando todos terminan

**3.4 — Propagación de estado**
- Si un hijo falla, el padre puede: ignorar, fallar también, o activar una rama de compensación
- Configuración en `StageDefinition.ChildFailurePolicy`

---

### Fase 4 — Versionado de CaseDefinitions
**Objetivo:** Mantener múltiples versiones de un CaseType activas simultáneamente. Los Cases existentes siguen la versión con la que fueron creados; los nuevos usan la última.  
**Prioridad:** 🟡 Alta — esencial en entornos productivos donde los procesos evolucionan sin detener instancias en vuelo.

#### Tareas

**4.1 — Clave compuesta de registro**
```csharp
// Cambiar la clave del registry de TypeId a (TypeId, Version)
registry.Register(definition);         // registra v2.0.0
registry.Get("LoanApproval");          // devuelve la última versión
registry.Get("LoanApproval", "1.0.0"); // devuelve v1.0.0 específica
```

**4.2 — Snapshot de definición en el Case**
- Al crear un Case, serializar y guardar la `CaseDefinition` completa dentro del Case
- Así, aunque la definición cambie, el Case siempre tiene la versión con la que fue creado

**4.3 — Migration paths**
- API para migrar Cases en vuelo de una versión a otra (con mapping de etapas)
- Registro de auditoría `DefinitionVersionMigrated`

---

### Fase 5 — Notificaciones y eventos de dominio
**Objetivo:** Emitir eventos cuando el Case cambia de estado, etapa o urgencia SLA, para que sistemas externos (email, Slack, webhooks) reaccionen sin polling.  
**Prioridad:** 🟡 Alta.

#### Tareas

**5.1 — Eventos de dominio**
```
Events/
├── CaseCreatedEvent.cs
├── CaseStageAdvancedEvent.cs
├── CaseStatusChangedEvent.cs
├── CaseSlaBreachedEvent.cs
├── CaseResolvedEvent.cs
└── CaseCancelledEvent.cs
```

**5.2 — ICaseDomainEventPublisher**
```csharp
public interface ICaseDomainEventPublisher
{
    Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : ICaseDomainEvent;
}
```
- Implementación con MediatR (in-process) para empezar
- Implementación con MassTransit / Azure Service Bus para producción distribuida

**5.3 — Integración en CaseService**
- El servicio publica eventos al final de cada operación, después de `store.SaveAsync`
- Los eventos son transaccionales: si el save falla, no se publican (Outbox Pattern recomendado en fase 5.4)

**5.4 — Outbox Pattern (opcional avanzado)**
- Tabla `CaseOutboxMessages` en la misma DB que `Cases`
- `OutboxRelayBackgroundService` lee y publica mensajes pendientes
- Garantía de entrega at-least-once

---

### Fase 6 — Búsqueda y reporting operacional
**Objetivo:** Dar visibilidad sobre todos los Cases en vuelo: cuántos están por etapa, cuántos incumplieron SLA, throughput diario, etc.  
**Prioridad:** 🟠 Media.

#### Tareas

**6.1 — Read models (proyecciones)**
```
Reporting/
├── Projections/
│   ├── ActiveCasesByStageProjection.cs   // { stageId → count }
│   ├── SlaBreachedCasesProjection.cs     // lista de casos vencidos
│   ├── CaseThroughputProjection.cs       // completados por día/semana
│   └── AverageCycleTimeProjection.cs     // tiempo promedio por tipo de Case
└── IProcessReportingService.cs
```

**6.2 — Endpoint de métricas**
```
GET /cases/metrics/summary           → totales por status y tipo
GET /cases/metrics/sla               → % cumplimiento SLA por tipo
GET /cases/metrics/throughput?days=7 → completados por día
```

**6.3 — Exportación**
- `GET /cases/export?format=csv` para reportes operacionales
- Integración con plantillas Excel via módulo `FlowForge.Reporting`

---

### Fase 7 — Portal web de operación (Bandeja de trabajo)
**Objetivo:** Interfaz visual para que operadores vean, gestionen y tomen decisiones sobre Cases sin necesidad de API directa.  
**Prioridad:** 🟠 Media — depende de Fase 2 y Fase 5.

#### Tareas

**7.1 — Bandeja de casos**
- Lista paginada y filtrable por tipo, estado, etapa, urgencia SLA
- Indicadores visuales de urgencia (verde / amarillo / rojo / rojo parpadeante)
- Acceso directo al expediente individual

**7.2 — Detalle del expediente**
- Vista completa del Case: datos, etapa actual, historial de auditoría
- Acciones: avanzar, cancelar, actualizar datos, agregar comentario
- Timeline visual de etapas completadas y pendientes

**7.3 — Dashboard de operación**
- KPIs en tiempo real: abiertos, pendientes, vencidos, resueltos hoy
- Gráficas de throughput y cycle time
- Alertas activas de SLA

**Tecnología sugerida:** Blazor Server (misma plataforma .NET) o React con la API de Fase 2.

---

### Fase 8 — Designer visual de CaseDefinitions
**Objetivo:** Que analistas de negocio puedan crear y modificar CaseTypes sin escribir código. Equivalente al "App Studio" de PEGA.  
**Prioridad:** 🔵 Baja — máximo valor, máximo esfuerzo.

#### Tareas

**8.1 — Serialización JSON de CaseDefinition**
- `CaseDefinitionSerializer`: `CaseDefinition` ↔ JSON
- Endpoint `GET /case-types/{id}/export` y `POST /case-types/import`

**8.2 — UI drag-and-drop de etapas**
- Canvas con etapas como nodos arrastrables
- Panel de propiedades por etapa: nombre, SLA, workflow asociado, auto-advance outcomes
- Validación en tiempo real (etapas duplicadas, SLA inválido, etc.)

**8.3 — Versionado desde el designer**
- Guardar nueva versión sin afectar Cases en vuelo
- Diff visual entre versiones

---

## Dependencias entre fases

```
Fase 1 (EF Core)
   └─→ Fase 2 (API HTTP)
           └─→ Fase 3 (Sub-Cases)
           └─→ Fase 5 (Eventos)
                   └─→ Fase 6 (Reporting)
                   └─→ Fase 7 (Portal)
                           └─→ Fase 8 (Designer)
Fase 4 (Versionado) → independiente, puede hacerse en paralelo con Fase 2
```

---

## Priorización recomendada

| Sprint | Fases | Resultado entregable |
|---|---|---|
| Sprint 1 | Fase 1 | Persistencia real, módulo apto para staging |
| Sprint 2 | Fase 2 | API REST completa, integrable con cualquier frontend |
| Sprint 3 | Fase 4 + Fase 5 | Versionado + eventos — módulo apto para producción |
| Sprint 4 | Fase 3 | Sub-Cases, soporte para procesos complejos |
| Sprint 5 | Fase 6 + Fase 7 | Visibilidad operacional + portal de operadores |
| Sprint 6 | Fase 8 | Designer visual para analistas de negocio |

---

## Integración con los otros módulos BPM

| Módulo | Dependencia con Case Management |
|---|---|
| **Human Tasks** | Consume `ICaseService.ResumeAsync` cuando un operador toma una decisión en la bandeja |
| **Rules Engine** | Sus `IRuleEngine` se usan dentro de los workflows de las etapas; los resultados se guardan en `Case.Data` |
| **Governance / Observability** | Consume los domain events de Fase 5 para construir las proyecciones de reporting |
| **Notifications** | Se suscribe a `CaseSlaBreachedEvent` y `CaseStatusChangedEvent` para enviar alertas |
