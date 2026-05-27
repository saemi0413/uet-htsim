# AGENTS.md

## Project goal

We are studying UEC multipath behavior in ultraethernet/uet-htsim.

Primary goals:
- Understand existing UEC multipath implementations.
- Preserve baseline behavior unless explicitly asked to change it.
- Add instrumentation for multipath experiments.
- Later design and evaluate an RTT-based multipath policy.

## Important files

Read these first:
- htsim/sim/uec_mp.h
- htsim/sim/uec_mp.cpp
- htsim/sim/uec.cpp
- htsim/sim/uecpacket.h
- htsim/sim/uecpacket.cpp
- htsim/sim/datacenter/main_uec.cpp
- htsim/sim/datacenter/fat_tree_switch.*
- htsim/sim/datacenter/fat_tree_topology.*
- htsim/sim/route.*

## Multipath mental model

- uec_mp.cpp does not directly choose physical links.
- It chooses packet entropy/path_id.
- UecSrc calls nextEntropy() and writes the result into packet path_id.
- ACK/NACK/TIMEOUT feedback flows back into processEv().
- Existing algorithms:
  - Oblivious: uniform spraying with XOR permutation.
  - Bitmap: blacklist/cooldown using skip penalties.
  - REPS: whitelist/recycle good paths.
  - Mixed: REPS good-path reuse first, Bitmap fallback.

## Coding rules

- Do not rewrite unrelated simulator code.
- Keep existing baseline algorithms unchanged.
- Add new behavior behind a separate flag, option, or class.
- Prefer small, reviewable patches.
- After code changes, build and run a small scenario.
- Record the exact command used for every experiment.

## Experiment rules

For every experiment, record:
- git branch and commit hash
- command line
- topology file
- traffic matrix
- routing strategy
- path count
- random seed if available
- output log path

Metrics of interest:
- FCT mean / p50 / p95 / p99
- goodput
- retransmissions
- NACK/trim count
- timeout count
- selected path distribution
- feedback per path