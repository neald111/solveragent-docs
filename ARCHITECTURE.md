# SolverAgent Architecture

**Version**: 2.5.0  
**Runtime**: .NET Framework 4.8 / x86 (COM interop with Synergi Gas SDK)

---

## Purpose

SolverAgent is a headless automation layer for Synergi Gas hydraulic modeling. It replaces manual Synergi Desktop workflows with a CLI pipeline that:

1. Downloads models from network shares to a local tier-based folder structure
2. Imports demand data (DFC exchange, profiles, flow categories) into Access MDB files
3. Invokes the Synergi Gas SDK (via reflection) to solve the model
4. Enriches results with HIS (High Integrity System) data from Oracle
5. Extracts solved results to GeoJSON for consumption by GIS/mapping systems
6. Supports parallel execution across multiple models bounded by license seats

---

## Solution Structure

```
SolverAgent.sln
├── SolverAgent.Core        # Domain models, MDB schema discovery (OLEDB), MDB writing
├── SolverAgent.Engine      # SDK layer, demand writers, extraction, OML, HIS enrichment
├── SolverAgent.Cli         # CLI entry point (SolverAgent.exe) — single-model pipeline
├── SolverAgent.Orchestrator# Multi-model parallel execution (spawns CLI child processes)
├── SolverAgent.Publisher   # GeoJSON→Oracle publisher (planned)
├── SolverAgent.Tests       # Unit + integration tests (MSTest)
└── SolverAgent.Tools.*     # Developer utilities
```

**Dependency chain**: Core ← Engine ← Cli / Orchestrator / Publisher / Tests

---

## Pipeline Flow (Steady-State)

