# FRL-PSII System Preparation and NAMD MD Tutorial

> **Purpose:** Document the reproducible workflow for preparing an FRL-PSII membrane system, from the initial PDB structure through CHARMM setup, lipid addition, solvation, minimization, equilibration, and production MD.
>
> **Status:** Working protocol. Commands and parameters below reflect the current workflow; items marked **Verify** should be checked against the actual topology, CHARMM/NAMD version, and system geometry before production runs.


### Important principles

- Keep a copy of every validated intermediate structure; do not overwrite the only known-good version.
- Keep protein coordinates from the **FRL structure**. The WT structure is used as a source of membrane lipids, not as a replacement for the FRL protein.
- PDB atom serial numbers, residue IDs, chain IDs, segment IDs, residue names, and atom names are different fields. Check them separately.
- Do not proceed to MD if any atom still has placeholder coordinates such as `9999.000 9999.000 9999.000`, or if the structure contains visibly abnormal lipid geometry.
- Preserve CHARMM and NAMD logs for every stage.

## 2. Prepare the initial structure

### 2.1 Copy the reference setup structure

The existing WT setup is a reference for the workflow and membrane environment:

```bash
cp /NAMD2/ps2_membrane_wt.pdb .
```

**Important:** This is the WT reference structure. Use the actual FRL-PSII structure for the FRL protein and cofactors.

### 2.2 Split chains

Run the existing chain-splitting script:

```bash
bash 2.Split_chains.sh
```

Expected components:

- Individual protein chains
- Cofactors
- Crystal waters

**Check before continuing:**

- All expected FRL protein chains are present.
- Cofactors and crystal waters have been retained.
- Residue and atom names have not been unintentionally changed.
- Residue numbering does not create ambiguities when chain IDs are later removed or ignored by CHARMM.

## 3. Prepare crystal lipids

### 3.1 Create the coordinate directory and copy input files

```bash
mkdir -p coordinates_psii_lipids
```

Copy the following files into `coordinates_psii_lipids/`:

```text
DGD.pdb
LHG.pdb
LMG.pdb
SQD.pdb
```

Check the source paths and filenames before running the following commands.

### 3.2 Rename lipid residue/segment identifiers

The original substitutions were:

```bash
sed -i 's/  DGD /  DGDL/g' DGD.pdb
sed -i 's/  LHG /  LHGL/g' LHG.pdb
sed -i 's/  LMG /  LMGL/g' LMG.pdb
sed -i 's/LSQD/SQDL/g' SQD.pdb
```

Expected identifiers:

| Coordinate file | Intended identifier |
|---|---|
| `DGD.pdb` | `DGDL` |
| `LHG.pdb` | `LHGL` |
| `LMG.pdb` | `LMGL` |
| `SQD.pdb` | `SQDL` |

**Verify the PDB columns before relying on these substitutions.** PDB residue names and CHARMM segment IDs occupy different fields. A global text substitution can alter the wrong field or fail to update the intended field, especially for `SQD.pdb`. Confirm the resulting residue name, segment ID, atom names, and residue numbering using a PDB-aware tool or a fixed-column inspection script. The names must match the actual CHARMM RTF/topology definitions.

## 4. Update OEC S2 parameters

Use the newer OEC parameter files as the starting point:

```text
/toppar_extra2026/oec.prm
/toppar_extra2026/oec.rtf
```

Create the S2-specific files:

```text
oec_s2.prm
oec_s2.rtf
```

### 4.1 OEC water naming

The intended OEC water order is:

| Residue name | Intended chemical state |
|---|---|
| `WT1` | H2O |
| `WT2` | OH |
| `WT3` | H2O |
| `WT4` | H2O |

The current workflow requires swapping the WT1 and WT2 water definitions relative to the source files. When making this change, check all relevant topology and parameter entries, including:

