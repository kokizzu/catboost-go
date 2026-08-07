# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A minimal Go wrapper around the CatBoost C library's `model_calcer_wrapper.h` (the "dynamic C++ wrapper" API), used to load a trained CatBoost model and run predictions from Go via cgo. The entire package is two files: `model.go` (low-level bindings) and `classifier.go` (a binary-classification convenience wrapper). Package name is `catboost`.

There is no `go.mod` in this repository — it predates Go modules and is consumed as a plain GOPATH-style import (`go get github.com/ma3axaka/catboost-go`). Don't add a `go.mod` unless explicitly asked to modernize the project; doing so silently would change how downstream consumers `go get` this package.

## Build requirements

This package **cannot compile without the CatBoost C library installed on the machine** — `model.go` cgo-includes `model_calcer_wrapper.h` and links `-lcatboostmodel`. Building without it fails immediately with `fatal error: model_calcer_wrapper.h: No such file or directory`. This is expected in most sandboxes; it is not a bug in the Go code.

To actually build/vet/test this package, the CatBoost model_interface library must be built and discoverable, per the README:

```sh
git clone https://github.com/catboost/catboost.git
cd catboost/catboost/libs/model_interface && ../../../ya make -r .
export CATBOOST_DIR=$(pwd)
export C_INCLUDE_PATH=$CATBOOST_DIR:$C_INCLUDE_PATH
export LIBRARY_PATH=$CATBOOST_DIR:$LIBRARY_PATH
export LD_LIBRARY_PATH=$CATBOOST_DIR:$LD_LIBRARY_PATH
```

(Alternatively, install the compiled `.so`/`.h` files into `/usr/local/lib` / `/usr/local/include`.)

With the library available, standard commands apply (module mode is off since there's no `go.mod`):

```sh
GO111MODULE=off go build .
GO111MODULE=off go vet .
GO111MODULE=off go test .
```

There are currently no test files in the repo.

## Architecture

Two layers, both in package `catboost`:

- **`model.go` — low-level wrapper over `ModelCalcerHandler`.**
  - `Model` wraps the C handler as `unsafe.Pointer`.
  - `LoadFullModelFromFile(filename)` allocates a handler (`ModelCalcerCreate`) and loads a model file into it.
  - `CalcModelPrediction(floats, floatLength, cats, catLength)` runs raw prediction over a batch of samples. It marshals Go `[][]float32` and `[][]string` into C arrays (`**C.float`, `***C.char`), calling `CalcModelPrediction` in the C API, and returns raw `[]float64` predictions (one per sample). Categorical feature arrays are converted with the local `makeCStringArrayPointer`/`makeCharArray`/`freeCharArray` C helpers defined in the cgo preamble, and are freed via `defer` after the call.
  - `getError()` pulls the last error from the C side via `GetErrorString` and wraps it as a Go `error` — called whenever a C API call returns a false/failure status.
  - `Close()` must be called to release the C-side handler (`ModelCalcerDelete`); there is no finalizer, so leaking a `Model` leaks C memory.

- **`classifier.go` — binary classification convenience layer.**
  - `BinaryClassifer` (note: spelled without the second "s" — matches the existing exported API, don't "fix" the typo without checking call sites) wraps a `*Model`.
  - `LoadBinaryClassifierFromFile` loads a model and wraps it.
  - `PredictProba` calls `Model.CalcModelPrediction` and applies a `sigmoid` to each raw score, turning CatBoost's raw margin/probit output into a 0-1 probability-like score. This is standard for CatBoost binary-classification models trained with `Logloss`.
  - `Close()` delegates to the underlying `Model.Close()`.

### Key conventions to preserve

- Feature inputs are always passed as parallel `[][]float32` (numeric) and `[][]string` (categorical) slices, one inner slice per sample, plus explicit `floatLength`/`catLength` counts — these counts must match `Model.GetFloatFeaturesCount()` / `GetCatFeaturesCount()` and are not currently validated inside `CalcModelPrediction`, so callers are responsible for getting them right.
- All C-side fallibility is surfaced as Go `error` via `getError()`, not panics.
- Any new C API surface added to `model.go` should follow the same pattern: thin Go method on `Model`, convert Go slices to C arrays in the method body, check the C call's bool return, call `getError()` on failure.
