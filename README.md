# Semantic Model Edge Bundling for Power BI

Hierarchical edge bundling visualization that maps all dependencies within a Power BI semantic model — relationships, measure dependencies, and calculated column dependencies — rendered as an interactive radial graph using Deneb (Vega 6.2.0) inside Power BI.

> **⚠️ XMLA Endpoint Required**
>
> This solution queries the semantic model via DMVs (`DISCOVER_CALC_DEPENDENCY` and `TMSCHEMA_RELATIONSHIPS`) through the XMLA endpoint. The XMLA endpoint is only available with:
>
> - **Power BI Premium Per User (PPU)** licensing, or
> - **Power BI Premium** capacity (P SKUs), or
> - **Microsoft Fabric** capacity (F SKUs)
>
> Without one of these, the Power Query queries that feed the visualization will not be able to connect to the semantic model's XMLA endpoint. A community version using Python and Altair (no Premium required) is on the roadmap.

---

## What It Does

The visualization arranges every column, measure, and calculated column in your semantic model as leaf nodes around a radial layout, grouped by table. Bundled curves connect nodes that have dependencies, color-coded by type:

- **Blue** (`#4682b4`) — Relationships (e.g., `OrderDate` → `Date`)
- **Red** (`#b22222`) — Measure dependencies (e.g., a measure referencing a column)
- **Orange** (`#ff8c00`) — Calculated column dependencies

Hovering over a node highlights all its connections, showing inbound edges in red and outbound edges in green. A hover info panel displays the node name, type, and edge counts. Slicers let you filter by dependency type and by table/object. Clicking a node cross-filters a detail table showing every dependency involving that node.

---

## Architecture Overview

### Data Flow

```
XMLA Endpoint
  ├── DISCOVER_CALC_DEPENDENCY (DMV)
  └── TMSCHEMA_RELATIONSHIPS (DMV)
        │
        ▼
Power Query (7 queries)
  ├── TreeNodes (staging) ──────────────┐
  ├── Dependencies (staging) ───────────┤
  ├── EdgeBundlingData (loaded) ◄───────┘  ← Deneb dataset
  ├── DepTypeFilter (loaded) ◄──── disconnected slicer
  ├── TableFilter (loaded) ◄────── disconnected slicer
  ├── DependencyDetails (loaded) ◄─ detail table
  └── NodeBridge (loaded) ◄──────── cross-filter bridge
        │
        ▼
DAX Measures (4)
  ├── SelectedDepTypes ──── slicer → Vega signal
  ├── SelectedTables ────── slicer → Vega signal
  ├── SelectedObjects ───── slicer → Vega signal
  └── ShowDetailRow ─────── cross-filter + slicer → detail table filter
        │
        ▼
Deneb / Vega 6.2.0 (v12 spec)
  └── Hierarchical edge bundling with interactive cross-filtering
```

### The Disconnected Slicer Pattern

The slicers (dependency type, table/object) are deliberately **not related** to the main dataset in the Power BI model. Instead, DAX measures read slicer selections and pass them as constant column values into the Deneb dataset. Inside Vega, `pluck(data('dataset'), 'MeasureName')[0]` extracts these values into reactive signals that drive edge and node opacity. This avoids the filtering side-effects that normal relationships would introduce — Power BI would filter the tree nodes themselves, breaking the radial layout.

### The Cross-Filter Bridge

When a user clicks a leaf node in the Deneb visual, the click needs to filter the `DependencyDetails` table to show matching rows. But `EdgeBundlingData` and `DependencyDetails` are intentionally disconnected. The solution is a `NodeBridge` table:

```
Deneb click → EdgeBundlingData[id] → NodeBridge[nodeId]
                                          │
                     ShowDetailRow reads SELECTEDVALUE(NodeBridge[nodeId])
                                          │
                     Manually checks DependencyDetails[sourceId] and [targetId]
```

Only **one relationship** exists in the entire model: `EdgeBundlingData[id]` → `NodeBridge[nodeId]` (many-to-one, single direction, active). All other filtering is handled purely through DAX measures.

---

## Power Query Queries

### Query 1: TreeNodes (staging — not loaded)

Builds the hierarchical tree structure from `DISCOVER_CALC_DEPENDENCY`. Extracts all unique objects (measures, calculated columns, columns, partitions) as leaf nodes, groups them under table nodes, and adds a single root node (`Model`).

