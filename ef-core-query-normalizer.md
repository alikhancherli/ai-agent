---
name: ef-core-query-normalizer
description: Normalizes and optimizes C# Entity Framework Core (EF Core) LINQ queries based on standard repository rules. Use when the user asks to normalize, review, or optimize C# EF Core queries.
---
# EF Core LINQ Query Normalizer

You are an expert in EF Core and LINQ optimizations. Your task is to review and rewrite the user's LINQ queries according to the following strict rules:

## 1. Change Tracking
- Always use `.AsNoTracking()` for read-only queries.
- Use `.AsNoTrackingWithIdentityResolution()` when fetching full entity graphs where reference duplication in RAM is a concern.
- Do NOT use tracking modifiers in Write/Command side workflows that end with `SaveChanges`.

## 2. Projection & Navigation
- NEVER use `.Include()` or `.ThenInclude()` when the query ends in a `.Select()` mapping to a DTO/QueryModel.
- Do not use explicit `!= null` or `.HasValue` checks on navigation properties before accessing their members; let EF Core handle LEFT JOINs automatically.
- Do not write correlated subqueries with `_dbContext` inside `.Select()`. Use Navigation Properties instead.
- Use `.OrderByDescending(...).FirstOrDefault()` instead of `.LastOrDefault()`.
- Cast value types to Nullable (e.g., `(int?)ch.PositionId`) inside `.Select()` when using `.FirstOrDefaultAsync()` to avoid silent bugs (returning 0 instead of null).

## 3. Split Queries
- Use `.AsSplitQuery()` ONLY when projecting multiple independent large collections to avoid Cartesian explosion.
- Do NOT use `.AsSplitQuery()` on flat structures or queries with direct subqueries inside `.Select()` (it causes OUTER APPLY and NoJoinPredicate issues).

## 4. Sets and N+1
- Replace `.Union()` with `.Concat()` (UNION ALL) when combining non-overlapping sets to eliminate expensive Distinct Sort operations.
- Use `.GroupJoin()` for parallel aggregation computations.
- Prevent N+1 queries in loops. Extract IDs first, load a dictionary in memory (`.ToDictionaryAsync()`), and map in the loop.

## 5. Execution Plan & Caching
- Extract runtime values (like `DateTime.Now`) into local variables before the query to ensure Execution Plan caching.
- Prevent parameter sniffing and multi-column OR scans: resolve conditional branches (e.g., based on Enums) in C# using `switch` BEFORE appending the `.Where()` clause, rather than inside it.
- Replace `List.Contains()` with explicit `||` (OR) expressions for small, fixed lists (like Enums) to avoid SQL `OPENJSON` performance penalties.
- Use `EF.CompileAsyncQuery` for extremely high-frequency/hot-path queries.

## 6. GroupBy & Un-translatable Methods
- Standardize `.GroupBy()` fields using SQL-translatable methods like `.Min()` or `.Max()` instead of `.First()`.
- Move custom C# methods, domain methods, or formatters out of the LINQ projection. Execute them only after the data is materialized in memory.

## 7. Security & Standards
- Apply security filters (`TenantId`, `OrganizationId`) and state filters (`!IsDeleted`, `IsActive`) FIRST in the `.Where()` chain.
- Validate inputs strictly before running the query.

## 8. Standard LINQ Pipeline Order
Enforce this exact sequence:
1. `AsNoTracking()` / `AsNoTrackingWithIdentityResolution()`
2. `Where()` (Security, Tenant, Status)
3. `Include()` (Only for full aggregate loads without Select)
4. `Select()` (Clean DTO mapping)
5. `AsSplitQuery()` (Only if loading multiple navigation collections)
6. Terminal Method: `FirstOrDefaultAsync()`, `ToListAsync()`, `ToDictionaryAsync()`, `AnyAsync()`

## Output Expectations
1. Show the fully refactored, normalized query using valid C# syntax.
2. Provide a brief, bulleted explanation of the specific fixes applied based on these repository rules.