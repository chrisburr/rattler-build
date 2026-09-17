# How to Build a New-Style Conda-Forge Package with Rattler-Build

This guide explains how to create conda-forge compatible packages using rattler-build, the modern Rust-based conda package builder.

## Prerequisites

1. **Install rattler-build:**
   ```bash
   pixi global install rattler-build
   # or
   conda install rattler-build -c conda-forge
   ```

2. **Required system dependencies:**
   - **Linux/macOS:** `patchelf` (Linux), `install_name_tool` (macOS), `git`, `patch`
   - **Windows:** MSVC compiler, `git`, `patch` (via `m2-patch`)

## Basic Package Structure

Create a directory for your package with these files:
```
my-package/
├── recipe.yaml           # Main recipe file
├── variants.yaml         # Optional: build variants
└── patches/              # Optional: source patches
    └── fix-something.patch
```

## Recipe Format

### 1. Basic Recipe Structure

```yaml
context:
  name: my-package
  version: "1.0.0"

package:
  name: ${{ name|lower }}
  version: ${{ version }}

source:
  url: https://github.com/owner/repo/archive/v${{ version }}.tar.gz
  sha256: abcd1234...  # Use `openssl dgst -sha256 filename` to get hash

build:
  number: 0
  script:
    - if: win
      then: |
        mkdir build && cd build
        cmake -G "Ninja" -DCMAKE_INSTALL_PREFIX=%LIBRARY_PREFIX% ..
        ninja install
    - if: unix
      then: |
        mkdir build && cd build
        cmake ${CMAKE_ARGS} -DCMAKE_INSTALL_PREFIX=$PREFIX ..
        make -j${CPU_COUNT} install

requirements:
  build:
    - ${{ compiler('c') }}        # Compiler for target platform
    - ${{ compiler('cxx') }}      # C++ compiler if needed
    - cmake
    - if: win
      then: ninja
    - if: unix
      then: make
  host:
    - libxml2                     # Libraries to link against
  run:
    - libxml2                     # Runtime dependencies

tests:
  - script:
      - if: unix
        then:
          - test -f $PREFIX/bin/my-executable
          - my-executable --version
      - if: win
        then:
          - if not exist %LIBRARY_BIN%\\my-executable.exe exit 1
  - python:                       # Python import tests
      imports:
        - my_package
      pip_check: true

about:
  homepage: https://github.com/owner/repo
  license: MIT
  license_file: LICENSE
  summary: Short description of the package
  description: |
    Longer description explaining what this package does
    and why someone would want to use it.
  documentation: https://my-package.readthedocs.io
  repository: https://github.com/owner/repo

extra:
  recipe-maintainers:
    - your-github-username
```

### 2. Python Packages

For Python packages, use this pattern:

```yaml
context:
  version: "2.1.0"

package:
  name: python-mypackage
  version: ${{ version }}

source:
  url: https://pypi.io/packages/source/m/mypackage/mypackage-${{ version }}.tar.gz
  sha256: abcd1234...

build:
  noarch: python                 # Platform-independent Python package
  script:
    - python -m pip install . -vv --no-deps --no-build-isolation

requirements:
  host:
    - python >=3.8
    - pip
    - setuptools                  # or hatchling, poetry-core, etc.
  run:
    - python >=3.8
    - numpy >=1.19
    - requests

tests:
  - python:
      imports:
        - mypackage
      pip_check: true
  - script:
      - python -c "import mypackage; print(mypackage.__version__)"
```

### 3. Multi-Output Packages

For packages that generate multiple outputs:

```yaml
context:
  version: "1.0.0"

package:
  name: mylib
  version: ${{ version }}

source:
  url: https://example.com/mylib-${{ version }}.tar.gz
  sha256: abcd1234...

# First output (default)
build:
  number: 0

outputs:
  - package:
      name: libmylib              # Library package
    build:
      script:
        - cmake --build . --target install_lib
    requirements:
      build:
        - ${{ compiler('c') }}
        - cmake
      run:
        - libxml2

  - package:
      name: mylib-tools           # Tools package
    build:
      script:
        - cmake --build . --target install_tools
    requirements:
      build:
        - ${{ compiler('c') }}
        - cmake
      run:
        - libmylib ==${{ version }}
    tests:
      - script:
          - mytool --help
```

## Build Variants and Matrix Builds

Create `variants.yaml` for building across multiple configurations:

```yaml
# variants.yaml
python:
  - "3.9"
  - "3.10"
  - "3.11"
  - "3.12"

numpy:
  - "1.19"
  - "1.21"
  - "1.24"

# Pin numpy versions to Python versions
zip_keys:
  - [python, numpy]
```

## Common Patterns

### 1. Conditional Dependencies

```yaml
requirements:
  build:
    - ${{ compiler('c') }}
    - if: linux
      then: sysroot_linux-64 2.17    # For old glibc compatibility
  host:
    - if: win
      then: vs2019_win-64
    - if: osx
      then: clang_osx-64
  run:
    - if: py>=3.9
      then: typing_extensions >=4.0
```