Each node gets an `id` in the format `Model.TableName.ObjectName`, a `parent` reference (`Model.TableName` for leaves, `Model` for tables), a display `name`, and a `nodeType` (measure, calc_column, column, partition, table, root).

```powerquery-m
let
    Source = DISCOVER_CALC_DEPENDENCY,

    SourceObjects = Table.SelectRows(Source, each
        [OBJECT_TYPE] = "MEASURE" or [OBJECT_TYPE] = "CALC_COLUMN"),
    SourceLeaves = Table.SelectColumns(SourceObjects, {"TABLE", "OBJECT", "OBJECT_TYPE"}),
    SourceRenamed = Table.RenameColumns(SourceLeaves, {
        {"TABLE", "tableName"}, {"OBJECT", "objectName"}, {"OBJECT_TYPE", "rawType"}}),
    SourceDistinct = Table.Distinct(SourceRenamed),

    ReferencedObjects = Table.SelectRows(Source, each [REFERENCED_OBJECT_TYPE] <> "TABLE"),
    ReferencedLeaves = Table.SelectColumns(ReferencedObjects, {
        "REFERENCED_TABLE", "REFERENCED_OBJECT", "REFERENCED_OBJECT_TYPE"}),
    ReferencedRenamed = Table.RenameColumns(ReferencedLeaves, {
        {"REFERENCED_TABLE", "tableName"}, {"REFERENCED_OBJECT", "objectName"},
        {"REFERENCED_OBJECT_TYPE", "rawType"}}),
    ReferencedDistinct = Table.Distinct(ReferencedRenamed),

    AllLeaves = Table.Distinct(Table.Combine({SourceDistinct, ReferencedDistinct})),

    AddNodeType = Table.AddColumn(AllLeaves, "nodeType", each
        if [rawType] = "MEASURE" then "measure"
        else if [rawType] = "CALC_COLUMN" then "calc_column"
        else if [rawType] = "COLUMN" then "column"
        else if [rawType] = "PARTITION" then "partition"
        else "column"
    , type text),

    AddId = Table.AddColumn(AddNodeType, "id", each
        "Model." & [tableName] & "." & [objectName], type text),
    AddParent = Table.AddColumn(AddId, "parent", each "Model." & [tableName], type text),
    AddName = Table.AddColumn(AddParent, "name", each [objectName], type text),
    LeafNodes = Table.SelectColumns(AddName, {"id", "name", "parent", "nodeType"}),

    TablesSource = Table.SelectColumns(
        Table.Distinct(Table.SelectColumns(Source, {"TABLE"})), {"TABLE"}),
    TablesRef = Table.Distinct(Table.SelectColumns(Source, {"REFERENCED_TABLE"})),
    TablesRefRenamed = Table.RenameColumns(TablesRef, {{"REFERENCED_TABLE", "TABLE"}}),
    AllTables = Table.Distinct(Table.Combine({TablesSource, TablesRefRenamed})),

    TableNodes = Table.AddColumn(
        Table.AddColumn(
            Table.AddColumn(
                Table.AddColumn(AllTables, "id", each
                    "Model." & [TABLE], type text),
                "name", each [TABLE], type text),
            "parent", each "Model", type text),
        "nodeType", each "table", type text),
    TableNodesFinal = Table.SelectColumns(TableNodes, {"id", "name", "parent", "nodeType"}),

    RootNode = #table(
        type table [id = text, name = text, parent = text, nodeType = text],
        {{"Model", "Model", null, "root"}}
    ),

    Result = Table.Combine({RootNode, TableNodesFinal, LeafNodes})
in
    Result
```

### Query 2: Dependencies (staging — not loaded)

Extracts all dependency edges. Measure and calculated column dependencies come directly from `DISCOVER_CALC_DEPENDENCY`. Relationship dependencies are constructed by grouping `ACTIVE_RELATIONSHIP` rows by their GUID (the `OBJECT` column) and pairing the two referenced columns.

