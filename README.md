# PW Benchmark - Plane Wave SCF Benchmark

In this repository are plane-wave basis set benchmarks that may be used by software developers working on plane-wave electronic-structure codes. **PW Benchmark** provides compact, reproducible inputs and measured reference results for testing periodic Kohn–Sham density-functional theory (DFT) implementations and their self-consistent field (SCF) procedures.

## Test cases

The suite contains **48 geometries across 16 crystal families**, spanning metals, covalent semiconductors, and ionic insulators. Each family is represented by an ideal primitive cell containing one or two atoms, sampled at three volumes.

| Crystal prototype | Materials |
| --- | --- |
| Face-centered cubic (fcc) | Al, Cu, Ag |
| Body-centered cubic (bcc) | Li, Na |
| Hexagonal close-packed (hcp) | Mg |
| Diamond | C, Si, Ge |
| Zincblende | SiC, BN, AlP, GaAs |
| Rocksalt | NaCl, MgO, LiF |

The geometries were generated with [`ase.build.bulk`](https://docs.ase-lib.org/_modules/ase/build/bulk.html), using chosen prototype lattice parameters; the resulting lattice and positions are fully specified in each `geometry.xyz`. They are **ideal crystal prototypes with chosen test parameters**, rather than experimentally fitted structures or relaxed PBE equilibrium geometries. The three volume ratios are `0.94`, `1.00`, and `1.06`; lattice vectors are scaled by the cube root of the volume ratio, preserving fractional atomic coordinates and cell shape.

The use of volume-dependent energy differences follows the approach of the [Delta Codes DFT comparison project](https://github.com/molmod/DeltaCodesDFT). **The structures themselves were generated independently; this suite is not the Delta dataset or a full Delta-factor equation-of-state benchmark.**

These cases exercise reciprocal-space basis construction, Brillouin-zone sampling, metallic and insulating occupations, SCF convergence, finite-temperature energies, stress, electron counting, and reproducibility under volume changes. Forces are included, although ideal high-symmetry structures provide limited coverage of nonzero-force behavior. The suite is a *fixed-setting regression benchmark*; it does not establish cutoff or k-point convergence, or cover spin polarization, spin–orbit coupling, surfaces, molecules, or strongly correlated materials.

## Directory structure

Each case is named `<material>_<prototype>_<volume>`, where `v094`, `v100`, and `v106` denote `V/V₀ = 0.94`, `1.00`, and `1.06`, respectively.

```text
release/
├── README.md
├── Al_fcc_v094/
│   ├── in.json          # Physical settings, numerical parameters, and metadata
│   ├── out.json         # Measured reference results and compact diagnostics
│   └── geometry.xyz     # Periodic lattice, chemical species, and atomic positions
├── Al_fcc_v100/
│   ├── in.json
│   ├── out.json
│   └── geometry.xyz
├── Al_fcc_v106/
│   ├── in.json
│   ├── out.json
│   └── geometry.xyz
├── Si_diamond_v100/
│   ├── in.json
│   ├── out.json
│   └── geometry.xyz
└── …                    # 48 case directories in total
```

Each **case directory contains exactly three files**. `geometry.xyz` uses the extended XYZ (*extxyz*) format: its comment line specifies the lattice vectors, periodicity, and atomic-property columns. Coordinates and lattice vectors are in ångströms. The `geometry` field in `in.json` resolves relative to the case directory.

## Calculation specification

All reference calculations use **PBE with the projector-augmented-wave (PAW) method**, neutral spin-paired cells, fixed ions, and periodic boundary conditions in all three directions. Spin–orbit coupling and symmetry reductions are disabled. The wavefunction kinetic-energy cutoff is **600 eV**, with a **Γ-centered `4 × 4 × 4` k-point mesh** and **Fermi–Dirac occupations of width `0.05 eV`**. The intended energy-convergence target is `1e-7 eV` per cell, with at most 200 SCF iterations.

`in.json` expresses common physical and numerical settings, rather than VASP input tags. A code adapter should map these settings to its own interface. Identification fields are metadata, not additional solver parameters. Native convergence definitions differ: VASP uses `EDIFF` for free-energy and band-energy changes, whereas GPAW uses `Energy(1e-7, relative=False)` for the variation of extrapolated energy over three iterations, together with its default density and eigenstate criteria. Band counts are selected by the respective engines and recorded in the results.

For example, the following **input excerpt** specifies the common basis and occupation settings:

```json
{
  "geometry": "geometry.xyz",
  "method": "PAW",
  "xc": "PBE",
  "plane_wave_cutoff_eV": 600.0,
  "kpoints": {"mesh": [4, 4, 4], "centering": "gamma"},
  "occupations": {"method": "fermi-dirac", "width_eV": 0.05}
}
```

## `in.json` — input fields

- **`schema_version`** — Integer identifying the input schema revision.
- **`id`** — Unique case identifier, matching the directory name.
- **`geometry`** — Relative path to the periodic extxyz geometry, normally `geometry.xyz`.
- **`family`** — Material/prototype identifier linking the three volume variants.
- **`volume_ratio`** — Cell volume divided by the family's chosen reference volume, `V/V₀`.
- **`method`** — Ionic/core-electron treatment; `PAW` in this release.
- **`xc`** — Exchange–correlation functional; `PBE` in this release.
- **`periodic`** — Three Boolean flags specifying periodicity along the lattice directions.
- **`charge_e`** — Net cell charge in elementary-charge units; positive values denote removed electrons, and all supplied cases are neutral.
- **`spin_polarized`** — Whether separate spin channels are requested; `false` here.
- **`spin_orbit`** — Whether spin–orbit coupling is requested; `false` here.
- **`plane_wave_cutoff_eV`** — Kinetic-energy cutoff for the wavefunction plane-wave basis, in eV.
- **`kpoints.mesh`** — Number of uniform Brillouin-zone sampling points along each reciprocal-lattice direction.
- **`kpoints.centering`** — Mesh centering convention; `gamma` includes Γ.
- **`occupations.method`** — Occupation distribution; `fermi-dirac` in this release.
- **`occupations.width_eV`** — Electronic temperature expressed as an energy, `kBT`, in eV.
- **`convergence.energy_change_eV_per_cell`** — Intended total-cell SCF energy-convergence threshold, in eV; the adapter must document its native stopping definition.
- **`convergence.maximum_scf_iterations`** — Maximum permitted SCF iteration count.
- **`symmetry`** — Whether symmetry reductions are enabled; `false` here.

## `out.json` — reference result fields

`out.json` contains the **measured VASP reference result**, along with the physical quantities and compact diagnostics needed to use the reference. Extensive quantities, including `free_energy_eV`, refer to the entire cell unless their field name explicitly states otherwise.

- **`schema_version`** — Integer identifying the result schema revision.
- **`id`** — Case identifier corresponding to `in.json` and the directory name.
- **`backend`** — Engine that generated the reference; `vasp` in the released case files.
- **`converged`** — Whether the calculation satisfied the engine's SCF convergence criterion.
- **`natoms`** — Number of atoms in the cell.
- **`input_sha256`** — SHA-256 digest of the released `in.json` physical specification; private execution records retain their original input-file hashes.
- **`geometry_sha256`** — SHA-256 digest of the exact `geometry.xyz` file used for the calculation.
- **`free_energy_eV`** — Electronic free energy of the complete cell, in eV, at the specified occupation width; this is not a zero-temperature extrapolated energy.
- **`forces_eV_per_angstrom`** — Cartesian force components as an `natoms × 3` array, in the atom order of `geometry.xyz`.
- **`stress_eV_per_angstrom3`** — Symmetric Cartesian `3 × 3` stress tensor in eV/Å³, using the ASE convention of positive tensile stress.
- **`plane_wave_counts`** — Number of basis G vectors at each k point, ordered consistently with `kpoints_fractional`; these are not FFT-grid sizes.
- **`number_of_bands`** — Number of electronic bands used by the engine.
- **`fermi_energy_eV`** — Fermi level in the engine's energy-zero convention, in eV; values are not generally comparable across different PAW datasets.
- **`number_of_electrons`** — Number of explicitly treated valence electrons in the cell, determined by the ionic datasets and charge.
- **`kpoints_fractional`** — Actual sampled k-point coordinates in the reciprocal-lattice basis.
- **`kpoint_weights`** — Corresponding integration weights, normalized to sum to one.
- **`plane_wave_count_definition`** — Engine-specific interpretation and ordering of the reported basis counts.
- **`density.kind`** — Description of the PAW-dependent charge-density quantity summarized by the engine.
- **`density.grid_shape`** — Three dimensions of the sampled charge-density grid.
- **`density.unit`** — Unit of the density extrema, normally `electron/angstrom^3`.
- **`density.integral_electrons`** — Recorded integral of the charge-density grid over the cell, in electrons.
- **`density.minimum`** — Minimum sampled grid value; PAW charge grids need not be pointwise positive.
- **`density.maximum`** — Maximum sampled grid value.
- **`units.energy`** — Energy unit used by the results.
- **`units.length`** — Length unit used by the geometry and derived quantities.
- **`units.force`** — Force unit used by the results.
- **`units.stress`** — Stress unit used by the results.
- **`units.stress_sign`** — Stress sign convention, explicitly recorded as positive tensile.

**Density is summary-only.** Full density arrays, PAW one-center occupancy records, and reconstructed all-electron densities are not embedded in this release.

## Using the reference results

Use the same geometry, exchange–correlation functional, occupation convention, and numerical settings. Reference results were generated with **VASP 5.4.4**. The finite-temperature electronic free energy is the energy quantity recorded by this release.

**Absolute energies depend on the ionic datasets and their energy-zero conventions.** Matching physical settings alone does not guarantee matching absolute energies across different PAW implementations or valence spaces. For cross-code studies, volume-relative energies within a material family can reduce arbitrary energy offsets, but differences between ionic datasets still matter. Plane-wave counts refer to the explicit k-point ordering supplied in each output.

The public files contain no runtime measurements, SCF iteration counts, performance-comparison tables, or per-case provenance blocks. They retain numerical results and input/geometry checksums. Licensed VASP source code, executables, and potential files are not included.
