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
| `analysis.ipynb` | Wide-area monitoring example notebook (PyRAMSES); currently configured for the Nordic test system data and kept here as a starting point — adapt the data-file names to the GB files before use |
| `GBSystemLCC.pdf`, `GBSystemLCC.png` | One-line diagram of the system |

## Quick Start

With [PyRAMSES](https://stepss.sps-lab.org/):

```python
import pyramses
case = pyramses.cfg()
case.addData('GBdyn.dat')
case.addData('GBhvdc.txt')
case.addData('GBvoltrat.dat')
case.addData('settings.dat')
case.addDst('disturb.dst')
case.addObs('obs.dat')
sim = pyramses.sim()
sim.execSim(case)
```

Or run the RAMSES executable directly with `cmd.txt`. Run from the repository root so that relative paths to the data files resolve.

## Documentation

The data formats are documented in the STEPSS user guide at [stepss.sps-lab.org](https://stepss.sps-lab.org/).

## License

This repository is licensed under the [Apache License 2.0](LICENSE).

## Authors

Developed and maintained by the [Sustainable Power Systems Laboratory (SPS-L)](https://sps-lab.org/) at the Cyprus University of Technology, under the direction of Dr. Petros Aristidou.
