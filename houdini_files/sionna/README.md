# Scene format, materials and messages

Reference for `sionna_sop.py` (the Houdini side) and `sionna_run.py` (the solver), with the material library in `materials.json`. How to set up and use the plugin is in [../README.md](../README.md).

## Scene: supported attributes

Everything Sionna receives comes from the attributes below. Any other attribute (`Cd`, `N`, `uv`, groups) is ignored.

### Geometry (primitive attributes, triangles only)

| Attribute | Type | Required | Meaning |
|---|---|---|---|
| `name` | string | yes | Object name. One mesh and one Sionna object per value. Letters, digits, `_` and `-` only. |
| `material` | string | yes | An ITU material, a name from `materials.json`, `measure` for a radio map surface, or `display` for display-only geometry |
| `thickness` | float | no | Slab thickness in metres used for reflection and transmission. 0 or absent: library default, else 0.1. Not the mesh thickness. |
| `scattering_coefficient` | float | no | 0 to 1. Share of reflected energy that is scattered diffusely. Only has an effect with `diffuse_reflection` on. 0 or absent: library default, else 0. |
| `moisture` | float | no | Percent water, only for vegetation materials (`leaf`, `stem`: % of fresh mass, living leaves 70 to 90) and soil materials (`soil_*`: % of volume, 0 to 50). 0 or absent: library default. Ignored on other materials (`W011`); outside 0 to 100 is an error (`V012`). |

All primitives with the same `name` must have the same material attributes (`V006`). `moisture` may vary: it is rounded to whole percent and each value becomes its own Sionna object, named `<name>_m<percent>` (e.g. `leaves_m80`); that is the name in the results and in `object` of `sionna_paths`. Every surface is a slab of the given thickness, so a closed box is a hollow shell: a ray through it crosses two slabs.

### Devices (point attributes on loose points)

| Attribute | Type | Required | Meaning |
|---|---|---|---|
| `P` | vector | yes | Position |
| `role` | string | yes | `tx` or `rx` |
| `name` | string | yes | Unique name, carried through to all results |
| `orientation` | vector | no | Yaw, pitch, roll in radians, Sionna convention (not converted from Houdini axes). Sets the antenna direction; only has an effect with an antenna other than `iso`. At (0, 0, 0) a `tr38901` panel looks along Sionna +x. |
| `antenna_pattern` | string | no | `iso`, `dipole`, `hw_dipole` or `tr38901`. Empty or absent: `base_tx_antenna_pattern` or `base_rx_antenna_pattern`. All TX points must end up with the same antenna, and all RX points (`V108`, `V109`): Sionna uses one antenna per role. |
| `polarization` | string | no | `V` or `H`. Empty or absent: the base parm of its role. Same rule per role. |
| `velocity` | vector | no | m/s. Exported, but not yet passed to Sionna: no effect. |
| `power_dbm` | float | no | TX power. Passed to Sionna, but no current output depends on it: all results are path gain. |

### Display-only geometry

Primitives with `material = "display"` are kept for viewing but never sent to Sionna: validate skips them entirely (any primitive type, polygons of any size, curves, no `name` needed, not counted in the scene size) and export leaves them out of the meshes and the scene XML. They are saved with everything else in `scene.bgeo.sc` of the run or work item folder, so after a batch fetch `sionna_batch_scene` shows them unchanged. On a remote target, `scene.bgeo.sc` stays on this machine, as always. Only primitives can be marked: loose points are read as devices (`V106` without a valid `role`).

### Measurement surfaces (radio maps)

Any triangulated mesh with `material = "measure"` and its own `name`. Each triangle is one cell of the radio map. Measurement surfaces are not part of the scene: they neither block nor reflect. Keep them off other surfaces, e.g. 1.5 m above a floor.

Solve computes paths when there is at least one RX, and a radio map when there is at least one measurement surface. Without either, validation fails (`V107`).

### Not settable per primitive

