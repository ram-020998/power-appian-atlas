---
name: "appian-atlas"
displayName: "Appian Atlas"
description: "Universal knowledge base for Appian applications. Browse application bundles, trace dependencies, search objects, and understand architecture across all Appian solutions."
keywords: ["appian", "sail", "record type", "process model", "expression rule", "interface", "integration", "web api", "connected system", "cdt", "appian bundle", "appian dependency", "appian parser", "low-code"]
---

# Onboarding

## Step 1: Validate knowledge base access
Call the `list_applications` tool to verify available Appian applications. If no applications are found, an Appian application package needs to be parsed first:

```bash
cd /path/to/gam-appian-knowledge-base
source .venv/bin/activate
python -m appian_parser dump <package.zip> ./data/<AppName>
```

## Step 2: Understand the knowledge base
This is a universal repository for Appian application documentation. Each parsed application contains:

### Core Files (Single-Fetch Orientation)
- **app_overview.json** (~100-150KB) — Package metadata, bundle index, dependency summary, coverage stats, enrichment metadata — everything needed to navigate the app in ONE call
- **search_index.json** (~200KB) — Flat object name lookup: `name → {type, uuid, bundles[], deps_out, deps_in}` — instant search, no scanning

### Enrichment Data (NEW)
- **enrichment/metadata.json** (~100B) — Enrichment summary: total objects, objects with depth, version
- **enrichment/object_depths.json** (~50-100KB) — Dependency depth for all objects: `uuid → depth`
- **enrichment/object_enrichments.json** (~300-500KB) — Full enrichment: depth, tags, statistics per object

### Bundle Structure (Layered Detail)
- **bundles/\<bundle_id\>/structure.json** (5-50KB) — Flow structure, object relationships, NO code — fast preview
- **bundles/\<bundle_id\>/code.json** (50KB-2MB) — SAIL code only, keyed by UUID — loaded on demand when you need implementation details

### Per-Object Data
- **objects/\<uuid\>.json** (1-5KB) — Individual object dependencies: `{calls[], called_by[], bundles[]}` — no need to load monolithic dependency file
- **orphans/_index.json** (~50KB) — Catalog of unbundled objects by type
- **orphans/\<uuid\>.json** (1-10KB) — Per-orphan detail with code

Bundle types:
| Type | Entry Point | Contents |
|---|---|---|
| action | Record Type Action | Action → process model → form interface → all deps |
| process | Standalone Process Model | PM → subprocesses → interfaces → deps |
| page | Record Type Views | Summary/detail views → interfaces → supporting objects |
| site | Site | Navigation → all page targets → interfaces |
| dashboard | Control Panel | Dashboard → interfaces → record types |
| web_api | Web API | Endpoint → all called rules/integrations |

## Recommended workflow
1. `list_applications` → see all Appian apps
2. `get_app_overview(app)` → **ONE call** gets full map: bundles, deps, object counts, coverage
3. `search_objects(app, name)` → instant lookup via search_index.json (cached after first use)
4. `get_bundle(app, bundle_id, "summary")` → loads structure.json only (5-50KB) — flow + relationships, no code
5. `get_bundle(app, bundle_id, "full")` → loads structure.json + code.json — complete implementation
6. `get_object_detail(app, uuid)` → loads objects/\<uuid\>.json — full dependency info for one object
7. `list_orphans(app)` → browse unbundled objects by type
8. `get_orphan(app, uuid)` → load individual orphan with code

## Key Improvements Over Previous Structure
- **Fewer round-trips**: `app_overview.json` replaces 3-4 separate file fetches
- **Smaller payloads**: Structure files are 5-50KB vs 2MB+ monolithic bundles
- **Faster search**: Direct name lookup vs scanning 364KB manifest
- **On-demand code**: Only load SAIL code when you need it
- **Per-object deps**: 1-5KB targeted files vs 4.6MB monolithic dependency graph

# MCP Tool Reference

## Tool: `list_applications`
**Purpose**: Discover all available Appian applications with high-level stats.

**Returns**: Array of applications with:
- `name`: Application folder name
- `total_objects`: Count of parsed objects
- `total_errors`: Parse error count
- `bundle_coverage`: `{total_objects, bundled, orphaned}`
- `bundles_by_type`: Count per bundle type

