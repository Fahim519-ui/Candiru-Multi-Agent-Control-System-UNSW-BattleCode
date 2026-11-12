# Candiru

## Executive Summary & Value Proposition

Candiru is a **deterministic, low-latency multi-agent control research system** for the UNSW Battlecode environment. It converts a local 7x7 partially observed grid into survival, resource-acquisition, and adversarial-execution decisions under a **~25 ms per-agent turn budget**—a useful analogue for constrained online decision systems where a locally optimal action can create delayed inventory, liquidity, or path-dependence risk.

The current `hammerhead_port` policy combines **shortest-path resource routing**, **bounded reachability/liveness checks**, and **selective adverse-selection-aware head-to-head trades**. Its persisted tournament ledger records a **46/63 (73%)** aggregate win rate for the current variant, including **20/36 (55.6%)** against its immediate predecessor.

## System Architecture & Design

Each dragon runs in an isolated Python process and receives only a live 7x7 observation. There is no shared memory: the policy constructs a per-turn local state view, scores feasible actions, commits exactly one action, and optionally exchanges compact tagged sonar messages.

```text
                 UNSW Battlecode engine
                           |
                           v
             Live 7x7 observation / Controller API
                           |
                           v
      +--------------------------------------------+
      | Per-dragon decision process                |
      |                                            |
      |  View index -> path / region search        |
      |       |                  |                 |
      |       v                  v                 |
      | pearl target        safety & liveness      |
      | enemy / team        portal handling        |
      +--------------------+-----------------------+
                           |
                           v
            Tiered candidate filters + score
                           |
                           v
      make_move | make_moves | do_split | send_sonar
```

| Component | Responsibility |
| --- | --- |
| `hammerhead_port/` | Primary policy: portal-aware BFS, bounded flood-fill safety gate, population control, and favorable head-to-head selection. |
| `hammerhead/`, `hammerhead_kami/`, `tiger_shark/`, `escape_only/` | Controlled strategy variants used to isolate the value of routing, liveness, and collision-policy changes. |
| `mako_shark/` | Role-based policy: protected mothership, expendable sprint-capable bombers, persistent map/body memory, and tagged sonar coordination. |
| `reef_shark/` | ID-stable aggressive/conservative policy mix with sticky targets and local action scoring. |
| `bot1`–`bot5`, `mybot/` | Baselines ranging from random walk through greedy collection, splitting, and untagged sonar coordination. |
| `*/helper.py` | Identical vendored Battlecode SDK in every bot package; owns the controller protocol and game types. |
| `.claude/skills/tournament/` | Serial/head-to-head and round-robin runners, replay-log parser, result ledger, and matchup matrix. |
| `test_portal_seek.py` | Deterministic regression harness for the portal-escape failure mode. |

The architecture deliberately favors **process isolation and bounded local work** over cross-agent mutable state. `mako_shark` is the exception at the strategy level: it reconstructs a persistent view of static blocked edges and its own trail within a process, while sonar communicates only tagged position/sighting payloads.

## Quantitative & Algorithmic Highlights

- **Shortest-path resource acquisition:** `hammerhead_port` runs BFS through visible empty, non-portal cells and executes the first edge of the shortest path to a pearl. On a window of \(V \leq 49\) cells and \(E \leq 4V\) directed edges, this is **\(O(V + E) = O(1)\)** per turn with respect to map size.
- **Liveness-constrained action selection:** each candidate move is subjected to a capped flood fill. A destination must expose at least `min(length + 1, 12)` reachable cells; when every option fails, the policy takes the roomiest feasible pocket. This converts delayed self-trapping into an explicit local constraint with **bounded \(O(V + E)\)** work.
- **Portal escape detection:** portal edges are treated as exits rather than as the adjacent geometric cell (which is not the teleport destination). The policy triggers portal-seeking only when the agent is locally boxed in or has a long per-ID-jittered starvation signal, then rate-limits re-entry with a 25-turn cooldown.
- **Adversarial trade selection:** `hammerhead_kami` and `hammerhead_port` count visible enemy segments by dragon ID and intentionally take an adjacent head-on only when the observed enemy length exceeds their own. This is a conservative, lower-bound estimate of trade value under partial observability.
- **Population control:** the base family splits a two-segment child whenever legal before round 400, targeting the engine’s 64-unit cap; the split cutoff avoids late-game self-congestion. `mako_shark` instead uses threat-conditioned cadence, a rising retained-length floor, and periodic backup-mothership creation.
- **Structured coordination:** `mako_shark` encodes sonar as `(kind << 24) | (x << 12) | y`, validates message tags before use, and expires observations using TTLs. Bombers use bounded BFS to sprint an enemy head within `length - 1` steps, then screen the mothership rather than clustering in its forward lane.
- **Multi-objective action scoring:** `mako_shark` applies successive hard filters—portal avoidance, enemy/friendly-head clearance, region capacity, onward exits—before maximizing a weighted score for pearl distance, enemy distance, local crowding, free exits, and reachable region size.

