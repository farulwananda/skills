# Runnable Go HTTP Reference

## Contract and limits

`go-api-http-example` is a complete `main.go` using only the standard library. It needs Go 1.22+ syntax/APIs; for production select a currently supported patched toolchain, not the oldest version capable of compiling it.

It demonstrates a public, bounded arithmetic endpoint—not persistence, authentication, a provider integration, a production business application or a universal bootstrap. Import no new packages, create no database, and make no outbound calls. Keep existing application contracts instead of replacing them with this example.

- Default bind: `127.0.0.1:8080`; `HTTP_ADDR` can override it. Local loopback is intentional. Production exposure, TLS and ingress policy require a separate host decision.
- `POST /v1/square` accepts one JSON object with required integer `value` between -1,000,000 and 1,000,000; returns its exact int64 square. The 1 KiB body budget includes trailing data/whitespace consumed while checking EOF.
- Unknown fields, malformed JSON, wrong field types, missing/null values and extra JSON values are rejected. Standard encoding/json permits duplicate keys and matches struct field names case-insensitively; this is not a strict duplicate-key or case-sensitive parser.
- Parse/shape/range errors use 400, oversized consumed bodies 413 and unsupported media types 415. `ServeMux` retains its standard text 404/405 and Allow behavior; there is no claim of a global JSON error envelope. GET patterns also serve HEAD in Go's modern mux.
- `/healthz` is process liveness; `/readyz` represents only this dependency-free example's drain state, not database readiness. Both use GET/HEAD.
- Header/body/write/idle and drain limits are explicit example values, not deployment defaults. Streaming/upload routes need different policy. The arithmetic handler has no blocking downstream work; real downstream work must use request context/deadlines and appropriate admission limits.
- SIGINT/SIGTERM stops readiness and drains ordinary HTTP requests. No signal-derived BaseContext is assigned, so drain does not immediately cancel in-flight requests. A second signal restores the OS default. No hijacked sockets, background jobs or shared application dependencies exist in this example.

## Complete source

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log/slog"
	"mime"
	"net"
	"net/http"
	"os"
	"os/signal"
	"sync/atomic"
	"syscall"
	"time"
)

func writeJSON(w http.ResponseWriter, status int, value any) {
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Cache-Control", "no-store")
	w.Header().Set("X-Content-Type-Options", "nosniff")
	w.WriteHeader(status)
	// All callers supply small, known-serializable values. A failed write
	// cannot be repaired by sending a second HTTP status.
	if err := json.NewEncoder(w).Encode(value); err != nil {
		slog.Warn("response write failed")
	}
}

func writeError(w http.ResponseWriter, status int, code string) {
	writeJSON(w, status, struct {
		Error string `json:"error"`
	}{Error: code})
}

func rejectJSON(w http.ResponseWriter, err error) {
	var tooLarge *http.MaxBytesError
	if errors.As(err, &tooLarge) {
		writeError(w, http.StatusRequestEntityTooLarge, "body_too_large")
		return
	}
	writeError(w, http.StatusBadRequest, "invalid_json")
}

func square(w http.ResponseWriter, r *http.Request) {
	mediaType, _, err := mime.ParseMediaType(r.Header.Get("Content-Type"))
	if err != nil || mediaType != "application/json" {
		writeError(w, http.StatusUnsupportedMediaType, "unsupported_media_type")
		return
	}
	r.Body = http.MaxBytesReader(w, r.Body, 1024)
	decoder := json.NewDecoder(r.Body)
	decoder.DisallowUnknownFields()
	var input struct {
		Value *int64 `json:"value"`
	}
	if err := decoder.Decode(&input); err != nil {
		rejectJSON(w, err)
		return
	}
	if err := decoder.Decode(new(any)); err != io.EOF {
		rejectJSON(w, err)
		return
	}
	if input.Value == nil || *input.Value < -1_000_000 || *input.Value > 1_000_000 {
		writeError(w, http.StatusBadRequest, "invalid_value")
		return
	}
	writeJSON(w, http.StatusOK, struct {
		Square int64 `json:"square"`
	}{Square: *input.Value * *input.Value})
}

