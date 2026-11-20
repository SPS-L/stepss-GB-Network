# GB Network Test System

RAMSES/STEPSS model of a reduced Great Britain (GB) transmission system with HVDC interconnections (three LCC links and two VSC links), for dynamic simulation and wide-area monitoring/control studies.

The system is a 50 Hz reduced-order representation of the GB grid with 87 buses and 36 synchronous machines (thermal and CCGT units with AVR/PSS and governor models), shunt compensation, and HVDC links modelled as two-port converter injectors. A one-line diagram is provided in `GBSystemLCC.pdf`/`GBSystemLCC.png`.

## Repository Files

| File | Description |
|------|-------------|
| `GBdyn.dat` | Dynamic data: buses, machines, exciters/PSS, governors, loads, shunts |
| `GBvoltrat.dat` | Power-flow solution (bus voltages and transformer ratios), solved by Helios from `GBpf.dat` |
| `GBpf.dat` | Helios power-flow input for the AC network, with the HVDC terminals as constant P/Q loads |
| `GBhvdc.txt` | HVDC interconnection data (two-port converter models) |
| `settings.dat` | Solver settings |
| `disturb.dst` | Disturbance scenario: 180 s run with commented templates for line/machine trips, bus faults, and HVDC parameter changes |
| `nothing.dst` | No-disturbance run |
| `obs.dat` | Observables selection |
| `cmd.txt` | RAMSES command file listing the data files above |
| `GB-Network.cfg` | STEPSS scenario file: opens the `cmd.txt` case in the STEPSS GUI with **File > Load configuration**. Paths are relative to this folder, so it travels with a copy or a move |
| `GBSystemLCC.pdf`, `GBSystemLCC.png` | One-line diagram of the system |

## Quick Start

