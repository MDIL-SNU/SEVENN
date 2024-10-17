# Changelog
All notable changes to this project will be documented in this file.

## WIP
### Added
- energy_key, force_key, stress_key options for `sevenn_graph_build` @thangckt
### Changed
- Read EFS of atoms from y_* keys of .info or .arrays dict, instead of caclculator results
### Fixed
- [e3gnn_serial]: can continue simulation even when atom tag becomes not consecutive (removing atom dynamically) @gasplant64
- [e3gnn_parallel]: undefined behavior when there is no atoms to send/recv (for non pbc system)
- [e3gnn_parallel]: incorrect force/stress in some edge cases (too small simulation cell & 2 process)
- [e3gnn_parallel]: revert commit 14851ef, now e3gnn_parallel is sane.
- [e3gnn_*]: += instead of = when saving virial stress and forces @gasplant64

## [0.9.5]
### Note
This version is not stable, but I tag it as v0.9.5 before making further changes.
LAMMPS `pair_e3gnn_parallel.*` should be re-compiled for the below changes regarding LAMMPS parallel.
This is the first changelog and may not reflect all the changes.
### Added
- Stress compute for LAMMPS sevennet parallel
- `sevenn_inference` now takes .extxyz input
- `sevenn_inference` gives MAE error
- Experimental `sevenn_inference` on the fly graph build option
### Changed
- **[Breaking]** Parallel LAMMPS model changed, old deployed parallel models will not work
- **[Breaking]** Parallel LAMMPS takes the directory of potentials as input. Accordingly, `sevenn_get_model -p` creates a folder with potentials.
- **[Breaking]** Except for serial LAMMPS models, force and stress are computed from gradients of edge vectors, not positions.
- Separate interaction block from model build
- Add typing for most of functions
- Remove clang pre-commit hook as it breaks lammps pair files
- `torch.load` with `weights_only=False`
- Line length limit 80 -> 85
- Refactor
### Fixed
- Correct batch size for SevenNet-0(11July2024)

## [0.9.4] - 2024-08-26
### Added
- D3 correction (contributed from dambi3613) for LAMMPS serial
