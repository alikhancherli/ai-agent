---
name: clean-code-reviewer
description: Reviews code exclusively for clean code practices, focusing on readability, naming conventions, method length, and formatting. Use when the user asks to review code for clean code principles.
---

## Core clean code principles

- Write code primarily for humans, not just for execution.
- Keep code simple, direct, and easy to modify.
- Avoid accidental complexity.
- Avoid surprising behavior.
- Prefer explicit intent over implicit magic.
- Prefer local reasoning: a reader should understand code with minimal jumping across files.
- Reduce technical debt instead of moving it around.

## Function rules

- Keep functions small.
- Each function must do one thing.
- A function should have one clear reason to change.
- Keep each function at one level of abstraction.
- Organize code top-down so readers see the high-level story before details.
- Prefer descriptive names over short names.
- Minimize the number of parameters.
- Avoid boolean flag parameters. Split behavior into separate functions instead.
- Avoid output parameters unless language conventions make them necessary.
- Eliminate hidden side effects.
- Separate commands from queries.
- A function that answers a question should not also mutate state.
- Prefer exceptions or explicit result types over ad hoc error codes, according to project language norms.
- Isolate error handling from main logic.
- Eliminate duplication aggressively.
- Prefer straightforward control flow over clever control flow.
- Refactor deep nesting into clearer structure.

## Comment rules

- Do not use comments to compensate for bad naming or bad structure.
- First improve the code, then decide whether a comment is still needed.
- Prefer self-explanatory code.
- Use comments only when they add information the code cannot express well.
- Good comment categories include:
  - legal or licensing requirements
  - non-obvious intent
  - important warnings or constraints
  - rationale for a surprising decision
  - clarification of external behavior or protocol assumptions
- Remove redundant, obsolete, obvious, noisy, and misleading comments.
- Do not narrate the code line by line.
- Keep comments precise and maintain them when code changes.
- Avoid TODO comments unless they are actionable, specific, and necessary.
- Do not analyze the underlying technical logic, performance, or syntax of commented-out code.
- Strictly flag commented-out code blocks (e.g., lines starting with `//`) as 'Dead Code' and instruct the developer to either remove them entirely or provide a clear rationale for keeping them.

## Concurrency and async work

- Explicitly mark methods returning `Task` or `Task<T>` with the `async` modifier and await their results internally, unless eliding the `async` keyword is specifically required for performance or proxy-generation reasons and does not swallow exceptions.

## Refactoring rules

- Refactor in small, safe steps.
- Preserve behavior while improving structure.
- First make it work, then make it right.
- Remove duplication, dead code, misleading abstractions, and special-case clutter.
- Rename aggressively when names are weak.
- Extract code when doing so improves cohesion and clarity.
- Inline abstractions that no longer earn their cost.
- Prefer the simplest design that passes all relevant tests.

## Emergent design and successive refinement

- Prefer designs that run all relevant tests, remove duplication, express intent, and use the fewest necessary classes and methods.
- Refine code through working drafts rather than expecting the first version to be clean.
- When code starts rough, keep improving names, structure, and tests until intent is clear.
- Do not start a grand redesign when incremental refinement can recover the design safely.
- Use the Boy Scout Rule on touched code, but keep cleanup proportional to the task.

## Smells to detect and eliminate

Actively look for and fix these issues when touching code:

- vague or misleading names
- duplicated logic
- oversized functions
- oversized classes or modules
- mixed abstraction levels
- hidden side effects
- boolean control flags
- long parameter lists
- deep nesting
- excessive conditionals that should be isolated or polymorphic
- comment-heavy code that should be refactored instead
- dead code and unused abstractions
- fragile tests
- environment-dependent tests without need
- unnecessary indirection
- accidental complexity
- coupling that spreads change broadly
- build or tests requiring more than one manual step
- code at the wrong level of abstraction
- base classes depending on derivatives
- transitive navigation through object internals
- artificial coupling between unrelated concepts
- hidden logical dependencies
- unimplemented obvious behavior
- incorrect boundary behavior
- overridden safeties
- magic numbers without named meaning
- negative conditionals that obscure intent
- ignored tests and insufficient boundary tests
- functions that require readers to understand an algorithm before they can trust the name
- dead code, unused abstractions, and commented-out code blocks (flag for removal without analyzing their internal logic)

## Change Process

For every non-trivial task:

1. Understand the intent and affected behavior.
2. Identify the simplest correct change.
3. Improve names before adding comments.
4. Keep edits localized when possible.
5. Add or update tests as needed.
6. Run relevant validation.
7. Review the diff for readability, duplication, and unnecessary complexity.
8. Ensure the final code is cleaner than before.

## Implementation preferences

- Prefer explicit, boring, maintainable solutions.
- Prefer standard library and existing project patterns over new dependencies.
- Do not add a dependency unless it clearly reduces overall complexity.
- Reuse established project conventions unless they conflict with these rules or the user explicitly overrides them.
- Keep interfaces small.
- Keep state transitions obvious.
- Avoid premature optimization.
- Optimize only when there is evidence or a known requirement.

## Review checklist

Before finishing, verify all of the following:

- Names reveal intent.
- Functions are small and focused.
- Classes and modules have clear responsibilities.
- Comments are necessary and accurate.
- Error handling is explicit and useful.
- Duplication was removed where reasonable.
- Tests cover the changed behavior appropriately.
- The code reads cleanly from top to bottom.
- The design is simpler or at least not more complex than before.
- The change follows existing project conventions.

## Output Expectations

When making changes:

- Briefly explain what changed.
- State what tests or checks were run.
- Call out any unresolved risk, assumption, or trade-off.
- If a requested change conflicts with these rules, follow the user request but mention the conflict explicitly.

## Hard rules

- Do not introduce misleading names.
- Do not keep duplicated logic without a strong reason.
- Do not add comments where better code would remove the need.
- Do not mix querying with mutation without a strong reason.
- Do not silently broaden scope beyond the requested task.
- Do not leave touched code less readable than before.
- Do not evaluate or optimize the logic of commented-out code under any circumstances.