`xpd_coefficient`, and the model inputs `temperature_c`, `ionic_conductivity` and the soil texture (`sand`, `clay`, `silt`, densities), exist only per entry in `materials.json`.

## Materials

Every material name is looked up in `sionna/materials.json`. Values are computed at export for the run `frequency` and written to `<hip>_materials.json`, so changing the frequency requires a new export (`S008`). Every entry has a `source` and a `status`; a status other than `confirmed` gives warning `W005`. Entries with a measured frequency range give `W010` when the run frequency lies outside it: the values are then extrapolated and the solve still runs.

### Kinds

| Kind | Fields | Model |
|---|---|---|
| `itu` | `ranges_ghz`: `[lo, hi, a, b, c, d]` per range | ITU-R P.2040-3: `er = a * f^b`, `sigma = c * f^d`, f in GHz. Outside every range the nearest range is used (`W010`) |
| `cole_cole` | `eps_inf`, `eps_s`, `sigma_s`, `tau`, `alpha` | Cole-Cole fit to measurements |
| `soil` | `sand`, `clay`, `silt` (%), `specific_gravity`, `bulk_density`, `moisture` (% volume), `temperature_c` | ITU-R P.527-6 section 5.2 |
| `conductor` | `conductivity` | metal: permittivity 1, conductivity constant over frequency |
| `fixed` | `relative_permittivity` plus one of `conductivity`, `loss_tangent`, `loss_factor` | constant permittivity; loss converted at the run frequency |
| `water` | `temperature_c`, `ionic_conductivity` | Debye model (Liebe, Hufford, Manabe 1991, same equations as ITU-R P.527-6) plus ionic conductivity |
| `vegetation` | `moisture` (% of fresh mass) | dual-dispersion model (Ulaby and El-Rayes 1987) |
| `gabriel` | `eps_inf`, `terms` (`[delta, tau in s, alpha]` per dispersion), `sigma` | multi-pole Cole-Cole model for body tissue (Gabriel, Lau, Gabriel 1996) |

### Entries

