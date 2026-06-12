# SolverAgent User's Guide

**Version**: 2.5.0  
**Audience**: Engineers and developers building applications on top of SolverAgent  
**Prerequisites**: Windows x86, .NET Framework 4.8, Synergi Gas SDK installed

---

## What is SolverAgent?

SolverAgent is a headless command-line tool that automates Synergi Gas hydraulic modeling. It takes a gas network model (Access MDB file), imports demand data, runs a hydraulic solve via the Synergi SDK, and exports results as GeoJSON files ready for GIS consumption.

Think of it as middleware between:
- **Input**: Model library (MDB files) + DFC demand exchange files
- **Output**: GeoJSON feature collections (nodes, pipes, valves, regulators, compressors) + JSON summary

---

## Quick Start

### 1. Download a model

```bash
SolverAgent.exe oml-download --tier D --model CC_SALINAS_H
```

### 2. Run the full solve pipeline

```bash
SolverAgent.exe --model "path\to\model_CWD.mdb" ^
                --scenario APD ^
                --scenario-dir "path\to\DFC" ^
                --output "output\dir"
```

### 3. Find your results

```
output\dir\
├── nodes.geojson        # All nodes with pressures, flows, HIS data
├── pipes.geojson        # All pipes with flow, velocity, diameter
├── valves.geojson       # Valve positions and flows
├── regstations.geojson  # Regulator stations with set pressures
├── regunits.geojson     # Regulator units
├── summary.json         # Run metadata, counts, tier
└── .success             # Pipeline completion marker
```

---

## CLI Commands

### Solve Pipeline (default)

```
SolverAgent.exe --model <path.mdb>
                [--scenario <name>]        # Default: MDB filename stem
                [--scenario-dir <dir>]     # DFC exchange file location
                [--output <dir>]           # Output directory
                [--mode steady|usm|opt]    # Default: steady
                [--no-import]              # Skip demand import, just re-balance
                [--oracle-conn <connstr>]  # Oracle for HIS enrichment
                [--model-name <name>]      # Model name for Oracle HIS lookup
```

**Exit codes**: 0=success, 1=bad args, 2=import failed, 3=SDK init failed, 4=solve failed, 5=extraction failed

### OML Download

```
SolverAgent.exe oml-download [options]

Options:
  --tier D|LT|BB           Filter by tier
  --division <name>        Filter by division
  --model <name>           Filter by model folder name
  --all                    Download everything
  --force                  Re-download even if unchanged
  --oml-root <path>        Override local root
  --refresh-whitelist      Scan source and write model list CSV
```

### Orchestrator (multi-model parallel)

```
SolverAgent.Orchestrator.exe --source <dir> --max-seats <1-11>
                             [--manifest <file.json>] [--mode steady] [--timeout 10]
```

Manifest format:
```json
[
  {"mdbPath": "model.mdb", "scenario": "CWD", "scenarioDir": "dfc_dir", "mode": "steady"}
]
```

---

## Output Format Reference

### nodes.geojson

Point features with these properties:

| Property | Type | Description |
|----------|------|-------------|
| NodeName | string | Synergi node name |
| NodeID | int | Internal node ID |
| ResultPressure | number | Solved pressure (PSIG) |
| Flow | number | Node flow (SCFH) |
| PressureUnits | string | Always "psig" |
| IsLPNode | bool | True if low-pressure service |
| Subsystem | string | PressureSegment name |
| HIS | string | High Integrity System name |
| MAOP | number | Maximum Allowable Operating Pressure |
| NOP | number | Normal Operating Pressure |
| MDP | number | Maximum Design Pressure |
| PressureClassification | string | PCAT code (HP, SHP, etc.) |
| WXSTA | string | Weather station code |
| SubsystemId | int | Subsystem identifier |
| Elevation | number | Node elevation (feet) |
| CustomerCount | int | Customers served at node |
| IsMinPressureNode | bool | Lowest pressure in segment |

### pipes.geojson

LineString features:

| Property | Type | Description |
|----------|------|-------------|
| PipeName | string | Pipe element name |
| ElementId | int | Internal element ID |
| FromNode | string | Upstream node name |
| ToNode | string | Downstream node name |
| ResultFlow | number | Solved flow (SCFH) |
| Velocity | number | Flowing velocity (ft/s) |
| Diameter | number | Inside diameter (inches) |
| Length | number | Pipe length (feet) |
| Roughness | number | Pipe roughness factor |
| Efficiency | number | Pipe efficiency factor |
| Material | string | Pipe material |
| HIS | string | HIS name (from upstream node) |
| StatusId | int | Status code |

