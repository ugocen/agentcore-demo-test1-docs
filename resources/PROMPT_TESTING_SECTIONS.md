# Testing Sections for All Prompts
# This file documents the pytest testing requirements added to each prompt.

## Test Structure
```
tests/
  infra/          # Prompt_00 — Gate 0: AWS infrastructure
  agents/         # Prompt_01 — Gate 1: Agent deployment + unit tests
  telemetry/      # Prompt_02 — Gate 2: V5 payload + OTel
  backend/        # Prompt_03 — Gate 3: FastAPI health + DB
  temporal/       # Prompt_04 — Gate 4: Workflow + HITL + A2A
  frontend/       # Prompt_05 — Gate 5: CopilotKit + streams (Playwright)
  e2e/            # Prompt_06 — Gate 6: Full end-to-end (11 tests)
```

## Running All Tests
```bash
# Ensure .venv is active
source .venv/bin/activate

# Run all tests
pytest tests/ -v

# Run specific phase
pytest tests/infra/ -v        # Phase 0
pytest tests/agents/ -v       # Phase 1
pytest tests/telemetry/ -v    # Phase 2
pytest tests/backend/ -v      # Phase 3
pytest tests/temporal/ -v     # Phase 4
pytest tests/frontend/ -v     # Phase 5
pytest tests/e2e/ -v          # Phase 6

# Run with coverage
pytest tests/ -v --cov=app --cov=agents --cov-report=term-missing
```

## CRITICAL RULE (added to all prompts)
Every gate checkpoint MUST have a corresponding pytest test.
Write test functions that assert the gate condition programmatically.