```
┌─────────────────────────────────────────────────────────────────┐
│  OML Download (one-time setup)                                   │
│  Network share → local OML folder organized by tier              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Phase 0b: MdbSchema.Discover()                                  │
│  OLEDB table/column enumeration → ModelState detection           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Phase 2A: SteadyStateWriter.Import() — OLEDB bulk (pre-SDK)    │
│  DFC exchange .xlsx → GasNodeBaseFlow (DELETE + INSERT)          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Phase 1: SDK Initialize → LoadModel                             │
│  SolverWrapper.dll loaded via Assembly.LoadFrom + reflection     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Phase 2B: SteadyStateWriter.ImportSdkData() — SDK writes       │
│  Profiles + flow category multipliers via SetValueByTransaction  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Solve: SolverEngine.Balance()                                   │
│  Populates GasNodeResults, GasPipeResults, GasValveResults, etc. │
│  ErrorCode=0 → success (regardless of bool return)               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  SaveModel → CheckinLicense → Re-discover schema                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  HIS Enrichment: HisEnricher.Enrich()                            │
│  Oracle query → SubsystemId trace → MDB custom attrs             │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Extraction: ExtractResults.Extract()                            │
│  OLEDB read → GeoJSON output (nodes, pipes, valves, regs, comps) │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Classes

### SolverAgent.Core

| Class | Purpose |
|-------|---------|
| `MdbSchema` | OLEDB schema discovery — tables, columns, NameIdMaps, ModelState |
| `MdbWriter` | OLEDB write operations — bulk insert, column ensure, custom attrs |
| `ModelState` | Enum: Empty → DemandsLoaded → Balanced → BalancedAndSaved |
| `SolverMode` | Enum: SteadyState / TimeSimulation / Optimizer |
| `TierLookup` | Case-insensitive model→tier dictionary with JSON cache |
| `OmlConfiguration` | Source paths, division lists, model mapping |
| `OmlFilter` | Tier/division/model filtering for discovery |
| `HisRecord` | DTO for one Oracle HIS reference row |
| `HisNodeData` | Per-node HIS enrichment data |
| `HisResult` | Container for all HIS enrichment results |

### SolverAgent.Engine

| Class | Purpose |
|-------|---------|
| `SolverEngine` | Reflection-based SDK — Balance, TimeSimulation, Optimize, GetValue, SetValue |
| `SteadyStateWriter` | DFC exchange import (Phase A: OLEDB, Phase B: SDK transactions) |
| `ExtractResults` | Post-solve OLEDB extraction → GeoJSON (tier-aware, 7 output files) |
| `TierDetector` | Model tier classification (lookup-first, sentinel node fallback) |
| `OmlDiscovery` | Network share scanning — finds MDB + DFC per model |
| `OmlDownloader` | Copies models to local OML root with manifest tracking |
| `HisEnricher` | Oracle HIS enrichment orchestrator |
| `OracleHisReader` | Query Oracle for HIS reference data |
| `SubsystemIdResolver` | GasNodeResults SubsystemId → NodeId mapping |
| `FallbackResolver` | Prior-GeoJSON-based HIS resolution |
| `HisValidator` | Topology comparison + stale attribute detection |
| `HisNameDecoder` | HTML entity decoding for PressureSegmentName values |

---

## SDK Reference (SolverEngine)

The SDK is loaded via reflection (COM-based, x86 only).

### Available Operations

| Method | Purpose | Status |
|--------|---------|--------|
| `Initialize()` | Load SDK, create solver instance | Working |
| `LoadModel(path)` | Load MDB into SDK context | Working |
| `SaveModel(path)` | Persist results back to MDB | Working |
| `SaveModelAs(path)` | Save to new path (scenario variants) | Working |
| `Balance()` | Steady-state hydraulic solve | Working |
| `TimeSimulation(hourSets)` | USM loop: Init→Run×N→Exit | Runs (capture TBD) |
| `Optimize()` | Constraint-based optimization | Working (extraction TBD) |
| `GetValue(itemType, name, attr)` | Read SDK in-memory attribute | Working |
| `SetValue(itemType, name, attr, val)` | Write single attribute | Working |
| `BeginTransaction(size)` | Start bulk SDK write | Working |
| `SetProfilePoint(...)` | Write demand profile point | Working |
| `SetFlowCatMultiplier(...)` | Write flow category multiplier | Working |
| `CommitTransaction()` | Commit bulk SDK writes | Working |
| `DoesItemExist(itemType, name)` | Check node/element existence | Working |
| `CheckinLicense()` | Release license seat | Working |

### License Seats

| Type | Seats | Used By |
|------|-------|---------|
| SLVG | 11 | Base requirement (all modes) |
| SSM | 11 | Steady-state balance |
| USM | 11 | Time simulation |
| OPT | 7 | Optimization |

---

## Model Tiers

| Tier | PSIG Range | Solve Mode | Count | Characteristics |
|------|-----------|------------|-------|-----------------|
| Distribution | < 60 (mostly) | Steady-State | ~200 | Neighborhood networks, DFC demand loading |
| Local Transmission (LT) | 60-400 | Steady-State | ~30 | Transmission feeders, dual regulator control |
| Backbone (BB) | System-wide | Optimizer | 3 | Compressor stations, constraint optimization |

---

## Output Files

Per model/scenario extraction produces:

| File | Geometry | Key Properties |
|------|----------|----------------|
| `nodes.geojson` | Point | NodeName, ResultPressure, Flow, IsLPNode, Subsystem, HIS, MAOP, NOP, MDP, PCAT, WXSTA, SubsystemId, Elevation, CustomerCount, IsMinPressureNode |
| `pipes.geojson` | LineString | PipeName, ResultFlow, Velocity, Diameter, Length, Material, HIS, StatusId |
| `valves.geojson` | Point | ValveName, Cmax, Cmin, SetPressure, NominalSize, Flow, ValveStatus, HIS |
| `regstations.geojson` | Point | SetPressure, Constant, PressureSegmentName_Up/Down, OutletHIS |
| `regunits.geojson` | Point | Regulator unit details |
| `compressors.geojson` | Point/null | Station + Unit features (BB only) |
| `summary.json` | — | Tier, modelState, deviceCounts, outputFiles, CRS, version |

**CRS**: `urn:ogc:def:crs:EPSG::26910` (NAD83 UTM Zone 10N, units: feet)  
**Numeric values**: Unquoted (critical for ArcGIS compatibility)

---

## What's Remaining

### USM (Transient / Time Simulation)
- Demand import for 3 hour-sets not yet built
- Per-hour-set SDK capture (GetValue between Run iterations) not yet built
- Mode-aware extraction needed

### Backbone / Optimizer
- Constraint parsing and demand import not yet built
- Solve itself works; extraction behavior under validation

### Oracle Publisher
- GeoJSON → Oracle table push not yet implemented
