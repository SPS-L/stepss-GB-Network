# GB Network Test System

RAMSES/STEPSS model of a reduced Great Britain (GB) transmission system with HVDC-LCC interconnections, for dynamic simulation and wide-area monitoring/control studies.

The system is a 50 Hz reduced-order representation of the GB grid with 87 buses and 37 synchronous machines (thermal and CCGT units with AVR/PSS and governor models), shunt compensation, and HVDC links modelled as two-port converter injectors. A one-line diagram is provided in `GBSystemLCC.pdf`/`GBSystemLCC.png`.

## Repository Files

| File | Description |
|------|-------------|
| `GBdyn.dat` | Dynamic data: buses, machines, exciters/PSS, governors, loads, shunts |
| `GBvoltrat.dat` | Power-flow solution (bus voltages and transformer ratios) |
| `GBhvdc.txt` | HVDC interconnection data (two-port converter models) |
| `settings.dat` | Solver settings |
| `disturb.dst` | Disturbance scenario: 180 s run with commented templates for line/machine trips, bus faults, and HVDC parameter changes |
| `nothing.dst` | No-disturbance run |
| `obs.dat` | Observables selection |
| `cmd.txt` | RAMSES command file listing the data files above |
| `analysis.ipynb` | Wide-area monitoring example notebook (stepss); currently configured for the Nordic test system data and kept here as a starting point, adapt the data-file names to the GB files before use |
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

**Runs**, following the model migrations described below. The case previously stopped at
two-port reading and never reached the network solution.

Three migrations were needed, all of them cases where the model name still resolved or nearly
resolved but its parameter list had moved on:

| Record | Was | Now |
|--------|-----|-----|
| `INJEC WT3` (9) and `INJEC WT4` (10) in `GBdyn.dat` | 53 parameters, the earlier DFIG formulation | 128 (WT3) and 103 (WT4) parameters of the current WECC composite model |
| `TWOP HVDC_VSC` (4) in `GBhvdc.txt` | compiled but registered under no name | `TWOP HVDC_VSC_SC`, the registered VSC model |
| `TWOP HVDC_LCC` (3) in `GBhvdc.txt` | 20 parameters | 17: `alphamin`, `Idred` and `gammamin` are no longer user inputs, the `HVDC_LIM` controller sets them |

For the wind farms, rating and drivetrain carry over and the control blocks take WECC
defaults. For the VSC links, everything maps across except that the old `Kpd`/`Kid` become
`Kpp`/`Kip` under a new name, the old `alpha1`/`beta1`/`alpha2`/`beta2` characteristic
coefficients have no counterpart and are dropped, and a new gain `Kwi` appears.

**`Kwi` is set to 0 deliberately.** It multiplies `(1 - omega)` in the outer active-power
loop, so zero leaves the converter on pure P/Q control, which is what these grid-following
links represented. Raising it turns them into grid-forming, frequency-supporting devices,
which is a modelling decision rather than a migration.

Validated on a no-disturbance run: every bus holds its power-flow voltage, and the wind
farms sit at their initial power and rotor speed. Bus 5 settles 0.001 pu above the
power-flow value within 0.2 s, which is the HVDC control reaching its own equilibrium.

**`analysis.ipynb` in this folder still does not run**, for an unrelated reason: it loads
`dyn_A.dat`, `volt_rat_A.dat` and `settings1.dat`, which are the *Nordic* test system's
filenames and are not present here, and it calls `plt.grid(b=True, ...)`, a keyword removed
in Matplotlib 3.5. It appears to be an unadapted copy of the Nordic tutorial.
`CurrentStatus.ipynb` is the working notebook for this system.

## Documentation

The data formats are documented in the STEPSS user guide at [stepss.sps-lab.org](https://stepss.sps-lab.org/).

## License

This repository is licensed under the [Apache License 2.0](LICENSE).

## Authors

Developed and maintained by the [Sustainable Power Systems Laboratory (SPS-L)](https://sps-lab.org/) at the Cyprus University of Technology, under the direction of Dr. Petros Aristidou.