- Residue definitions and atom names in the RTF
- Atom charges and total residue charge
- Bond, angle, dihedral, and nonbonded parameters in the PRM
- Coordinate atom names and residue names
- Consistency with the intended OEC S2 protonation/coordination state

**Do not perform a blind text replacement.** Ensure the molecular identity, atom naming, charge, and parameter assignments remain internally consistent. Confirm that the final OEC model matches the intended S2 state.

## 5. Build the initial CHARMM structure

### 5.1 Combine protein, cofactors, and crystal waters

Combine the FRL protein chains, cofactors, and crystal waters into one PDB file for the initial CHARMM build.

Run:

```bash
charmm < charmm_gen_01.sh > charmm_gen_01.log
```

Inspect `charmm_gen_01.log` for:

- Missing residues or atoms
- Unrecognized residue or atom names
- Missing topology or parameter entries
- Failed IC construction
- Warnings or errors during coordinate generation

### 5.2 Add crystal lipids and build missing atoms

Add the crystal lipids to the structure:

- `DGDL`
- `LHGL`
- `LMGL`
- `SQDL`

The relevant CHARMM operations in the current workflow are:

```text
IC GENERATE
IC FILL PRESERVE
IC BUILD
HBUILD
```

Use the ordering appropriate for the existing CHARMM script and topology. These commands generate/use internal coordinates and build missing coordinates where the required IC information is available; `HBUILD` builds hydrogen coordinates.

If required by the topology and current setup, generate angles and dihedrals:

```text
AUTOGENERATE ANGLES DIHEDRALS
```

**Verify:** `AUTOGENERATE` must be appropriate for the topology and must not duplicate or conflict with existing terms. Check the CHARMM log and energy evaluation.

Write the coordinates for inspection:

```text
OPEN WRITE CARD UNIT 10 NAME control.pdb
WRITE COOR PDB UNIT 10
```

### 5.3 Minimize crystal lipids

Add the necessary missing-atom construction and minimization instructions to `charmm_min_02.sh`. The goal is to complete missing lipid atoms and optimize the lipids into reasonable local geometries.

After minimization:

1. Inspect the CHARMM log.
2. Search for placeholder coordinates.
3. Visualize all added lipids in VMD or ChimeraX.
4. Check bond geometry and severe overlaps.
5. Repair and re-minimize any lipid that remains abnormal.

## 6. Add membrane lipids

The WT membrane is used as a source of membrane-lipid coordinates. Keep the FRL protein and cofactors from the FRL structure.

### 6.1 Align WT to FRL

In VMD:

1. Load the FRL structure as the reference.
2. Select corresponding, structurally conserved protein backbone atoms in WT and FRL.
3. Fit the WT structure to the FRL coordinate frame.
4. Save the aligned WT structure.
5. Remove the WT protein and unwanted components, retaining only the membrane lipids.

Use equivalent atom selections with matching atom counts for the alignment. Avoid fitting on the lipids themselves, since their thermal motions can distort the alignment.

### 6.2 Merge membrane lipids with FRL-PSII

Combine the extracted membrane lipids with the FRL protein, cofactors, crystal waters (if retained at this stage), and crystal lipids.

Inspect:

- Membrane orientation and coverage
- Lipid–protein and lipid–lipid overlaps
- Lipid residue names and numbering
- Duplicate atom serials
- Whether the membrane and FRL structure share the same coordinate frame

### 6.3 Renumber PDB atom serial numbers

After merging files, some atom serial numbers may exceed the width of the traditional PDB serial field. For example:

```text
ATOM  *****  O5E DGD    26
```

Use `pdb_reatom` or another PDB-aware tool to renumber atom serials.

**Important:** Atom serial numbers are not residue IDs. Renumbering atom serials does not fix residue-number collisions or incorrect residue names. After renumbering, verify the PDB fixed-column layout and confirm that atom names, residue names, residue IDs, coordinates, and record types are preserved.

## 7. Validate the complete structure

Before solvation, validate the combined protein–cofactor–crystal-lipid–membrane-lipid structure.

