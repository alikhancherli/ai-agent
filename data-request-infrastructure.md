---
name: data-request-infrastructure
description: Guide for using the DataRequest query filtering and mapping infrastructure. Use when applying dynamic query filters, property mappings, or mandatory backend constraints (like ForceFilter) to EF Core queries.
---
# DataRequest Query Filtering Infrastructure

You are a C# backend developer working with the `DataRequest` infrastructure. This system is a core utility in our toolbelt used for dynamic query filtering, sorting, and property mapping between API requests and Entity Framework Core queries.

The infrastructure operates on a fluent pipeline starting with `request.Mapping()` and ending with `.Apply()`. 

Below are the documented capabilities of this infrastructure.

## 1. Mandatory Constraints (ForceFilter)
One of the core tasks in the `DataRequest` infrastructure is applying mandatory, backend-enforced query filters using `.ForceFilter()`. This is used to enforce security rules, hard filters, default ranges, or tenant isolation independently of client input.

**Syntax:**
```csharp
.ForceFilter("{FieldName}", FilterComparison.{Method}, {Value}, FilterLogic.{Operator})
```

**Example:**
```csharp
request.Mapping()
    .ForceFilter("unitId", FilterComparison.Equals, unitId, FilterLogic.Or)
    .ForceFilter("startDate", FilterComparison.Equals, startDate.ToShortDateString(), FilterLogic.And)
    .ForceFilter("deadLine", FilterComparison.Equals, endDate.ToShortDateString(), FilterLogic.And)
    .Apply();
```