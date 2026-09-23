# QWRF Morrison shadow experiment

The opt-in code in `phys/module_mp_morr_two_moment.F` records the first column
whose provisional rain density exceeds `1e-4 kg m-3` somewhere in the column
and `1e-6 kg m-3` at the bottom. The selected scheme is
standard Morrison (`mp_physics=10`). Run this experiment in a **serial WRF
build**, from a dedicated case directory, with `QWRF_SHADOW=1` in the WRF
process environment. The output is `qwrf_shadow.dat` in that directory.

The original WRF explicit sedimentation still updates the forecast. The shadow
code independently solves the rain mass and rain number backward-Euler systems
using their frozen WRF fall speeds, and records both solutions. It times 100,000
repeated pairs of Fortran bidiagonal solves; this timing excludes PSD work,
transport, file I/O, and all quantum work. With `QWRF_SHADOW` absent, no file is
written. This first version writes one fixed filename and is intended only for
serial runs, not concurrent MPI ranks or OpenMP threads.

Each `QWRF_SHADOW_QR` or `QWRF_SHADOW_NR` row contains the WRF level index,
`dz` (m), air density (kg m-3), downward fall speed (m s-1), provisional
density variable `b`, original explicit result, and shadow implicit result.
For rain mass, `b` and results have units kg m-3; for rain number, m-3. The
header records the number of levels, WRF microphysics `DT` (s), explicit
substeps, benchmark repetitions, and microseconds per rain mass and number
solve pair. The record does not yet include horizontal location or model time.

Analyze the trace with `Py-Morrison/run_wrf_shadow_milestone.py` in the separate
research repository. It checks explicit replay and Fortran implicit solutions
against independent Python calculations, reports conservation and the
explicit-implicit discretization difference, and runs statevector VQLS on a
small contiguous window of the recorded rain system. When the window ends
below model top, its upper boundary value comes from the full Fortran implicit
solution; the VQLS task is therefore a local subproblem, not a full-column
quantum solve. No QPU execution or speedup is implied.

For a reproducible serial comparison, start two case directories with the
same `wrfinput_d01` and `namelist.input`. Run one with `QWRF_SHADOW` unset and
the other with `QWRF_SHADOW=1`; compare their `wrfout` files. The Python
analysis records a SHA-256 of the trace and the edited Morrison source. Its
acceptance gates are relative replay/reference error below `5e-5`, relative
column-budget error below `5e-5`, nonnegative density variables to `1e-10`,
and VQLS residual and window error below `1e-3`. The difference between the
explicit and implicit solutions is reported separately because they use
different time discretizations.
The report also requires positive rain bottom fallout and matching SHA-256
hashes for the paired forecast output files.

## Phase and substep ensemble

With `QWRF_ENSEMBLE=1`, the same serial Morrison build also writes
`qwrf_ensemble.dat`. This is a separate opt-in diagnostic and does not require
`QWRF_SHADOW=1`. It stores at most two separated columns in each exclusive
selection class: warm rain, cold dominated, mixed phase, and `NSTEP>1`.
The four class IDs are 1, 2, 3, and 4. Selection prioritizes `NSTEP>1`, then
mixed, cold dominated, and warm. Thresholds are on provisional density after
multiplication by air density: mass peak above `1e-5 kg m-3`; opposite-phase
mass peak below `1e-6 kg m-3` for a pure-phase label. The records have an
active-call index but no horizontal coordinate or timestamp.

Each `QWRF_ENSEMBLE_BEGIN` row has class ID, call index, number of levels,
microphysics time step, and WRF explicit `NSTEP`. Each `QWRF_ENSEMBLE_FIELD`
row has field ID, WRF level, layer thickness, air density, frozen downward
fall speed, provisional density, WRF explicit output, and frozen-speed
one-step backward-Euler output. Field IDs 1–10 correspond to rain mass,
rain number, ice mass, ice number, snow mass, snow number, graupel mass,
graupel number, cloud mass, and cloud number. `QWRF_ENSEMBLE_END` closes each
column. Python reads and validates these records with
`Py-Morrison/run_wrf_ensemble.py` in the separate research repository.

The ensemble trace and reports are in `Py-Morrison/benchmark_outputs` and
`Py-Morrison/reports/wrf_v48_ensemble_shadow.md`. A paired no-env/control run
is required to verify unchanged forecasts; a successful WRF completion
message alone is insufficient to rule out NaNs. The 120-level, 40-minute
stress run did develop NaNs after 15 minutes and was discarded. Its retained
15-minute paired run has finite outputs and includes a real `NSTEP=2` column.
