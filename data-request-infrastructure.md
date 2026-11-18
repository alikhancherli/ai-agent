---
name: data-request-infrastructure
description: Guide for using the DataRequest query filtering and mapping infrastructure. Use when applying dynamic query filters, property mappings, or mandatory backend constraints (like ForceFilter) to EF Core queries.
---
# DataRequest Query Filtering Infrastructure

You are a C# backend developer working with the `DataRequest` infrastructure. This system is a core utility in our toolbelt used for dynamic query filtering, sorting, and property mapping between API requests.

The infrastructure operates on a fluent pipeline starting with `request.Mapping()` and ending with `.Apply()`. 

## ARCHITECTURAL LAYER CONSTRAINT (CRITICAL)
The `DataRequest` manipulation MUST strictly follow Clean Architecture layer separation:
- **Service/Application Layer:** ALL configurations like `.Mapping()`, `.ForceFilter()`, `.Item()`, and `.Apply()` MUST be executed here (e.g., in Application Services, Query Handlers, or Use Cases) before passing the `request` to the Repository.
- **Persistence/Repository Layer:** The EF Core Repository MUST NOT contain `.Mapping()` or `.ForceFilter()` logic. The repository's only job is to receive the already-configured `DataRequest` and apply it to the `IQueryable` (e.g., via `.ToDataSourceResultAsync(request)`).

*Reviewer Instruction:* If a mandatory business filter is missing, you must flag the omission in the **Service Layer** file. If a developer attempts to add `.ForceFilter()` inside a Repository, flag it as an architectural violation and instruct them to move it to the Service layer.

## 1. Mandatory Constraints (ForceFilter)
One of the core tasks in the `DataRequest` infrastructure is applying mandatory, backend-enforced query filters using `.ForceFilter()`. This is used to enforce security rules, hard filters, default ranges, or tenant isolation independently of client input.

**Syntax:**
```csharp
.ForceFilter("{FieldName}", FilterComparison.{Method}, {Value}, FilterLogic.{Operator})
```

**Example (Inside Service Layer):**
```csharp
request.Mapping()
    .ForceFilter("unitId", FilterComparison.Equals, unitId, FilterLogic.Or)
    .ForceFilter("startDate", FilterComparison.Equals, startDate.ToShortDateString(), FilterLogic.And)
    .ForceFilter("deadLine", FilterComparison.Equals, endDate.ToShortDateString(), FilterLogic.And)
    .Apply();
    
// The pre-configured request is then passed to the repository
var result = await _repository.GetSummaries(request);
```