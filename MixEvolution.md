# Mix Evolution ERP — Estado de Desarrollo

## Proyecto
ERP web. Blazor Web App (InteractiveServer) + .NET 10 REST API. Solo desarrollador. Máx 50 usuarios, 10 empresas. Módulos como assemblies independientes, carga dinámica en runtime vía `AssemblyLoadContext.Default.LoadFromAssemblyPath`.

## Stack
.NET 10, Blazor Web App (InteractiveServer global), MudBlazor, SQL Server 2022, Docker, GitFlow AVH.

## Migración WASM→Server (COMPLETA, no reabrir)
`ERP.Host.UI` (WASM) fue ELIMINADO. Causa: `LoadFromAssemblyPath` fallaba en WASM Release (`TypeLoadException`/VTable). Reemplazado por `ERP.Host.Server`. Confirmado mergeado a `develop` (commit `81c88059` feat(blazor-server): migrate frontend to Blazor Web App with InteractiveServer). NO usar datos de sesiones pre-migración (puertos, DI lifetimes Singleton, `ERP.Host.UI`) sin verificar.

**Nota real de esta sesión:** "COMPLETA" se refería a que compila y el flujo de auth/tabs funciona — no que cada módulo migrado fue probado en runtime. Se encontró un bug real (ver sección "Bug AppState/IClientEmpresaState" abajo) en una pantalla (`AsientoList`) que nunca se había abierto desde la migración.

## Estructura
`src/ERP.Core.Contracts`, `ERP.Core.Contracts.Shared` (DTOs, `MenuItem`, `ITabService`, `IClientEmpresaState`), `ERP.Core.Infrastructure`, `ERP.Core.Security`, `ERP.Core.UI.Shared` (`CrudPage`, `MasterDetailPage`), `ERP.Host.Api`, `ERP.Host.Server` (Blazor, `TabsState`, `MenuBuilderService`, `TabContainer`, `AppState`)
`modules/ERP.Module.Contabilidad.Api/.UI`, `ERP.Module.Admin.Api/.UI`
`tools/Pack-Module.ps1`, `Install-Module.ps1`

## Arquitectura Clave

### Módulos
- Assemblies independientes bajo `modules/{Nombre}/{core|ui}/` + `module.json`
- `IAppModule`: `RegisterServices`, `MapEndpoints`, `GetPermissions`
- `IUIModule`: `ModuleId`, `Title`, `Icono?`, `Version`, `RegisterServices`, `GetMenuItems`
- Distribución via `.mixm` (zip con core/ ui/ module.json)
- Post-build Debug copia DLLs a `ERP.Host.Api/bin/Debug/.../modules/` **y** `ERP.Host.Server/bin/Debug/.../modules/`
- `InputFile` y componentes de archivos deben vivir en el host, no en módulos dinámicos (restricción de resolución de assembly)
- `ModuloList` vive en el host por la misma razón. **No usa `CrudPage` ni `MasterDetailPage` — decisión explícita, se queda fuera del refactor de manejo de errores (ver abajo).**

### Auth/Seguridad
- **JWT en cookie HttpOnly**, no en localStorage ni sessionStorage
- Middleware custom valida la cookie JWT en cada request y construye `ClaimsPrincipal`
- `PersistentComponentState` transfiere claims del request HTTP inicial al circuito SignalR
- `ApiAuthorizationHandler` lee la cookie e inyecta `Authorization: Bearer` en llamadas a la API. **NO debe inyectar servicios Scoped de la app (como `AppState`) por constructor — ver "Bug AppState/IClientEmpresaState" abajo.**
- Login: página SSR sin `@rendermode`, form POST plano — necesario para que el server pueda setear la cookie HttpOnly en la respuesta HTTP directamente
- `JwtParser`/`JwtClaimsReader` centralizan parseo de claims. Claim del JWT es `"empresa"` (singular), un objeto por claim (multi-empresa = múltiples claims `"empresa"`), leído vía `user.FindAll("empresa")`. `"empresaDefault"` es claim separado, opcional.
- Permisos: `Rol` → `RolPermiso` → `Permiso` (registrado por módulos), granular a pantalla/acción, aditivos (unión de roles por empresa + overrides por usuario)
- `SyncPermissionsAsync` al arranque sincroniza permisos de todos los módulos

### DI Lifetimes (confirmado en `Program.cs` de `ERP.Host.Server`)
`AppState`, `PermissionState`, `TabsState`, `ITabService`, `IClientEmpresaState`, `MenuBuilderService` → **Scoped** (uno por circuito SignalR/usuario). `ApiAuthorizationHandler` → **Transient** (patrón correcto para `HttpClientFactory`, confirmado no es la causa del bug de scope — ver abajo).