```powerquery-m
let
    Source = DISCOVER_CALC_DEPENDENCY,

    MeasureAndCalcDeps = Table.SelectRows(Source, each
        ([OBJECT_TYPE] = "MEASURE" or [OBJECT_TYPE] = "CALC_COLUMN")
        and [REFERENCED_OBJECT_TYPE] <> "TABLE"
    ),
    AddSource_A = Table.AddColumn(MeasureAndCalcDeps, "source", each
        "Model." & [TABLE] & "." & [OBJECT], type text),
    AddTarget_A = Table.AddColumn(AddSource_A, "target", each
        "Model." & [REFERENCED_TABLE] & "." & [REFERENCED_OBJECT], type text),
    AddDepType_A = Table.AddColumn(AddTarget_A, "depType", each
        if [OBJECT_TYPE] = "MEASURE" then "measure_dep"
        else "calc_column_dep"
    , type text),
    PartA = Table.SelectColumns(AddDepType_A, {"source", "target", "depType"}),

    Relationships = Table.SelectRows(Source, each [OBJECT_TYPE] = "ACTIVE_RELATIONSHIP"),
    Grouped = Table.Group(Relationships, {"OBJECT"}, {
        {"Rows", each _, type table}
    }),
    AddEdge = Table.AddColumn(Grouped, "edge", each
        let
            rows = [Rows],
            row1 = rows{0},
            row2 = rows{1},
            src = "Model." & row1[REFERENCED_TABLE] & "." & row1[REFERENCED_OBJECT],
            tgt = "Model." & row2[REFERENCED_TABLE] & "." & row2[REFERENCED_OBJECT]
        in
            [source = src, target = tgt, depType = "relationship"]
    ),
    ExpandEdge = Table.ExpandRecordColumn(AddEdge, "edge", {"source", "target", "depType"}),
    PartB = Table.SelectColumns(ExpandEdge, {"source", "target", "depType"}),

    Result = Table.Combine({PartA, PartB})
in
    Result
```

### Query 3: EdgeBundlingData (loaded — Deneb dataset)

Appends `TreeNodes` and `Dependencies` into a single flat table with a unified schema. Tree node rows have null `source`/`target`/`depType`; dependency rows have null `id`/`name`/`parent`/`nodeType`. Vega separates them internally using `filter` transforms.

```powerquery-m
let
    Tree = TreeNodes,
    Deps = Dependencies,

    TreeWithBlanks = Table.AddColumn(
        Table.AddColumn(
            Table.AddColumn(Tree, "source", each null, type text),
            "target", each null, type text),
        "depType", each null, type text),

    DepsWithBlanks = Table.AddColumn(
        Table.AddColumn(
            Table.AddColumn(
                Table.AddColumn(Deps, "id", each null, type text),
                "name", each null, type text),
            "parent", each null, type text),
        "nodeType", each null, type text),

    TreeFinal = Table.SelectColumns(TreeWithBlanks, {
        "id", "name", "parent", "nodeType", "source", "target", "depType"}),
    DepsFinal = Table.SelectColumns(DepsWithBlanks, {
        "id", "name", "parent", "nodeType", "source", "target", "depType"}),

    Result = Table.Combine({TreeFinal, DepsFinal})
in
    Result
```

### Query 4: DepTypeFilter (loaded — disconnected slicer)

Three rows: `relationship`, `measure_dep`, `calc_column_dep` — each with a friendly label for the slicer display. No relationships to any other table.

```powerquery-m
let
    Source = Dependencies,
    DistinctTypes = Table.Distinct(Table.SelectColumns(Source, {"depType"})),
    AddLabel = Table.AddColumn(DistinctTypes, "depTypeLabel", each
        if [depType] = "relationship" then "Relationship"
        else if [depType] = "measure_dep" then "Measure Dependency"
        else if [depType] = "calc_column_dep" then "Calc Column Dependency"
        else [depType]
    , type text)
in
    AddLabel
```

### Query 5: TableFilter (loaded — disconnected hierarchical slicer)

Distinct leaf nodes with `tableName` and `objectName` columns for a hierarchical slicer (table → object). No relationships to any other table.

```powerquery-m
let
    Source = TreeNodes,
    LeavesOnly = Table.SelectRows(Source, each
        [nodeType] <> "root" and [nodeType] <> "table"),
    AddTableName = Table.AddColumn(LeavesOnly, "tableName", each
        Text.AfterDelimiter([parent], "Model."), type text),
    AddObjectName = Table.AddColumn(AddTableName, "objectName", each [name], type text),
    Result = Table.SelectColumns(AddObjectName, {"tableName", "objectName"})
in
    Result
```

