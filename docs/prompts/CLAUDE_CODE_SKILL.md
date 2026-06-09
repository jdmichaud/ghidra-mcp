# Ghidra RE Assistant

You are performing reverse engineering tasks using Ghidra via the GhidraMCP server.
The user's request: $ARGUMENTS

## Ghidra MCP Connectivity

### Architecture

The GhidraMCP stack has three layers:
1. **MCP bridge** (`bridge_mcp_ghidra.py`) — translates MCP protocol to HTTP calls. Provides `mcp__ghidra__*` tools.
2. **HTTP server** (port 8089) — REST API exposed by either the GUI plugin or the headless server.
3. **Ghidra backend** — GUI (with plugin) or headless (no GUI).

The MCP bridge talks to port 8089 regardless of which backend serves it.
**Always prefer `mcp__ghidra__*` tools** over raw curl — they provide structured parameters, validation, and discoverability.
Fall back to `curl` only if the MCP bridge is unavailable.

### Headless Mode (preferred — fully autonomous)

The bridge can launch the headless Ghidra backend with `--ghidra-home`:

```bash
bridge_mcp_ghidra.py --ghidra-home /path/to/ghidra_12.0.3_PUBLIC
```

This starts the headless server (no binary loaded) and registers all MCP tools,
including headless-specific tools like `load_program` and `close_program`.
The server is cleaned up automatically on exit.

Optional: `--java-opts "-Xmx8g"` for large binaries.

In this mode the bridge also manages the backend JVM: `restart_headless_server`
restarts it (all open programs are closed — save/export first, then re-load).
Use it to pick up newly installed language definitions or to change JVM options
(`restart_headless_server(java_opts="-Xmx8g")`).

### GUI Backend (alternative)

If the Ghidra GUI is already running with the MCP plugin and a program loaded in CodeBrowser, the bridge connects to it the same way — no special flags needed.

### Verifying Connection

```
mcp__ghidra__check_connection()
```

## Headless Workflow (load → analyze → work)

In headless mode, no binary is loaded at startup. You must load one explicitly:

```
# Step 1: Load a binary (absolute path)
mcp__ghidra__load_program(file="/path/to/binary.exe")

# Step 2: Switch to the loaded program (makes it the active context)
mcp__ghidra__switch_program(name="binary.exe")

# Step 3: Run auto-analysis (required before decompilation works)
mcp__ghidra__run_analysis(program="binary.exe")

# Step 4: Now all other tools work — list functions, decompile, rename, etc.
mcp__ghidra__list_functions(program="binary.exe")
```

When done, unload with:
```
mcp__ghidra__close_program(name="binary.exe")
```

## Key Operations (MCP Tools)

Always use `mcp__ghidra__*` tools. These work identically whether the backend is GUI or headless.

### Listing Functions
```
mcp__ghidra__list_functions(program="binary.exe")
```

### Decompiling
```
mcp__ghidra__decompile_function(name="FUN_1000_065c", program="binary.exe")
# For large functions, paginate:
mcp__ghidra__decompile_function(name="...", offset=0, limit=200, program="...")
# After type changes, force re-decompile:
mcp__ghidra__decompile_function(name="...", force=true, program="...")
```

### Renaming Functions
```
mcp__ghidra__rename_function_by_address(function_address="0x1065c", new_name="check_protection", program="...")
```

### Cross-References
```
mcp__ghidra__get_function_callers(name="...", program="...")
mcp__ghidra__get_function_callees(name="...", program="...")
```

### Disassembly
```
mcp__ghidra__disassemble_function(address="0x1065c", program="...")
```

### Strings
```
mcp__ghidra__list_strings(filter="*", program="...", limit=100)
```

### Program Metadata
```
mcp__ghidra__get_current_program_info(program="...")
```

### Saving (GUI mode only — headless is in-memory)
```
mcp__ghidra__save_program(program="...")
```

### Choosing a language / compiler spec at load time

`load_program` auto-detects the language and compiler spec, which sometimes picks
the wrong **compiler convention** (e.g. a Watcom DOS/PE binary detected as a generic
x86). To override it, pass `language_id` and (optionally) `compiler_spec_id`:

