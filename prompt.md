Refactor this repository according to the attached
TANSTACK_AI_EXPERIMENTS_REFACTOR_HANDOFF.md.

Use the existing project under tempalte/ as the starting point.
Do not generate a second starter or discard useful existing code.

First inspect the current checkout and record baseline build,
type-check, and lint results. Then normalize the workspace at the
repository root and execute the refactor milestones.

Preserve the existing shared UI, Better Auth, Bun workspace
structure, and useful tooling. Convert the frontend to actual
TanStack Start and the device backend to Effect 4 on Bun.
Use local SQLite with the documented database ownership rules.

Replace or explicitly supersede the stale PostgreSQL specification
already checked into this repository.

Continue into the complete collaborative-chat implementation.
Keep the scope to chat, not a workbench.

Maintain progress and verification records. Run the tests and
distinguish passing, failing, and blocked checks. Do not stop
after producing another plan.