### Query 6: DependencyDetails (loaded — detail table)

Full detail table for the cross-filter drill-through. Combines measure/calc column dependencies with relationship dependencies. Relationship rows are enriched with cardinality and cross-filter direction metadata from `TMSCHEMA_RELATIONSHIPS`.

```powerquery-m
let
    Source = DISCOVER_CALC_DEPENDENCY,

    // PART A: Measure & Calc Column dependencies
    MeasureAndCalcDeps = Table.SelectRows(Source, each
        ([OBJECT_TYPE] = "MEASURE" or [OBJECT_TYPE] = "CALC_COLUMN")
        and [REFERENCED_OBJECT_TYPE] <> "TABLE"
    ),
    AddFields_A = Table.AddColumn(
        Table.AddColumn(
            Table.AddColumn(
                Table.AddColumn(
                    Table.AddColumn(MeasureAndCalcDeps, "sourceId", each
                        "Model." & [TABLE] & "." & [OBJECT], type text),
                    "targetId", each
                        "Model." & [REFERENCED_TABLE] & "." & [REFERENCED_OBJECT], type text),
                "depType", each
                    if [OBJECT_TYPE] = "MEASURE" then "measure_dep"
                    else "calc_column_dep", type text),
            "depTypeLabel", each
                if [OBJECT_TYPE] = "MEASURE" then "Measure Dependency"
                else "Calc Column Dependency", type text),
        "relInfo", each null, type text),
    PartA = Table.SelectColumns(AddFields_A, {
        "TABLE", "OBJECT", "OBJECT_TYPE",
        "REFERENCED_TABLE", "REFERENCED_OBJECT", "REFERENCED_OBJECT_TYPE",
        "sourceId", "targetId", "depType", "depTypeLabel", "relInfo"}),

    // PART B: Relationship dependencies
    Relationships = Table.SelectRows(Source, each
        [OBJECT_TYPE] = "ACTIVE_RELATIONSHIP"),
    Grouped = Table.Group(Relationships, {"OBJECT"}, {
        {"Rows", each _, type table}}),
    AddGUID = Table.AddColumn(Grouped, "relGUID", each [OBJECT], type text),
    DropGroupedObject = Table.RemoveColumns(AddGUID, {"OBJECT"}),
    BuildRelRows = Table.AddColumn(DropGroupedObject, "relRow", each
        let
            rows = [Rows],
            row1 = rows{0},
            row2 = rows{1},
            src = "Model." & row1[REFERENCED_TABLE] & "." & row1[REFERENCED_OBJECT],
            tgt = "Model." & row2[REFERENCED_TABLE] & "." & row2[REFERENCED_OBJECT]
        in
            [
                TABLE = row1[REFERENCED_TABLE],
                OBJECT = row1[REFERENCED_OBJECT],
                OBJECT_TYPE = "ACTIVE_RELATIONSHIP",
                REFERENCED_TABLE = row2[REFERENCED_TABLE],
                REFERENCED_OBJECT = row2[REFERENCED_OBJECT],
                REFERENCED_OBJECT_TYPE = "COLUMN",
                sourceId = src,
                targetId = tgt,
                depType = "relationship",
                depTypeLabel = "Relationship"
            ]),
    ExpandRel = Table.ExpandRecordColumn(BuildRelRows, "relRow", {
        "TABLE", "OBJECT", "OBJECT_TYPE",
        "REFERENCED_TABLE", "REFERENCED_OBJECT", "REFERENCED_OBJECT_TYPE",
        "sourceId", "targetId", "depType", "depTypeLabel"}),
    CleanRel = Table.RemoveColumns(ExpandRel, {"Rows"}),

    // PART C: Merge with TMSCHEMA_RELATIONSHIPS for metadata
    MergedRel = Table.NestedJoin(
        CleanRel, {"relGUID"},
        TMSCHEMA_RELATIONSHIPS, {"Name"},
        "RelMeta", JoinKind.LeftOuter),
    ExpandRelMeta = Table.ExpandTableColumn(MergedRel, "RelMeta", {
        "IsActive", "CrossFilteringBehavior", "FromCardinality", "ToCardinality"}),
    AddRelInfo = Table.AddColumn(ExpandRelMeta, "relInfo", each
        let
            fromCard = if [FromCardinality] = 2 then "*" else "1",
            toCard   = if [ToCardinality] = 2 then "*" else "1",
            dir      = if [CrossFilteringBehavior] = 2 then "↔" else "→",
            active   = if [IsActive] = true then "" else " (inactive)"
        in
            fromCard & " " & dir & " " & toCard & active
    , type text),
    PartB = Table.RemoveColumns(AddRelInfo, {
        "relGUID", "IsActive", "CrossFilteringBehavior",
        "FromCardinality", "ToCardinality"}),

    // PART D: Combine
    Result = Table.Combine({PartA, PartB}),
    AddSourceTable = Table.AddColumn(Result, "sourceTable", each
        Text.BetweenDelimiters([sourceId], ".", ".", 0, 0), type text),
    AddTargetTable = Table.AddColumn(AddSourceTable, "targetTable", each
        Text.BetweenDelimiters([targetId], ".", ".", 0, 0), type text),
    Final = Table.SelectColumns(AddTargetTable, {
        "depType", "depTypeLabel", "relInfo",
        "TABLE", "OBJECT", "OBJECT_TYPE",
        "REFERENCED_TABLE", "REFERENCED_OBJECT", "REFERENCED_OBJECT_TYPE",
        "sourceId", "targetId", "sourceTable", "targetTable"})
in
    Final
```