**When to use**: First call to see what's available.

**Example**:
```
User: "What Appian applications are available?"
→ Call list_applications()
→ Returns: [{"name": "SourceSelection", "total_objects": 2327, ...}]
```

---

## Tool: `get_app_overview`
**Purpose**: Get complete application map in ONE call — replaces multiple file fetches.

**Args**:
- `app_name` (required): Application name from `list_applications`

**Returns**: Single JSON with:
- `_metadata`: Parser version, generation timestamp
- `package_info`: Filename, object counts, error count
- `object_counts`: Count per object type (Interface, Expression Rule, etc.)
- `bundles[]`: Array of all bundles with:
  - `id`: Bundle identifier (use this for `get_bundle`)
  - `bundle_type`: action | process | page | site | dashboard | web_api
  - `root_name`: Human-readable name
  - `parent_name`: Parent object (e.g. Record Type for actions)
  - `object_count`: Total objects in bundle
  - `key_objects[]`: Top 5 most-connected objects
- `dependency_summary`: Top depended-on objects, dependency type breakdown
- `coverage`: Bundled vs orphaned object counts

**When to use**: After selecting an app, before drilling into specifics. This gives you the full map.

**Example**:
```
User: "Give me an overview of SourceSelection"
→ Call get_app_overview("SourceSelection")
→ Returns: {package_info: {...}, bundles: [...], dependency_summary: {...}}
```

---

## Tool: `search_objects`
**Purpose**: Fast object name lookup using the search index (cached after first use).

**Args**:
- `app_name` (required): Application name
- `query` (required): Case-insensitive substring to match
- `object_type` (optional): Filter by type (e.g. "Interface", "Expression Rule", "Process Model")

**Returns**: Array of matching objects (max 50) with:
- `name`: Object name
- `uuid`: Object UUID
- `type`: Object type
- `description`: Object description (may be null)
- `bundles[]`: Bundle IDs containing this object
- `deps_out`: Outbound dependency count
- `deps_in`: Inbound dependency count

**When to use**: Finding specific objects by name, checking if an object exists.

**Example**:
```
User: "Find the addVendors interface"
→ Call search_objects("SourceSelection", "addVendors", "Interface")
→ Returns: [{"name": "AS_GSS_FM_addVendors", "type": "Interface", ...}]
```

---

## Tool: `search_bundles`
**Purpose**: Find bundles by name or parent name.

**Args**:
- `app_name` (required): Application name
- `query` (required): Case-insensitive substring to match against bundle root_name or parent_name
- `bundle_type` (optional): Filter by type (action | process | page | site | dashboard | web_api)

**Returns**: Array of matching bundles (same structure as in `app_overview.bundles[]`)

**When to use**: Finding specific functionality without browsing the full bundle list.

**Example**:
```
User: "Show me evaluation-related actions in SourceSelection"
→ Call search_bundles("SourceSelection", "evaluation", "action")
→ Returns: [{id: "action__SourceSelection__Complete_Evaluation", ...}]
```

---

## Tool: `get_bundle`
**Purpose**: Load bundle content at the requested detail level.

**Args**:
- `app_name` (required): Application name
- `bundle_id` (required): Bundle ID from `search_bundles` or `get_app_overview`
  - Can also use `root_name` with spaces — auto-resolved
- `detail_level` (optional, default "summary"):
  - `"summary"`: Metadata + entry_point + flow + object names only (fastest, ~5KB)
  - `"structure"`: Full structure.json — flow + relationships, NO code (~5-50KB)
  - `"full"`: Structure + SAIL code merged (~50KB-2MB)

**Returns**:
- **summary**: `{_metadata, entry_point, flow, objects: [{name, type, description}]}`
- **structure**: Full structure.json with:
  - `_metadata`: Bundle type, root name, object count
  - `entry_point`: Type-specific entry point details
  - `flow`: Process flow graph (if applicable, else null)
  - `objects[]`: Full object metadata with `{uuid, name, type, description, parameters, calls[], called_by[]}`
- **full**: Same as structure but each object has `sail_code` field added

**When to use**:
- Start with `"summary"` to preview
- Use `"structure"` to understand flow and relationships without loading code
- Use `"full"` only when you need to see implementation details

