# PLAN — Lab 11 "Scenario Forge" (complete the lab)

## Project overview (read this first)

- **Stack:** Python 3.11 + numpy (+ pytest). No other dependencies (`requirements.txt`). Windows machine: run commands with `python` from the repository root.
- **Layout:**
  - `avsim/` — the simulator (`core.py` rollouts/actors/`DEFAULTS`/`HAZARD_DEFAULTS`, `space.py` natural distribution and towns, `studio.py` hazard-file loader with `LIMITS`, `check_space`, `hazard_ok`). **READ-ONLY. Never edit.**
  - `run_all.py` — the pipeline (`python run_all.py {world,replay,mc,hunt,estimate,fix,export,mine,all} [hazard.json] [--quick]`). Each step writes `evidence/<step>.json`. **READ-ONLY.** It reads the MSSV from the first line of `REPORT.md` (`MSSV: 202602673`), which sets the town and seeds — never change that line.
  - `tests/test_todos.py` — 31 public tests for the 10 TODOs (a hidden grader suite has the same shape but different numbers). `tests/` is **READ-ONLY**.
  - `lab11/*.py` — the student code: 10 functions that currently `raise NotImplementedError("TODO n: ...")`. Everything else in those files is given; do not change given helpers or signatures.
  - `policy_candidates.json` — candidate planner policies for step E (may only be extended with entries whose keys/values lie inside `run_all.POLICY_LIMITS`; do not pick candidates by looking at TEST numbers).
  - `README.md` (Vietnamese lab instructions), `RUBRIC.md` (grading), `REPORT.md` (the report template, 9 questions, Vietnamese).
  - `viewer/` — the browser 3D viewer (chapter 9 = evidence viewer, chapter 10 = hazard studio). Only humans use it.
- **Conventions:** match the surrounding code style (type hints, short docstrings, vectorised numpy, `from __future__ import annotations`). Implement each TODO **exactly as its docstring says** — no hard-coding numbers from the tests. Return plain Python `float`/`int`/`str`/`dict` where the docstring returns scalars.
- **Hard rules (grading caps):** do NOT modify `avsim/`, `run_all.py`, `tests/`, the natural distribution or seeds; never use `--quick` for submitted evidence; never choose a policy from TEST data; never report a probability of 0 for an unseen failure mode (report the upper bound instead).
- **Test command** (compact output, 31 tests):

TEST_CMD: python -m pytest -q tests/test_todos.py

---

## Task 1: All 10 TODOs (metrics, feasible, estimate, search, modes, policy)

TEST_CMD runs the whole suite, so all 10 TODOs are done in this one task. Sub-parts A–C below.

### Part A: Environment check + metrics TODOs (TODO 1, 2, 9)

**Files to modify:** `lab11/metrics.py` only (replace the three `raise NotImplementedError` bodies).

Steps:
1. Make sure `python -m pytest -q tests/test_todos.py` runs (collects 31 tests). If `numpy`/`pytest` are missing, run `python -m pip install -r requirements.txt`. Do not add config files.
2. **TODO 1 `min_ttc(R, cap=10.0)`** — fully vectorised over (K, NF), no Python loop over frames:
   - `F = R.F` (K, NF, N, 7; columns x, z, hx, hz, v, a, d); `e = F[:, :, core.IX["ego"]]`.
   - `lead = R.EGO[:, :, 3].astype(int)`; `valid = lead >= 0`; `j = np.where(valid, lead, 0)`.
   - Leader state: `L = np.take_along_axis(F, j[:, :, None, None], axis=2)[:, :, 0]`.
   - Leader half-length per actor: `acts = R.actors or core.ACTORS`; `half = np.array([0.3 if a.kind == "ped" else a.dims[0] / 2 for a in acts])` (for the base actors `a.dims[0] == core.LEN[j]`; using `R.actors` also covers hazard actors). `hl = half[j]`.
   - `lon = (L.x - e.x) * e.hx + (L.z - e.z) * e.hz`; `gap = np.maximum(lon - 2.3 - hl, 0.0)` (use `HALF_EGO`).
   - `dv = e.v - L.v`; `ok = valid & (dv > 0.1)`; `ttc = np.where(ok, gap / np.maximum(dv, 1e-6), np.inf)`; `out = np.minimum(cap, ttc.min(axis=1))`.
   - For every k with `ego_events(R, k)` non-empty, set `out[k] = 0.0`. Return float array of shape (K,).