func run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	addr := os.Getenv("HTTP_ADDR")
	if addr == "" {
		addr = "127.0.0.1:8080"
	}
	listener, err := net.Listen("tcp", addr)
	if err != nil {
		return fmt.Errorf("listen: %w", err)
	}
	var ready atomic.Bool
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, http.StatusOK, struct {
			Status string `json:"status"`
		}{Status: "ok"})
	})
	mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
		if !ready.Load() {
			writeError(w, http.StatusServiceUnavailable, "draining")
			return
		}
		w.WriteHeader(http.StatusNoContent)
	})
	mux.HandleFunc("POST /v1/square", square)
	server := &http.Server{
		Handler:           mux,
		ReadHeaderTimeout: 3 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
		MaxHeaderBytes:    16 << 10,
	}
	served := make(chan error, 1)
	ready.Store(true)
	go func() { served <- server.Serve(listener) }()
	slog.Info("listening", "addr", listener.Addr().String())
	select {
	case err := <-served:
		return err
	case <-ctx.Done():
		stop()
	}
	ready.Store(false)
	slog.Info("draining")
	cleanup, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(cleanup); err != nil {
		closeErr := server.Close()
		<-served
		return errors.Join(fmt.Errorf("drain: %w", err), closeErr)
	}
	if err := <-served; err != nil && !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	slog.Info("stopped")
	return nil
}

func main() {
	slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil)))
	if err := run(); err != nil {
		slog.Error("server failed", "error", err)
		os.Exit(1)
	}
}
```

Incoming bodies remain owned/closed by net/http. Decoder allocations are limited by the consumed body budget. If real application output can fail serialization before transport, marshal/validate it before committing status; do not generalize this known-output helper into an unchecked arbitrary-response encoder.

The normal drain waits for Shutdown, not merely Serve. On drain timeout this example force-closes sockets and returns failure; Close is not proof that arbitrary handlers/goroutines terminated. A real app must separately join its workers and close shared clients/pools only after their users finish. Do not claim this sample implements those absent resources.

## Local execution and verification

Save the fence as `main.go` in a disposable directory and use the repository's supported Go toolchain. `go run main.go` starts the local example; use a built binary when exercising signal/exit behavior so signals target the server process rather than a go-run wrapper. No install, migration, external request or deployment is needed.

Exercise success and range boundaries, malformed/multiple/null/unknown-key/wrong-type JSON, missing content type, oversize input including trailing whitespace, method/unknown-route responses, HEAD, bind failure and clean exit. For drain, begin a request body before SIGTERM, then finish it during the drain and confirm its response plus process completion. An HTTP Expect: 100-continue exchange can establish that the handler is reading the body before sending the signal. Separately exercise a body that never finishes to observe the finite drain deadline/nonzero result; do not equate force-closing a socket with joining arbitrary work.

Use actual local requests as smoke evidence. Permanent regression tests should target consumer-visible contracts and uncertain boundaries, not log wording, copied constants, directory layout or source text. Do not infer database/ORM/auth/ingress/load guarantees from this public, dependency-free example.

## Recorded smoke evidence

On 2026-10-15, the exact source above passed `go vet ./...` and `go build -race` on Go 1.27.1, darwin/arm64. The built binary was exercised through real loopback HTTP: arithmetic/range/type/null/shape validation, the exact body boundary and oversized trailing whitespace, content types, 404/405/HEAD, bind failure, an in-flight request completing after SIGTERM, drain-timeout failure, and second-signal termination. No race report was observed.

This is sample evidence, not a production toolchain pin: the upstream release page already lists Go 1.27.2 with security fixes. Use the current supported patch for a production build. MySQL/PostgreSQL or other database/ORM integrations, authentication, ingress/TLS, deployment, load and backup restoration were not exercised by this dependency-free smoke.

## Sources and provenance

Original example, created 2026-10-15; informed by [net/http](https://pkg.go.dev/net/http), [routing changes](https://go.dev/blog/routing-enhancements), [encoding/json](https://pkg.go.dev/encoding/json), [signal.NotifyContext](https://pkg.go.dev/os/signal#NotifyContext), [slog](https://go.dev/blog/slog) and the selected toolchain's `go doc`. No copied upstream implementation or implied production certification.
