# Progress Log

## Task 1: All 10 TODOs (metrics, feasible, estimate, search, modes, policy)
﻿- Implemented all 10 Task 1 TODOs across the six student modules.
- Added vectorised metrics, contact classification, and feasibility checks.
- Implemented Wilson intervals, defensive weights, and importance-sampling estimates.
- Added stable CEM updates, failure-mode labels, and constrained policy selection.
- Validation: python -m pytest -q tests/test_todos.py: 31 passed.

## Task 2: Design the custom hazard file `hazard.json` (step G)
- Created hazard.json with a feasible L1 hard-braking event and all default scene parameters, with civil traffic enabled.
- Added four bounded hazard ranges and Vietnamese explanations citing NHTSA and FHWA, explicitly identifying exposure and timing assumptions.
- Verified studio validation returns [] True and ran the full 1,500-scenario mining command: 1,492 accepted (99.47%).
- No attributed planner failures were observed; the reported conditional upper bound is approximately 0.00257 (per-trip upper bound 0.000205).
- Ran python -m pytest -q tests/test_todos.py using the existing virtual environment: all 31 tests passed.
