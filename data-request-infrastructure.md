---
name: data-request-expert
description: Guide for using the DataRequest query filtering and mapping infrastructure. Use when applying dynamic query filters, property mappings, or mandatory backend constraints (like ForceFilter) to EF Core queries.
---
# Data Request Infrastructure

Senior architect guide for Entity Framework Core, ASP.NET Core, and dynamic LINQ queries using the `DataRequest` ecosystem. This skill enforces strict architectural boundaries, correct layer-by-layer responsibilities, and precise usage of filtering, sorting, projection, aggregation, grouping, selected keys, and Excel exports.

# When to Use

- When applying dynamic query filters or property mappings.
- When enforcing mandatory backend constraints like `ForceFilter`.
- When reviewing or refactoring controller actions, application services, or repositories using `DataRequest`.
- When handling data exports (e.g., Excel) that require pagination removal.

## Architectural Boundaries & Layer Responsibilities

### 1. Presentation Layer (Controllers)
- Must remain "thin".
- Use the `[DataRequest]` attribute on action parameters for automatic model binding.
- Forward the `DataRequest` object directly to the Application/Service layer without any manual manipulation, business logic, or filtering.

### 2. Application Layer (Services / Business Logic)
- The **only** layer permitted to modify, configure, or extend `DataRequest`.
- Responsible for defining mappings, filters, and sorting rules using Fluent Builder or extension methods.
- **Allowed Operations:**
  - **Mapping:** `.Mapping<TSource, TQuery>()` or `.Mapping()` to map DTO properties to entity/query model properties.
  - **Filtering Rules:**
    - `ForceFilter(...)`: Enforce mandatory security/business conditions that the client cannot override.
    - `Require()`: Make a specific filter mandatory from the client side (throws `ArgumentException` if missing).
    - `Ignore()`: Skip automatic filtering on specific properties when manually handled in the service.
  - **Sorting Rules:**
    - `ApplyDefaultSort(..., "asc"/"desc")`: Apply default sorting only if the client did not send explicit sorting.
    - `AppendSort(...)` / `PrependSort(...)`: Add sorting rules at the beginning or end of the sort collection.
  - **Projection & Selection:**
    - `.Project(...)` to map projection properties.
    - `.WithSelectionLookupField(...)` to specify the key property for multi-select and lookup scenarios.
  - **Export / Unlimited:**
    - Call `request.Unlimited()` in export methods (e.g., `GetListExport`) to remove pagination limits before fetching data and generating Excel files via `.ExcelResult(...)`.

### 3. Infrastructure Layer (Repositories / DbContext / Queryables)
- Responsible for building the base `IQueryable<T>` (using `AsNoTracking()`, includes, filters specific to persistence, etc.).
- **Strict Rule:** Never manually manipulate `request.Filter`, `request.Sort`, or business rules in this layer.
- Concludes execution by calling `.ToDataSourceResultAsync(request, allowedFields, asSplitQuery)` to let the infrastructure handle automatic filtering, sorting, projection, aggregation, grouping, and paging efficiently in the database.

## Gotchas (Code Review Anti-patterns)
- **Modifying `request.Filter` in Repository:** Do not modify `request.Filter` or add filters inside the Repository/Infrastructure layer. *Fix:* Move all filter configurations, `ForceFilter`, or mapping logic to the Application service layer.
- **Fat Controllers:** Do not put business logic or filter manipulation inside Controller actions. *Fix:* Keep controllers clean; let services handle the `DataRequest` configuration.
- **Paginated Exports:** Forgetting `request.Unlimited()` in Excel export methods results in exporting only a single page (e.g., 15 records). *Fix:* Always invoke `request.Unlimited()` in export services.
- **Missing Lookup Fields:** Sending `selectedKeys` from the client without configuring `.WithSelectionLookupField(...)` in the application layer. *Fix:* Define the lookup field using the builder to enable `EnableToShowSelectedData`.