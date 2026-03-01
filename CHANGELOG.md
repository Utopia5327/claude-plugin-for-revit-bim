# Changelog

All notable changes to the `revit-bim` plugin are documented here.

This project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0] — 2025-01-01

### Added
- `generate-dynamo` skill: generate complete Dynamo Python scripts for any Revit automation task
- `audit-model` skill: comprehensive model health report (warnings, element counts, families, views, worksets)
- `query-model` skill: read-only filtered element collection and data extraction scripts
- `analyze-rooms` skill: GIA/NIA calculation, space program comparison, unbounded room detection
- `manage-parameters` skill: batch read/write Revit parameters with type detection
- `clash-detection` skill: geometric intersection check between two element sets
- `create-schedule` skill: programmatic Revit schedule creation with fields, filters, and sheet placement
- `export-ifc` skill: IFC 2x3 / IFC 4 export with configurable options
- `revit-api-code` skill: C# external commands, macros, .addin manifests, and pyRevit scripts
- `manage-sheets` skill: batch sheet creation, view placement, view template application
- `manipulate-elements` skill: move, rotate, copy, change type of Revit elements
- `bim-agent`: specialist Claude agent with deep Revit/Dynamo/BIM knowledge, activates automatically
- Default to IronPython 2.7 for maximum Dynamo 2.x compatibility
- Full unit conversion coverage (feet ↔ mm/m)
- Comprehensive try/except and transaction handling in all generated scripts
