---
name: backend-architecture-rules
description: DDD, Clean Architecture, and Clean Code rules for the C# .NET backend. Use when generating, reviewing, or refactoring domain entities, application services, EF Core repositories, REST APIs, or DTOs.
---
# DDD, Clean Architecture and Clean Code rules for this backend

Rules are expressed in terms of the types that actually exist in `Framework/`. Examples point at `Notification2/` (see `reference-module.md`).

## 1. Clean Architecture — dependency rule

Dependencies point inward only:

```
Interface.RestApi ─► Application.Contract ─► Domain ◄─ Application ◄─ Infrastructure
                                              ▲
                                  Persistence.EF (implements Domain repository interfaces)
Config = composition root (references everything, contains no logic)
```

- **Domain**: no reference to EF Core, ASP.NET, Autofac, HTTP, `IConfiguration`, other modules. Only `Framework.Core` / `Framework.Utility`.
- **Application.Contract**: interfaces + DTOs + mappers + `I<M>UnitOfWork`. No implementation logic.
- **Application**: use-case orchestration. Talks to repositories and ports via interfaces; owns port interfaces for external systems (`Application/Common/I…`).
- **Infrastructure**: adapters for external systems (SMS, SignalR, files, LLM, …) implementing Application ports.
- **Persistence.EF**: `DbContext`, `IEntityTypeConfiguration<T>`, repositories, unit of work. The only layer that knows EF.
- **Interface.RestApi**: thin controllers — bind, authorize, take tenant/user from claims, call one service method, return. No business rules, no EF, no repositories.
- **Cross-module**: call the other module's `Application.Contract` interface or an `Infrastructure/ACL` project. Never reference another module's Domain/Application/Persistence project. Read-only foreign data mapped into your context must implement `IReadOnlyEntity` (the base context forces such entries back to `Unchanged`). Example: `Notification2.Domain.Entities.User` maps `user.person` for read-only joins.
- Verify with `.csproj` `ProjectReference`s, not with assumptions.

## 2. DDD tactical patterns

### Aggregates and entities
- Aggregate root: `public sealed class X : EntityId<Guid>, IAggregateRoot` (or `EntityId<int>`; keys are `int` or `Guid` only).
- `private X() { }` for EF; a public constructor (or static factory) that receives everything needed and **leaves the object valid** — validate and throw a domain exception otherwise.
- Constructors with 3+ parameters take an input/options object (`ObjectiveInputModel` style in Okr) — team convention.
- State changes only through intention-revealing methods (`MarkAsRead()`, `Deactivate(...)`, `Update(...)`); setters are `private set` or `init`. No public setters on aggregate state, no anemic "bag of properties" for new aggregates.
- Methods enforce invariants and are idempotent where the business allows (see `InAppNotification.MarkAsRead`).
- Child entities are created and modified only through the root. Expose collections as `IReadOnlyCollection<T>` backed by a private field.
- Reference other aggregates by id (`Guid UserId`), not by navigation property, unless they are in the same aggregate.
- Time: accept the timestamp (or `IDateTimeProvider`) from the caller; do not call `DateTime.Now` inside the domain. Name date-only vs date-time explicitly (`BirthDate`, `CreatedOnUtc`).

### Value objects
- Immutable `sealed record` (`Recipient`, `NotificationContent`) or `Framework.Core.ValueObject<T>` when custom equality is needed.
- Validate in a static `Create(...)` factory; return new instances via `with` for changes.
- Map with `OwnsOne`/`OwnsMany` in EF.

### Domain services
- Only when a rule does not belong to one aggregate (e.g. `INotificationPolicyEngineService` combines a config and a request into a valid `Notification`).
- Interface in `Domain/Service(s)`; implementation in Domain if it is pure, in `Application/Services` if it needs repositories.

### Repositories
- One interface per aggregate root in `Domain/Repositories`, deriving from `Framework.Core.Interface.IRepository` (required for auto-registration).
- Write methods take/return aggregates. Read methods for screens return query models (`Domain/QueryModel`) or `DataSourceResult<TQueryModel>` — never DTOs, never `IQueryable`.
- Implementation: `sealed class XRepository : BaseRepository<XDbContext>, IXRepository`.

### Specifications
- For reusable business predicates use `Framework.Core.Interface.Specification<T>` (`ToExpression()` works both in memory and in EF queries; compose with `And/Or/Not`).

### Domain events
- `Framework.Core.Interface.IEvent` exists, but there is no generic domain-event dispatcher in Notification2. Do not invent an event bus inside a feature; for cross-module notifications follow the existing `MessageChannel<INotificationEvent>` pattern used by `NotificationService`, or ask.

### Exceptions
- `public sealed class <Event><Domain><Act>Exception : BaseException` with a real code `Ex<subsystem><submodule><D|I|S|C><seq>` from `Exceptions.txt`. Read that file first; reuse an existing exception if it fits; add new codes to `Exceptions.txt` in the same change.
- Domain throws `D` codes, application services `S`, controllers `C`. Never `throw new Exception(...)`.