### Query 7: NodeBridge (loaded — cross-filter bridge)

Distinct leaf node IDs used to bridge the cross-filter from `EdgeBundlingData` to `DependencyDetails`. Excludes Power BI auto-generated internal date tables and removes duplicates that would break the many-to-one relationship.

```powerquery-m
let
    Source = TreeNodes,
    LeavesOnly = Table.SelectRows(Source, each
        [nodeType] <> "root" and [nodeType] <> "table"),
    ExcludeInternal = Table.SelectRows(LeavesOnly, each
        not Text.Contains([id], "DateTableTemplate_")
        and not Text.Contains([id], "LocalDateTable_")),
    Result = Table.SelectColumns(ExcludeInternal, {"id"}),
    #"Removed Duplicates" = Table.Distinct(Result),
    Renamed = Table.RenameColumns(#"Removed Duplicates", {{"id", "nodeId"}})
in
    Renamed
```

---

## DAX Measures

### SelectedDepTypes

Reads the disconnected `DepTypeFilter` slicer and passes the selection into the Deneb dataset as a comma-separated string. Returns `"ALL"` when no slicer selection is active.

```dax
SelectedDepTypes =
IF(
    NOT ISFILTERED(DepTypeFilter[depTypeLabel]),
    "ALL",
    CONCATENATEX(VALUES(DepTypeFilter[depType]), [depType], ",")
)
```

### SelectedTables

Same pattern for the table slicer.

```dax
SelectedTables =
IF(
    NOT ISFILTERED(TableFilter[tableName]),
    "ALL",
    CONCATENATEX(VALUES(TableFilter[tableName]), [tableName], ",")
)
```

### SelectedObjects

Same pattern for the object-level slicer.

```dax
SelectedObjects =
IF(
    NOT ISFILTERED(TableFilter[objectName]),
    "ALL",
    CONCATENATEX(VALUES(TableFilter[objectName]), [objectName], ",")
)
```

### ShowDetailRow

Evaluates per row in `DependencyDetails` to determine visibility. Handles three filter dimensions simultaneously: dependency type slicer, table slicer, and Deneb cross-filter (clicked node). Uses `MAX()` instead of `SELECTEDVALUE()` because the measure evaluates in the table visual's row context.

```dax
ShowDetailRow =
VAR _depFilter =
    IF(
        NOT ISFILTERED(DepTypeFilter[depTypeLabel]),
        TRUE(),
        MAX(DependencyDetails[depType]) IN VALUES(DepTypeFilter[depType])
    )
VAR _tableFilter =
    IF(
        NOT ISFILTERED(TableFilter[tableName]),
        TRUE(),
        MAX(DependencyDetails[sourceTable]) IN VALUES(TableFilter[tableName])
        || MAX(DependencyDetails[targetTable]) IN VALUES(TableFilter[tableName])
    )
VAR _selectedNode = SELECTEDVALUE(NodeBridge[nodeId])
VAR _visualFilter =
    IF(
        ISBLANK(_selectedNode),
        TRUE(),
        MAX(DependencyDetails[sourceId]) = _selectedNode
        || MAX(DependencyDetails[targetId]) = _selectedNode
    )
RETURN
IF(_depFilter && _tableFilter && _visualFilter, 1, 0)
```

