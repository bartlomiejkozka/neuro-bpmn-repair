# neuro-bpmn-repair

> **Targeted BPMN Process Model Repair via Local SLMs, Conformance Checking, and Formal Verification**

`neuro-bpmn-repair` is an interactive neurosymbolic framework designed for the formal verification and targeted repair of BPMN 2.0 process models. It extends the verification pipeline by combining dynamic event-log analysis (*conformance checking*), Petri-net soundness checking, and privacy-preserving, locally hosted Small Language Models (SLMs), alongside cloud-hosted LLMs.

Part of a master's thesis: *Integrating Small Language Models, Conformance Checking, and Formal Verification for Targeted Repair of BPMN Process Models* — AGH University of Krakow.

---

## Origins and Attribution

This project is an independent continuation and hard fork of **bpmn-refine** originally developed by Michał Jurzak at AGH University of Krakow (https://github.com/bartlomiejkozka/neuro-bpmn-repair).

Key extensions introduced in `neuro-bpmn-repair`:
- **Dynamic Conformance Checking:** Integration of event-log analysis (deviations, alignments) into the multi-tiered formal validation pipeline.
- **Privacy-Preserving SLMs:** Primary reasoning layer utilizing local Small Language Models (e.g., via Ollama/vLLM) to process sensitive business operational data without external exposure, with continued support for hosted LLM providers (OpenAI, Anthropic).
- **Closed-Loop Neurosymbolic Repair:** Generation of atomic repair plans derived from execution discrepancies, backed by automated formal revalidation (Petri-net soundness and structural integrity).

Both the original codebase and all subsequent modifications are licensed under the **GNU Affero General Public License v3.0 (AGPLv3)**.

---

## What it does

- **Validates** BPMN diagrams with deterministic structural rules (R001–R008), Petri-net soundness checking, event-log conformance checking, and semantic review.
- **Repairs** diagrams by synthesizing targeted, atomic fixes based on structural defects and observed execution log alignments.
- **Refines** diagrams through chat: users describe intent or provide operational constraints, and the system suggests verifiable changes.
- **Dual Model Support:** Built with a primary focus on private, locally hosted Small Language Models (SLMs) for confidential operational data, while retaining full compatibility with state-of-the-art hosted Large Language Models (LLMs) for comparative benchmarking.

Repair and refinement proposals require acceptance by default. A user can explicitly enable Auto mode to apply a proposal and continue the repair loop.

The supported local harnesses include Ollama / local SLM runtimes alongside CLI harnesses (`codex_cli`, `claude_cli`) and hosted API providers. See [LLM integration](docs/llm-integration.md) for executable paths, authentication, isolation, and version-pinning guidance.


## Architecture

```
frontend/          React + bpmn-js — diagram editor, validation panel, chat
backend/app/
  model/           canonical BPMN IR + swappable converters (XML, JSON, YAML, Mermaid)
  validation/      tier 1 deterministic rules, tier 2 Woflan adapter (no LLM dependency)
  llm/             provider-agnostic client, model routing, versioned prompts
  api/             FastAPI endpoints: /diagrams, /validate, /repair, /chat, /history
```

See [`docs/architecture.md`](docs/architecture.md) for a detailed walkthrough.

## Quick start

Run all commands from the repository root. You need Python 3.12 or newer, `uv`, `make`, and Node.js 22.x (22.22.2 or newer) with `npm`.

Install the locked dependencies:

```bash
uv sync --extra dev --frozen
source .venv/bin/activate
npm --prefix frontend ci
```

Start the backend in one terminal:

```bash
make backend
```

Start the frontend in another terminal, again from the repository root:

```bash
make frontend
```

Open `http://localhost:5173`. The backend is available at `http://localhost:8000`, with OpenAPI at `http://localhost:8000/docs` and a health check at `http://localhost:8000/health`.

For a first check, import `data/rule_cases/R000_valid_baseline.bpmn` and click **Verify Rules**. The panel should show **No issues found**. This check needs no `.env` file or LLM provider. Semantic validation, chat, and LLM-based repair need a configured provider. Create the local configuration without overwriting an existing file, then replace the relevant placeholder with a valid credential:

```bash
test -e .env || cp .env.example .env
```

Restart the backend after changing `.env`. Use the **Models** control in the toolbar to choose the provider and model for each LLM interaction. Hosted providers require the matching API key. Ollama requires a local server at `OLLAMA_BASE_URL`; Codex CLI and Claude Code use existing local authentication when their executable is available.

Docker Compose requires a root `.env` file. Create it without overwriting an existing file, then run:

```bash
test -e .env || cp .env.example .env
docker compose up --build
```

The containerized frontend uses `http://localhost:3000`.

## Repository map and retention

| Path | Role | Retention policy |
|---|---|---|
| `backend/app/`, `frontend/src/` | Runtime application code | Active source; versioned |
| `backend/tests/`, `evaluation/tests/` | Automated verification | Active source; versioned |
| [`data/`](data/README.md) | Benchmark, ground truth, source-corpus metadata, and test fixtures | Versioned selectively; see the data map |
| `data/import_cases/`, `data/rule_cases/`, `data/test_cases/` | Import, deterministic-rule, and integration fixtures | Active test data; versioned |
| `data/pmo-dataset/` | Locally fetched source corpus | Local input cache; only provenance files are versioned |
| `evaluation/` | Dataset generation and audit tooling | Active research code; versioned |
| `experiments/specs/runs/` | Exact executed configurations and selections | Reproducibility evidence; versioned |
| `experiments/results/` | Canonical E1–E9 outputs, manifests, analyses, and reviewed derivatives | Reproducibility evidence; versioned; only per-trial resume checkpoints are ignored |
| `experiments/notebook/` | Analysis notebook and thesis figure generator | Reproducibility evidence; versioned |
| `workspaces/` | Interactive session snapshots | Local runtime state; ignored, but not treated as a disposable build cache |
| `.venv/`, `frontend/node_modules/`, `frontend/dist/`, Python and pytest caches | Installed or generated artifacts | Recreatable and ignored |

The detailed experiment layout and the exceptions for manually reviewed or
rescored outputs are documented in [`experiments/README.md`](experiments/README.md).

## Running tests

```bash
make test             # backend + frontend
make test-backend
make test-frontend
```

## CLI

Run the backend-local CLI without starting FastAPI:

```bash
PYTHONPATH=backend .venv/bin/python -m app.cli validate data/rule_cases/R000_valid_baseline.bpmn
PYTHONPATH=backend .venv/bin/python -m app.cli validate data/rule_cases/R000_valid_baseline.bpmn --json
PYTHONPATH=backend .venv/bin/python -m app.cli roundtrip data/rule_cases/R000_valid_baseline.bpmn --out /tmp/roundtrip.bpmn
PYTHONPATH=backend .venv/bin/python -m app.cli batch-validate data/rule_cases --recursive --format jsonl --out /tmp/validation.jsonl
```

With a configured LLM provider, chat and repair can use the same versioned fixture:

```bash
PYTHONPATH=backend .venv/bin/python -m app.cli chat data/rule_cases/R000_valid_baseline.bpmn --message "add a manager approval step" --out /tmp/refined.bpmn
PYTHONPATH=backend .venv/bin/python -m app.cli repair data/rule_cases/R000_valid_baseline.bpmn --out /tmp/repaired.bpmn
```

With `.venv` activated, the CLI is also exposed as:

```bash
bpmn-refine validate data/rule_cases/R000_valid_baseline.bpmn
```

Experiment-oriented commands support metadata such as runtime, dataset path, provider, model, and prompt filename:

```bash
bpmn-refine validate data/rule_cases/R000_valid_baseline.bpmn --json
bpmn-refine repair data/rule_cases/R000_valid_baseline.bpmn --json
bpmn-refine batch-validate data/rule_cases --recursive --format json --out /tmp/results.json
```

## Documentation

| Document | Description |
|---|---|
| [`docs/README.md`](docs/README.md) | Documentation map, ownership, and update rules |
| [`docs/development.md`](docs/development.md) | Local setup, configuration, tests, and API discovery |
| [`docs/architecture.md`](docs/architecture.md) | System design, module responsibilities, data flow |
| [`docs/validation-rules.md`](docs/validation-rules.md) | Tier 1 deterministic rules (R001–R008) |
| [`docs/formal-checkers.md`](docs/formal-checkers.md) | Tier 2 formal checkers and the shared issue shape |
| [`docs/llm-integration.md`](docs/llm-integration.md) | LLM client layer, model routing, prompt versioning |
| [`docs/converter-format.md`](docs/converter-format.md) | Diagram model, converter protocol, adding new formats |
| [`docs/repair-loop.md`](docs/repair-loop.md) | Repair modes, dispatcher, edit ops |
| [`docs/experiments.md`](docs/experiments.md) | `ExperimentConfig`, metrics, running a sweep |
| [`docs/run.md`](docs/run.md) | The `run` reproducibility envelope |
| [`evaluation/README.md`](evaluation/README.md) | Dataset construction and audit entry point |
| [`experiments/README.md`](experiments/README.md) | Executed E1–E9 runs and retained artifacts |

## License

This project is licensed under the **GNU Affero General Public License version 3 (AGPL-3.0-only)**.

- Original codebase: Copyright (C) 2026 Michał Jurzak.
- Modifications and extensions (`neuro-bpmn-repair`): Copyright (C) 2026-2027 Bartłomiej Kózka.

See the full [LICENSE](LICENSE) file for complete terms and [LICENSING.md](LICENSING.md) for scope and third-party notices. The derived evaluation dataset retains its separate [source attribution and license](data/eval/v1.0/ATTRIBUTION.md).