**Example**:
```
User: "How does the Add Vendors action work?"
→ Call get_bundle("SourceSelection", "action__SourceSelection__Add_Vendors", "structure")
→ Returns: {entry_point: {action_type: "RECORD_ACTION", target_process: "AS_GSS_PM_addVendors"}, flow: {...}, objects: [...]}

User: "Show me the SAIL code for that form"
→ Call get_bundle("SourceSelection", "action__SourceSelection__Add_Vendors", "full")
→ Returns: Same structure but objects now include sail_code
```

---

## Tool: `get_dependencies`
**Purpose**: Get dependency graph for a specific object by name.

**Args**:
- `app_name` (required): Application name
- `object_name` (required): Case-insensitive object name

**Returns**: Object dependency data:
- `uuid`: Object UUID
- `name`: Object name
- `type`: Object type
- `description`: Object description
- `bundles[]`: Bundle IDs containing this object
- `calls[]`: Outbound dependencies `[{name, type, dep_type}]`
- `called_by[]`: Inbound dependencies `[{name, type, dep_type}]`

**When to use**: Tracing what an object depends on or what depends on it.

**Example**:
```
User: "What depends on AS_GSS_BL_validateVendors?"
→ Call get_dependencies("SourceSelection", "AS_GSS_BL_validateVendors")
→ Returns: {calls: [...], called_by: [{name: "AS_GSS_FM_addVendors", type: "Interface"}]}
```

---

## Tool: `get_object_detail`
**Purpose**: Get dependency and bundle info for an object by UUID (faster than name lookup).

**Args**:
- `app_name` (required): Application name
- `object_uuid` (required): Object UUID

**Returns**: Same structure as `get_dependencies`

**When to use**: When you already have the UUID (e.g. from search results or bundle objects).

---

## Tool: `list_orphans`
**Purpose**: Browse objects not reachable from any entry point (unbundled objects).

**Args**:
- `app_name` (required): Application name

**Returns**:
- `_metadata`: Description, total orphan count
- `by_type`: `{type: [{uuid, name}]}` — orphans grouped by object type

**When to use**: Finding legacy/unused objects, understanding coverage gaps.

**Example**:
```
User: "Are there any unused expression rules in SourceSelection?"
→ Call list_orphans("SourceSelection")
→ Returns: {by_type: {"Expression Rule": [{uuid: "...", name: "AS_GSS_BL_legacyHelper"}]}}
```

---

## Tool: `get_orphan`
**Purpose**: Get full detail (including code) for an orphaned object.

**Args**:
- `app_name` (required): Application name
- `object_uuid` (required): Orphan UUID from `list_orphans`

**Returns**:
- `uuid`: Object UUID
- `name`: Object name
- `type`: Object type
- `description`: Object description
- `sail_code`: SAIL code (if applicable)
- `calls[]`: Outbound dependencies
- `called_by[]`: Inbound dependencies (typically empty for orphans)

**When to use**: Inspecting specific orphaned objects to understand why they're not bundled.

---

## Tool: `get_enrichment_metadata`
**Purpose**: Get enrichment summary statistics for an application.

**Args**:
- `app_name` (required): Application name

**Returns**:
- `total_objects`: Total objects enriched
- `objects_with_depth`: Objects with calculated dependency depth
- `enrichment_version`: Enrichment layer version

**When to use**: Check if enrichment data is available and get summary statistics.

**Example**:
```
User: "Is enrichment data available for SourceSelection?"
→ Call get_enrichment_metadata("SourceSelection")
→ Returns: {total_objects: 2461, objects_with_depth: 2021, enrichment_version: "0.1.0"}
```

---

## Tool: `get_object_enrichment`
**Purpose**: Get enrichment data for a specific object (depth, tags, statistics).

**Args**:
- `app_name` (required): Application name
- `object_uuid` (required): Object UUID

**Returns**:
- `uuid`: Object UUID
- `dependency_depth`: Depth from entry points (0 = entry point, null = orphaned)
- `tags[]`: Classification tags (e.g. "dashboard", "approval_workflow", "read_only")
- `statistics`: Object statistics
  - `dependency_count`: Number of objects this depends on
  - `dependent_count`: Number of objects that depend on this
  - `complexity_score`: Complexity metric (for process models)
  - `node_count`: Number of nodes (for process models)