---

## Model Relationships

Only **one relationship** exists:

| From | To | Cardinality | Direction | Active |
|------|----|-------------|-----------|--------|
| `EdgeBundlingData[id]` | `NodeBridge[nodeId]` | Many-to-one | Single | Yes |

**Delete all auto-created relationships** involving `DepTypeFilter`, `TableFilter`, `DependencyDetails`, `EdgeBundlingData`, and `NodeBridge`. Auto-created relationships will break the disconnected slicer pattern (symptoms include `(Blank)` entries appearing in slicers).

---

## Deneb Configuration

### Values Well

**Columns** (from EdgeBundlingData): `id`, `name`, `parent`, `nodeType`, `source`, `target`, `depType`

**Measures**: `SelectedDepTypes`, `SelectedTables`, `SelectedObjects`

### Settings

| Setting | Value |
|---------|-------|
| Provider | Vega |
| Render mode | SVG |
| Tooltip handler | ✅ |
| Resolve data points in context menu | ✅ |
| Expose cross-highlight values for measures | ☐ |
| Expose cross-filtering values for dataset rows | ✅ |
| Cross-filtering management | **Advanced** |

### Config Tab

```json
{ "autoSize": false }
```

---

## Vega Specification (v12)

The complete Vega spec is in `deneb-specification-v12.json`. Key architectural decisions in the spec:

### Data Pipeline Inside Vega

The flat `dataset` from Power BI is split inside Vega:

1. **`nodes`** — filtered to rows where `id != null` (tree nodes only)
2. **`edges`** — filtered to rows where `source != null` (dependency edges only)
3. **`tree`** — `stratify` transform builds a tree from nodes using `id`/`parent`
4. **`treeLayout`** — `tree` layout computes radial x/y positions (cluster layout, 360° extent)
5. **`leaves`** — filtered to nodes with no children (the visible labels)
6. **`dependencies`** — `treePath('tree', source, target)` computes bundled paths for each edge, with `"initonly": true` to survive Deneb re-initialization
7. **`selected`** — edges matching the current slicer selection (for opacity control)
8. Hover-derived datasets: `hoverOut`, `hoverIn`, `connectedNodes`, `hoverOutCount`, `hoverInCount`

### Signals

Signals bridge three worlds: Power BI slicers → Vega reactivity → visual encoding.

- **`selectedDepTypes` / `selectedTables` / `selectedObjects`** — extracted from DAX measure columns using `pluck(data('dataset'), 'MeasureName')[0]`
- **`active`** — tracks the currently hovered node ID
- **`pbiCrossFilterSelection`** — handles click-to-cross-filter using `pbiCrossFilterApply()` and `pbiCrossFilterClear()`
- **`tension`** — user-controllable bundle tightness (0 = straight lines, 1 = fully bundled)

### Opacity Strategy

Three-layer cascading opacity controls visual state:

| State | Edge Opacity | Node Opacity |
|-------|-------------|--------------|
| Hovered/connected | 1.0 | 1.0 |
| Slicer match (no hover) | 0.2 | 1.0 |
| Non-matching (slicer active) | 0.03 | 0.15 |
| No slicer, no hover | 0.2 | 1.0 |

### Cross-Filtering Events

Advanced mode uses explicit event handlers:

- **Click on leaf label**: `pbiCrossFilterApply(event, 'datum.id == ...')` — filters the detail table via the NodeBridge relationship
- **Click on empty canvas**: `pbiCrossFilterClear()` — removes the filter
- The clear handler includes a filter `"!event.item || event.item.mark.name != 'leafLabels'"` because Vega doesn't prevent event propagation from marks to the view

---

## Setup Guide

### Step 1: Configure the Connection Parameters

The PBIX includes two Power Query parameters that you need to set for your environment:

- **`theAnalysisServiceServer`** — your workspace XMLA endpoint (e.g., `powerbi://api.powerbi.com/v1.0/myorg/YourWorkspace`)
- **`theDatabase`** — the name of your semantic model