3. **TODO 2 `contact_type(R, k, event)`**: other actor id = `event["b"] if event["a"] == "ego" else event["a"]`; index via `R.ids.index(other)`; `f = event["f"]`. With ego row `E = R.F[k, f, core.IX["ego"]]` and other row `O`: `rx, rz = O[0]-E[0], O[1]-E[1]`; `lon = rx*E[2] + rz*E[3]`; `lat = -rx*E[3] + rz*E[2]`. Return `"front"` if `lon > 2.3 and abs(lat) < 1.6`, `"rear"` if `lon < -2.3 and abs(lat) < 1.6`, else `"side"`.
4. **TODO 9 `progress_metrics(R)`**: `e = R.F[:, :, core.IX["ego"]]`; `dist = np.hypot(np.diff(e[..., 0], axis=1), np.diff(e[..., 1], axis=1)).sum(axis=1)`; `harsh = (e[..., 5] < -3.0).sum(axis=1)` (int). Return `(dist, harsh)`.

Part A check: `-k "min_ttc or contact_type or progress"` → 10 pass.

### Part B: Feasibility + estimation TODOs (TODO 3, 4, 6, 7)

**Files to modify:** `lab11/feasible.py`, `lab11/estimate.py`.

1. **TODO 3 `is_feasible(p)`** (returns a real `bool`, test uses `is`): `mu = p["mu"]`; return `False` if any of: `(math.pi / p["cutDur"])**2 * 3.5 / 2 > mu * G`; `p["cutBrake"] > mu * G`; `p["pedV"] > 2.5`; `not (0 < p["cutV"] <= 22)`. Otherwise `True`. Strict inequalities exactly as written.
2. **TODO 4 `wilson_ci(k, n, z=1.96)`**: if `n == 0` return `(0.0, 1.0)`. `ph = k/n`; `den = 1 + z*z/n`; `c = (ph + z*z/(2n)) / den`; `h = z*sqrt(ph(1-ph)/n + z*z/(4n*n)) / den`; `lo = max(0.0, c - h)`, `hi = min(1.0, c + h)`; when `k == 0` force `lo = 0.0` exactly. Return a tuple of floats.
3. **TODO 6 `log_weights(Z, Bn, nat, prop, beta=0.2)`**: `lp = nat.logpdf(Z, Bn)`; `lq_prop = proposal_logpdf(prop, Z, Bn)`; `lq = np.logaddexp(np.log1p(-beta) + lq_prop, np.log(beta) + lp)` (wrap in `np.errstate(divide="ignore")` so beta = 0 or 1 gives -inf terms safely); return `lp - lq`. Must be 0 when the proposal equals the natural distribution and always `<= log(1/beta)`.
4. **TODO 7 `is_estimate(logw, y, z=1.96)`**: `w = np.exp(np.asarray(logw, float))`, `y = np.asarray(y, float)`, `n = len(y)`, `wy = w*y`; `p = wy.mean()` (NOT self-normalised); `se = wy.std(ddof=1)/sqrt(n)` if `n > 1` else `0.0`; `lo = max(0.0, p - z*se)`, `hi = p + z*se`; `ess = w.sum()**2 / (w**2).sum()` (0.0 if the denominator is 0). Return `dict(p=, lo=, hi=, se=, ess=, n=)` with floats and `n` as int.

Part B check: `-k "feasible or wilson or log_weights or is_estimate"` → all pass.

### Part C: Search, mode labelling and policy choice TODOs (TODO 5, 8, 10)

**Files to modify:** `lab11/search.py`, `lab11/modes.py`, `lab11/policy.py`.

1. **TODO 5 `cem_update(Z, Bn, score, prev, rho=0.1, smooth=0.7, min_std=0.25, p_clip=(0.03, 0.97))`**: `score = np.asarray(score, float)`; `n = len(score)`; `m = max(2, math.ceil(rho * n))`; `idx = np.argsort(-score, kind="stable")[:m]`; `E, EB = Z[idx], Bn[idx]`;
   `mean = smooth*E.mean(0) + (1-smooth)*prev["mean"]`; `std = np.maximum(min_std, smooth*E.std(0) + (1-smooth)*prev["std"])` (population std, ddof=0); `p_bin = np.clip(smooth*EB.mean(0) + (1-smooth)*prev["p_bin"], *p_clip)`; `level = float(score[idx].min())`. Return `dict(mean=, std=, p_bin=, level=)` (numpy arrays for the first three).