## 3. Application layer (use cases)

- One service per aggregate/use-case group: `XService : IXService`. The class name **must end with `Service`** to be picked up by `RegisterServicesByAssemblies`; anything else (`Dispatcher`, `Channel`, `Mapper`, `Policy`) must be registered explicitly in `Config/Bootstrapper.cs`.
- Typical command flow: load aggregate(s) → call domain behaviour → persist via repository (inside `I<M>UnitOfWork` when more than one write) → dispatch side effects → return id/DTO.
- Typical query flow: `DataRequest` mapping → repository projection → `DataSourceResult(Mapper.ToDto)`.
- Every async method takes and forwards `CancellationToken ct`. No `async void`, no `.Result`/`.Wait()`, no fire-and-forget without logging.
- Current user / organisation come in as parameters (from the controller) or from the module's user-context service where the module has one — never from DTO fields the client can set.

## 4. DTOs and mapping

- `sealed record` DTOs in `Application.Contract/DTOs`; request records may use positional parameters (`SendNotificationRequest(Guid UserId, string EventCode, string Body)`).
- Mapping is explicit: static extension classes in `Application.Contract/Mapping` (`ToDto`, `ToModel`). No AutoMapper in new code unless the module already uses it.
- Enum titles exposed as computed properties (`StatusTitle => Status.GetDescription()`).
- Do not expose entities from controllers.

## 5. Interface.RestApi

```csharp
[ApiController]
[Tags("<Module> - <Area>")]
[Route("<module>/<plural-resources>")]
public sealed class WidgetsController : ControllerBase
{
    private readonly IWidgetService _service;
    public WidgetsController(IWidgetService service) => _service = service;

    /// <summary>
    /// Permission : <Module>_Widgets
    /// دریافت لیست ...
    /// </summary>
    [HttpGet]
    [Authorize([RootsAuthenticationConst.Module_Widgets])]
    public async Task<DataSourceResult<WidgetDto>> GetList([DataRequest] DataRequest request)
        => await _service.GetList(request, User.GetOrganizationId());

    [HttpGet("{id}")]
    [Authorize([RootsAuthenticationConst.Module_Widgets])]
    public async Task<WidgetDto?> Get(Guid id, CancellationToken ct)
        => await _service.Get(id, User.GetOrganizationId(), ct);
}
```

- Controllers must be `public` (MVC discovery) and their class name must end with `Controller` (auto-registration).
- Every action has `[Authorize]` (`Framework.Infrastructure.Attribute.AuthorizeAttribute`) with the permission constant; writes use `[PermissionAction.Create|Update|Delete]`. New permission keys go in `Const/RootsAuthenticationConst.cs` and must exist in the permission seed the team uses — ask how it is deployed.
- Standard action names `Get` / `GetList`; plural routes.

## 6. Config / DI

```csharp
public static void WireUp(ContainerBuilder builder)
{
    builder.Register(ctx =>
    {
        var config = ctx.Resolve<IConfiguration>();
        var options = new DbContextOptionsBuilder<XDbContext>();
        options.UseSqlServer(config.GetConnectionString("Server"));
        return new XDbContext(options.Options);
    }).AsSelf().InstancePerLifetimeScope();

    builder.RegisterServicesByAssemblies([
        typeof(XService).Assembly, typeof(XRepository).Assembly, typeof(XController).Assembly ]);

    builder.RegisterType<XUnitOfWork>().As<IXUnitOfWork>().InstancePerLifetimeScope(); // not caught by the scan
}
```

- Scan rules (`Framework.Utility/Extentions/AutofacExtensionMethods.cs`): classes implementing `IRepository` → interfaces; class names ending `Service` → interfaces; ending `Controller` → self; implementing `IJob` → interfaces + self. All `InstancePerLifetimeScope`.
- A new module also needs a `WireUp` call in `ServiceHost/AutofacModule.cs` and project references from `ServiceHost`.
- Missing registration fails at runtime, not compile time — check it.

## 7. Clean code checklist

- Names in the ubiquitous language of the module (use the Persian business term's established English name in the module; do not rename existing concepts). No abbreviations except established ones (`Okr`, `Kpi`, `Dto`).
- Small methods, one level of abstraction; guard clauses instead of nested `if`s.
- `sealed` by default; `internal` unless another assembly needs it; `readonly` fields; `private set`/`init`.
- No magic strings/numbers — constants, enums, options.
- No commented-out code, no `TODO` without ticket, no empty `catch`.
- Nullable reference types are enabled in new projects (`<Nullable>enable</Nullable>`); respect them, do not silence with `!` without reason.
- Comments explain *why*; Persian comments are fine and must be preserved.
- Follow `.editorconfig`; do not reformat untouched code.