These parameters are already referenced by the two DMV queries (`DISCOVER_CALC_DEPENDENCY` and `TMSCHEMA_RELATIONSHIPS`). Update them in Power Query under **Manage Parameters** before refreshing.

### Step 2: Create the Power Query Queries

Create all 7 queries as documented above. `TreeNodes` and `Dependencies` should be staging queries (connection only, not loaded). The other 5 are loaded.

### Step 3: Delete Auto-Created Relationships

In Model view, delete every auto-created relationship. Power BI will try to create relationships between tables that share column names — these must all be removed.

### Step 4: Create the Single Relationship

Create: `EdgeBundlingData[id]` → `NodeBridge[nodeId]` (many-to-one, single direction, active).

### Step 5: Create DAX Measures

Add all four measures (`SelectedDepTypes`, `SelectedTables`, `SelectedObjects`, `ShowDetailRow`) to the `EdgeBundlingData` table or a dedicated measures table.

### Step 6: Add the Deneb Visual

Add a Deneb visual to your report page. Configure the Values well with the 7 columns and 3 measures listed above. Paste the v12 Vega spec. Set Deneb config to `{ "autoSize": false }`. Configure all Deneb settings as documented.

### Step 7: Add Slicers

- **Dependency type slicer**: field = `DepTypeFilter[depTypeLabel]`
- **Table/Object slicer** (hierarchical): fields = `TableFilter[tableName]`, then `TableFilter[objectName]`

### Step 8: Add the Detail Table Visual

Add a Table visual with columns from `DependencyDetails`: `depTypeLabel`, `TABLE`, `OBJECT`, `REFERENCED_TABLE`, `REFERENCED_OBJECT`, `relInfo`. Apply a visual-level filter: `ShowDetailRow = 1`.

---

## Key Technical Learnings

### treePath() Requires Raw String IDs

Vega's `treePath('tree', source, target)` returns `null` when passed Vega tuple objects from `lookup` transforms. The tuples carry internal `_id` properties that don't match stratify keys. Always pass raw string fields (`datum.source`, `datum.target`) directly.

### treePath() Survives Deneb Re-initialization

Deneb re-initializes the entire Vega view when cross-filtering is applied. The `"initonly": true` flag protects the `treePath()` computation — edges remain intact after click interactions.

### Cross-Filtering Requires the Relationship Graph

Power BI cross-filtering propagates through the data model's relationship graph. Without a relationship path between `EdgeBundlingData` and `DependencyDetails`, cross-filter events have no effect. The `NodeBridge` table provides this path with minimal model impact.

### DAX Measures as Vega Signals

Measures in Deneb's Values well appear as constant columns in the dataset. `pluck(data('dataset'), 'MeasureName')[0]` extracts them into Vega signals that update reactively when slicers change.

### SELECTEDVALUE vs MAX in Row Context

`SELECTEDVALUE` doesn't work in a table visual's row context because each row contains one value per column, making it ambiguous. Use `MAX()` instead for columns in `DependencyDetails` when building the `ShowDetailRow` measure.

### Auto-Generated Date Tables

Power BI creates internal date hierarchy tables with GUID-based names (e.g., `DateTableTemplate_ec992640-...`). These produce duplicate node IDs and must be filtered out in the `NodeBridge` query (and optionally in `TreeNodes` and `Dependencies`).

---

## Spec Version History

| Version | Changes |
|---------|---------|
| v1–v4 | Initial attempts, treePath() debugging |
| v5 | First working version — 123 edges rendered |
| v6 | Dependency type coloring (blue/red/orange) |
| v7 | Disconnected slicer filtering via opacity |
| v8 | Node dimming for unaffected nodes |
| v9 | Table arc marks attempt (parked) |
| v10 | Hover info panel with node details and edge counts |
| v11 | Responsive layout, enriched hover legend, dep type legend |
| v12 | Cross-filtering to detail table via NodeBridge and Advanced mode |

---

## Roadmap

- **Table arc marks**: Sunburst-style inner ring grouping nodes by table (parked — angle calculation issue)
- **FUAM scale-up**: Scale from 1 model to ~23,000 semantic models via FUAM lakehouse
- **Python/Altair version**: Community-shareable version that doesn't require XMLA/Premium
- **Richer metadata**: Integration with `TMSCHEMA_COLUMNS` / `TMSCHEMA_TABLES`