**Available tags**:
- Architecture: `record_driven`, `integration_heavy`, `subprocess_heavy`, `form_heavy`
- Workflow: `conditional_workflow`, `parallel_workflow`, `approval_workflow`, `linear_workflow`
- Data: `read_only`, `write_heavy`, `query_heavy`
- Complexity: `simple`, `moderate`, `complex`
- UI: `dashboard`, `form_interface`, `report`
- Integration: `has_integrations`, `has_web_api`, `has_database`

**When to use**: Understanding object characteristics, prioritizing by depth, filtering by tags.

**Example**:
```
User: "What's the dependency depth of AS_GSS_FM_addVendors?"
→ First: search_objects("SourceSelection", "addVendors") to get UUID
→ Then: get_object_enrichment("SourceSelection", uuid)
→ Returns: {dependency_depth: 0, tags: ["form_interface"], statistics: {...}}
```

---

## Tool: `get_dependency_depths`
**Purpose**: Get dependency depths for all objects, optionally filtered by max depth.

**Args**:
- `app_name` (required): Application name
- `max_depth` (optional): Maximum depth to include (e.g. 2 for entry points + first two levels)

**Returns**: Dictionary mapping UUID to depth: `{"uuid": depth}`

**When to use**: Understanding architecture layers, finding entry points, analyzing depth distribution.

**Example**:
```
User: "Show me the dependency depth distribution for SourceSelection"
→ Call get_dependency_depths("SourceSelection")
→ Returns: {"uuid-1": 0, "uuid-2": 1, "uuid-3": 2, ...}

User: "Show me only entry points and first-level dependencies"
→ Call get_dependency_depths("SourceSelection", max_depth=1)
→ Returns: {"uuid-1": 0, "uuid-2": 0, "uuid-3": 1, ...}
```

---

## Tool: `search_by_depth`
**Purpose**: Find all objects at a specific dependency depth.

**Args**:
- `app_name` (required): Application name
- `depth` (required): Dependency depth to search for (0 = entry points)

**Returns**: Array of objects (max 100) with:
- `name`: Object name
- `uuid`: Object UUID
- `type`: Object type
- `depth`: Dependency depth

**When to use**: Finding entry points (depth 0), exploring architecture layers, prioritizing context.

**Example**:
```
User: "What are the entry points in SourceSelection?"
→ Call search_by_depth("SourceSelection", 0)
→ Returns: [{name: "AS_GSS_FM_addVendors", type: "Interface", depth: 0}, ...]

User: "Show me objects at the second dependency level"
→ Call search_by_depth("SourceSelection", 2)
→ Returns: [{name: "AS_CO_UT_filterCdt", type: "Expression Rule", depth: 2}, ...]
```

---

## Tool: `search_by_tags`
**Purpose**: Find objects with specific classification tags.

**Args**:
- `app_name` (required): Application name
- `tags` (required): Array of tags to search for (objects must have ALL specified tags)

**Returns**: Array of objects (max 100) with:
- `name`: Object name
- `uuid`: Object UUID
- `type`: Object type
- `tags[]`: All tags for this object
- `depth`: Dependency depth (may be null)

**When to use**: Finding specific patterns (dashboards, approval workflows, etc.), filtering by characteristics.

**Example**:
```
User: "Find all read-only dashboards in SourceSelection"
→ Call search_by_tags("SourceSelection", ["dashboard", "read_only"])
→ Returns: [{name: "AS_GSS_Dashboard", type: "Interface", tags: ["dashboard", "read_only"], depth: 0}]

User: "Find all approval workflows"
→ Call search_by_tags("SourceSelection", ["approval_workflow"])
→ Returns: [{name: "AS_GSS_PM_approveVendor", type: "Process Model", tags: ["approval_workflow", "simple"], depth: 1}]

User: "Find complex integration-heavy processes"
→ Call search_by_tags("SourceSelection", ["integration_heavy", "complex"])
→ Returns: [...]
```

---

## Tool: `get_statistics` 🆕
**Purpose**: Get aggregated statistics without loading full data. Instant answers to "how many" questions.

