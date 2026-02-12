---
name: "gam-appian"
displayName: "GAM Appian Knowledge Base"
description: "Central knowledge base for all Government Acquisition Management (GAM) Appian solutions. Browse application bundles, trace dependencies, search objects, and understand architecture across all GAM applications."
keywords: ["appian", "gam", "government acquisition", "acquisition management", "sail", "record type", "process model", "expression rule", "interface", "integration", "web api", "connected system", "cdt", "source selection", "requirements management", "appian bundle", "appian dependency"]
---

# Onboarding

## Step 1: Validate knowledge base access
Call the `list_applications` tool to verify available GAM applications. If no applications are found, a GAM application package needs to be parsed first:

```bash
cd /path/to/gam-appian-knowledge-base
source .venv/bin/activate
python -m appian_parser dump <package.zip> ./data/<AppName>
```

## Step 2: Understand the knowledge base
This is the central repository for all GAM Appian solution documentation. Each parsed application contains:

### Core Files (Single-Fetch Orientation)
- **app_overview.json** (~100-150KB) — Package metadata, bundle index, dependency summary, coverage stats — everything needed to navigate the app in ONE call
- **search_index.json** (~200KB) — Flat object name lookup: `name → {type, uuid, bundles[], deps_out, deps_in}` — instant search, no scanning

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
1. `list_applications` → see all GAM apps
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
**Purpose**: Discover all available GAM applications with high-level stats.

**Returns**: Array of applications with:
- `name`: Application folder name
- `total_objects`: Count of parsed objects
- `total_errors`: Parse error count
- `bundle_coverage`: `{total_objects, bundled, orphaned}`
- `bundles_by_type`: Count per bundle type

**When to use**: First call to see what's available.

**Example**:
```
User: "What GAM applications are available?"
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

# Typical Query Patterns

## Pattern 1: Explore an application
```
1. list_applications() → see all apps
2. get_app_overview("AppName") → full map
3. Review bundles[] and dependency_summary
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
3. For each dependency, repeat step 2 to build full graph
```

## Pattern 4: Impact analysis
```
1. get_dependencies("AppName", "shared_utility") → see called_by[]
2. For each caller, get_object_detail() to see which bundles use it
3. Assess impact scope across bundles
```

## Pattern 5: Find unused code
```
1. list_orphans("AppName") → see unbundled objects
2. Filter by_type for specific object types
3. get_orphan("AppName", uuid) → inspect individual orphans
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
