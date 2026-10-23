# Molecular Docking and MD Simulations

Structure-based virtual screening and GPU molecular dynamics across four real drug-discovery targets —
antibacterial, antiviral, anticancer, and an Ebola virus matrix protein — run end-to-end on Kaggle (T4 GPU).

## Aim

Given a real protein target with a computationally detected (not blind-guessed) binding pocket, which
candidate small molecule binds best, and does that computationally "best" docked pose actually stay bound
once real atomic motion is allowed to happen?

## Objective

For three single-target systems (an antibacterial target, an antiviral target, and an anticancer target),
dock one real candidate ligand each into its own detected pocket and run production MD to check pose
stability. For a fourth target (Ebola virus VP40 matrix protein), run a real large-scale virtual screen —
1000+ real compounds per class across 3 mechanistic classes — narrowed through drug-likeness filtering, a
known-binder similarity check, and a two-stage docking funnel down to a single final candidate, then run the
same MD stability check on that one winner. Where a real ground-truth co-crystallized pose exists for the
screened ligand itself (one of the four systems), validate the docking protocol directly against it before
trusting the others' relative rankings.

## Data fetch

- **Receptors:** real PDB structures fetched live from RCSB — 6ML0 (antibacterial target), 5T2P (antiviral
  target, HBV core protein), 5EHR (anticancer target, SHP2/PTPN11), 4LDB (Ebola virus VP40 matrix protein
  dimer) — via RCSB's own file-download REST API.