**Args**:
- `app_name` (required): Application name
- `stat_type` (required): Type of statistics:
  - `tag_distribution` - Count of objects per classification tag
  - `depth_distribution` - Count of objects per dependency depth
  - `type_distribution` - Count of objects per type
  - `bundle_complexity` - Bundles sorted by object count
  - `object_reuse` - Objects sorted by dependent_count (most reused first)
  - `orphan_summary` - Orphan counts by type
- `filters` (optional): Filters like `{"limit": 10, "min_count": 5}`

**When to use**: Counting, distribution analysis, finding most/least complex bundles.

**Example**:
```
User: "How many approval workflows are there?"
→ Call get_statistics("SourceSelection", "tag_distribution")
→ Returns: {"tag_distribution": {"approval_workflow": 15, "dashboard": 8, ...}}

User: "What are the most complex bundles?"
→ Call get_statistics("SourceSelection", "bundle_complexity", {"limit": 5})
→ Returns: {"bundle_complexity": [{id: "...", root_name: "...", object_count: 637}, ...]}

User: "Which objects are most reused?"
→ Call get_statistics("SourceSelection", "object_reuse", {"limit": 10, "min_count": 5})
→ Returns: {"object_reuse": [{name: "...", type: "...", dependent_count: 42}, ...]}
```

---

## Tool: `batch_get` 🆕
**Purpose**: Get multiple objects/bundles in one call. Reduces N calls to 1.

**Args**:
- `app_name` (required): Application name
- `operation` (required): Type of batch operation:
  - `objects` - Get multiple object details by UUID
  - `bundles` - Get multiple bundles by ID
  - `enrichments` - Get enrichment data for multiple objects
  - `dependencies` - Get dependencies for multiple objects by name
- `identifiers` (required): List of UUIDs, bundle IDs, or object names
- `options` (optional): Settings like `{"detail_level": "summary"}`

**When to use**: Loading multiple objects for comparison, getting details for a list.

**Example**:
```
User: "Compare these three interfaces"
→ Call batch_get("SourceSelection", "objects", [uuid1, uuid2, uuid3])
→ Returns: [{uuid: "...", data: {...}}, ...]

User: "Load enrichment data for these objects"
→ Call batch_get("SourceSelection", "enrichments", [uuid1, uuid2, uuid3])
→ Returns: [{uuid: "...", data: {depth: 2, tags: [...]}}, ...]
```

---

## Tool: `smart_query` 🆕
**Purpose**: Common query patterns in one call. Combines search + load operations.

**Args**:
- `app_name` (required): Application name
- `query_type` (required): Type of smart query:
  - `find_and_load_bundle` - Search for bundle by name and load it
  - `find_and_get_object` - Search for object by name and get full details
  - `get_bundle_summary` - Get just bundle metadata without loading full data
  - `count_by_tag` - Count objects with specific tags
  - `most_reused` - Get top N most-reused objects with details
- `**params`: Query-specific parameters (query, bundle_type, detail_level, tags, limit, etc.)

**When to use**: Common find-and-load patterns, counting by tag, finding most reused objects.

**Example**:
```
User: "Show me the Add Vendor action"
→ Call smart_query("SourceSelection", "find_and_load_bundle", query="Add Vendor", detail_level="summary")
→ Returns: {search_results: [...], loaded_bundle: {...}, bundle_id: "..."}

User: "How many approval workflows?"
→ Call smart_query("SourceSelection", "count_by_tag", tags=["approval_workflow"])
→ Returns: {tags: ["approval_workflow"], count: 15, sample_objects: [...]}

User: "What are the top 5 most reused objects?"
→ Call smart_query("SourceSelection", "most_reused", limit=5)
→ Returns: {most_reused: [{name: "...", dependent_count: 42, depth: 3, tags: [...]}, ...]}
```

---

# Typical Query Patterns

## Pattern 1: Explore an application
```
1. list_applications() → see all apps
2. get_app_overview("AppName") → full map (includes enrichment metadata if available)
3. Review bundles[] and dependency_summary
4. get_enrichment_metadata("AppName") → check enrichment availability
```