### 7.1 Check names, numbering, and coordinates

Confirm:

- FRL protein chains and cofactors are complete.
- OEC uses the intended S2 topology and parameters.
- `WT1`–`WT4` correspond to the intended chemical states.
- Crystal and membrane lipids match the CHARMM topology.
- Atom serial numbers and residue IDs are valid and unambiguous.
- No placeholder coordinates remain.
- Lipid geometry is reasonable.
- No severe steric clashes are present.

### 7.2 Apply required patches

The current workflow includes this patch command:

```text
PATCH CASN CCHN 35 CHLC 511 SETUP
```

**Verify before applying:** Confirm the patch definition and the identities of `CASN`, `CCHN`, `CHLC`, residues 35 and 511 against the actual topology and structure. Do not copy this patch into a different structure without checking that the relevant residues and patch syntax match.

After applying the patch, inspect the CHARMM log and verify that all required topology and parameter terms are available.

## 8. Add the water box

Two options are available.

### Option A: Reuse an existing water box

Reuse a previously prepared water box only if its dimensions, orientation, and coordinate frame match the current system.

Check that:

- The solvent covers both sides of the membrane.
- Water molecules do not overlap with protein, cofactors, or lipids.
- Water and ion residue/atom names match the topology.
- The final periodic box is compatible with the water-box coordinates.

### Option B: Generate a new water box using VMD

Use the existing script:

```text
model_psii_with_membrane/03-solvate.tcl
```

The FRL-PSII structure was oriented using OPM, and the structure center is intended to be at `(0, 0, 0)`. Check the script's translations, box boundaries, and the final coordinate ranges rather than assuming the solvent box is automatically centered correctly.

After solvation, inspect the water distribution, membrane coverage, and any overlaps. Add ions if required by the intended simulation setup, using a consistent and documented protocol.

## 9. Configure the periodic cell

The current `min0.conf` cell parameters are:

```tcl
set a        176.000
set b        176.000
set c        160.000
set alpha     90.0
set beta      90.0
set gamma     90.0
set zcen     -10.0

cellBasisVector1     $a   0.0   0.0
cellBasisVector2    0.0    $b   0.0
cellBasisVector3    0.0   0.0    $c
cellOrigin          0.0   0.0 $zcen
```

| Parameter | Current value |
|---|---:|
| `a` | 176 Å |
| `b` | 176 Å |
| `c` | 160 Å |
| `alpha`, `beta`, `gamma` | 90° |
| `cellOrigin` z coordinate | −10 Å |

These values document the current setup; they are not universal dimensions for all FRL-PSII systems.

**Verify:** The cell dimensions and origin must be consistent with the final solvated coordinates and membrane orientation. `cellOrigin` is the periodic cell origin, not automatically the geometric center of the system. Check that the coordinates fit the intended periodic cell and that sufficient solvent remains around the system.

## 10. Troubleshooting

### 11.1 Placeholder coordinates: `9999.000 9999.000 9999.000`

If these coordinates remain after CHARMM building or minimization, do not proceed to MD.

Investigate:

1. Missing IC definitions for the atom.
2. Atom names that do not match the RTF or PDB.
3. Incorrect residue names.
4. Residue-number collisions after chain IDs are dropped or ignored.
5. Missing topology or parameter definitions.

Example collisions encountered previously include residue labels such as `E101`, `J101`, and `I101`. If the chain identifier is lost during processing, residues from different chains can become ambiguous.

A possible repair in a particular case was to rename one `J101` to `J102` and one `I101` to `I103`. These are examples, not universal replacements. Identify the correct residue first and make consistent changes in all relevant files.

### 11.2 Lipid geometry remains abnormal

1. Identify the affected lipid in VMD or ChimeraX.
2. Determine which atoms have incorrect positions or geometry.
3. Extract correct coordinates from a validated structure, if available.
4. Update the relevant coordinates in `coordinates_psii_lipids/` using a PDB-aware method or a carefully checked script.
5. Re-run CHARMM building/minimization.
6. Visualize and validate again.