- **Single-ligand candidates:** real PubChem compounds (CID 16829777 for 6ML0; Clemizole, CID 2782, for
  5T2P; SHP099, CID 118238298, for 5EHR — SHP099 is also the real co-crystallized ligand in 5EHR's own
  structure, making it the project's one ground-truth-validated pair), fetched as 3D SDF via PubChem's
  PUG-REST API.
- **The Ebola library:** for each of 3 mechanistic classes (triterpenoid/betulinic-acid derivatives,
  microtubule/cytoskeletal disruptors, SERM-class allosteric-wedge candidates), PubChem's own 2D
  similarity-search endpoint (`fastsimilarity_2d`, Tanimoto ≥70%) was queried live, anchored on that class's
  documented representative compound, targeting 1000+ real compounds per class. 2D properties (SMILES,
  molecular weight, LogP, H-bond counts, rotatable bonds, TPSA) for the whole resulting library were then
  fetched in bulk (chunks of 100 CIDs per request) — real 3D structures were only built later, for the
  compounds that survived the cheap filters below.

## Data describe

4477 real, distinct PubChem compounds were returned across the 3 Ebola-library classes; 3273 of those had
usable bulk 2D property data. Each of the 4 receptor structures is a real X-ray crystal structure (resolution
and refinement statistics available directly from each PDB entry); 6ML0 has no genuine co-crystallized small
molecule (only crystallization buffer components), while 5T2P, 5EHR, and 4LDB each have their own real native
ligand or inhibitor on file.

## Methods / Workflow — what we did

1. **Receptor prep:** strip crystallographic waters/buffer components (and any real native ligand not being
   screened), rebuild missing heavy atoms (PDBFixer), convert to PDBQT with Gasteiger partial charges.
2. **Real binding-site detection (fpocket):** never blind docking — every search box comes from a real,
   computationally detected pocket.
3. **Ligand prep:** PubChem SDF → Meeko protonation/3D embedding → PDBQT.
4. **Ebola library only — cheap filters before any docking:** a Lipinski/Veber drug-likeness filter (RDKit,
   computed directly from each compound's SMILES) narrowed 3273 candidates to 2911; an independent 2D
   fingerprint similarity screen against salicylic acid (a real, documented VP40-pocket binder from published
   crystallographic fragment screening) ranked the ADMET-passing compounds, and the top 15 by similarity were
   carried forward as the real docking shortlist.
5. **Docking (AutoDock Vina):** single-ligand systems docked directly against their detected pocket. The
   Ebola shortlist went through a two-stage funnel — a fast coarse pass on all shortlisted compounds with
   usable 3D structures, then exhaustive, high-settings redocking on only the top 3 coarse hits.
6. **Redocking validation (5EHR + SHP099 only):** superposed the docked pose onto the real crystallographic
   ligand position and computed RMSD — the one pair in this project with a genuine ground truth to check
   against.
7. **GROMACS MD (GPU):** `pdb2gmx` (amber99sb-ildn + TIP3P) for the protein, ACPYPE/GAFF for each ligand,
   merged into one solvated system, energy-minimized, equilibrated (NVT then NPT), then a real 5 ns
   GPU-accelerated production run — once per system, on exactly one final ligand per system (never more than
   one compound carried into MD for the library screen).
8. **Trajectory analysis:** ligand RMSD-vs-time relative to the protein backbone, for every system.

## Results

- **5EHR + SHP099 redocking RMSD: 0.60 Å** — well under the conventional 2 Å success threshold, real
  evidence the full pipeline (pocket detection + docking) recovers a genuinely correct binding mode when a
  known answer exists to check against.
- **Ebola library screen:** 4477 real compounds fetched → 3273 with usable properties → 2911 passed
  drug-likeness filtering → top 15 by similarity to a known VP40 binder → 7 with usable real 3D structures →
  top 3 of those carried to exhaustive redocking → final winner **CID 249659**, exhaustive affinity **-9.10
  kcal/mol**, similarity to the known binder **0.30** (a modest, honestly-reported partial agreement between
  the two independent signals).
- All four systems (6ML0, 5T2P, 5EHR, and the Ebola library's single winner) completed a full, GPU-accelerated
  5 ns production MD run, with GROMACS's own ligand index group (`UNL`) confirmed to match each system's real
  ligand atom count exactly — the precondition for trusting each system's own ligand-RMSD-vs-time trajectory
  as a genuine stability read.

## Biology interpretation of results

A Vina docking score only answers whether a rigid pose geometrically and electrostatically fits a pocket — it
says nothing about whether that complex is physically stable once real thermal motion, water competition, and
sidechain flexibility are allowed. The 0.60 Å redocking RMSD for SHP099 against its own real SHP2/PTPN11
pocket is direct evidence this project's pipeline can recover a true allosteric binding mode when one is
known, which is the strongest basis for trusting the other three systems' rankings even without their own
ground truth. For 6ML0 and 5T2P, neither screened ligand is known to bind its target from independent
literature — a completed, stable MD run is accordingly this project's primary, not secondary, evidence for or
against those pairings being real. For the Ebola VP40 screen, narrowing 4477 real candidates down to one
through drug-likeness, known-binder similarity, and a two-stage docking funnel mirrors real early-stage
drug-discovery triage, where MD — the most expensive, physics-based stage — is reserved for a single
best-evidenced lead, not run indiscriminately across a library.

## Learning through project

The hardest problems in this project clustered at file-format boundaries, not inside any single tool's own
math — a symptom that looks like one kind of problem (a charge calculation failing) can actually be a
format-conversion step silently dropping or rearranging information earlier in the pipeline, and the fix is
to verify the structure itself at each boundary (the real atom list, the real bond graph, rebuilt from a
verified source like SMILES rather than guessed from incomplete coordinates) instead of chasing the
symptom where it appears. The project also surfaced a real, honest data-availability gap: not every
shortlisted compound had a usable real 3D structure available, handled by disclosing and skipping rather
than forcing a fixed count.

## Caveats

Each production MD run is scoped to 5 real simulated nanoseconds to fit a single Kaggle GPU session's
wall-clock budget — this shows short-timescale pose stability, not a converged, rigorously sampled
free-energy landscape. Three of the four target/ligand pairings (6ML0, 5T2P, and the Ebola library's winner)
are exploratory, without independent experimental confirmation of binding; only 5EHR's SHP099 pairing has a
real, independent ground truth to validate against.
