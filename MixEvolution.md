# Mix Evolution ERP — Estado de Desarrollo

## Proyecto
ERP web. Blazor Web App (InteractiveServer) + .NET 10 REST API. Solo desarrollador. Máx 50 usuarios, 10 empresas. Módulos como assemblies independientes, carga dinámica en runtime vía `AssemblyLoadContext.Default.LoadFromAssemblyPath`.

## Stack
.NET 10, Blazor Web App (InteractiveServer global), MudBlazor, SQL Server 2022, Docker, GitFlow AVH.

## Migración WASM→Server (COMPLETA, no reabrir)
`ERP.Host.UI` (WASM) fue ELIMINADO. Causa: `LoadFromAssemblyPath` fallaba en WASM Release (`TypeLoadException`/VTable). Reemplazado por `ERP.Host.Server`. Confirmado mergeado a `develop` (commit `81c88059` feat(blazor-server): migrate frontend to Blazor Web App with InteractiveServer). NO usar datos de sesiones pre-migración (puertos, DI lifetimes Singleton, `ERP.Host.UI`) sin verificar.

## Estructura
`src/ERP.Core.Contracts`, `ERP.Core.Contracts.Shared` (DTOs, `MenuItem`, `ITabService`), `ERP.Core.Infrastructure`, `ERP.Core.Security`, `ERP.Core.UI.Shared` (`CrudPage`), `ERP.Host.Api`, `ERP.Host.Server` (Blazor, `TabsState`, `MenuBuilderService`, `TabContainer`)
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
- `ModuloList` vive en el host por la misma razón

### Auth/Seguridad
- **JWT en cookie HttpOnly**, no en localStorage ni sessionStorage
- Middleware custom valida la cookie JWT en cada request y construye `ClaimsPrincipal`
- `PersistentComponentState` transfiere claims del request HTTP inicial al circuito SignalR
- `ApiAuthorizationHandler` lee la cookie e inyecta `Authorization: Bearer` en llamadas a la API
- Login: página SSR sin `@rendermode`, form POST plano — necesario para que el server pueda setear la cookie HttpOnly en la respuesta HTTP directamente
- `JwtParser` centraliza parseo de claims (corrige bug donde solo se leía el primer claim `empresa` en usuarios multi-empresa)
- Permisos: `Rol` → `RolPermiso` → `Permiso` (registrado por módulos), granular a pantalla/acción, aditivos (unión de roles por empresa + overrides por usuario)
- `SyncPermissionsAsync` al arranque sincroniza permisos de todos los módulos

### DI Lifetimes (confirmado en `Program.cs` de `ERP.Host.Server`)
`AppState`, `PermissionState`, `TabsState`, `ITabService`, `MenuBuilderService` → **Scoped** (uno por circuito SignalR/usuario). Intencional, ya corregido desde el Singleton original de WASM — con Singleton en Server, todos los usuarios habrían compartido la misma lista de tabs.

### Sistema de Tabs MDI
- `TabsState` (Scoped): `OpenTab(MenuItem)`, `CloseTabAsync`, `SetDirty`, `SetActive`
- `TabContainer.razor` (en `ERP.Host.Server`): usa `MudDynamicTabs` con `KeepPanelsAlive=true`
- `DynamicComponent` recibe parámetros vía `MenuItem.Parametros` (`Dictionary<string,object>?`, **nullable sin default** — el `Concat` en `TabContainer` debe usar `tab.Parametros ?? []`, si no, cualquier `MenuItem` sin `Parametros` explícito rompe el tab al abrirlo)
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

### CrudPage Genérico (`ERP.Core.UI.Shared`)
- `CrudPage<TItem, TForm>` encapsula `ViewMode` (List/New/Edit), dirty state, dialogs confirmar/cancelar/eliminar
- Callbacks: `OnNew`, `OnSave`, `OnCancel`
- `ConfirmarEliminar(Func<Task<bool>>)` para eliminar con confirmación
- **No usar `CrudPage` para pantallas sin List/New/Edit** — ej. `RolPermisoList` (checkboxes + guardar) se implementó como página simple con `ITabService.SetDirty` manual, no como instancia de `CrudPage`

## Módulos Implementados

### ERP.Module.Contabilidad
**API**: `CuentaContableService`, `CentroCostoService`, `AsientoService`. BD: schema `contabilidad`.
**UI**: `CuentaContableList`, `CentroCostoList` usando `CrudPage`.

