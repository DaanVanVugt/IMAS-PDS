# prescribed_transport

## What it does

A minimal MUSCLE3 chain -- `source -> waveform_editor -> equilibrium -> sink` (+
`equilibrium.recorder_equilibrium`) -- that produces a "prescribed" (fixed, not self-consistently
coupled to transport) NICE free-boundary equilibrium dataset for one scenario. The
waveform editor overlays the designed Ip(t)/B0 boundary target onto the source's raw
equilibrium and supplies the static machine description (wall, pf_passive, iron_core) plus
the coil-current seed (pf_active), all imported from the scenario's waveform config rather
than carried through `source`. NICE-inverse solves the free-boundary equilibrium per time
slice; `sink` stores the resulting equilibrium and coil currents; `recorder_equilibrium` distills the
same data (config: `visualization/nice_inv.py`) for the muscle3-dashboard.

`equilibrium` is the `nice_inverse` submodel (defined in `workflow.ymmsl`): one NICE inverse
actor that receives the whole trace and solves it in batch mode (chunks of consecutive slices in
parallel over its threads, each slice warm-starting from its time neighbour; `batch_chunk_slices`
overrides the default of one chunk per thread), plus `recorder_equilibrium` recording its output.
The former load balancer (per-slice scatter over N workers) is gone; its psi re-gauging was a
no-op here (the waveform editor already writes psi[-1] = psi_boundary) and stays in
`inverse_convergence`, where the Picard state needs it.

## Running it

Build a case folder for a shot, then hand it to SLURM:

```bash
export SCENARIOS_REPO=/path/to/pds-scenarios   # defaults to /work/projects/pds/pds-scenarios

bin/pds-create-case prescribed_transport 105084       # -> cases/prescribed_transport_105084
sbatch --time=00:20:00 --cpus-per-task=8 bin/pds-run-case.sbatch cases/prescribed_transport_105084
```

`pds-create-case` stacks `workflow.ymmsl`, this workflow's `settings.ymmsl` (resources,
solver config, this workflow's own `waveforms_no_transport.yaml` pulse-design template, and
the input DBEntry, all templated from `${SHOT}`), and
`cases/overrides/prescribed_transport_<shot>.ymmsl` if it exists into numbered files under
the case folder; `pds-run-case.sbatch` runs that folder under `muscle_manager`, writing to
`cases/runs/<case>`.

Scenarios available: 105078, 105084, 105092, 105099.

Only the input DBEntry (`data/in`, `data/in_md`) is read live from `pds-scenarios`; the
pulse design (`waveforms_no_transport.yaml`) lives in this workflow's own directory, so it's
versioned and editable here. See `pds-scenarios`' `GENERATING.md` for how `data/` itself is
produced from DINA and machine-description sources.

## Assumptions

- Transport is **not** solved and is **not** self-consistent with the equilibrium: the plasma
  shape, Ip(t)/B0, and profile shape (`p'`, `FF'`) are fixed externally in the scenario's
  waveform config rather than produced by a transport code.
- Each time slice is an independent NICE-inverse solve -- there is no time coupling between
  slices, and no outer iteration (contrast with `inverse_convergence`, which wraps a similar
  NICE-inverse solve in a Picard loop against TORAX).
- The static machine description (wall, pf_passive, iron_core) and the coil-current seed
  (pf_active) never change across the pulse; they come from the scenario's machine-description
  entry, not from `source`.
- `equilibrium.nice` runs with 8 threads (`settings.ymmsl` resources); batch mode uses them
  for chunks of slices, so the solve time scales with the thread count up to the number of
  slices.

## Input requirements

Produced by `tools/prepare <shot>` in the `pds-scenarios` repository.

- A DINA-derived source supplying the equilibrium boundary/target trace:
  `vacuum_toroidal_field/r0` and `/b0`, `global_quantities/ip` and `/psi_boundary`, and
  `boundary/outline/r`/`/z`, plus `psi`, `f_df_dpsi`, and `dpressure_dpsi` on `profiles_1d`
  to seed NICE's first solve.
- Machine descriptions for `pf_active`, `pf_passive`, `wall`, and `iron_core`.
- The `pf_active` machine description must have exactly one element per coil in the `elements`
  array of structures, and each coil must carry a resistance value -- both required by NICE
  inverse.

## Output

The sink writes into the run directory itself:

- `sink` writes an IMAS HDF5 dataset at `<run_dir>/out_nice`, containing the
  per-time-slice free-boundary `equilibrium` and the solved `pf_active` coil currents.
- `recorder_equilibrium` writes a distilled copy of the same data for the muscle3-dashboard (see
  `visualization/nice_inv.py`).
