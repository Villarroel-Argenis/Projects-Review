# Projects-Review
Visualizador de Proyectos


# FlowForge 🔥

> Un motor de workflows moderno, extensible y durable para .NET 10

[![.NET 10](https://img.shields.io/badge/.NET-10-512BD4)](https://dotnet.microsoft.com/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)](https://github.com/yourusername/FlowForge)

## 📋 Descripción

**FlowForge** es un motor de workflows liviano, extensible y de alto rendimiento diseñado para aplicaciones .NET modernas. Permite definir, ejecutar y gestionar workflows complejos con soporte para:

- ✅ **Ejecución asíncrona** con suspensión/reanudación mediante bookmarks
- ✅ **Control de flujo avanzado** (loops, condicionales, iteraciones)
- ✅ **Gestión de estado** con persistencia de instancias
- ✅ **Arquitectura extensible** basada en actividades reutilizables
- ✅ **Workers durables** para procesamiento en background
- ✅ **Stack-based execution** para bloques anidados (foreach, while, if)

## 🚀 Características Principales

### Motor de Workflows
- **WorkflowEngine**: Motor principal de ejecución
- **WorkflowInstance**: Instancia de workflow con estado persistible
- **WorkflowDefinition**: Definición declarativa de workflows
- **ActivityPipeline**: Pipeline de middleware para actividades

### Actividades Disponibles

#### Control de Flujo
- **`IfActivity`**: Ejecución condicional con ramas true/false
- **`WhileActivity`**: Bucle con condición evaluable
- **`ForEachActivity`**: Iteración sobre colecciones (strings, números, objetos)

#### Utilidades
- **`DelayActivity`**: Pausa el workflow por un tiempo determinado
- **`SetVariableActivity`**: Establece variables en el contexto
- **`IncrementVariableActivity`**: Incrementa contadores numéricos
- **`WriteLineActivity`**: Salida a consola (debugging)

### Runtime
- **`WorkflowWorker`**: Worker durable que procesa workflows en background
- **`InMemoryWorkflowInstanceStore`**: Almacenamiento de instancias en memoria
- **`WorkflowBookmark`**: Sistema de bookmarks para suspensión/reanudación

## 📦 Instalación

```bash
# Clonar el repositorio
git clone https://github.com/yourusername/FlowForge.git
cd FlowForge

# Restaurar dependencias
dotnet restore

# Compilar
dotnet build

# Ejecutar pruebas
dotnet test
```

## 💻 Uso

### Ejemplo Básico: Workflow con Loop

```csharp
using FlowForge.Core;
using FlowForge.Runtime;
using FlowForge.Activities;
using FlowForge.Activities.ControlFlow;

// 1. Crear almacenamiento de instancias
var store = new InMemoryWorkflowInstanceStore();

// 2. Crear pipeline y motor
var pipeline = new ActivityPipeline([]);
var engine = new WorkflowEngine(
    nextStrategy: new DefaultNextStrategy(),
    pipeline: pipeline
);

// 3. Definir actividades
var incrementActivity = new IncrementVariableActivity(
    id: "increment_counter",
    variableName: "counter",
    amount: 1,
    returnToScope: true
);

var writeLineActivity = new WriteLineActivity(
    id: "write_line",
    messageFunc: ctx => $"Contador: {ctx.Get<int?>("counter") ?? 0}",
    nextActivityId: "write_end"
);

var whileActivity = new WhileActivity(
    id: "loop_start",
    condition: ctx => (ctx.Get<int?>("counter") ?? 0) < 5,
    bodyStartId: "increment_counter",
    nextActivityId: "write_line"
);

var endActivity = new WriteLineActivity(
    id: "write_end",
    message: "Workflow completado!",
    nextActivityId: null
);

// 4. Crear definición del workflow
var definition = new WorkflowDefinition(
    id: "loop_demo_workflow",
    activities: [incrementActivity, writeLineActivity, whileActivity, endActivity],
    startActivityId: "loop_start"
);

// 5. Crear instancia
var instance = WorkflowInstance.Create(definition.Id);
instance.CurrentActivityId = definition.StartActivityId;
await store.SaveAsync(instance, CancellationToken.None);

// 6. Ejecutar con worker
var worker = new WorkflowWorker(engine, definition, store);
using var cts = new CancellationTokenSource();
_ = worker.RunAsync(cts.Token);

Console.WriteLine("Presiona Ctrl+C para salir...");
Console.CancelKeyPress += (s, e) => cts.Cancel();
await Task.Delay(-1, cts.Token);
```

**Salida:**
```
Contador: 1
Contador: 2
Contador: 3
Contador: 4
Contador: 5
Workflow completado!
```

### Ejemplo: ForEach con Objetos Complejos

```csharp
var users = new[]
{
    new { Name = "Alice", Age = 30 },
    new { Name = "Bob", Age = 25 },
    new { Name = "Charlie", Age = 35 }
};

var foreachActivity = new ForEachActivity(
    id: "foreach_users",
    collectionFunc: ctx => users,
    itemVariableName: "currentUser",
    bodyStartId: "process_user",
    nextActivityId: "done"
);

var processActivity = new WriteLineActivity(
    id: "process_user",
    messageFunc: ctx => {
        var user = ctx.Get<dynamic>("currentUser");
        return $"Procesando usuario: {user.Name}, edad: {user.Age}";
    },
    nextActivityId: null // returnToScope
);

var definition = new WorkflowDefinition(
    id: "users_workflow",
    activities: [foreachActivity, processActivity],
    startActivityId: "foreach_users"
);
```

### Ejemplo: Delay Activity (Workflow Suspendido)

```csharp
var delayActivity = new DelayActivity(
    id: "wait_5_seconds",
    delay: TimeSpan.FromSeconds(5),
    nextActivityId: "continue"
);

var continueActivity = new WriteLineActivity(
    id: "continue",
    message: "5 segundos después...",
    nextActivityId: null
);

var definition = new WorkflowDefinition(
    id: "delay_workflow",
    activities: [delayActivity, continueActivity],
    startActivityId: "wait_5_seconds"
);

// El WorkflowWorker detectará automáticamente los timers vencidos
// y reanudará el workflow cuando corresponda
```

## 🏗️ Arquitectura

### Componentes Principales

```
FlowForge/
├── src/
│   ├── FlowForge.Core/              # Motor central y modelos
│   │   ├── Engine/
│   │   │   └── WorkflowEngine.cs
│   │   ├── Models/
│   │   │   ├── WorkflowDefinition.cs
│   │   │   ├── WorkflowInstance.cs
│   │   │   ├── ActivityResult.cs
│   │   │   └── WorkflowBookmark.cs
│   │   ├── Context/
│   │   │   └── ActivityExecutionContext.cs
│   │   └── Abstractions/
│   │       └── IActivity.cs
│   │
│   ├── FlowForge.Runtime/           # Runtime y workers
│   │   ├── WorkflowWorker.cs
│   │   ├── WorkflowRuntimeEngine.cs
│   │   └── InMemoryWorkflowInstanceStore.cs
│   │
│   ├── FlowForge.Activities/        # Actividades base
│   │   ├── DelayActivity.cs
│   │   ├── SetVariableActivity.cs
│   │   ├── IncrementVariableActivity.cs
│   │   ├── WriteLineActivity.cs
│   │   ├── WhileActivity.cs
│   │   └── ForEachActivity.cs
│   │
│   └── FlowForge.Activities.ControlFlow/  # Control de flujo
│       └── IfActivity.cs
│
└── Tests/
    └── FlowForge.Tests/             # Pruebas unitarias (xUnit + Shouldly)
        └── Core/
            └── WorkflowEngineTests.cs
```

### Flujo de Ejecución

1. **Definición**: Se crea un `WorkflowDefinition` con actividades
2. **Instancia**: Se crea un `WorkflowInstance` desde la definición
3. **Ejecución**: El `WorkflowEngine` ejecuta actividades secuencialmente
4. **Stack Management**: Bloques como `ForEach` y `While` usan el `ExecutionStack`
5. **Suspensión**: `DelayActivity` crea un `WorkflowBookmark` para pausar
6. **Reanudación**: El `WorkflowWorker` detecta timers vencidos y reanuda

### Estados del Workflow

```csharp
public enum WorkflowStatus
{
    Running,      // En ejecución activa
    Suspended,    // Suspendido (esperando timer o señal)
    Completed     // Finalizado
}
```

## 🧪 Testing

El proyecto incluye una suite completa de pruebas unitarias con **xUnit** y **Shouldly**:

```bash
dotnet test --verbosity normal
```

**Cobertura actual: 11 pruebas**
- ✅ WorkflowDefinition
- ✅ DelayActivity
- ✅ ForEachActivity (strings, números, objetos complejos, colección vacía)
- ✅ IncrementVariableActivity (casos múltiples)

## 🗺️ Roadmap

### ✅ Completado (v0.1)

- [x] Core workflow engine
- [x] Actividades básicas (Delay, SetVariable, WriteLine)
- [x] Control de flujo (If, While, ForEach)
- [x] Sistema de bookmarks para suspensión
- [x] WorkflowWorker para ejecución durable
- [x] InMemoryWorkflowInstanceStore
- [x] Suite de pruebas unitarias

### 🚧 En Progreso (v0.2)

- [X] Persistencia a base de datos (Entity Framework Core)
- [X] Logging y telemetría (OpenTelemetry)
- [ ] Middleware pipeline para actividades
- [ ] Validación de workflows

### 📅 Próximos Pasos (v0.3+)

#### Nuevas Actividades
- [ ] `ParallelActivity`: Ejecución paralela de ramas
- [ ] `SwitchActivity`: Switch/case para múltiples condiciones
- [ ] `TryCatchActivity`: Manejo de excepciones
- [ ] `HttpRequestActivity`: Llamadas HTTP
- [ ] `SendSignalActivity` / `WaitForSignalActivity`: Event-driven workflows

#### Características Avanzadas
- [ ] Event Bus para señales externas
- [ ] Compensación y transacciones (Saga pattern)
- [ ] Versionado de workflows
- [ ] Workflow designer visual (Blazor)
- [ ] Dashboard de monitoreo
- [ ] Métricas y observabilidad
- [ ] Distributed tracing

#### Infraestructura
- [ ] Soporte para DI (Dependency Injection)
- [ ] Plugins y extensibilidad
- [ ] Azure Durable Functions compatibility layer
- [ ] Kubernetes operators
- [ ] CLI para gestión de workflows

## 🤝 Contribuir

¡Las contribuciones son bienvenidas! Por favor:

1. Fork el proyecto
2. Crea una rama para tu feature (`git checkout -b feature/AmazingFeature`)
3. Commit tus cambios (`git commit -m 'Add some AmazingFeature'`)
4. Push a la rama (`git push origin feature/AmazingFeature`)
5. Abre un Pull Request

### Guías de Contribución

- Seguir las convenciones de código C# (.editorconfig incluido)
- Agregar pruebas unitarias para nuevas funcionalidades
- Actualizar la documentación según sea necesario
- Los commits deben seguir [Conventional Commits](https://www.conventionalcommits.org/)

## 📄 Licencia

Este proyecto está bajo la licencia MIT. Ver el archivo [LICENSE](LICENSE) para más detalles.

## 👨‍💻 Autor

**Tu Nombre** - [@yourusername](https://github.com/yourusername)

## 🙏 Agradecimientos

- Inspirado por [Azure Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/)
- Inspirado por [Temporal.io](https://temporal.io/)
- Built with ❤️ using .NET 10

---

⭐ Si este proyecto te resulta útil, considera darle una estrella en GitHub!