### 2. Cross-Compilation

```yaml
build:
  script:
    - if: build_platform != target_platform
      then: |
        # Cross-compilation specific steps
        export CC=$CC_FOR_BUILD
        ./configure --host=$HOST --build=$BUILD
    - if: build_platform == target_platform
      then: |
        # Native compilation
        ./configure
```

### 3. Source Patches

```yaml
source:
  url: https://example.com/source.tar.gz
  sha256: abcd1234...
  patches:
    - fix-compilation.patch
    - if: win
      then: windows-specific.patch
```

## Building Commands

### 1. Basic Build
```bash
# Build for current platform
rattler-build build recipe.yaml

# Specify output directory
rattler-build build recipe.yaml --output-dir ./output

# Build with variants
rattler-build build recipe.yaml -m variants.yaml
```

### 2. Cross-Platform Builds
```bash
# Build for different platforms
rattler-build build recipe.yaml --target-platform linux-64
rattler-build build recipe.yaml --target-platform osx-64
rattler-build build recipe.yaml --target-platform win-64

# Build for multiple architectures
rattler-build build recipe.yaml --target-platform linux-aarch64
```

### 3. Development Workflow
```bash
# Render recipe without building (useful for debugging)
rattler-build build recipe.yaml --render-only

# Render with dependency resolution
rattler-build build recipe.yaml --render-only --with-solve

# Keep build environment for debugging
rattler-build build recipe.yaml --keep-build

# Skip tests during development
rattler-build build recipe.yaml --test skip

# Build with debug output
rattler-build build recipe.yaml --debug
```

### 4. Testing
```bash
# Test an existing package
rattler-build test my-package-1.0.0-py311h123456_0.conda

# Test with debug environment
rattler-build test my-package-1.0.0-py311h123456_0.conda --debug
```

## Conda-Forge Specific Considerations

### 1. Build Numbers
- Start with `build: number: 0`
- Increment for recipe-only changes (no version bump)
- Reset to 0 when version changes

### 2. Naming Conventions
- Use lowercase package names
- Prefix Python packages with `python-` if they conflict with other packages
- Use `lib` prefix for C/C++ libraries

### 3. License Requirements
- Always specify `license` and `license_file`
- Use SPDX license identifiers when possible
- Include license file in package

### 4. Dependencies
- Pin to specific versions only when necessary
- Use `>=` for minimum versions
- Specify Python version ranges: `python >=3.8,<3.13`

### 5. Tests
- Always include meaningful tests
- Test imports for Python packages
- Test executables and basic functionality
- Use `pip_check: true` for Python packages

## Troubleshooting

### Common Issues

1. **Missing Dependencies:**
   ```bash
   # Check what dependencies are available
   rattler-build build recipe.yaml --render-only --with-solve
   ```

2. **Build Failures:**
   ```bash
   # Keep build directory for inspection
   rattler-build build recipe.yaml --keep-build --debug
   ```

3. **Test Failures:**
   ```bash
   # Test specific package with debug output
   rattler-build test package.conda --debug
   ```

4. **Cross-compilation Issues:**
   ```bash
   # Check if tools are correctly set up
   echo $CC $CXX $AR $RANLIB
   ```

### Debugging Tips

1. **Check rendered recipe:**
   ```bash
   rattler-build build recipe.yaml --render-only > rendered.json
   ```

2. **Inspect build environment:**
   ```bash
   rattler-build debug recipe.yaml
   # Then manually run build commands in the created environment
   ```

3. **Validate against conda-forge standards:**
   ```bash
   # Use conda-smithy tools if available
   conda-smithy recipe-lint recipe.yaml
   ```

## Advanced Features

### 1. Recipe Generation
```bash
# Generate recipe from PyPI
rattler-build generate-recipe pypi numpy

# Generate from CRAN
rattler-build generate-recipe cran ggplot2

# Generate from CPAN
rattler-build generate-recipe cpan DBI
```

### 2. Upload to Staging
```bash
# Upload to conda-forge staging (requires tokens)
rattler-build upload conda-forge package.conda --staging-token $TOKEN
```

### 3. Using TUI
```bash
# Interactive build interface
rattler-build build recipe.yaml --tui
```

This guide provides the foundation for creating conda-forge compatible packages with rattler-build. For more complex scenarios, refer to:

- **Local examples directory:** `/home/cburr/Development/rattler-build/examples/` - Contains real-world recipe examples including:
  - `curl/` - C library with platform-specific build scripts
  - `rich/` - Python noarch package example
  - `xtensor/` - C++ header-only library
  - `mamba/` - Complex package with variants
  - `recipe-folder/` - Multi-package recipes
- **Recipe file reference:** `/home/cburr/Development/rattler-build/docs/reference/recipe_file.md` - Complete recipe format specification
- **Full documentation:** https://prefix-dev.github.io/rattler-build/ - Comprehensive rattler-build documentation