Repeat until there are no placeholder coordinates and the lipid geometry is reasonable. Avoid using broad `sed` replacements on fixed-column PDB files without checking the resulting columns.

### 11.3 PDB atom serial field overflows

If an atom serial is displayed as `*****`, renumber atom serials using `pdb_reatom` or another PDB-aware utility. Then validate the fixed-column formatting and confirm that atom/residue identities have not changed.

### 11.4 CHARMM reports missing atoms or parameters

Check, in order:

- Residue and atom names in the PDB
- RTF/topology residue definitions
- PRM parameter coverage
- IC definitions for missing heavy atoms
- Patch application and residue numbering
- The first error in the CHARMM log (later errors may be consequences)

### 11.5 Unexpected instability during equilibration

Check:

- Initial steric overlaps
- Lipid geometry and membrane continuity
- Solvation and periodic cell dimensions
- OEC atom names, charges, protonation state, and coordination
- Temperature and pressure coupling settings
- Restraint definitions and units
- Whether the structure passed all validation steps before MD

## 11. Pre-production checklist

- [ ] All FRL protein chains, cofactors, and crystal waters are present.
- [ ] OEC S2 RTF/PRM files and WT1–WT4 definitions are verified.
- [ ] Crystal lipid names match the topology.
- [ ] Membrane lipids are aligned and merged correctly.
- [ ] No duplicate or ambiguous residue IDs remain.
- [ ] PDB atom serial numbers are valid.
- [ ] No `9999.000 9999.000 9999.000` coordinates remain.
- [ ] Lipid geometry and membrane integrity look reasonable.
- [ ] Required patches are confirmed and successfully applied.
- [ ] Water box, ions (if used), and periodic cell are consistent.
- [ ] CHARMM logs show no unresolved errors.
- [ ] NVT/NPT equilibration trajectories have been inspected.
- [ ] NAMD settings have been checked for the installed version.
- [ ] The final structure and configuration files are backed up before production.

## 12. Suggested directory layout

```text
FRL_PSII/
├── 01_initial_structure/
│   ├── 2.Split_chains.sh
│   └── protein_cofactors_waters.pdb
├── 02_lipids/
│   └── coordinates_psii_lipids/
│       ├── DGD.pdb
│       ├── LHG.pdb
│       ├── LMG.pdb
│       └── SQD.pdb
├── 03_oec_parameters/
│   ├── oec_s2.prm
│   └── oec_s2.rtf
├── 04_charmm_build/
│   ├── charmm_gen_01.sh
│   ├── charmm_min_02.sh
│   └── charmm_gen_01.log
├── 05_membrane/
│   └── aligned_WT_membrane.pdb
├── 06_solvation/
│   └── 03-solvate.tcl
├── 07_namd/
│   ├── mini0.conf
│   ├── nvt1.conf
│   ├── npt2.conf
│   ├── npt3.conf
│   └── npt4.conf
└── 08_validation/
    └── structure_check_notes.md
```

This is a suggested layout. Adapt it to the actual project directories, but keep validated intermediate structures separate from files still under repair.

## 13. Run record

Record the following for each system build:

| Field | Value |
|---|---|
| System / structure ID | |
| FRL structure source and version | |
| CHARMM version | |
| NAMD version | |
| OEC parameter version | |
| Lipid parameter version | |
| Water/ion model | |
| Periodic cell dimensions | |
| Minimization date and log | |
| Equilibration date and logs | |
| Production trajectory path | |
| Known issues / deviations | |


## 14. MD simulation: minimization, equilibration, and production

The planned sequence is:

```text
mini0.conf -> nvt1.conf -> npt2.conf -> npt3.conf -> npt4.conf
```

### 10.1 Stage summary