### valves.geojson

Point features (midpoint between From/To nodes):

| Property | Type | Description |
|----------|------|-------------|
| ValveName | string | Valve name |
| ElementId | int | Internal element ID |
| Cmax | number | Maximum constant |
| Cmin | number | Minimum constant |
| SetPressure | number | Set pressure |
| NominalSize | number | Nominal valve size |
| Flow | number | Solved flow |
| ValveStatus | string | Open/Closed/etc. |
| HIS | string | HIS name |

### summary.json

```json
{
  "solverVersion": "SolverAgent v2.4.2",
  "scenario": "APD",
  "tier": "Distribution",
  "modelState": "Balanced",
  "crs": "EPSG:26910",
  "isModelBuilder": false,
  "hasPressureSegments": true,
  "outputFiles": ["nodes.geojson", "pipes.geojson", ...],
  "deviceCounts": {"nodes": 907, "pipes": 895, "valves": 13, "regulators": 15},
  "extractTimestamp": "2026-06-11T19:00:00.000Z"
}
```

---

## Integration Patterns

### Python (subprocess)
```python
import subprocess, json

result = subprocess.run([
    "SolverAgent.exe",
    "--model", mdb_path,
    "--scenario", "APD",
    "--scenario-dir", dfc_dir,
    "--output", output_dir
], capture_output=True, text=True)

if result.returncode == 0:
    with open(f"{output_dir}/summary.json") as f:
        summary = json.load(f)
    print(f"Tier: {summary['tier']}, Nodes: {summary['deviceCounts']['nodes']}")
```

### Python (geopandas — read results)
```python
import geopandas as gpd

nodes = gpd.read_file("output/nodes.geojson")
low_pressure = nodes[nodes["ResultPressure"] < 25]
print(f"Nodes below 25 PSIG: {len(low_pressure)}")
min_node = nodes.loc[nodes["ResultPressure"].idxmin()]
print(f"Min pressure: {min_node['NodeName']} at {min_node['ResultPressure']:.1f} PSIG")
```

### C# (direct .NET reference)
```csharp
using SolverAgent.Core;
using SolverAgent.Engine;

var schema = new MdbSchema(mdbPath);
schema.Discover(includeRowCounts: true);
var writer = new MdbWriter(schema);
var extractor = new ExtractResults(schema, writer);
extractor.Extract(outputDir, "APD");
// Results now in outputDir as GeoJSON files
```

### JavaScript (read GeoJSON for web maps)
```javascript
// Note: CRS is EPSG:26910 (UTM) — reproject to WGS84 for web maps
const response = await fetch('output/nodes.geojson');
const geojson = await response.json();
// Use proj4js to reproject from UTM Zone 10N to WGS84
const proj4 = require('proj4');
proj4.defs('EPSG:26910', '+proj=utm +zone=10 +datum=NAD83 +units=ft');
```

### Batch processing manifest
```json
[
  {"mdbPath": "model1.mdb", "scenario": "CWD", "scenarioDir": "dfc1", "mode": "steady"},
  {"mdbPath": "model2.mdb", "scenario": "APD", "scenarioDir": "dfc2", "mode": "steady"}
]
```
```bash
SolverAgent.Orchestrator.exe --manifest batch.json --max-seats 4
```

---

## Model Tiers

| Tier | PSIG Range | Solve Mode | Count | Notes |
|------|-----------|------------|-------|-------|
| Distribution | < 60 (mostly) | Steady-State | ~200 | Neighborhood networks, DFC demand loading |
| Local Transmission (LT) | 60-400 | Steady-State | ~30 | Transmission feeders, dual regulator control |
| Backbone (BB) | System-wide | Optimizer | 3 | Compressor stations, constraint optimization |

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| Exit code 2 | Demand import failed | Check DFC exchange xlsx path and format |
| Exit code 3 | SDK init failed | Verify SDK installed, check license server |
| Exit code 4 | Balance failed | Model may have topology errors |
| Exit code 5 | Extraction failed | Check model state (must be Balanced) |
| "Model state is 'Empty'" | Extraction refused | Model needs demands + balance first |

---

## Limitations (Current Version)

- **USM/Transient mode**: Not yet implemented — time simulation results require per-hour-set SDK capture
- **Optimizer/Backbone**: Solve works but demand import and constraint parsing not yet built
- **Oracle Publisher**: GeoJSON→Oracle push not yet implemented
- **CRS**: Fixed at EPSG:26910 (UTM Zone 10N) — no reprojection
- **x86 only**: Required by the COM-based Synergi SDK
