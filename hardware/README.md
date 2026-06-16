# Hardware

Physical device design — schematics, PCB layout, enclosure, and BOM.

| Path | Contents |
|---|---|
| `kicad/` | KiCad schematics and PCB layout (RF keep-out zones, 50 Ω routing) |
| `openscad/` | Parametric enclosure models (`main_body.scad`, `sled_18650.scad`, `sled_2xAA.scad`) |
| `bom/` | PCBway BOM files and sourcing notes |

Compute SoC, storage, and power-component choices are deferred until dev-platform
benchmarks complete — see [`ROADMAP.md`](../ROADMAP.md) and requirements §10.