| Stage | Purpose | Timestep | Steps | Duration | Positional restraint |
|---|---|---:|---:|---:|---:|
| `mini0.conf` | Minimization + heating | MD heating timestep must be checked | 10,000 minimization steps + 10,000 heating steps | Heating duration depends on its timestep | 10 kcal mol^-1 Å^-2 |
| `nvt1.conf` | Initial NVT equilibration | 1 fs | 100,000 | 0.1 ns | 5 kcal mol^-1 Å^-2 |
| `npt2.conf` | First NPT equilibration | 1 fs | 100,000 | 0.1 ns | 5 kcal mol^-1 Å^-2 |
| `npt3.conf` | Reduced-restraint NPT equilibration | 1 fs | 100,000 | 0.1 ns | 1 kcal mol^-1 Å^-2 |
| `npt4.conf` | Production NPT MD | 1 fs | 40,000,000 | 40 ns | None |

The restraint units above assume a harmonic positional restraint force constant. Check that the actual NAMD configuration uses the intended restraint mechanism and units.

### 10.2 `mini0.conf`: minimization and heating

Current plan:

- Minimize for 10,000 steps.
- Use positional restraints of 10 kcal mol^-1 Å^-2.
- Heat from 0 K to the target temperature in a separate MD heating stage of 10,000 steps.
- Keep GPU-resident mode off.

Minimization steps do not correspond to physical simulation time. Heating is MD and its duration depends on the heating timestep.

### 10.3 `nvt1.conf`: initial equilibration

- Ensemble: NVT
- Timestep: 1 fs
- Steps: 100,000
- Duration: 0.1 ns
- Restraint: 5 kcal mol^-1 Å^-2
- GPU-resident mode: off

Check temperature stability, structural integrity, and obvious clashes or abnormal lipid motions.

### 10.4 `npt2.conf`: first pressure equilibration

- Ensemble: NPT
- Timestep: 1 fs
- Steps: 100,000
- Duration: 0.1 ns
- Restraint: 5 kcal mol^-1 Å^-2
- GPU-resident mode: off

Monitor pressure, cell dimensions, membrane integrity, and solvent distribution.

### 10.5 `npt3.conf`: reduce restraints

- Ensemble: NPT
- Timestep: 1 fs
- Steps: 100,000
- Duration: 0.1 ns
- Restraint: 1 kcal mol^-1 Å^-2
- GPU-resident mode: off

Continue equilibration while allowing the membrane, protein, and lipids to relax.

### 10.6 `npt4.conf`: production MD

- Ensemble: NPT
- Timestep: 1 fs
- Steps: 40,000,000
- Duration: 40 ns
- Positional restraints: none
- GPU-resident mode: on

Before production, inspect the end of `npt3` for membrane integrity, lipid geometry, OEC coordination, energy stability, and any abnormal structural changes.

### 10.7 Membrane-related NPT settings

The current protocol proposes:

```tcl
useFlexibleCell       yes
useConstantArea       yes
```

The intended distinction in the current notes is:

| System | `useFlexibleCell` | `useConstantArea` |
|---|---|---|
| Water box without membrane | `no` | `no` |
| Membrane system | `yes` | `yes` |

**Verify against the documentation for the exact NAMD version and the intended barostat/ensemble.** Confirm that these settings are compatible with each other and with the desired membrane-area and pressure-coupling protocol. Do not assume that this combination is universally appropriate for every membrane simulation.

For a 2 fs timestep, the current notes also mention:

```tcl
useGroupPressure yes
```

Check the NAMD version and pressure-calculation requirements before enabling this option. A 2 fs timestep also requires appropriate constraints for fast motions, especially bonds involving hydrogen. The current 1 fs timestep for short equilibration stages is a conservative starting point.
---

## Notes for future updates

When changing lipid parameters, OEC protonation, residue naming, water-box dimensions, or NAMD settings, update the relevant section and record what changed. Keep the previous validated structure and configuration files so that differences between WT and FRL systems can be traced to deliberate model changes rather than accidental setup inconsistencies.
