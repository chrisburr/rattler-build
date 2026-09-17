# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

rattler-build is a fast, standalone conda-package builder written in Rust that creates cross-platform relocatable binaries/packages from simple recipe formats. It's inspired by conda-build and boa but has no Python dependencies and works as a standalone binary.

## Development Commands

### Building and Testing
- `pixi run build-release` - Build release version
- `pixi run install` - Install the binary locally
- `pixi run test` - Run unit tests
- `pixi run test-slow` - Run tests in release mode
- `pixi run test-end-to-end` - Run end-to-end Python tests (requires build-release)
- `pixi run test-all` - Run all tests (slow + end-to-end)
- `pixi run test-ci` - Run CI tests with single thread
- `pixi run test-patch-extra` - Run tests with patch-test-extra feature

### Linting and Formatting
- `pixi run lint` - Run all linters and formatters
- `pixi run lint-fast` - Run fast linters (no clippy)
- `pixi run lint-slow` - Run slow linters including clippy
- `pixi run cargo-clippy` - Run clippy with project-specific settings
- `pixi run cargo-fmt` - Format Rust code
- `pixi run ruff-lint` and `pixi run ruff-format` - Python linting/formatting
- `pixi run typos` - Fix typos
- `pixi run pre-commit-install` - Install lefthook git hooks

### Documentation
- `pixi run --environment docs build-docs` - Build documentation with mkdocs
- `pixi run --environment docs docs` - Serve docs locally
- `pixi run generate-cli-docs` - Generate CLI reference documentation

### Development Tools
- `pixi run generate-test-data` - Generate test data for patches
- `pixi run update-snapshots` - Update test snapshots

## Architecture Overview

### Core Components

**Main Entry Points:**
- `src/main.rs` - CLI entry point with async runtime and command dispatching
- `src/lib.rs` - Library exports and core build orchestration functions
- `src/opt.rs` - Command-line argument parsing and configuration

**Recipe System:**
- `src/recipe/` - Recipe parsing, validation, and Jinja2 templating
  - `parser/` - YAML parsing for different recipe sections (build, requirements, tests, etc.)
  - `jinja.rs` - Custom Jinja2 environment for recipe templating
  - `error.rs` - Recipe-specific error handling with miette integration

**Build Pipeline:**
- `src/build.rs` - Core build orchestration and workflow management
- `src/metadata.rs` - Build metadata, output configuration, and package identification
- `src/source/` - Source handling (URL downloads, Git repos, local files, patching)
- `src/render/` - Dependency resolution and recipe rendering with variants
- `src/script/` - Build script generation and execution across platforms
- `src/packaging.rs` - Final package creation and compression

**Platform Support:**
- `src/linux/`, `src/macos/`, `src/windows/` - Platform-specific linking and environment setup
- `src/unix/` - Shared Unix functionality
- `src/post_process/` - Post-build processing (relinking, Python fixes, etc.)

**Testing and Quality:**
- `src/package_test/` - Package testing framework
- `test/end-to-end/` - Python-based integration tests using pytest

**Additional Systems:**
- `src/upload/` - Package upload to various registries (conda-forge, prefix.dev, etc.)
- `src/recipe_generator/` - Recipe generation from PyPI, CRAN, CPAN, LuaRocks
- `src/tui/` - Terminal user interface (optional feature)
- `src/variant_config.rs` - Multi-variant builds and configuration matrix

### Key Design Patterns

**Error Handling:** Uses miette for rich error reporting with source locations and suggestions.

**Async Architecture:** Built on tokio with async/await for concurrent operations like downloads and builds.

**Cross-Platform:** Extensive platform-specific code for linking, environment setup, and tool execution.

**Recipe Processing Pipeline:**
1. Parse YAML recipe with custom selectors and Jinja2 templating
2. Resolve variants and build matrix
3. Download/prepare sources
4. Resolve dependencies via rattler solver
5. Execute build scripts in isolated environments
6. Post-process binaries (relinking, etc.)
7. Package and compress outputs
8. Run tests if enabled

## Recipe Format

Recipes are YAML files with sections: `context`, `package`, `source`, `build`, `requirements`, `tests`, `about`, and `extra`. Key features:

- **Jinja2 templating** with `${{ variable }}` syntax
- **Conditional selectors** like `if: win` or `if: py>=3.8`
- **Multi-output recipes** for creating multiple packages from one recipe
- **Variant support** via variant_config.yaml files for build matrices

## Testing Strategy

**Unit Tests:** Rust tests with extensive snapshot testing using insta crate.
**Integration Tests:** Python pytest suite in `test/end-to-end/` testing actual recipe builds.
**Test Data:** Comprehensive test recipes in `test-data/recipes/` covering edge cases.

## Key Dependencies

- **rattler ecosystem:** Core conda functionality (rattler, rattler_solve, etc.)
- **tokio:** Async runtime
- **clap:** CLI parsing
- **miette:** Error reporting
- **serde:** Serialization
- **minijinja:** Jinja2 templating
- **reqwest:** HTTP client

## Development Notes

- Uses Rust edition 2024 with MSRV 1.86.0
- Pixi is the preferred package manager (see pixi.toml)
- Extensive use of feature flags for optional functionality (tui, recipe-generation)
- Cross-compilation support with platform-specific build logic
- Lefthook for git hooks with comprehensive linting pipeline

## Building New-Style Conda-Forge Packages

For conda-forge package creation:
1. Use `rattler-build build recipe.yaml` with appropriate variant configs
2. Recipe should follow conda-forge conventions with proper selectors
3. Test thoroughly with `--test native` or `--test skip` options
4. Leverage `examples/` directory for reference implementations
5. Use `rattler-build upload conda-forge` for staging uploads (hidden command)

The tool supports the full conda-forge workflow including variant matrices, cross-compilation, and package validation.