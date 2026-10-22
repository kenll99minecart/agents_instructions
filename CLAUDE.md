# Code style

- Max 3 levels of indentation. When a 4th appears, return early with guard clauses, or extract the nested block into a named function.
- No comments that explain *what* the code does. When you want one, extract the code into a function and let the name replace the comment.
- Comments explain *why* only, and only for: a workaround (with a ticket number), a counterintuitive business rule, or a deliberately-avoided obvious approach.
- Public APIs and complex data structures must have doc comments (params, return, exceptions, side effects).
- Don't commit commented-out code. No `TODO` / `FIXME` — file a ticket, put the number in the code.
- Banned names: `data`, `info`, `temp`, `obj`, `val`, `res`, `stuff`. No invented abbreviations. Don't put the type in the name (no `strName`, `userList`).
- An `and` in a function name means split the function.
- No `utils` / `helpers` / `common` / `manager` files or classes. Group by domain concept.
- One concept, one word across the whole codebase (once you pick `fetchUser`, no `getUser`).
- Boolean names are affirmative statements: `isValid`, `hasPermission`. No negated names like `isNotReady`.
- Variable names must not be single character and must represent what does the variable contains in plain english
- Max 2 levels of bracket, i.e. for readability, do not include multiple functions at each line, e.g. plot(concat(preprocess(A))), separate them into different lines
- Functions should only be functionally meaningful, i.e. for very simple functions steps that can be done in 1-4 lines, avoid collapsing them into a function. Instead, generally for important meaningful steps, such as specific preprocessing step, imputation or calculation, make that as a function, and in the end of the code I can understand whatthe code is running by looking at the main pipeline functions.

# Architecture constraints

- Default to composition, not inheritance. `extends` only for pure abstract base classes/interfaces, max 1 level deep.
- No boolean switches or empty method bodies in a parent to accommodate individual subclasses. Use composition.
- A class doesn't create its own dependencies. Database, HTTP, storage, clock, randomness — all constructor-injected and declared as interface types.
- No `new Date()`, `Date.now()`, or `Math.random()` in business code. Inject the clock and randomness source.
- Don't read global singletons, static instances, or service locators. Object wiring happens only at the entry point.
- Abstract on the third occurrence. Keep two similar pieces duplicated. Don't merge coincidental duplication (looks alike but will evolve separately).
- Don't add a boolean parameter to an existing function to fit a new caller. Redraw the abstraction boundary.
- No abstraction layer that only forwards and makes no decisions.
- Write business logic as pure functions; push side effects to the edges. Don't mutate arguments — return new values.
- Transform data with `map` / `filter` / `reduce`, not hand-written accumulator loops.

# Performance

- No optimization without profiling data. A performance PR must attach before/after benchmarks.
- But choosing the right data structure and algorithm is not premature optimization: hash maps for lookups, no queries inside loops (N+1). Get these right from the start.
- No micro-optimizations (bit ops instead of division, manual loop unrolling).
- Readability is the default; trading it for performance requires benchmarks plus a comment stating why
- Do not put assertions and try except functions everywhere. The file loading code should be 1-2 lines only, and always ask if the file is non-standard.
# Packages

- use scanpy internal function as much as possible, avoid using h5py and anndata only if specific operations are only available there.

# Workflow

- **IMPORTANT: If you're uncertain, don't write the code. Ask first, or write a minimal check first.**
- Before writing a file, Look at what functions need to be written first, group every overlapping functions and look for existing implmentations of the simpler versions of the code in each functions and at each function, then write the code changes. 
- Classes are permitted, but only make a class of the central object if there are many attributes added to that class. If the main object in that class is directly from another known package class, i.e. the variable msot used is already an object, avoid making it as a class.
- Alway use the herdr interface to spawn sub-agents.
- Spawn a separate agent to review the code using the rules from CODEFORMAT.md. Flag the code and made changes while still consistent with the code asethetics in this .md file.
<!-- - After a series of code changes, run the typecheck or define the variable type at each step. -->
<!-- - Run the single relevant test file, not the whole suite. -->