### ERP.Module.Admin
**API**: `EmpresaEndpoints`, `UsuarioEndpoints`, `RolEndpoints` (con `GET/PUT /api/admin/roles/{id}/permisos`), `PermisoEndpoints` (`GET /api/admin/permisos`).
**UI**: `EmpresaList`, `UsuarioList`, `RolList`, `ModuloList`, `RolPermisoList` (checkboxes agrupados por `ModuleId`/`Pantalla`).

`AdminApiService` — **verificado contra el archivo real**, todos los métodos existen con estos nombres exactos: `GetPermisosAsync`, `GetPermisosRolAsync`, `ActualizarPermisosRolAsync`, además de `GetPermisosUsuarioAsync`/`ActualizarPermisosUsuarioAsync`, `GetRolesUsuarioAsync`/`ActualizarRolesUsuarioAsync`, `GetEmpresasUsuarioAsync`/`ActualizarEmpresasUsuarioAsync`.

## Branch Activa
`feature/blazor-server-migration` — mergeada a `develop` (verificado con git log en sesión previa).

## RolPermisoList — Estado Real (esta sesión)

**Confirmado, no asumido:**
- `PermisoEndpoints.MapPermisoEndpoints()` **sí está registrado** en `AdminModule.MapEndpoints()` — verificado contra el archivo real.
- Nombres de métodos en `AdminApiService` **coinciden** con lo asumido — verificado contra el archivo real.
- Dirty tracking reescrito: compara el set de permisos actual contra el estado original (`_permisosOriginales`), no marca dirty por cualquier toggle si el resultado neto es igual al estado guardado. `SetDirty(TabId, false)` solo se llama si `Guardar()` recibe `true` del backend — antes limpiaba el dirty flag sin verificar si el guardado tuvo éxito.

**Gaps reales confirmados (no "por verificar" — ya se vio que faltan):**
- No hay `ISnackbar` inyectado en `RolPermisoList`. Si `ActualizarPermisosRolAsync` devuelve `false`, el usuario no ve ningún mensaje de error.
- `OnInitializedAsync` no tiene `try/catch`. Si `GetPermisosAsync` o `GetPermisosRolAsync` fallan (401/500/timeout), no hay manejo — la página puede quedar en blanco o el circuito SignalR puede romperse sin mensaje.

**Sin verificar todavía (riesgo real, no cosmético):**
- `PermisoDto` — no se ha visto el record. `PermisoEndpoints` lo construye con constructor posicional `(IdPermiso, ModuleId, Pantalla, Accion, Descripcion)`; si el orden real no coincide, compila pero mapea campos mal (ej. `Descripcion` termina en `Accion`) sin ningún error visible.
- Cómo `RolList` arma `MenuItem.Parametros` al abrir el tab de `RolPermisoList` — si la clave no es exactamente `"RolId"` o el tipo no es `int`, el parámetro llega en 0 silenciosamente y se cargan/guardan permisos del rol equivocado.
- No compilado ni probado en runtime todavía.

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
16. Asignación de permisos a roles — código escrito y con dirty tracking corregido, pendiente compilar/probar, faltan Snackbar y try/catch (ver sección RolPermisoList arriba)
17. Asignación de roles a usuarios por empresa
18. Asignación de empresas a usuarios
19. Empresa por defecto por usuario
20. Campos `Debitos1`/`Creditos1` — pendiente info dueños
21. Trazabilidad `Origen`/`IdOrigen` — pendiente info dueños
22. Presupuesto por mes — pendiente info dueños
23. Modelo Multimoneda — pendiente info dueños
24. `AsientoList` con formulario de renglones dinámicos
25. Menú filtrado por permisos y empresa activa
26. `beforeunload` guard eliminado en la migración a Server — evaluar `NavigationManager.RegisterLocationChangingHandler`
27. Docker para arquitectura Server no existe todavía — el commit `d5d225f1` (feat(docker): serve Blazor WASM from API container as static files) es obsoleto, sirve el proyecto WASM eliminado. Sin esto no hay forma de desplegar el estado actual.

## Datos de Conexión (Dev)
- `ERP.Host.Api`: `http://localhost:5041`
- `ERP.Host.Server`: `http://localhost:5128`
- SQL Server: `localhost,1433`, Docker `mcr.microsoft.com/mssql/server:2022-latest`
- BD Security: `MixEvolution_Security`
- BD Empresas: `MixEvolution_Empresas`