### Bug AppState/IClientEmpresaState (diagnosticado y corregido en Contabilidad, esta sesión)
**Síntoma:** `AsientoList` fallaba al cargar/guardar con `InvalidOperationException: EmpresaId no encontrado en el header X-Empresa-Id`, pese a que `AppState.EmpresaId` tenía el valor correcto en `MainLayout`.

**Causa raíz real (confirmada con hashcodes de instancia):** `ApiAuthorizationHandler` (DelegatingHandler, Transient, registrado vía `AddHttpMessageHandler<T>()`) inyectando `AppState` por constructor no es confiable — `HttpClientFactory` resuelve sus `HttpMessageHandler` desde un pool/scope interno propio, distinto al scope del circuito Blazor. Confirmado con logs: dos circuitos distintos mostraron hashcodes de `AppState` diferentes entre `MainLayout` y el handler.

**Intento fallido #1:** leer `AppState` dentro del handler vía `_httpContextAccessor.HttpContext?.RequestServices.GetService<AppState>()` en vez de constructor injection. Funcionó en una prueba, falló en la siguiente (intermitente) — `IHttpContextAccessor.HttpContext` en Blazor Server interactivo no siempre referencia el contexto de servicio correcto del circuito activo, especialmente pasado el primer render. **No usar este patrón.**

**Fix real y validado:** sacar la dependencia de `AppState` del `DelegatingHandler` por completo. Crear `IClientEmpresaState` (solo lectura, `int EmpresaId { get; }`) en `ERP.Core.Contracts.Shared.Services`. `AppState : IClientEmpresaState`. Registrar `builder.Services.AddScoped<IClientEmpresaState>(sp => sp.GetRequiredService<AppState>());` en `Program.cs`. Los servicios API de cada módulo (`ContabilidadApiService`, etc.) inyectan `IClientEmpresaState` directo por constructor (Scoped normal, resuelve correctamente porque no pasa por el mecanismo interno de `HttpClientFactory`), construyen el `HttpRequestMessage` manualmente (no se puede usar `PostAsJsonAsync`/`GetFromJsonAsync` porque no exponen el request para agregar headers) y agregan `X-Empresa-Id` ahí.

**Deliberadamente de solo lectura:** `IClientEmpresaState` NO expone `SetEmpresaId`. Cambiar la empresa activa dispara efectos colaterales (cerrar tabs, revalidar permisos, reconstruir menú) que solo `MainLayout.OnEmpresaChanged` debe orquestar. Un módulo no debe poder cambiar la empresa activa directamente.

**Aplicado:** `ContabilidadApiService` — corregido y validado en runtime (`AsientoList` carga y guarda correctamente).
**Pendiente (deuda técnica):** `AdminApiService` — mismo riesgo latente, no aplicado a propósito. Ningún endpoint de Admin depende hoy de `IEmpresaContext.EmpresaId`, así que no falla, pero si alguno llega a depender de él, el header no se enviará. Ver deuda #28.

**Efecto colateral detectado y revertido:** durante el diagnóstico, `AppState` se cambió accidentalmente a `AddSingleton` (autocompletado de IntelliSense). Se revirtió a `AddScoped` de inmediato. Si alguna vez se ve comportamiento de "un usuario ve datos de otro" en `AppState`, revisar este registro primero.

### Sistema de Tabs MDI
- `TabsState` (Scoped): `OpenTab(MenuItem)`, `CloseTabAsync`, `SetDirty`, `SetActive`
- `TabContainer.razor` (en `ERP.Host.Server`): usa `MudDynamicTabs` con `KeepPanelsAlive=true`
- `DynamicComponent` recibe parámetros vía `new Dictionary<string, object> { { "TabId", tab.Id } }.Concat(tab.Parametros ?? []).ToDictionary()` — `TabId` se inyecta manualmente en el contenedor, no depende de que el emisor del tab lo incluya en `Parametros`. **Verificado contra archivo real.**
- `MenuItem.Parametros` es `Dictionary<string,object>?` nullable sin default — el `Concat` en `TabContainer` usa `tab.Parametros ?? []`, si no, cualquier `MenuItem` sin `Parametros` explícito rompe el tab al abrirlo
- Menú llama `TabsState.OpenTab(item)` en vez de navegar
- `ITabService` (namespace real: `ERP.Core.Contracts.Shared.Services`; `MenuItem` en `ERP.Core.Contracts.Shared.Navigation`) expone:
  ```csharp
  void SetDirty(string tabId, bool isDirty);
  bool OpenTab(MenuItem item);
  bool HasDirtyTabs { get; }
  ```

