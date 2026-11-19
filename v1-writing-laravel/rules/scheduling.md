# Scheduling Decisions

A scheduler selects when work starts; it does not automatically require a queued job or a background process. Reuse existing registration conventions for the installed Laravel version.

- Use withoutOverlapping() when concurrent executions of the same scheduled task would violate an invariant or waste significant resources. Inspect existing locks, expected duration, expiry and crash recovery; safe overlap or independent partitions can be intentional.
- Use onOneServer() when several servers run the scheduler but the task should run once globally. Confirm a shared supported lock store and stable task identity. Do not add it when only one scheduler exists or per-server execution is required.
- Consider runInBackground() for scheduled command/exec tasks only when independent work may safely run concurrently and blocking later tasks is a demonstrated issue. Account for process capacity, completion/error reporting and overlap locks. Preserve sequential execution when order matters. It does not make the work a durable queue job.
- Use environment restrictions when the workflow must run only in selected environments. Preserve intended staging/test behavior; do not apply production-only restrictions to every task.
- Bound large processing with appropriate batching, checkpoints and time budgets. takeUntilTimeout() is a LazyCollection iteration tool, not a scheduler/worker timeout or a way to cancel a blocked HTTP call.
- Use schedule groups for actual repeated options when supported by the installed version. A few explicit declarations may be clearer than an abstraction.

Review timezone/DST, due times, retry/replay, locks and deployment registration before changing a schedule. Existing scheduler and queue flows must remain compatible; do not add worker infrastructure merely because a task is scheduled.

Source: Laravel 12.x [scheduling](https://laravel.com/docs/12.x/scheduling). Verify installed APIs, scheduler registration, shared lock support and process capacity. Do not run schedule:run or start workers as a harmless inspection command: due tasks may write data, send messages or incur provider charges.
