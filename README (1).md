# devdiag — Packaged CLI Diagnostics Tool

`devdiag` is a distributable, dependency-free Python command-line tool that inspects a machine and produces a **deterministic developer-environment health report** in both human-readable text and structured JSON.

## What it checks

| Check | Description |
|-------|-------------|
| Python version | Compares the running interpreter against a configurable minimum (default 3.9) |
| Disk space | Free space at a configurable path against a minimum in GB (default 1 GB) |
| Environment variables | Required vars (missing = **fail**) and optional vars (missing = **warn**) |
| Developer tools | Required / optional tools on `PATH` (e.g. `git`, `node`, `docker`), with detected versions |

## Installation

Requires Python 3.9+. Use an isolated virtual environment:

```bash
git clone <your-repo-url> devdiag
cd devdiag
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -e ".[dev]"          # use `pip install .` for a plain install
```

This registers the `devdiag` console entry point (`devdiag = devdiag.cli:main`). `python -m devdiag` also works.

## Usage

```bash
devdiag                                    # text report with defaults
devdiag -f json                            # JSON only
devdiag -f both                            # text followed by JSON
devdiag -c samples/config.example.json     # custom configuration
devdiag -o report.json                     # also write JSON report to a file
devdiag --strict                           # warnings become failures
devdiag --help
```

| Option | Meaning |
|--------|---------|
| `-c, --config FILE` | JSON configuration file |
| `-f, --format {text,json,both}` | Output format (default `text`) |
| `-o, --output FILE` | Write the JSON report to FILE as well |
| `--strict` | Exit with code 1 if there are warnings |
| `--version` | Print version |

### Exit codes

| Code | Meaning |
|------|---------|
| `0` | Healthy (or degraded with warnings only, without `--strict`) |
| `1` | One or more checks failed (or warnings with `--strict`) |
| `2` | Usage error (bad command-line arguments, unwritable output file) |
| `3` | Configuration error (missing, malformed or invalid config file) |

### Configuration

All keys are optional. Unknown keys are rejected so typos are caught early.

```json
{
  "min_python": "3.9",
  "disk_path": ".",
  "min_free_gb": 1,
  "required_env": ["PATH", "HOME"],
  "optional_env": ["VIRTUAL_ENV"],
  "required_tools": ["git", "python3"],
  "optional_tools": ["pip", "node", "docker"]
}
```

### Sample text output

```
devdiag 1.0.0 - developer environment report
======================================================
[ OK ] python             Python 3.13.18 (>= 3.9)
[ OK ] disk_space         29.98 GB free (>= 1.0 GB)
[ OK ] env:HOME           HOME is set
[ OK ] tool:git           git found (git version 2.43.0)
------------------------------------------------------
Overall: HEALTHY  (4 passed, 0 warnings, 0 failed)
```

### JSON report shape

```json
{
  "tool": "devdiag",
  "version": "1.0.0",
  "overall": "healthy | degraded | unhealthy",
  "summary": { "total": 0, "passed": 0, "warnings": 0, "failed": 0 },
  "checks": [
    { "name": "python", "status": "pass | warn | fail", "message": "...", "details": {} }
  ]
}
```

## Determinism

For the same machine state and configuration, output is byte-for-byte identical: checks run in a fixed order, names within each group are sorted, JSON keys are sorted, and no timestamps are embedded.

## Project layout

```
devdiag/
├── pyproject.toml
├── README.md
├── src/devdiag/
│   ├── cli.py        # argument parsing, exit codes
│   ├── checks.py     # Python / disk / env / tool checks (Probe abstraction)
│   ├── config.py     # config loading + validation
│   └── report.py     # JSON and text rendering
├── tests/test_devdiag.py
└── samples/          # example configs and generated reports
```

## Tests

```bash
pytest -v
```

The suite (22 tests) covers three required paths, using a fake `Probe` so results do not depend on the host machine:

- **Success** – healthy report, deterministic output, text/JSON formats, output file.
- **Missing dependency** – missing required tool/env var (fail, exit 1), missing optional tool (warn, exit 0), `--strict`, old Python, low disk.
- **Malformed configuration** – invalid JSON, wrong types, unknown keys, negative values, missing file (exit 3).

## Sample reports

See the `samples/` directory:

- `sample-report.txt` / `sample-report.json` – healthy run
- `sample-report-missing-dependency.txt` – missing required tool (exit 1)
- `sample-report-malformed-config.err` – malformed config (exit 3)

## License

MIT