**Building materials in situ (kind `itu`, ITU-R P.2040-3 Table 3, coefficients as in Sionna's `itu.py`, status confirmed).** Inside their range the values are identical to Sionna's built-in ITU materials (checked); outside it they are extrapolated instead of stopping the solve:

| Material | Measured range (GHz) | At 868/915 MHz |
|---|---|---|
| `vacuum`, `wood` | 0.001–100 | in range |
| `glass` | 0.1–100 | in range |
| `concrete`, `plasterboard`, `ceiling_board`, `chipboard` | 1–100 | extrapolated, `W010` |
| `brick`, `plywood`, `vinyl_tile`, `carpet_tile`, `asphalt_concrete` | 1–40 | extrapolated, `W010` |
| `marble` | 1–60 | extrapolated, `W010` |
| `very_dry_ground`, `medium_dry_ground`, `wet_ground` | 1–10 | extrapolated, `W010`; below 1 GHz prefer `soil_*` |
| `floorboard` | 50–100 | far outside, avoid |
| `clear_acrylic` | 110–330 | far outside, use `pmma` |

**Lab samples, air-dried (kind `cole_cole`, Zhekov, Franek, Pedersen 2020, IEEE Antennas Propag. Mag. 62(1), Table II, measured 0.2–67 GHz):** `pmma`, `polypropylene`, `ptfe`, `polystyrene` (solid, not foam), `pine_dry`, `hardboard_dry`, `plywood_dry`, `mdf_dry`, `chipboard_dry`, `wood_cement_board_dry`, `gypsum_dry`, `plasterboard_dry` (confirmed); `brick_red_dry`, `brick_yellow_dry`, `concrete_dry` (small gravel), `concrete_coarse_dry` (large gravel) give `W005`, because the paper reports a less reliable fit for brick and concrete.

**Metals (kind `conductor`, no frequency limit, confirmed):** `metal` (1e7 S/m, as ITU), `steel` (6.99e6, carbon steel), `stainless_steel` (1.45e6), `aluminium` (3.5e7), `zinc` (1.69e7, galvanised coating), `copper` (5.96e7). Any of them acts as a near-perfect reflector at these frequencies.

**Soils (kind `soil`, ITU-R P.527-6 section 5.2 with the textures of its Table 2, confirmed):** `soil_sandy_loam`, `soil_loam`, `soil_silty_loam`, `soil_silty_clay`. Default moisture 20 %, set per primitive with `moisture`. Range 0.3–18 GHz, the measurements the model is built on (Peplinski et al. 1995, Dobson et al. 1985). The implementation reproduces the reference values of MathWorks `earthSurfacePermittivity` (P.527-6) exactly. Example `soil_loam` at 868 MHz: er 4.1 / 11.2 / 20.9 and sigma 0.02 / 0.06 / 0.10 S/m at 5 / 20 / 35 % moisture.

**Greenhouse materials:**

| Material | Kind | Status |
|---|---|---|
| `water` | water | confirmed (equations checked against ITU-R P.527-6) |
| `nutrient_solution` | water | `W005`: ionic conductivity from an assumed EC of 2.5 mS/cm |
| `leaf`, `stem` | vegetation | `W005`: coefficients checked against a reproduction of the model (Peden et al. 2021), not the original paper; not validated for these crops; thicknesses (leaf 0.3 mm, stem 1 cm) assumed; valid 0.2–20 GHz |
| `fruit` | fixed | `W005`: strawberry values used for tomato |
| `polycarbonate` | fixed | `W005`: permittivity from datasheets at 1 MHz only |
| `polyethylene_film`, `pvc` | fixed | `W005`: permittivity confirmed, loss assumed zero |
| `rockwool_wet` | fixed | `W005`: placeholder, must be measured |

**People (kind `gabriel`, confirmed):** `human`, dry skin from Gabriel, Lau, Gabriel 1996 (Phys. Med. Biol. 41, Table 1), the outer layer a signal meets first. er / sigma: 41.6 / 0.86 S/m at 868 MHz, 38.0 / 1.46 at 2.45 GHz (the published values for dry skin), 35.1 / 3.72 at 5.8 GHz. Default thickness 0.15 m: a body blocks nearly everything that passes through it. Real signals also bend around a person; that needs `diffraction` on, otherwise the shadow behind a person is too deep.

### In situ versus lab values

The two sources disagree most for concrete and brick: the lab samples were dried, real walls hold water. Loss through a 23 cm concrete wall at normal incidence, computed with Sionna (CPU), against the 3GPP TR 38.901 formula for real walls (5 + 4 f dB):

| Material | 868 MHz | 2.4 GHz | 5 GHz |
|---|---|---|---|
| `concrete` (ITU, in situ) | 8.0 dB | 16.5 dB | 28.2 dB |
| `concrete_dry` (lab) | 1.8 dB | 1.5 dB | 2.6 dB |
| `concrete_coarse_dry` (lab) | 1.9 dB | 2.7 dB | 4.3 dB |
| 3GPP TR 38.901 | 8.5 dB | 14.6 dB | 25.0 dB |

The ITU values match real walls within 0.5 to 3 dB, also when extrapolated to 868 MHz. Use the ITU names for real buildings; the `_dry` entries describe very dry material and are a lower bound on loss.

### When a value is confirmed

A value becomes `confirmed` only when it comes from a measurement or a source that states it for the frequency range in use, with the reference in `source`. Ways to get there, in order of strength:

1. Measure it: dielectric probe or free-space measurement with a VNA, or fit it from channel measurements (CELSA T3; Sionna is differentiable for this).
2. Peer-reviewed values at the right frequency: ITU-R P.2040 and P.527, Zhekov et al. 2020, Ulaby and El-Rayes 1987, Peng et al. 2013 (tomato), manufacturer data at GHz frequencies.
3. Datasheet values at 1 MHz (plastics) are not enough for 433 MHz to 5 GHz.

## Messages

Codes starting with N are information, W warnings (do not block), all others are errors (stop the button).

| Code | Message | Meaning / what to do |
|---|---|---|
| N001 | scene size | Bounding box of the geometry in Sionna axes. Check units and orientation. |
| N002 | exported to | Run folder written |
| N003 | sionna dump written | Previz succeeded |
| N004 | paths, result written | Solve found paths |
| N005 | imported paths, links | Import succeeded |
| N006 | radio map cells | Radio map computed or imported |
| N008 | target status | Result of check: Sionna version and the device it will run on |
| N009 | batch exported | Number of work items exported and failed |
| N010 | batch started | The job runs on the target; follow it with batch status |
| N011 | batch results | Work items with a result, failed, and not solved (yet) |
| N012 | cleared | Number and size of the deleted folders |
| N013 | nothing to clear | No runs or batches of this hip file (and `batch_name`) on the target |
| W001 | scene diagonal outside 1 m to 50 km | Probably wrong units |
| W002 | object smaller than one wavelength | Ray tracing is not valid for it; results for that object are unreliable |
| W003 | device far from the geometry | Device probably misplaced or in other units |
| W004 | no rx point | Only the radio map is computed |
| W005 | material not confirmed | Library value not verified; results depend on it |
| W006 | GPU in use by other processes | Runs slower; large jobs may run out of memory |
| W010 | material outside its measured range | The run frequency lies outside the range the material was measured for; values are extrapolated. Below 1 GHz this applies to most ITU materials; see Materials |
| W011 | moisture set on a material that does not use it | `moisture` only affects vegetation and soil materials |
| W008 | node exists but is not a File SOP | A node next to the Null already has the name `sionna_previz`, `sionna_paths`, `sionna_links`, `sionna_radiomap` or `sionna_batch_scene`, `_paths`, `_links`, `_radiomap`. The result file is written but not loaded; rename that node. |
| W015 | preset parm has an expression | The preset did not overwrite it, e.g. `frequency` driven by `@frequency`; set the value by hand or remove the expression |
| W016 | device with orientation (0, 0, 0) | Missing `orientation` point attribute is exported as upright without error; set it (yaw, pitch, roll in radians, Sionna axes) on devices that should tilt, e.g. a sensor along its stem |
| W014 | running batch not cleared | Clear again after it finished, or stop it first |
| W009 | no `batch` parm on the Null | Add a string parm `batch`; the `sionna_batch_*` File SOPs read the batch name from it |
| V013 | frequency not valid | The `frequency` parm is 0 or below, or an expression such as `@frequency` has no selected work item |
| V001 | no primitives to simulate | Nothing to export: no primitives, or only `display` primitives; add geometry or a measurement surface |
| V002 | missing prim attribute | Add `name` and `material` |
| V003 | not a triangle | Triangulate (Divide SOP) |
| V004 | zero area | Remove degenerate triangles (Clean SOP) |
| V005 | invalid name | Only letters, digits, `_`, `-` |
| V006 | material attributes differ within a name | Same `name` must have one material, thickness and scattering |
| V007 | unknown material | Not an ITU name and not in `materials.json` |
| V008 | thickness < 0 | Fix the attribute |
| V009 | scattering_coefficient outside 0 to 1 | Fix the attribute |
| V010 | empty name | Every primitive needs a `name` |
| V011 | material resolves to invalid values | Library entry gives permittivity < 1 or negative conductivity |
| V012 | moisture outside 0 to 100 | Fix the `moisture` attribute (percent) |
| V101 | missing point attribute | Add `role` and `name` to device points |
| V102 | no tx point | At least one TX is required |
| V103 | device with empty name | Name every device |
| V104 | duplicate device name | Device names must be unique |
| V105 | tx and rx at same position | Zero distance is undefined |
| V106 | loose point without valid role | A point that is not a device; remove it or set `role` |
| V108 | TX points with different antennas | Give all TX points the same `antenna_pattern` and `polarization`, or leave the attributes empty to use the base parms |
| V109 | RX points with different antennas | Same for the RX points |
| V110 | unknown antenna value | Use `iso`, `dipole`, `hw_dipole`, `tr38901` for `antenna_pattern`, `V` or `H` for `polarization` |
| V112 | base antenna parm empty | A `base_*` antenna parm on the Null is empty; set it, or pick an antenna preset. An empty base parm is an error, not a silent default (at solve: `S012`) |
| V111 | antenna attribute not a string | Create `antenna_pattern` and `polarization` as string point attributes |
| V107 | nothing to compute | Add an RX or a measurement surface |
| E001 | cannot create output directory | Check permissions and disk space |
| E002 | cannot write file | Check permissions and disk space |
| E003 | hip file is not saved | Save the hip file first; run folders live next to it |
| S001 | python executable not found | Local venv missing; check `python_exe` |
| S002 | solver script not found | `sionna/sionna_run.py` missing |
| S003 | solver exited with an error | Last line of the solver output; full output in `solver.log`. A missing Sionna installation in the venv also ends here (`No module named 'sionna'`). |
| S012 | setting not valid | A solver setting is out of range, e.g. `samples` 0 or an unknown `antenna_pattern`; for a work item with expression parms, check the wedge values |
| S011 | python on target failed | Check could not import Sionna; the last lines of the error are printed above it. On Python 3.10: `cannot import name 'Self'`, use Python 3.11 or later |
| S004 | solver timed out | Increase `timeout` or reduce the job |
| S005 | solver produced no result file | Solver ended without writing results |
| S006 | no exported run found | Export first |
| S008 | frequency changed since export | Export again |
| S009 | program not found | `ssh`, `rsync` or `nvidia-smi` missing on this machine |
| S010 | radio map exceeds the ray limit | `radiomap_samples` × number of TX is above Sionna's 4.29e9 rays per radio map. Lower `radiomap_samples` to the value shown. Checked before anything is sent. |
| T001 | target unreachable | VPN off, wrong SSH alias, or key not installed. The SSH reason is shown. |
| T002 | no python on target | No venv at `<remote_root>/venv` |
| T003 | no GPU on target | `use_gpu` is on but the target has no usable GPU. Switch `use_gpu` off to run on the CPU |
| T005 | no CPU backend on target | `use_gpu` is off but the LLVM library is missing on the target; install it (Ubuntu: `sudo apt install llvm`; Windows: see Windows) |
| T006 | no rsync on this machine | Needed for a remote target. Linux: `sudo apt install rsync`. Windows: install it in WSL, see Windows |
| T004 | file transfer failed | rsync error; reason shown |
| I001 | result file not found | Solve first |
| I002 | result file unreadable | Damaged file; solve again. Runs solved before the switch to `.npz` give `I001`; solve them again. |
| P001 | unknown preset | The `quality` or `band` parm holds a name that is not in `PRESETS` |
| W013 | old LLVM for the cpu backend | Shown by check when `use_gpu` is off and the target's LLVM is older than 17. LLVM 15 aborts cpu solves with `LLVM ERROR: Cannot select`; LLVM 20 works. On Ubuntu 22.04 install `libllvm20` from apt.llvm.org |
| W012 | preset parms missing on the Null | Those parms keep their defaults; add them to the Null to let the preset set them |
| B001 | TOP node not found | No `../topnet1`, no node with the display flag in it, or a wrong `top_node` |
| B002 | no work items in range | Check `batch_start` and `batch_end` against the work item indices |
| B003 | no batch found | Batch export first, or check the `batch` parm |
| B004 | batch already running | Wait for it, or stop it on the target |
| B010 | no work items in the fetch range | Check `fetch_start` and `fetch_end` against the work item indices of the batch |
| B009 | invalid `batch_name` | Use only letters, digits, `_` and `-` |
| C001 | clear failed on the target | The target was reached but listing or deleting failed; the error is shown |
| B007 | nothing to solve | Every work item failed at export or at the checks before sending (`S008`, `S010`) |
| B008 | input did not cook | The SOP network gave an error for this work item; it is skipped |
