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

## Task 3: Run the full pipeline (no `--quick`) and keep the evidence
- Removed only `evidence/` from .gitignore and completed all eight pipeline steps without --quick for town A; retained all generated JSON evidence and all four hunt modes. Export provenance is in meta.json; viewer.json has no _run field in the provided exporter.
- Replay used 400 scenarios (0 dropped): 103 replay crashes (10 side, 93 rear), 2 reactive crashes (1 front, 1 side), and 0 crashes in both.
- Monte Carlo: k/n = 3/3000, p = 0.001; 95% Wilson interval [0.000340, 0.002936]. Estimate seed totals: 0.000688585, 0.000446182, 0.000538259.
- TRAIN selected policy {"cutinLat": 3.0}; TEST failure estimate 7.53e-7, valid upper bound 0.0388 (bus bypass unseen), versus baseline estimate 0.000407 and upper bound 0.00227. Policy candidates were unchanged.
- Mining accepted 1492/1500 draws (99.47%); baseline/chosen avoidable counts were 0/0, total collisions 1/0, and unattributed counts 1/0. With no avoidable failures observed, report conditional upper bound 0.00256819 and per-trip upper bound 0.000205455 for both policies.
- Final validation: python -m pytest -q tests/test_todos.py passed all 31 tests; no changes to simulator, runner, tests, or student code were needed.

## Task 4: REPORT.md — town and questions 1–4
- Filled the town and Vietnamese answers for questions 1–4 in REPORT.md.
- Added evidence citations, replay and importance sampling tables, and uncertainty bounds.
- Preserved the student ID, template headings, and questions 5–9.
- Ran python -m pytest -q tests/test_todos.py: 31 passed.