## Performance & Benchmarks

**Reproducible functional regression:** `python test_portal_seek.py` constructs a sealed, pearl-less pocket whose sole exit is a portal. In the checked run, `hammerhead_port` escaped in **median 39 turns** and **worst-case 47 turns** over 20 seeds; `hammerhead_kami` failed to escape within the test’s **200-turn** cap on every seed.

**Persisted matchup ledger:** `python .claude/skills/tournament/results.py matrix` reports **225** completed games in the repository’s `results.csv`. For the current `hammerhead_port` row, the ledger records **46/63 wins (73%)**, including **20/36 (55.6%)** against `hammerhead_kami` and **9/9** against each of `bot4`, `reef_shark`, and `bot5`. These results are directional rather than a claim of statistical significance—the repository explicitly treats small per-map cells as noisy and stores source hashes with each recorded game.

| Measurement | Evidence | Result |
| --- | --- | --- |
| Decision-work bound | 7x7 observation; capped flood fill at 12 cells | **Constant-bounded per turn** |
| Relative engine work | `hammerhead` design notes | **~312k points/dragon-round** vs. ~331k for the earlier `v_meta` policy |
| Portal test | `test_portal_seek.py`, 20 deterministic seeds | **39-turn median**, **47-turn worst case** |
| Current policy vs. predecessor | `results.csv` matrix | **20/36 wins (55.6%)** |

Run games **serially** when benchmarking. One match can spawn up to 128 bot processes; concurrent matches can exhaust the host and cause turn-budget failures that invalidate results. Use both colors and multiple seeds before treating a matchup difference as signal.

## Engineering Rigor & Tech Stack

- **Python 3.11+**, standard-library-only policy code (`collections.deque`, `random`, typing syntax); `pyproject.toml` declares **no third-party Python dependencies**.
- **UNSW Battlecode CLI 0.3.6** is the external build/run dependency. The SDK is vendored as the same checksum-identical `helper.py` in each independently buildable bot directory.
- **Type-oriented interfaces:** policy code annotates controller/game globals and core strategy helpers; `mako_shark` uses `__slots__` for its hot-path `Step` state to avoid per-instance dictionaries.
- **Determinism controls:** stochastic policies seed from `BOT_SEED`; tournament scripts persist map, seed, side, winner, round count, resolution type, and source hash in `results.csv`.
- **Defensive simulation tooling:** runners reject incomplete logs and known build-cache races; `test_portal_seek.py` isolates a previously observed portal pathology with assertions; the autopsy utility renders fatal-turn board state for post-mortem analysis.
- **Trade-off:** the repository prioritizes an **\(O(1)\), bounded local decision path** over global optimality. No CI workflow, static type checker, profiler, or multithreaded execution is currently committed; independent dragons are deliberately process-isolated by the game engine.

## Quickstart & Reproducibility

**Prerequisites:** Python **3.11+** and the [UNSW Battlecode CLI](https://game.battlecode.au/docs/). The repository has no package dependencies, so setup is intentionally minimal.

```bash
git clone https://github.com/Fahim519-ui/candiru.git
cd candiru

# Verify the external simulator and Python version.
unswbc --version
python --version

# Deterministic portal-escape regression (20 seeds).
python test_portal_seek.py

# Compile every Python module.
python -m compileall -q .
```

Run a reproducible single match and inspect the ledger:

```bash
mkdir -p replays_local
BOT_SEED=1 unswbc run maps/default.map hammerhead_port hammerhead_kami \
  -o replays_local/default_port_vs_kami.replay \
  > replays_local/default_port_vs_kami.log 2>&1

python .claude/skills/tournament/results.py matrix
```

For a properly paired comparison across all bundled maps, colors, and seeds, run one game at a time:

```bash
SEEDS="1 2 3" OUT=replays_h2h \
  bash .claude/skills/tournament/head_to_head.sh hammerhead_port hammerhead_kami
python .claude/skills/tournament/results.py record replays_h2h
python .claude/skills/tournament/results.py matrix
```

`BOT_SEED` controls randomized policies; with a fixed seed and game state, `hammerhead_port` is reproducible. Generated replay/log directories are local artifacts, while `results.csv` is the durable, hash-attributed experiment ledger.