## Pattern 2: Find and analyze specific functionality
```
1. search_bundles("AppName", "keyword") → find relevant bundles
2. get_bundle("AppName", bundle_id, "summary") → preview
3. get_bundle("AppName", bundle_id, "structure") → see flow + relationships
4. get_bundle("AppName", bundle_id, "full") → load code if needed
```

## Pattern 3: Trace dependencies
```
1. search_objects("AppName", "object_name") → find object
2. get_dependencies("AppName", "object_name") → see calls/called_by
3. get_object_enrichment("AppName", uuid) → get depth and tags
4. For each dependency, repeat steps 2-3 to build full graph
```

## Pattern 4: Impact analysis
```
1. get_dependencies("AppName", "shared_utility") → see called_by[]
2. For each caller, get_object_detail() to see which bundles use it
3. get_object_enrichment() for each to understand depth and characteristics
4. Assess impact scope across bundles
```

## Pattern 5: Find unused code
```
1. list_orphans("AppName") → see unbundled objects
2. Filter by_type for specific object types
3. get_orphan("AppName", uuid) → inspect individual orphans
```

## Pattern 6: Explore architecture layers (NEW)
```
1. get_enrichment_metadata("AppName") → verify enrichment available
2. search_by_depth("AppName", 0) → find all entry points
3. search_by_depth("AppName", 1) → find first-level dependencies
4. get_dependency_depths("AppName", max_depth=3) → get shallow objects only
```

## Pattern 7: Find specific patterns (NEW)
```
1. search_by_tags("AppName", ["dashboard"]) → find all dashboards
2. search_by_tags("AppName", ["approval_workflow"]) → find approval processes
3. search_by_tags("AppName", ["integration_heavy", "complex"]) → find complex integrations
4. For each result, get_bundle() or get_object_detail() for more info
```

## Pattern 8: Prioritize context by depth (NEW)
```
1. search_by_depth("AppName", 0) → start with entry points
2. For each entry point, get_dependencies() → see what it calls
3. search_by_depth("AppName", 1) → explore first level
4. Continue level by level, prioritizing shallow objects
```

## Pattern 9: Analyze object characteristics (NEW)
```
1. search_objects("AppName", "object_name") → find object
2. get_object_enrichment("AppName", uuid) → get depth, tags, statistics
3. Use depth to understand position in architecture
4. Use tags to understand patterns and characteristics
5. Use statistics to understand complexity and reuse
```

## Pattern 10: Quick statistics and counts (NEW - Phase 1) 🆕
```
1. get_statistics("AppName", "tag_distribution") → count objects by tag
2. get_statistics("AppName", "depth_distribution") → count objects by depth
3. get_statistics("AppName", "bundle_complexity") → find most complex bundles
4. get_statistics("AppName", "object_reuse") → find most reused objects
→ All instant, no need to load full data
```

## Pattern 11: Batch operations for efficiency (NEW - Phase 1) 🆕
```
1. Collect list of UUIDs/IDs to fetch
2. batch_get("AppName", "enrichments", [uuid1, uuid2, ...]) → get all enrichments in one call
3. batch_get("AppName", "objects", [uuid1, uuid2, ...]) → get all object details in one call
→ Reduces N calls to 1, much faster
```

## Pattern 12: Smart queries for common tasks (NEW - Phase 1) 🆕
```
1. smart_query("AppName", "find_and_load_bundle", query="keyword") → search + load in one call
2. smart_query("AppName", "count_by_tag", tags=["approval_workflow"]) → instant count
3. smart_query("AppName", "most_reused", limit=10) → top reused objects with enrichment
→ Combines multiple operations, fewer round trips
```

---

# Performance Tips

1. **Cache app_overview**: First call fetches from GitHub (~200ms), subsequent calls are instant (cached)
2. **Use summary first**: `get_bundle(..., "summary")` is 10-40x smaller than "full"
3. **Search before scanning**: `search_objects` and `search_bundles` use indexes, not linear scans
4. **Per-object deps**: `get_dependencies` loads 1-5KB, not the entire 4.6MB dependency graph
5. **Batch context**: If analyzing multiple objects, load `app_overview` once and reference it

---

# When to Load Steering Files
- Exploring application structure, browsing bundles, understanding what an app does → `bundle-analysis.md`
- Tracing dependencies, impact analysis, finding shared utilities → `dependency-analysis.md`
