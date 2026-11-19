---
name: v1-writing-go
description: Use when writing or reviewing Go CLI tools, workers, utilities and services. Prefer standard library idioms, explicit resource ownership, bounded concurrency and cancellation while respecting the module's existing layout and Go version.
license: MIT
metadata:
  author: Farul Wananda Go conventions; pragmatic systems engineering
---

# V1 Writing Go

## Scope and repository checks

Use for Go implementation and runtime behavior, not to impose a directory template. Inspect `go.mod` (`go`/`toolchain` directives), build tags, existing package boundaries, CLI exit/output contracts and verification commands first. Match supported Go versions; do not introduce a newer language/library API silently.

A single executable may keep `main.go` at the root. Use `cmd/<tool>` for multiple entrypoints and `internal/` for actual private packages when useful. Go has no mandatory `cmd/internal/pkg` project layout or line-count limit. Do not add interfaces, constructors or dependencies without a real boundary.

For HTTP/REST routing, database/ORM persistence such as MySQL or PostgreSQL with GORM, authentication boundaries, server lifecycle or production delivery, also read [Go API guidance](../v1-writing-go-api/SKILL.md). Keep this skill responsible for Go language, errors, concurrency and resource ownership; the API skill does not mandate a router, database or application layer stack.

## Errors, resources and CLI behavior

- Return operational errors, adding useful context once. Use `%w` only when callers should retain `errors.Is`/`errors.As` access to the underlying error; wrapping may expose an implementation detail as part of the API. Generic `errors.AsType[E]` requires Go 1.26+; keep `errors.As` for older modules.
- Handle an error once: return it or log it at its owner, not at every layer. Preserve the error cause and actionable CLI context; keep machine-readable stdout separate from diagnostics on stderr.
- Only `main` chooses the exit code. Put cleanup in a returning `run` function so its defers execute before `os.Exit`; `os.Exit` skips deferred cleanup.
- Close an acquired resource after checking acquisition succeeded. Check close/flush errors when they determine data integrity, especially file writes; blindly deferring `Close` can lose those errors.
- For HTTP, use a reusable client with suitable timeouts, create requests with context, close response bodies and bound reads when input size is untrusted. A non-2xx status is not a transport error.
- Keep context propagation explicit for cancellable I/O/work; do not store request contexts or substitute `context.Background()` to escape caller cancellation.

## Bounded concurrency

Sequential work is the default until parallelism is useful. If concurrent, bound both active work and the number of waiting goroutines; a semaphore acquired inside one goroutine per item still creates unbounded goroutines.

Every goroutine needs an owner, cancellation path and join. The sender/owning producer closes a channel only after sends finish. A coordinator may close a multi-producer channel after all producers exit; consumers do not close shared input channels.

Complete reusable package example using Go 1.13+ APIs. When `go.mod` declares Go 1.25+, `wg.Go(func() { ... })` may replace the `Add`/`Done` pair; do not adopt it below that version. The callback must honor context for cancellation to interrupt its I/O; this pool cannot forcibly stop a callback that ignores it. First observed callback error wins; results are not ordered, completed side effects are not rolled back, and there are no automatic retries.

```go
package items

import (
	"context"
	"fmt"
	"sync"
)

func ProcessItems(
	ctx context.Context,
	items []string,
	workers int,
	handle func(context.Context, string) error,
) error {
	if workers < 1 {
		return fmt.Errorf("workers must be positive")
	}
	if err := ctx.Err(); err != nil {
		return err
	}
	if len(items) == 0 {
		return nil
	}
	if workers > len(items) {
		workers = len(items)
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	jobs := make(chan string)
	var wg sync.WaitGroup
	var once sync.Once
	var firstErr error

	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for item := range jobs {
				if ctx.Err() != nil {
					return
				}
				if err := handle(ctx, item); err != nil {
					once.Do(func() {
						firstErr = fmt.Errorf("item %q: %w", item, err)
						cancel()
					})
					return
				}
			}
		}()
	}

dispatch:
	for _, item := range items {
		select {
		case <-ctx.Done():
			break dispatch
		case jobs <- item:
		}
	}
	close(jobs)
	wg.Wait()
	if firstErr != nil {
		return firstErr
	}
	return ctx.Err()
}
```

The unbuffered job channel applies backpressure; goroutines and coordination storage scale with the configured worker bound, not input size. Cancellation is cooperative, not a guarantee that no work races with a cancellation signal. Use the installed `errgroup` only if the project already benefits from it; this example needs no extra module.

## Verification

Use existing table-driven tests for behavior and meaningful boundaries, not to require a particular directory layout. Run the focused package checks, then the actual CLI/worker path with isolated fixtures. For concurrent changes, exercise cancellation, first-error propagation, bounded active work and completion; run the race detector when supported. With Go 1.25+, `testing/synctest` gives deterministic fake-time tests for timeouts and goroutine quiescence. A race-free run does not prove cancellation or boundedness.

Run `go vet`; `go fix` (Go 1.26+ modernizers) only rewrites to APIs permitted by the module's `go` version, but review its diff as an intentional change rather than bundling it into an unrelated edit.

For CLIs, observe exit status, stdout/stderr separation and resource cleanup on error/signal. Do not spawn production workers or issue paid/provider operations for a smoke run. Never claim a benchmark improvement from fewer classes or files; measure the relevant allocation/I/O/CPU path if performance is in scope.

## Sources and provenance

Original project guidance, revised 2026-11-20; references inform principles rather than adding their dependencies:

- [Effective Go](https://go.dev/doc/effective_go) and [Code Review Comments](https://go.dev/wiki/CodeReviewComments).
- [Context](https://pkg.go.dev/context), [sync](https://pkg.go.dev/sync), [testing/synctest](https://pkg.go.dev/testing/synctest) and [os.Exit](https://pkg.go.dev/os#Exit): check documentation for the module's supported Go version.
- [Go release notes](https://go.dev/doc/devel/release) (1.25: `WaitGroup.Go`, `synctest`; 1.26: `errors.AsType`, `go fix` modernizers) for version gates.
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md): ownership, joining goroutines, cleanup and handling errors once. Uber-specific tooling and dependencies are not required.
