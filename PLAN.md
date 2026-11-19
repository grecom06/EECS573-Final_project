# EECS 573 Final Project Plan: Simulating Rowhammer and Evaluating DRAM Defenses in Ramulator2

## 1. Goal

Use Ramulator2 (this repo) to model Rowhammer access patterns and evaluate how the DRAM-level mitigations already implemented here (PARA, Graphene, BlockHammer, Hydra, TWiCe, TRR variants, PRAC/RFM) trade off attack prevention against performance and command overhead. The deliverable is a comparative study, not a hardware exploit.

## 2. Scope and honest framing

Ramulator2 is a cycle-level DRAM timing and command simulator. It does NOT model the analog charge leakage that produces a physical bit flip. So in simulation:

- "Hammering" = an access pattern that repeatedly activates one or more aggressor rows in a single bank.
- "Attack success" = a victim row accumulates more aggressor activations (ACT commands) within one refresh window (tREFW = 64 ms) than a configurable Rowhammer threshold N_RH (e.g. 50k for recent DRAM, 100k+ for older DDR3). Exceeding N_RH is our proxy for "a flip would occur."
- A mitigation "works" if, under the same attack, no victim row's aggressor ACT count reaches N_RH.

The physical flip itself (and a Rust victim demo) belongs to a hardware track and is explicitly out of scope here. This project is the simulation and defense-evaluation half.

## 3. Research questions

1. For a fixed double-sided hammer pattern, what is the maximum per-row aggressor ACT count per tREFW with no mitigation, and how does it scale with hammer intensity?
2. Which of the implemented mitigations keep every victim below N_RH, and down to what threshold do they hold?
3. What performance and command overhead does each mitigation impose (added cycles, average read latency, extra refresh/RFM/preventive-refresh commands, bandwidth loss) under (a) a pure attack and (b) an attack co-running with a benign workload?
4. How sensitive is each mitigation to its own parameters (PARA probability p, BlockHammer blacklist threshold, TWiCe table size, PRAC/RFM backoff)?

## 4. Background and related work

- Rowhammer: Kim et al., "Flipping Bits in Memory Without Accessing Them," ISCA 2014.
- Simulator: Luo et al., Ramulator 2.0 (IEEE CAL 2023) and Ramulator 2.1.
- Defenses (as mapped to this repo in Section 10): PARA (Kim 2014), TRR / TRRespass (Frigo 2020), Graphene (Park, MICRO 2020), BlockHammer (Yaglikci, HPCA 2021), TWiCe (Lee, ISCA 2019), Hydra (Qureshi, ISCA 2022), PRAC/RFM (JEDEC DDR5).
- Reliability framing (background papers in the project folder): "PinDrop: Breaking the Silence on SDCs in a Large-Scale Fleet" (HPCA 2026) and "Understanding Silent Data Corruption in LLM Training" (ACL 2025). Rowhammer is one source of the silent corruption these papers characterize, which motivates evaluating defenses.

## 5. Methodology

### 5.1 Baseline platform
- DRAM: DDR4 (org DDR4_8Gb_x8, timing DDR4_2400R) as the primary target; repeat key runs on DDR5 for a modern comparison.
- Controller: GenericDDR with FRFCFS scheduler, Open row policy, RoBaRaCoCh address mapper (so row bits are the high bits and aggressor/victim rows are easy to place in one bank).

### 5.2 Generating the attack
- Drive the hammer with a trace frontend (ReadWriteTrace or LoadStoreTrace). Generate a trace (small Python script in a new `attack/` folder) that alternately accesses addresses mapping to two aggressor rows R-1 and R+1 that bracket victim row R in the same bank, under the configured address mapper. This is double-sided hammering.
- Parameterize: number of hammer iterations, single vs double-sided, and an optional interleaved benign stream.

### 5.3 Measuring the attack
- Attach a per-row ACT counter. Start from the `CommandCounter` plugin (`python/ramulator/controller_plugin/command_counter.py`, `src/.../impl/command_counter.cpp`); if needed, add a small custom plugin that bins ACT commands by (bank, row) and reports the max count per tREFW window.
- Primary signal: max per-row aggressor ACT count per tREFW vs N_RH.

### 5.4 Defenses to evaluate (all already in the repo)
PARA, Graphene, BlockHammer, Hydra, RRS, TWiCe, ideal/Samsung/Hynix TRR, and PRAC + RFM (DDR5). Oracle_RH as an upper-bound reference.

### 5.5 Metrics
- Attack success proxy: does any victim reach N_RH? Max victim ACT count.
- Mitigation activity: preventive refreshes / RFMs / swaps injected.
- Performance overhead: controller cycles, avg read latency, row hits/misses/conflicts, sustained bandwidth, vs the no-mitigation and no-attack baselines.

### 5.6 Sweeps
Hammer intensity (ACTs per tREFW), N_RH threshold, tREFW, and each mitigation's own key parameter.

## 6. Experiments and figures
1. ACT-count vs hammer-intensity curve, no mitigation (shows the attack is modeled).
2. Max victim ACT count under each mitigation at a fixed strong attack (shows which defenses hold).
3. Threshold-sweep: lowest N_RH each mitigation still protects.
4. Overhead bar chart: latency and injected-command overhead per mitigation, attack-only and attack-plus-benign.

## 7. Milestones (target: end of term)
- Week 1: build Ramulator2 (container or local), reproduce `examples/example_config.py`, confirm per-row ACT counting works on a trivial trace.
- Week 2: implement the double-sided hammer trace generator and the no-mitigation baseline (RQ1).
- Week 3: run all mitigations, collect attack-prevention and overhead data (RQ2, RQ3).
- Week 4: parameter sweeps (RQ4), plots, writeup, slides.

## 8. Division of labor
TBD among team (grecom06 repo). Suggested split: (a) attack/trace generator + ACT-counter plugin, (b) mitigation runs + data collection, (c) sweeps + plots + writeup.

## 9. Risks and fallbacks
- Address mapping: confirm empirically which physical addresses land as adjacent rows in one bank under RoBaRaCoCh before trusting the "bracketing" assumption. Fallback: dump the mapper output and pick addresses from it.
- Custom plugin effort: if a per-row counter plugin is too much, approximate with CommandCounter on ACT plus offline binning of a `CmdTraceRecorder` trace.
- Scope creep: physical flips and any Rust victim are out of scope; keep to the ACT-threshold proxy.

## 10. Repo pointers
- Config entry points: `examples/example_config.py`, `python/ramulator/__init__.py`.
- Controller: `src/ramulator/controller/impl/generic_ddr_controller.cpp`.
- Mitigation plugins: `src/ramulator/controller/plugin/impl/` (para, graphene, hydra, rrs, twice, ideal_trr, samsung_trr, hynix_trr, oracle_rh, rfm_manager) and `src/ramulator/controller/impl/blockhammer_controller.cpp`, `prac_controller.cpp`.
- Counting/tracing: `command_counter.cpp`, `cmd_trace_recorder.cpp`, `bin_trace_recorder.cpp`.
- Address mapper: `src/ramulator/controller/addr_mapper/impl/`.
- New work lives in a new `attack/` folder (trace generator, run scripts, plot scripts).

## 11. References
See Section 4. Canonical Rowhammer and each defense paper, the two SDC papers in the project folder, and the Ramulator 2.0/2.1 citations in the repo README.