### Menú Dinámico
- `MenuBuilderService` itera módulos cargados, construye jerarquía
- `NavMenu.razor` recursivo
- Items con `Componente` abren tab; sin componente aparecen deshabilitados

### CrudPage y MasterDetailPage Genéricos (`ERP.Core.UI.Shared`) — refactor de manejo de errores esta sesión
- `CrudPage<TItem, TForm>` y `MasterDetailPage<TItem, TForm, TDetail>` encapsulan `ViewMode` (List/New/Edit), dirty state, dialogs confirmar/cancelar/eliminar
- **`OnSave` cambió de `EventCallback` a `Func<Task<bool>>` (no-nullable) en ambos componentes.** Antes: `Guardar()` invocaba `OnSave` y cerraba el formulario incondicionalmente, sin saber si el guardado tuvo éxito — mismo bug que tenía `RolPermisoList` antes de su fix, pero replicado en las seis pantallas que usan estos componentes genéricos.
- `Guardar()` ahora: `try/catch` alrededor de `await OnSave()`. Si `true` → Snackbar de éxito + `ShowList()`. Si `false` → Snackbar de error, formulario NO se cierra, dirty flag permanece. Si excepción → Snackbar de error genérico, mismo efecto.
- Se agregó `[Inject] ISnackbar` a ambos componentes.
- Botones "Guardar" y "Revertir" en el `.razor` de ambos: `Disabled="@(!_isDirty || _saving)"` — antes solo `!_isDirty`, permitía doble-submit o revertir en medio de un guardado en curso.
- `OnCancel` se dejó como `EventCallback` sin cambios — decisión explícita, no hay caso de uso hoy donde cancelar dispare I/O que pueda fallar. Si surge, se revisa entonces.
- **Callers actualizados** (todos cambiaron `Guardar()` de `async Task` a `async Task<bool>`, devolviendo el `bool` real de la llamada al API en vez de descartarlo): `RolList`, `EmpresaList`, `UsuarioList`, `AsientoList` (usa `MasterDetailPage`), `CentroCostoList`, `CuentaContableList`.
- **`ModuloList` no usa `CrudPage`/`MasterDetailPage`** — queda fuera de este refactor, decisión explícita.
- **No cubierto por este refactor:** guard de error en carga inicial (patrón de `RolPermisoList`: try/catch + flag de error + `MudAlert`). Ninguno de los dos componentes genéricos tiene `OnInitializedAsync` propio — la carga vive en cada pantalla consumidora (`CargarRoles()`, `CargarEmpresas()`, etc.), fuera del componente genérico. Ver deuda #29.
- **Validado en runtime, no solo compilado:** `CentroCostoList` (nuevo y editar, fallo forzado apagando la API) y `AsientoList` (editar, fallo forzado apagando la API, con asiento balanceado para descartar rechazo por validación de negocio) — ambos muestran Snackbar rojo, formulario no se cierra, dirty flag permanece, botón vuelve a estado usable.
- **No probado en runtime, solo compilado:** `RolList`, `EmpresaList`, `UsuarioList`, `CuentaContableList` — el mecanismo es el mismo componente genérico ya validado, pero cada `Guardar()` tiene su propia llamada al API que no se forzó a fallar individualmente.

## Módulos Implementados

