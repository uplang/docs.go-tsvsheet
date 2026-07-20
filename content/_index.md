---
title: Home
---

**The [tsvsheet](https://github.com/tsvsheet/tsvsheet) engine as an importable Go library.** A `.tsvt` file _is_ a spreadsheet — a single TAB-separated grid whose cells are literal values or `=formulas` that address other cells in A1 notation (`B2`, `D2:D5`), computed in place. `go-tsvsheet` parses that grid, evaluates every formula in dependency order, and hands you the computed values — with no filesystem or network access unless you inject it.

The library is the one engine behind the [`tsvsheet`](https://github.com/tsvsheet/tsvsheet.go) CLI, its browser editor, and its TUI. Import it to compute `.tsvt` sheets from your own Go programs, services, or tools.

## Install

```sh
go get github.com/tsvsheet/go-tsvsheet
```

API reference: **[pkg.go.dev/github.com/tsvsheet/go-tsvsheet](https://pkg.go.dev/github.com/tsvsheet/go-tsvsheet)**.

## Quickstart

```go
package main

import (
	"fmt"

	tsvsheet "github.com/tsvsheet/go-tsvsheet"
)

func main() {
	src := []byte("1\t2\n=A1+B1\t=A1*B1\n")

	sheet, err := tsvsheet.Parse(src)
	if err != nil {
		panic(err) // a formula syntax error, matchable with errors.Is(err, tsvsheet.ErrSyntax)
	}

	grid := sheet.Compute() // grid is a [][]string of computed cell text
	fmt.Println(grid[1][0], grid[1][1]) // 3 2
}
```

`Parse` compiles each `=formula` cell (literals pass through untouched); `Compute` evaluates them in dependency order and returns the value grid. A bad reference yields `#REF!`, a cycle yields `#CIRC!`, division by zero yields `#DIV/0!` — the error propagates as the cell's value, exactly like a conventional spreadsheet.

## The model

A `.tsvt` is the whole spreadsheet: one grid, each cell a literal or an `=formula`. Formulas address other cells in A1 notation — a single cell (`B2`), a range (`D2:D5`), or, when you inject a loader, a cross-sheet reference (`"prices"!A1`). The expression sublanguage is Excel-faithful: `^` (power), `&` (concatenation), postfix `%`, `TRUE`/`FALSE`, error-value literals, and a broad function library. Because the file is plain TSV, a sheet versions as text and diffs line by line. The language itself is specified in the [tsvsheet](https://github.com/tsvsheet/tsvsheet) repo.

### Pipes — shell-style formula composition

Formulas can compose like a pipeline: `expr | fn(arg, …)` is sugar for the call with the piped expression as its first argument — `=A1 | round(2)` is exactly `=round(A1, 2)`, and `=A2:A10 | sort() | unique() | count()` folds left into the composed calls. The engine desugars the pipe at parse time (SPECIFICATION §5.4), so evaluation, `Check`, `Explain`, and the dependency graph see only ordinary function application — there is no second execution path — while rendering (`Explain` traces, structural edits) preserves the author's pipe spelling.

## Core API

- **`Parse(src []byte) (Sheet, error)`** — compile a `.tsvt` grid into an immutable `Sheet`. A malformed formula is `ErrSyntax`, naming the offending cell.
- **`Sheet.Compute() Grid`** / **`ComputeAt(t time.Time) Grid`** — evaluate every formula and return the `Grid` (`[][]string`). `ComputeAt` injects the clock, so volatile functions (`TODAY`, `NOW`, `ISNOW`) are deterministic and testable.
- **`Sheet.ComputeWith(ComputeOptions) Grid`** — compute with injected collaborators (below).
- **`Check(s Sheet) []Diagnostic`** — static diagnostics (unknown functions, provable arity errors, non-A1 references) without computing.
- **`Sheet.Explain(...) Trace`** — trace how a cell's value was produced: its formula, the inputs it read, and the result.
- **`ReadTSV(io.Reader) (Grid, error)`** / **`WriteTSV(io.Writer, Grid) error`** — the tab-separated wire format.
- **Structural edits** — `Set`, `InsertRow`/`DeleteRow`/`InsertCol`/`DeleteCol` return a new `Sheet` with every A1 reference rewritten to follow the move. `Precedents`/`Dependents` expose the dependency graph.

`Sheet` is immutable and every method is a value receiver: computing, editing, or inspecting a sheet never mutates it, so sheets are safe to copy, alias, and share across goroutines.

### The document layer — comments and shebang round-trip

`Parse` reads the grid; **`ParseDocument(src []byte) (Document, error)`** reads the whole file, additionally recording its physical line layout. A `Document` is a `Sheet` plus that layout, so full-line comments (`# …`) and a `#!` shebang — which the grid drops — survive editing and are written back in position:

- **`Document.Sheet() Sheet`** — the parsed sheet the document wraps.
- **`Document.Text() []byte`** — serialize the document canonically: lines in layout order, comments verbatim, grid rows tab-joined, every line newline-terminated. For input already in canonical form, `ParseDocument` followed by `Text` is byte-identity.
- **`Document.SetCell`**, **`InsertRow`/`DeleteRow`/`InsertCol`/`DeleteCol`** — the structural edits at the document level; each returns a new `Document` with the layout kept in step.

`Text` is the one sanctioned way to serialize a sheet back to disk — writing a grid out by hand loses every comment line and the shebang. Like `Sheet`, `Document` is immutable: every editing operation returns a new `Document`.

### Standalone expressions — the engine without a sheet

A formula's expression can be compiled and evaluated on its own, detached from any sheet:

- **`CompileExpr(src []byte) (Expr, error)`** — compile one bare expression, the text that would follow `=` in a formula cell. A malformed expression is `ErrSyntax` carrying line/column detail.
- **`Expr.Eval(g Grid, opts ComputeOptions) Value`** — evaluate against a grid with the exact semantics of a formula cell in a sheet over that grid: reference resolution, literal coercion, ranges, dynamic arrays, error-value propagation, volatile functions from `opts.At`, `Limits` enforcement, and `Loader`/`Fetcher` gating. It returns error values, never Go errors.
- **`FormatValue(v Value) string`** — the canonical computed-cell text for a value, byte-identical to what `WriteTSV` emits for it in a computed grid; a 2-D array value reduces to its scalar-context (top-left) value before formatting.

A compiled `Expr` is an immutable value: compile once, then evaluate it against any number of grids, including concurrently. This is the surface the [tq](https://github.com/tsvsheet/tq) query language's engine ([go-tq](https://github.com/tsvsheet/go-tq)) evaluates every embedded expression through.

## Injected collaborators — the engine stays pure

The engine touches neither the filesystem nor the network on its own. Cross-file features work only through collaborators you inject via `ComputeOptions`, which keeps `go-tsvsheet` deterministic and trivially testable:

- **`Loader`** — resolves `SHEET("file")` / `"file"!A1` cross-sheet references. Without a loader, those cells resolve to `#REF!`. You control the base directory and containment.
- **`Fetcher`** — resolves content-typed `IMPORT*` cells (fetching values from an endpoint that speaks the tsvsheet media type). A `nil` fetcher disables imports (every `IMPORT*` is `#IMPORT!`). The concrete HTTP fetcher, allowlist, and caching live in the host program, not the engine.
- **`Limits`** — bounds every allocation (array-result cells, string bytes, grid dimensions) so an untrusted sheet cannot exhaust memory. `DefaultLimits()` is generous for a workstation; `BrowserLimits()` is sized for a WASM tab.

```go
grid := sheet.ComputeWith(tsvsheet.ComputeOptions{
	At:     time.Now(),
	Loader: myLoader,          // enable SHEET(...) / "file"!A1
	Fetcher: myFetcher,        // enable IMPORT*(...)
	Limits: tsvsheet.DefaultLimits(),
})
```

## Error values and sentinels

Two distinct kinds of error:

- **Spreadsheet error _values_** — a cell that fails to evaluate carries an `ErrorValue`: `#REF!`, `#VALUE!`, `#NAME?`, `#DIV/0!`, `#CIRC!`, `#N/A`, `#NUM!`, `#NULL!`, `#SPILL!`, `#IMPORT!`. These propagate through formulas as data; they are not Go errors.
- **Go error _sentinels_** — functions return `errs.Const` sentinels matchable with `errors.Is`: `ErrSyntax` (a malformed formula), `ErrInvalidValue`, `ErrReadInput`, `ErrWriteFile`, `ErrNotFound`.

## Design

The public surface is one package — `github.com/tsvsheet/go-tsvsheet` — a thin, documented facade. The evaluator, the A1 resolver, the function library, and the generated parser are all internal; no parser or grammar type escapes into the public API. Every allocation is bounded, every collaborator is injected, and the engine is `filesystem`- and `network`-free by construction.

## Related

- **[tsvsheet](https://github.com/tsvsheet/tsvsheet)** — the language and grammar specification.
- **[tsvsheet.go](https://github.com/tsvsheet/tsvsheet.go)** — the CLI, browser editor, and TUI built on this engine.
