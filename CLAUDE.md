# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

An Emacs minor mode that calculates and displays **Cognitive Complexity** metrics for source code using tree-sitter. Port of the tree-sitter `codemetrics` plugin. Implements the metric from [G. Ann Campbell's paper](https://www.sonarsource.com/docs/CognitiveComplexity.pdf). Also supports cyclomatic complexity as an alternative metric.

Requires Emacs 29.1+ (for built-in tree-sitter support).

## Build Commands

Uses [Eask](https://emacs-eask.github.io/) as the package manager. All commands assume you're in the project root.

```bash
make ci              # Full pipeline: clean → package → install → compile → test
make install         # Install dependencies (eask install-deps --dev)
make compile         # Byte-compile .el files
make test            # Run all ERT tests (eask test ert-runner)
make checkdoc        # Lint docstrings
make lint            # Run package-lint
make clean           # Clean generated files
```

**Run a single test file:**
```bash
eask test ert-runner test/c-test.el
eask test ert-runner test/rust-test.el
```

## Architecture

Two source files:

- **`cognitive-complexity.el`** — Main module. Analysis engine, tree traversal, overlay display, minor mode (`cognitive-complexity-mode`). Entry points: `cognitive-complexity-analyze`, `cognitive-complexity-region`, `cognitive-complexity-buffer`.
- **`cognitive-complexity-rules.el`** — Language-specific scoring rules. Each language has a `cognitive-complexity-rules-<lang>` function returning an alist of `(treesit-node-type . scoring-rule)`.

### Scoring Rules Format

Rules are alists mapping tree-sitter node types to either:
- **Simple rules:** `(node-type . (score-weight . increment-nested?))` — static score, optionally increases nesting depth
- **Function rules:** `(node-type . scoring-function)` — custom logic (e.g., recursion detection, logical operator sequences, class nesting)

Key special-case scoring functions are defined in `cognitive-complexity.el` and forward-declared in `cognitive-complexity-rules.el`:
- `cognitive-complexity-rules--class-declaration` — scores nested classes only
- `cognitive-complexity-rules--logical-operators` — scores sequences of 2+ operators
- `cognitive-complexity-rules--recursion` / `--recursion-using-node-name` — detects self-calls
- `cognitive-complexity-rules--method-declaration` — scores deeply nested methods

### Mode-to-Rules Mapping

The `cognitive-complexity-rules` defcustom maps major modes to rule sets. Both legacy modes (e.g., `c-mode`) and tree-sitter modes (e.g., `c-ts-mode`) are mapped. Languages: Bash, C, C++, C#, Elisp, Elixir, Go, Java, JavaScript, Julia, Kotlin, Lua, PHP, Python, Ruby, Rust, Scala, Swift, TypeScript.

## Test Structure

Tests use the `cognitive-complexity-test` macro from `test/test-helper.el`:

```elisp
(cognitive-complexity-test test-name "test/lang/File.ext" major-ts-mode
  '(expected-total-score
    (node-type . node-score)
    ...))
```

Each test opens a sample source file, runs `cognitive-complexity-buffer`, and compares the total score and per-node breakdown against expected values. Sample files live in `test/<language>/` subdirectories.

## CI

GitHub Actions matrix: Ubuntu/macOS/Windows × Emacs 29.4/30.2/snapshot. Runs `make ci`.