2. **TODO 8 `label_mode(R, k, event, contact_type)`**: `other = event["b"] if event["a"] == "ego" else event["a"]`. In this order:
   1. `other == "P1"` (block or hit) → `"crosswalk_block"`;
   2. `contact_type(R, k, event) == "rear"` → `"rear_ended"`;
   3. `other == "C2"`: `c = R.F[k, event["f"], core.IX["C2"]]`; if `c[0] < 112 and c[4] < 6` → `"bus_bypass"`, else `"cut_in"`;
   4. otherwise `"other"`.
3. **TODO 10 `choose_policy(table, baseline_progress, max_loss=0.05)`**: keep rows with `row["progress"] >= (1 - max_loss) * baseline_progress`; if none return `None`; else return the row (the original dict) minimising `(row["fail_hi"], -row["progress"])`.

**Acceptance:** `python -m pytest -q tests/test_todos.py` → **31 passed** (whole suite green). Only `lab11/*.py` changed.

## Task 2: Design the custom hazard file `hazard.json` (step G)

**Files to create:** `hazard.json` (repository root). Read `avsim/studio.py` (`FORMAT`, `LIMITS`, `check_space`, `hazard_ok`, `load`), `avsim/core.py` (`DEFAULTS`, `HAZARD_DEFAULTS`, `fired()` near line 386: a hazard fires once `t >= t0` and the ego is within `trig` m behind the hazard's x) and `run_all.py: step_mine` first.

Design (do NOT copy README's bicycle example — no `bike` hazard). Recommended: **lead car L1 brakes hard ("phanh gấp")**, a `brake` hazard triggered when the ego is within N m behind L1:
- File shape exactly as the chapter-10 viewer exports (`viewer/app.js: stuDoc`):
  ```json
  {
    "format": "avsim-studio/1",
    "params": { ...all keys of avsim.core.DEFAULTS with "civil": true (cutGap/pedGap may be null) },
    "policy": {},
    "hazards": [{"type": "brake", "who": "L1", "decel": 6.0, "dur": 2.5, "t0": 4.0, "trig": 40.0}],
    "ego": "planner",
    "space": {"h0_decel": [lo, hi], "h0_dur": [lo, hi], "h0_t0": [lo, hi], "h0_trig": [lo, hi]},
    "exposure": <number in (0, 1]>,
    "why": "<Vietnamese text>"
  }
  ```
- `space` keys must be `h0_<field>` with field in `LIMITS["brake"]` = decel (0.5–9.0), dur (0.1–10), t0 (0–24), trig (0–120), each `[lo, hi]` inside those limits. Suggested starting point: `h0_decel [3.0, 7.0]`, `h0_dur [1.0, 4.0]`, `h0_t0 [2, 18]`, `h0_trig [20, 60]` (trig must exceed the typical ego–L1 gap of ~20–35 m or the brake never fires). Keep the scene itself feasible (`studio.hazard_ok(hazards, params)` True: decel ≤ mu·g = 7.85 at mu 0.8).
- `exposure`: hard-braking-lead-vehicle events per trip, e.g. ~0.05–0.1, justified in `why`.
- `why` (Vietnamese) must justify **both** the frequency and each range with real, citable sources, honestly marked as approximations, e.g.: NHTSA Traffic Safety Facts (rear-end collisions are the most common crash type, ~29–33 % of police-reported crashes); NCHRP Report 400 / Fambro et al. 1997 (surprise emergency braking of drivers averages ≈ 0.55 g, range ~0.3–0.8 g ⇒ ≈ 3–7.5 m/s²); AASHTO Green Book (comfortable deceleration 3.4 m/s²); SHRP2 / 100-Car naturalistic driving studies (hard-braking events ≥ 0.3 g occur on the order of once per tens of km ⇒ the chosen per-trip exposure). State how each number in `space`/`exposure` follows from those sources. Do not invent precise statistics; give ranges and say they are estimates.
- Verify:
  1. `python -c "from avsim import studio; d=studio.load('hazard.json'); print(studio.check_space(d), studio.hazard_ok(d['hazards'], d['params']))"` → `[] True`.
  2. `python run_all.py mine hazard.json` (needs Task 1; runs ~1 500 scenarios; without `evidence/fix.json` it only evaluates the baseline policy — that is fine here). Read the printed `accepted` percentage. If `< 50 %`, narrow `h0_decel` (e.g. upper bound ≤ 4.9 so wet roads with mu 0.5 do not reject it) and rerun until `evidence/mine.json: accepted >= 0.5`. Update `why` if ranges change.

**Acceptance:** `hazard.json` loads with `studio.load`, `check_space` returns `[]`, `hazard_ok` is True, `exposure` in (0, 1], `why` non-empty and cites sources for frequency and every range; `evidence/mine.json: accepted >= 0.5`; TEST_CMD still 31 passed. Do not modify any other file.

## Task 3: Run the full pipeline (no `--quick`) and keep the evidence

**Files to modify:** `.gitignore` (remove only the line `evidence/` so `evidence/*.json` can be committed — it is a required deliverable). Generated: `evidence/*.json`.

1. Precondition: TEST_CMD → 31 passed; `hazard.json` exists (Task 2).
2. Run the full pipeline **without `--quick`**. Total ≈ 9–25 min; `fix` is the longest (3–10 min). To avoid tool timeouts run one step per command, in this order, each with a long timeout (≥ 600 s, or run in background and wait):
   `python run_all.py world`, `python run_all.py replay`, `python run_all.py mc`, `python run_all.py hunt`, `python run_all.py estimate`, `python run_all.py fix`, `python run_all.py mine hazard.json`, `python run_all.py export`
   (equivalent to `python run_all.py all`, which you may use instead if it fits in one run). `mine` must run after `fix` so `mine.json` contains both `baseline` and `chosen`.
3. If any step aborts, fix the responsible TODO in `lab11/` (never `avsim/`/`run_all.py`), re-run TEST_CMD, then re-run that step and every later step.
4. Do NOT add candidates to `policy_candidates.json` based on TEST numbers. (Leaving it unchanged is the expected choice.)
5. Verify: every file `meta.json, world/replay/mc/hunt/estimate/fix/mine/viewer` exists under `evidence/`; `evidence/meta.json` → every step has `"quick": false`; `evidence/mine.json: accepted >= 0.5` and has `baseline` and `chosen` keys; `evidence/fix.json: chosen` is not null (if it is null, report it, do not tune on TEST).
6. Print a short summary of key numbers (town from `_run.town`, replay counts, mc k/n/Wilson, estimate totals, fix chosen policy, mine avoidable counts) to the task summary for the report tasks.

**Acceptance:** all evidence files exist from a non-quick run (`_run.quick == false` in each), `hunt.json` has entries for each mode, TEST_CMD still 31 passed, no changes to `avsim/`, `run_all.py`, `tests/`, `lab11/` (unless a bug fix was needed — then TEST_CMD must still pass).

## Task 4: REPORT.md — town and questions 1–4

**Files to modify:** `REPORT.md` only. Keep line 1 exactly `MSSV: 202602673` and keep the template headings. Write answers in **Vietnamese**. Every number must be copied from `evidence/*.json` and cited as `file.json: field = value` (e.g. `mc.json: k = 4`). Read the needed JSON files with the Read tool (they can be large: read only needed parts or use a small `python -c` to print fields).

- Fill `Thị trấn của bạn (meta.json: _run.town): <town>` from `evidence/meta.json` (or any evidence file's `_run.town`).
- **Câu 1 (replay.json):** number dropped because the human driver crashed, collisions with replayed log vs reactive world, by contact type (a small table). Explain most replay collisions are `rear` and fake: the logged follower (TG) does not brake because the log does not know the planner brakes earlier than the human driver did. Pick one scenario from `replay_only` and narrate it.
- **Câu 2:** list the 4 rules of `is_feasible` with physical basis (lateral grip a_lat ≤ μg for a 3.5 m cosine lane change, braking ≤ μg tyre friction, pedestrian speed ≤ 2.5 m/s = running not walking, cut-in speed in (0, 22] m/s). From `hunt.json` say which hunt produced the most infeasible scenarios (cite the field) and explain CEM is pulled toward infeasible scenarios because they are the "most dangerous"; counting them would inflate the estimate with crashes that cannot happen.
- **Câu 3 (mc.json):** k, n, p̂, Wilson 95 %. Runs needed for ±20 % relative half-width: n ≈ 1.96²(1−p)/(0.2² p) ≈ 96/p, give the number. List modes with 0 occurrences in the 3 000 runs and report their upper bound (Wilson `hi` or rule of three 3/n), **never 0**.
- **Câu 4 (estimate.json):** table per mode × seed: `p`, `hits`, `ess`, `top_share`. Do the 3 seeds agree with each other and with MC? Where `top_share > 0.5` explain the estimate is essentially one sample's weight. Compare `estimate.json: budget` with MC runs needed (~96/p) for equal precision. `other` comes from `mc.json` (`estimate.json: other.source`) and is not hunted because it is not a single mechanism a proposal could target; note `upper_zero_hits`/bounds if present.

**Acceptance:** line 1 unchanged; town filled; questions 1–4 answered in Vietnamese; every number cites file and field and matches `evidence/` exactly (spot-check each with the JSON). TEST_CMD still 31 passed.

## Task 5: REPORT.md — questions 5–9 (with screenshot placeholders)

**Files to modify:** `REPORT.md` only (sections 5–9). Vietnamese; every number cited as `file.json: field = value` and matching `evidence/`. Do not create images; where a 3D screenshot is needed insert a placeholder line like `> TODO-ảnh: evidence/shots/q5-bus_bypass.png — mở viewer chương 9, chọn run <id>, kéo đến t = <s> s (lúc va chạm) rồi chụp.` naming the run id from `evidence/viewer.json` and the time.

- **Câu 5:** for each mode found (`hunt.json: found`), a screenshot placeholder plus one sentence of the event chain (who saw what, who waited for what: e.g. bus_bypass — C2 squeezes past the stopped bus, ego's view blocked; crosswalk_block — pedestrian steps out from behind the bus, ego stops with its body on the crosswalk; cut_in; rear_ended — TG tailgates and hits the braking ego). Which modes the hunt did not find and confidence. Explain why one hunt per mode is needed (a single general hunt collapses onto the easiest mode; cite `hunt.json: found` per hunt).
- **Câu 6 (fix.json):** chosen policy; TRAIN `fail`, `fail_hi`; TEST `fail_hi_seen`, `fail_hi` for chosen and for baseline (`chosen_test`, `baseline_test`); why the fix's `fail_hi` can exceed the baseline's `fail` though it is truly better (upper bound vs point estimate, unseen modes add bounds); progress and harsh-brake counts. Screenshot placeholder with the ghost (baseline) for a scenario the fix avoids. Name a knob that "sounds safer" but increases failures per `fix.json: table` (e.g. a lower `cautionV` or larger `T`) and explain via the reactive world (the tailgater TG rear-ends a car that slows/brakes harder; merging cars take the larger gap).
- **Câu 7:** three natural-distribution assumptions from `avsim/space.py` (e.g. cut-in accepted gap `cutGap` median, `pedLook` probability of peeking past the bus, `mu` wet-road share), each with a concrete real data source (e.g. NGSIM / highD lane-change datasets, pedestrian observation studies at bus stops, local weather/road-friction records) and the direction the estimate is biased if wrong.
- **Câu 8:** three limits specific to this world (one street, IDM-only drivers, perfect sensors except bus occlusion, brake-only oracle, fixed 24 s horizon…) and what you would measure next with one more week.
- **Câu 9 (mine.json + hazard.json):** the hazard and its real-world analogue; why the planner may handle it badly; `space`/`exposure` sources (quote `hazard.json: why`); a table for `baseline` and `chosen`: `avoidable`, `unavoidable`, `unattributed`, `low_speed`, `avoidable_foreknown`, `p` with Wilson `[lo, hi]`, `per_trip`/`per_trip_hi`; `mine.json: accepted`; compare per-trip rate with the bus-bypass rate (from `estimate.json` per-mode `p` × 1 trip). Distinguish avoidable / unavoidable / unattributed and explain "tránh được" is an optimistic standard in both directions (the oracle knows the hazard the instant it is visible and brakes at full μg ⇒ counts more avoidable than a real planner could; brake-only, no steering ⇒ counts fewer). Screenshot placeholder (chapter 9/10). "Bạn tự lái thì sao": placeholder `TODO-tự-lái` for the user. Propose a concrete fix: a knob from `POLICY_LIMITS` (e.g. larger `T`/`b`) or, if none helps, new logic (e.g. track the leader's deceleration / brake lights and pre-brake; larger gap when following closely with a tailgater).

**Acceptance:** all 9 sections answered in Vietnamese, line 1 still `MSSV: 202602673`, every cited number matches `evidence/`, every needed image is a `TODO-ảnh` placeholder pointing at `evidence/shots/`. TEST_CMD still 31 passed. No other file changed.