### ERP.Module.Contabilidad
**API**: `CuentaContableService`, `CentroCostoService`, `AsientoService`. BD: schema `contabilidad`. `AsientoEndpoints` usa `IEmpresaContext.EmpresaId` (backend) — depende de que el cliente envíe `X-Empresa-Id`.
**UI**: `CuentaContableList`, `CentroCostoList` (usan `CrudPage`), `AsientoList` (usa `MasterDetailPage`, renglones dinámicos — **deuda #24 cerrada**, el documento no se había actualizado). `ContabilidadApiService` inyecta `IClientEmpresaState` (ver sección Bug AppState arriba).

### ERP.Module.Admin
**API**: `EmpresaEndpoints`, `UsuarioEndpoints`, `RolEndpoints` (con `GET/PUT /api/admin/roles/{id}/permisos`), `PermisoEndpoints` (`GET /api/admin/permisos`).
**UI**: `EmpresaList`, `UsuarioList`, `RolList`, `ModuloList`, `RolPermisoList` (checkboxes agrupados por `ModuleId`/`Pantalla`, dirty tracking + Snackbar + try/catch — **COMPLETO**, verificado en runtime en sesión previa).

`AdminApiService` — verificado contra archivo real: `GetPermisosAsync`, `GetPermisosRolAsync`, `ActualizarPermisosRolAsync`, además de `GetPermisosUsuarioAsync`/`ActualizarPermisosUsuarioAsync`, `GetRolesUsuarioAsync`/`ActualizarRolesUsuarioAsync`, `GetEmpresasUsuarioAsync`/`ActualizarEmpresasUsuarioAsync`. **No implementa `IClientEmpresaState` — ver deuda #28.**

`PermisoDto` (record, orden confirmado): `(int IdPermiso, string ModuleId, string Pantalla, string Accion, string Descripcion)`. `RolAdminDto` (record, confirmado): `(int IdRol, string Nombre, string Descripcion)`.

## Branches Activas / Recientes
`feature/blazor-server-migration` — mergeada a `develop`.
`feature/crudpage-error-handling` — refactor `OnSave` en `CrudPage`/`MasterDetailPage`, seis pantallas, fix `IClientEmpresaState` en Contabilidad. Esta sesión.

## Deuda Técnica Registrada
1. Refresh Token
2. Persistencia de tabs al refresh (localStorage) — especialmente importante con dirty state
3. Filtrado de menú por permisos y empresa activa
4. Guard de cambio de empresa con dirty state
5. Spike técnico: `IDirtyCheck` interface
6. Tabs dinámicos cruzando módulos
7. Template de proyecto C# para módulos nuevos
8. Validación manifiesto vs assembly en ModuleLoader
9. Convención nombre DLL UI frágil
10. Notificación instalación módulos vía Redis (multi-instancia)
11. Docker con reinicio automático
12. Configuración por módulo
13. Soft delete en `CuentaContable`
14. Mapper por Source Generation
15. Gestión completa usuarios (empresas + roles en mismo formulario)
16. ~~Asignación de permisos a roles~~ — **CERRADO.**
17. Asignación de roles a usuarios por empresa
18. Asignación de empresas a usuarios
19. Empresa por defecto por usuario
20. Campos `Debitos1`/`Creditos1` — pendiente info dueños
21. Trazabilidad `Origen`/`IdOrigen` — pendiente info dueños
22. Presupuesto por mes — pendiente info dueños
23. Modelo Multimoneda — pendiente info dueños
24. ~~`AsientoList` con formulario de renglones dinámicos~~ — **CERRADO** (ya estaba resuelto, el documento no se había actualizado).
25. Menú filtrado por permisos y empresa activa
26. `beforeunload` guard eliminado en la migración a Server — evaluar `NavigationManager.RegisterLocationChangingHandler`
27. Docker para arquitectura Server no existe todavía — el commit `d5d225f1` es obsoleto, sirve el proyecto WASM eliminado. Sin esto no hay forma de desplegar el estado actual.
28. **(nuevo)** `AdminApiService` no implementa `IClientEmpresaState` (a diferencia de `ContabilidadApiService`, corregido esta sesión). Riesgo latente: si algún endpoint de Admin llega a depender de `IEmpresaContext.EmpresaId` en el backend, el header `X-Empresa-Id` no se enviará. Mismo bug ya diagnosticado y resuelto en Contabilidad (causa raíz: `ApiAuthorizationHandler` no puede leer `AppState` de forma confiable vía `IHttpContextAccessor`, por cómo `HttpClientFactory` resuelve sus handlers desde un scope propio distinto al del circuito Blazor). Fix ya probado: inyectar `IClientEmpresaState` directo en el servicio API, no en el handler.
29. **(nuevo)** Guard de error en carga inicial (patrón `RolPermisoList`: try/catch + flag de error + `MudAlert`) no está centralizado en `CrudPage`/`MasterDetailPage` — ninguno de los dos tiene `OnInitializedAsync` propio, la carga vive en cada pantalla consumidora. Aplica a las seis pantallas migradas en el refactor de `OnSave` más las que se agreguen después. Opciones a evaluar: (a) replicar manualmente en cada `CargarX()`, (b) agregar un parámetro `Func<Task>? OnLoad` a los componentes genéricos que centralice el guard.
30. **(nuevo)** `RolList`, `EmpresaList`, `UsuarioList`, `CuentaContableList` — refactor de `OnSave` compilado pero no probado en runtime contra fallo real (solo `CentroCostoList` y `AsientoList` se probaron forzando la API caída).

## Datos de Conexión (Dev)
- `ERP.Host.Api`: `http://localhost:5041`
- `ERP.Host.Server`: `http://localhost:5128`
- SQL Server: `localhost,1433`, Docker `mcr.microsoft.com/mssql/server:2022-latest`
- BD Security: `MixEvolution_Security`
- BD Empresas: `MixEvolution_Empresas`