With [stepss](https://stepss.sps-lab.org/):

```python
import stepss
case = stepss.cfg()
case.addData('GBdyn.dat')
case.addData('GBhvdc.txt')
case.addData('GBvoltrat.dat')
case.addData('settings.dat')
case.addDst('disturb.dst')
case.addObs('obs.dat')
sim = stepss.sim()
sim.execSim(case)
```

Or run the RAMSES executable directly with `cmd.txt`. Run from the repository root so that relative paths to the data files resolve.

## Status

**Runs flat for the full 240 s of `nothing.dst`** with RAMSES 3.84: every machine holds its operating point (largest speed excursion 8e-8 pu) and so does every AC bus away from the HVDC links. The buses of the EWIC and Nemo VSC converters do not: within the first second they move by up to 0.0039 pu (at 27IC2) and settle at a slightly different point, which they hold to the end. Tracked in [#2](https://github.com/SPS-L/stepss-test-systems/issues/2). At initialisation RAMSES creates a single `M_` mismatch load, 0.011 Mvar at bus 17, which is power-flow rounding.

That was not true before November 2026. Every version of this case up to then stopped with `Divergence` at t = 0.35 s and created 16 `M_` loads at initialisation. An earlier version of this section claimed the case ran; that came from a check that `stepss.sim().execSim()` returned without an exception, which it also does after a divergence, so a run that ended at 0.34 s was read as a pass. `CurrentStatus.ipynb` now checks how far the trajectory got.

Three faults were fixed.

| Fault | Effect | Fix |
|-------|--------|-----|
| `Er`/`Ei` of the three `HVDC_LCC` links were about 2.5 times too high (250, 450, 540 kV) | All three links started at firing and extinction angles near 70 deg, needing a large shunt capacitor at each weak inverter bus; the Moyle link diverged at 0.35 s and the other two inverter buses rose to 3.3 and 2.0 pu | `Er`/`Ei` sized for 15 deg firing and 18 deg extinction at the scheduled power: 87.4/88.4, 164.0/160.2 and 193.2/188.4 kV |
| The four VSC converters had been moved to `HVDC_VSC_SC`, which only controls active power | Neither end of a link held the DC voltage, and the Nemo DC buses collapsed to 0.5 pu by 1.4 s | Back to `HVDC_VSC`, callable by name since RAMSES 3.79, with the original control coefficients: the GB end on active-power control, the interconnector end on DC-voltage control |
| The power flow was solved with Moyle, EWIC and Nemo at 0 MW, and five generator buses gave their devices reactive shares summing to less than 1 | RAMSES created `M_` loads for the difference: -10, -500 and -1000 MW at buses 5, 12 and 27, the DC line losses at the four DC buses, and the unassigned reactive power at G1, G6, G19, G27 and G29 | Power flow re-solved with Helios from `GBpf.dat`, described below |

### Power flow

`GBpf.dat` is the Helios input for the AC network, and solving it gives the `LFRESV` records and transformer ratios in `GBvoltrat.dat`. Helios has no HVDC model, so each converter is a constant P/Q load at its AC bus. Built from the previous `GBvoltrat.dat` with the three links at 0 MW, it reproduces that file's voltages to within 0.00001 pu, which is how the input was checked.

Three choices in it change the operating point, and each is stated in the file header:

- The 1510 MW exported over Moyle, EWIC and Nemo is supplied by the non-nuclear, conventional-only stations G17, G18, G21, G22, G25, G26 and G28. Each machine takes 85.6% of its headroom below its rated power `Pnom`, so none exceeds its governor limit, and the shares `FP`/`FQ` of the two-machine stations G26 and G28 change in the sixth decimal. **G26 is now the slack bus** (it was G27, which carries only a wind farm).
- Every wind farm now takes the same share of its bus's reactive power as of its active power (`FQ = FP`). The five that did not are `WF3G1.05`, `WF3G6.05`, `CONVG19.06`, `CONVG27.06` and `CONVG29.06`.
- Each bus with wind farms is limited to 0.44 pu of each farm's rating. Only G27 reaches it: its farm absorbs 660 Mvar, and the bus rises to 1.023 pu.

The DC side is not in the power flow. On each link the interconnector-side DC bus keeps its voltage, the GB-side DC voltage is set so the line carries the scheduled power, and the receiving converter passes on what arrives after the line losses (496.171 MW of 500, 990.009 MW of 1000).

### Nemo converter rating

`HVDC_VSC` limits the d-axis current to `Pnom/Snom`. With Nemo's original `Pnom` of 1000 MW on 1110 MVA that is 0.901 pu, while exporting 1000 MW at the 0.977 pu of bus 27 needs 0.922 pu, so the link started on its limit and settled at 977 MW. Both Nemo converters now have `Pnom = Snom = 1110`, and the link holds its 1000 MW.

One small transient remains on the far side of each VSC link. The initial balance does not include the converters' phase-reactor losses, so the interconnector-side converters settle a little below the power that reaches their DC bus, and the Thevenin buses 12IC and 27IC2 move by up to 0.004 pu as they absorb the difference. The GB buses do not move.

### Earlier migrations

The case first stopped at reading, and two of the original migrations still stand.

| Record | Was | Now |
|--------|-----|-----|
| `INJEC WT3` (9) and `INJEC WT4` (10) in `GBdyn.dat` | 53 parameters, the earlier DFIG formulation | 128 (WT3) and 103 (WT4) parameters of the current WECC composite model |
| `TWOP HVDC_LCC` (3) in `GBhvdc.txt` | 20 parameters | 17: `alphamin`, `Idred` and `gammamin` are no longer user inputs, the `HVDC_LIM` controller sets them |

For the wind farms, rating and drivetrain carry over and the control blocks take WECC defaults. The third migration, of the VSC links to `HVDC_VSC_SC`, has been reversed as described above.

`CurrentStatus.ipynb` is the notebook for this system.

## Documentation

The data formats are documented in the STEPSS user guide at [stepss.sps-lab.org](https://stepss.sps-lab.org/).

## License

This repository is licensed under the [Apache License 2.0](LICENSE).

## Authors

Developed and maintained by the [Sustainable Power Systems Laboratory (SPS-L)](https://sps-lab.org/) at the Cyprus University of Technology, under the direction of Dr. Petros Aristidou.