```
# Discover valid IDs (filter is an optional case-insensitive substring):
mcp__ghidra__list_language_ids(filter="x86")
# -> [{ "language_id": "x86:LE:32:default",
#       "compiler_spec_ids": ["windows","gcc","borlandcpp","watcom",...] }, ...]

# Load with an explicit language + compiler spec instead of auto-detection:
mcp__ghidra__load_program(file="/path/bin.exe",
                          language_id="x86:LE:32:default",
                          compiler_spec_id="watcom")
# -> {"success": true, "program": "bin.exe",
#     "language": "x86:LE:32:default", "compiler_spec": "watcom"}
```

Rules: `compiler_spec_id` requires `language_id`; omit `compiler_spec_id` for the
language's default spec. An unknown ID returns a clear `error` (it does NOT silently
fall back to auto-detect) — call `list_language_ids` to find the right value. The
success response always reports the `language`/`compiler_spec` actually used, so you
can confirm the override took effect.

### Custom Compiler Specs (.cspec) — headless mode only

If the compiler spec you need is not in `list_language_ids` (e.g. a hand-written
convention), install a custom `.cspec` first. Ghidra only scans language definitions
(`.ldefs`/`.cspec`/`.pspec`) at JVM **startup**, so this requires a backend restart:

```
# 1. Write the .cspec into a scanned location and reference it from the .ldefs:
#    <ghidra_home>/Ghidra/Processors/<P>/data/languages/custom.cspec
#    Add a <compiler name="..." spec="custom.cspec" id="mycspec"/> entry to the
#    .ldefs in the same directory. Use run_script_inline (Java in the backend JVM)
#    for the file writes.

# 2. Restart the backend so Ghidra re-scans language definitions:
mcp__ghidra__restart_headless_server()

# 3. Confirm the new spec is now visible, then load with it directly:
mcp__ghidra__list_language_ids(filter="mycspec")   # should now list it
mcp__ghidra__load_program(file="/path/bin.exe",
                          language_id="x86:LE:32:default",
                          compiler_spec_id="mycspec")
```

No `setLanguage` scripting is needed — once the spec is installed and the backend
restarted, `load_program` applies it directly via `compiler_spec_id`.

**If the spec lists but `load_program` rejects it:** `list_language_ids` reads only the
`.ldefs`, so a malformed `.cspec` still shows up but fails to *build*. The `load_program`
error then names the file, line, and reason, e.g. `...(x86watcom.cspec:323): Missing
prototype model: __cdecl`. Every prototype model referenced in a `.cspec` must also be
defined in it. Fix the file, `restart_headless_server`, retry. Full backend output
(Ghidra's own parse errors) is in `<repo>/headless_server.log` — the path is reported by
`restart_headless_server`.

For a single function with an unusual calling convention, prefer custom variable
storage on the prototype (`set_function_prototype`) instead — no cspec or restart
needed.

### Curl Fallback

If MCP tools are unavailable (bridge down), the same endpoints are accessible via HTTP:
```bash
curl -s "http://127.0.0.1:8089/list_functions"
curl -s "http://127.0.0.1:8089/decompile_function?address=FUN_name"
curl -s "http://127.0.0.1:8089/rename_function?old_name=X&new_name=Y"
```

## Address Formats

Different Ghidra tools expect different address formats. This is a common source of errors.

- **Function names** (e.g., `FUN_1000_065c`, `main`, `entry`): Always work for decompilation.
- **Flat hex** (e.g., `0x1065c`, `0x401000`): Required for rename-by-address, comments, type application.
- **Segmented** (e.g., `1000:065c`): Display-only in function listings. Do NOT pass to tools — they will reject it.

### Segmented to Flat Conversion (16-bit real mode)
```
flat = (segment << 4) + offset
Example: 1000:065c → (0x1000 * 16) + 0x065c = 0x1065c
```

For 32/64-bit PE binaries, addresses are already flat (e.g., `0x00401000`).

## Tips

- **Batch operations**: When renaming many functions, batch them to avoid excessive round-trips.
- **Large functions**: Use pagination (offset/limit) for functions with hundreds of lines.
- **Force re-decompile**: After changing types, names, or signatures, always re-decompile with `force=true` to see the effect.
- **Headless is stateless**: The headless server doesn't persist changes to disk. Export/document findings in project files.
- **Port conflicts**: If port 8089 is busy, kill stale processes: `pkill -f GhidraMCPHeadless; sleep 3`
- **The decompiler is only as good as the analysis**: If output looks wrong, check if the function boundaries are correct, if the calling convention is right, and if types have been applied.
