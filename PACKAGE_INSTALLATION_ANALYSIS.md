# Package Installation Analysis

## Overview
This document analyzes package installations (pip) found in the NVIDIA Skills repository. The codebase uses Python package management in two primary contexts: dependency declarations and runtime installations.

## Key Findings

### 1. Requirements Files
Found **5 requirements.txt files** declaring Python package dependencies:

#### Medical Imaging Skills
- **skills/nv-generate-mr-brain/requirements.txt**
- **skills/nv-generate-mr/requirements.txt**
- **skills/nv-segment-ct/requirements.txt**
- **skills/nv-segment-ctmr/requirements.txt**

These files contain consistent dependencies:
```
nibabel>=4.0
numpy>=1.23
typer>=0.9
torch>=2.1
monai>=1.5
scipy>=1.10
scikit-image>=0.20
einops>=0.7
huggingface_hub>=0.20
tqdm>=4.65
fire>=0.5
tensorboard>=2.14
PyYAML>=6.0
```

#### Alert Management
- **skills/vss-manage-alerts/scripts/alert-notify/requirements.txt**

### 2. Runtime Package Installation Patterns

#### A. **omniverse-cad-to-simready** (Most Complex)
Located in: `skills/omniverse-cad-to-simready/references/`

Two main Python scripts perform dynamic pip installations:

**`preflight/scripts/preflight.py`** (Lines 815-822)
- Function: `_simready_pip_install_command()` - Creates pip install commands
- Supports two backends:
  - **uv** (modern Python package manager): `uv pip install --python <python>`
  - **pip** (traditional): `python -m pip install --disable-pip-version-check`
- Timeout: 1800 seconds (30 minutes)
- Used for:
  - Installing runtime requirements
  - Installing packages without dependencies: `--no-deps`
  - Installing from requirements.txt files

**`simready-validate/scripts/run.py`** (Lines 395-402)
- Function: `_run_pip_install()` - Executes pip install
- Uses: `python -m pip install --disable-pip-version-check`
- Timeout: 900 seconds (15 minutes)
- Captures output with `capture_output=True`
- Returns `subprocess.CompletedProcess[str]`

#### B. Installation Workflow
The installation pattern follows these steps:

1. **Virtual Environment Creation**
   - Creates venv at `{venv_root}/simready-validate`
   - Uses `uv venv --python 3.12` if available, otherwise `python -m venv`

2. **Dependency Installation**
   - Installs runtime requirements first
   - If USD-core fails to resolve, tries fallback with USD Exchange SDK
   - Installs SimReady package itself with `--no-deps` flag
   - Adds extra runtime requirements: `SIMREADY_RUNTIME_EXTRA_REQUIREMENTS`

3. **Fallback Strategy**
   - Function: `_should_try_simready_usd_exchange_fallback()` (line 812)
   - Checks for failure patterns:
     - "usd-core" in error message
     - "no matching distribution" in error message  
     - "resolutionimpossible" in error message
   - Falls back to alternative USD distribution if detected

#### C. Plugin Mirror
The exact same installation logic is mirrored in:
- `plugins/nvidia-skills/skills/omniverse-cad-to-simready/`
- Same structure, same functionality

### 3. Other Installation Mentions

**skill-card-generator/scripts/render_card.py** (Line 31)
- Hardcoded command: `pip install jinja2 --break-system-packages`
- Used in card rendering context

**vss-deploy-profile/scripts/normalize_resolved_yml.py** (Line 38)
- Comment: "ephemeral env on demand, so no `pip install` on the host is needed"
- Indicates intentional avoidance of host-level pip installation

### 4. Installation Context Patterns

#### When Installations Happen:
1. **Preflight Validation** - During `omniverse-cad-to-simready` preflight checks
2. **Runtime Discovery** - When resolving CLI executables that aren't on PATH
3. **Conditional Creation** - Only if not already installed or if check-only mode is disabled

#### Safety Features:
- **Version Check Disabled**: `--disable-pip-version-check` used to avoid delays
- **No Dependencies**: `--no-deps` used for controlled dependency installation
- **Isolated Environments**: Installations happen in project-specific venvs, not system-wide
- **Error Handling**: Comprehensive try/catch with fallback strategies
- **Timeout Protection**: 900-1800 second timeouts to prevent hanging
- **Output Capture**: All subprocess output captured for diagnostics

### 5. Special Flags and Configuration

| Flag | Purpose | Files |
|------|---------|-------|
| `--disable-pip-version-check` | Skip pip version check (faster) | preflight.py, run.py |
| `--no-deps` | Install package without dependencies | preflight.py, run.py |
| `--break-system-packages` | Allow installation in system Python | render_card.py |
| `--python` (uv only) | Specify Python interpreter | preflight.py |

### 6. Virtual Environment Usage

All runtime pip installations occur in isolated virtual environments:

```
{venv_root}/
└── simready-validate/          # Default location
    ├── bin/python              # Interpreter used for pip
    ├── lib/pythonX.Y/site-packages/
    └── ... (venv structure)
```

**Advantages**:
- No pollution of system Python
- Isolated dependency sets
- Easy cleanup/reinstall
- Per-project configuration

### 7. Distribution/Upstream Handling

The preflight script downloads and manages upstreams:
- SimReady Foundation checkout
- USD Convert CAD/gSplat tools
- Content Agents services

Pip installations are used to:
1. Build/install tools from upstream checkouts
2. Manage runtime dependencies for validation tools
3. Prepare environments for content generation

## Security Considerations

1. **No Untrusted Sources**: All installations from project-local requirements files
2. **Index Control**: Uses pip defaults (typically PyPI) - could be customized
3. **Version Pinning**: Requirements files can enforce specific versions
4. **Subprocess Safety**: Uses `check=False` to handle failures gracefully
5. **Timeout Protection**: Prevents DOS via long-running installs

## Recommendations for Analysis Tools

When analyzing this codebase for package installations:

1. **Focus Areas**:
   - `skills/omniverse-cad-to-simready/` - Most complex installation logic
   - `skills/*/requirements.txt` - Dependency declarations
   - `scripts/run.py` and `scripts/preflight.py` - Installation executors

2. **Key Functions**:
   - `_run_pip_install()` - Actual pip execution
   - `_simready_pip_install_command()` - Command builder
   - `_simready_pip_install_step()` - Step orchestrator

3. **Trace Points**:
   - Look for `subprocess.run()` with pip/python commands
   - Follow venv creation logic for environment isolation
   - Check requirements file resolution paths
   - Monitor fallback strategies for different platforms

4. **Test Cases**:
   - Verify venv isolation (no system Python pollution)
   - Test fallback paths (USD-core resolution failures)
   - Validate timeout behavior (long-running installs)
   - Check error message parsing for smart fallback
