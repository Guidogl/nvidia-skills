# Package Installation Analysis

## Overview
This document analyzes package installations (pip) found in the NVIDIA Skills repository. The codebase uses Python package management in two primary contexts: dependency declarations and runtime installations.

## How Installation Works in the skills.sh Flow

### Key distinction: `skills.sh` is a marketplace, not a runtime sandbox

`skills.sh` is the **distribution catalog** where these NVIDIA skills are
published (alongside [NVIDIA Build](https://build.nvidia.com/skills)). Users
install skills with the `skills` CLI:

```bash
npx skills add nvidia/skills
```

This copies **instruction files only** — `SKILL.md`, `skill_manifest.yaml`,
`scripts/`, and `references/` — into the agent's skill directory. It does
**not** install any Python packages, clone upstream repos, or download model
weights. Per the README: *"You do not need to clone this repo or copy skill
folders by hand."* The marketplace's job is to ship instructions plus
capability governance (signing/verification via `nv-agent-root-cert.pem`,
grouping via `skills.sh.json`).

**pip never runs "inside skills.sh."** Package installation happens later, at
**skill execution time**, on whatever host machine the agent runs on, when the
agent follows the SKILL.md instructions.

### The three-layer dependency architecture

Each skill declares dependencies in three coordinated places:

| Layer | File | Role |
|-------|------|------|
| **1. Declaration** | `skill_manifest.yaml` (`runtime.side_effects.pip_packages`) | Source-of-truth metadata; a declared "side-effect contract" the governance layer reads to disclose what will be installed/modified *before* you run it |
| **2. Instruction** | `SKILL.md` | Tells the agent to emit a bash block that runs `pip install` at runtime, fused to the run command |
| **3. Fulfillment** | `requirements.txt` | The actual pinned package list `pip install -r` resolves against |

The manifest also declares environment impact, e.g.:
```yaml
environment:
  modifies_active_python_environment: true
  clean_environment_recommended: true
  recommended_isolation: fresh venv or container for benchmarks
```

The SKILL.md deliberately fuses install + run because the runtime is assumed to
be **possibly-fresh and ephemeral** each time (see `nv-generate-mr/SKILL.md`
line 26: *"the runtime may be a fresh environment without nibabel/MONAI, so
dropping the install fails with ModuleNotFoundError"*).

### Two installation styles

**Style A — "thin wrapper, install at run"** (medical imaging skills:
`nv-generate-mr`, `nv-generate-mr-brain`, `nv-segment-ct`, `nv-segment-ctmr`)
- `git clone` an upstream NVIDIA repo into `.workbench_data/upstreams/`
- `pip install -r requirements.txt` into the **active environment** just before running
- Accepts polluting the active env only when the caller chose it; benchmarks should use a fresh venv/container

**Style B — "managed, isolated venv"** (`omniverse-cad-to-simready` preflight)
- Creates a **dedicated venv** (`{venv_root}/simready-validate`), preferring `uv venv --python 3.12`, falling back to `python -m venv`
- Installs into *that* venv via `uv pip install` or `python -m pip install --disable-pip-version-check`
- Has fallback dependency resolution (USD-core failures → USD Exchange SDK)
- Invoked via a POSIX shell shim `preflight.sh`: `exec "${PYTHON:-python3}" "$SCRIPT_DIR/preflight.py" "$@"`

### Installation lifecycle

```
skills.sh marketplace
   │  npx skills add nvidia/skills
   ▼
Agent's skill directory  ← only instructions land here (NO pip yet)
   │  agent loads SKILL.md when a task matches
   ▼
Agent emits the documented bash block on the HOST machine
   │
   ├─ Style A: pip install -r requirements.txt → active env (or run-chosen venv)
   └─ Style B: preflight.sh → preflight.py → creates venv → uv/pip install
   ▼
Skill's wrapper script runs against the now-satisfied dependencies
```

### Practical implications

- **Installing a skill is cheap and side-effect-free.** No GPU, torch, or multi-GB downloads at `npx skills add` time — those are deferred to first execution.
- **The `pip_packages` manifest field is your pre-flight disclosure** of exactly what will be installed and whether the active Python env is modified.
- **Environments are assumed ephemeral**, which is why every run re-runs `pip install` — fitting sandboxed/containerized agent runtimes.
- **For reproducible/benchmark runs, prefer isolation** (fresh venv or container); the omniverse skill enforces this by building its own venv.

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
