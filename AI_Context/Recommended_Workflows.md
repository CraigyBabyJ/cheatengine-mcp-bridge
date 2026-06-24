# Recommended Workflows

Common reverse-engineering workflows using the Cheat Engine MCP bridge tools. Prefer the `workflow_*` tools for trainer/game-update work because they keep session state and return summaries that are easier for an AI agent to act on.

---

## First Attach Checklist

1. Verify the bridge is alive.
   ```
   ping
   ```

2. Confirm the attached process.
   ```
   get_process_info
   ```

3. If needed, attach by process name.
   ```
   get_process_list
   open_process: process_id_or_name="game.exe"
   ```

4. Keep hardware debug slots clear before tracing.
   ```
   list_breakpoints
   clear_all_breakpoints
   ```

---

## Find a Game Value

Use this for visible values such as money, health, fuel, timer, or damage.

1. Start a named value hunt.
   ```
   workflow_value_hunt_start: name="money", value="1003219945", value_type="qword"
   ```

2. Ask the user to change the value in game.

3. Refine the session.
   ```
   workflow_value_hunt_refine: name="money", scan_type="exact", value="1002562293"
   ```

4. Repeat until there are only a few candidates.
   ```
   workflow_value_hunt_results: name="money", limit=20
   ```

5. Clean up when finished.
   ```
   workflow_value_hunt_destroy: name="money"
   ```

For unknown values, refine with `changed`, `unchanged`, `increased`, or `decreased` after each in-game action.

---

## Trace Who Writes to an Address

Use this when you know a candidate address but need the instruction that controls it.

1. Start a write-watch session over up to four addresses.
   ```
   workflow_write_watch_start: name="damage_writer", addresses=["0x12345678"], size=4, access_type="w", max_hits=20
   ```

2. Ask the user to trigger the value change.

3. Poll results and read the grouped writer summary first.
   ```
   workflow_write_watch_poll: name="damage_writer", limit=20
   ```

4. Stop the watch before trying a new trace.
   ```
   workflow_write_watch_stop: name="damage_writer"
   ```

If the target freezes or a slot is leaked, run `clear_all_breakpoints`.

---

## Recover a Stable Pointer Chain

Use this after a dynamic address has been validated.

1. Search for module/static roots pointing to the dynamic address.
   ```
   workflow_pointer_chain_find: target_address="0x12345678", max_depth=3, max_offset=1024, step=4, max_results=50
   ```

2. Prefer chains whose `root_symbol` is module-relative, e.g. `game.exe+2D4F118`.

3. Verify the best chain.
   ```
   read_pointer_chain: base="game.exe+2D4F118", offsets=[0x10, 0x10]
   ```

4. Restart/reload the target and verify the chain still resolves.

---

## Safely Patch and Revert

Use named patch sets instead of ad-hoc writes whenever possible.

1. Define the patch with original bytes.
   ```
   workflow_patch_define: name="stop_damage", patches=[
     {"name":"damage_write", "address":"game.exe+10C81F9", "original_bytes":"44 89 44 88 04", "patched_bytes":"83 64 88 04 00"}
   ]
   ```

2. Check whether bytes still match the known build.
   ```
   workflow_patch_status: name="stop_damage"
   ```

3. Apply only if status is `original` or `patched`.
   ```
   workflow_patch_apply: name="stop_damage"
   ```

4. Restore before testing a different patch.
   ```
   workflow_patch_restore: name="stop_damage"
   ```

Do not use `force=true` unless the user explicitly accepts the risk.

---

## Find a Moved Hook or Patch After an Update

Use this when a game update changes code offsets.

1. Check the old patch status.
   ```
   workflow_patch_status: name="known_patch"
   ```

2. If bytes are `unknown`, generate or use the old AOB signature.
   ```
   generate_signature: address="game.exe+old_offset"
   aob_scan_module_unique: module="game.exe", pattern="..."
   ```

3. Inspect the new hit.
   ```
   disassemble_range: start=<new_address>-32, end=<new_address>+64
   ```

4. Define a new patch set with the new address and original bytes.

5. Update the game-specific agent notes with version, offsets, signatures, original bytes, patched bytes, and test notes.

---

## Find a Changed Field Offset

Use this when an object base is stable but fields moved.

1. Resolve the object base.
   ```
   read_pointer_chain: base="game.exe+static_root", offsets=[0x10, 0x28]
   ```

2. Snapshot the structure region.
   ```
   memory_snapshot: name="obj_before", address=<base>, size=0x400
   ```

3. Trigger the in-game change.

4. Diff the region.
   ```
   memory_diff: name="obj_before"
   ```

5. Read candidates in one call.
   ```
   workflow_read_typed_batch: reads=[{"address":<base>+0x80,"type":"float"},{"address":<base>+0x84,"type":"float"}]
   ```

6. If values need resetting, write them as a verified batch.
   ```
   workflow_write_typed_batch: writes=[{"address":<base>+0x80,"type":"float","value":0}], verify=true
   ```

---

## Analyze a Function

Use this when a write/watch or AOB scan lands on code.

1. Get metadata.
   ```
   function_info: address=<addr>
   ```

2. Disassemble the full function.
   ```
   disassemble_range: start=<function_start>, end=<function_end>
   ```

3. Find callers and code references.
   ```
   xref_summary: address=<addr>, max_refs=50
   ```

4. Generate a signature if the instruction will be used in a trainer.
   ```
   generate_signature: address=<addr>
   ```

---

## When to Use Low-Level Tools

The older tools are still useful when the workflow wrapper is too broad:

- `scan_all`, `next_scan`, `get_scan_results` for raw CE scans.
- `set_data_breakpoint`, `get_breakpoint_hits`, `remove_breakpoint` for single-slot manual tracing.
- `memory_snapshot`, `memory_diff` for byte-level before/after comparisons.
- `restore_bytes`, `write_memory`, `write_integer` for quick experiments.
- `start_dbvm_watch`, `stop_dbvm_watch` when DBVM is available and stealth is important.
