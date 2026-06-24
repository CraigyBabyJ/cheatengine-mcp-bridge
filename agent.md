# Cheat Engine MCP Agent Guide

This file is for AI agents using the Cheat Engine MCP bridge. It explains the practical workflow, not just the raw API list.

## First Checks

1. Call `ping`.
2. Call `get_process_info`.
3. If no process is attached, use `get_process_list`, then `open_process`.
4. Confirm the target process name and architecture before scanning or patching.
5. Prefer `+W-C` protection for value scans and `+X` for code/AOB scans unless there is a reason to widen scope.

Cheat Engine setup matters:

- Load `MCP_Server/ce_mcp_bridge.lua` in Cheat Engine.
- Disable Cheat Engine setting: `Settings -> Extra -> Query memory region routines`.
- DBVM tools require DBK/DBVM to be working. If DBVM is unavailable, use hardware breakpoints.
- Hardware breakpoints are limited to four slots. Always stop/clear watch sessions when done.

## Tool Choice

Use workflow tools first when doing game trainer work:

- `workflow_value_hunt_*` for guided value scans.
- `workflow_write_watch_*` for finding what writes/accesses candidate addresses.
- `workflow_pointer_chain_find` for recovering a stable pointer chain from a dynamic address.
- `workflow_patch_*` for code patches that need original-byte verification and restore.
- `workflow_read_typed_batch` and `workflow_write_typed_batch` for reading/writing multiple values safely.

Use low-level tools when you already know the exact operation:

- `read_memory`, `read_integer`, `read_pointer_chain`
- `write_integer`, `write_memory`
- `scan_all`, `next_scan`, `get_scan_results`
- `set_data_breakpoint`, `get_breakpoint_hits`, `remove_breakpoint`
- `aob_scan`, `generate_signature`, `disassemble`

## Value Hunt Flow

Use this when the user gives a visible game value such as money, health, fuel, or damage.

1. `workflow_value_hunt_start(name="money", value="1003219945", value_type="qword")`
2. Ask the user to change the value in-game.
3. `workflow_value_hunt_refine(name="money", scan_type="exact", value="1002562293")`
4. Repeat until there are few candidates.
5. `workflow_value_hunt_results(name="money", limit=20)`
6. Read nearby memory or watch writes on the best candidates.
7. `workflow_value_hunt_destroy(name="money")` when finished.

For unknown values use `changed`, `unchanged`, `increased`, or `decreased` after each in-game action.

## Write Watch Flow

Use this to find the instruction controlling a value.

1. Get candidate addresses from a value hunt or known pointer chain.
2. Start a session:

```json
{
  "name": "damage_writer",
  "addresses": ["0x12345678"],
  "size": 4,
  "access_type": "w",
  "max_hits": 20,
  "capture_stack": false
}
```

3. Ask the user to trigger the value change.
4. Poll:

```json
{"name": "damage_writer", "limit": 20}
```

5. Use the `summary` list first. It groups hits by writer instruction and is usually more useful than raw hit spam.
6. Stop the session:

```json
{"name": "damage_writer"}
```

If the game freezes, stop the watch or call `clear_all_breakpoints`.

## Pointer Chain Recovery

Use this once a dynamic address has been found and validated.

```json
{
  "target_address": "0x12345678",
  "max_depth": 3,
  "max_offset": 1024,
  "step": 4,
  "max_results": 50
}
```

Prefer results with a module/symbol root, for example `game.exe+2D4F118`, over heap-only roots.

After finding a candidate chain:

1. Resolve it with `read_pointer_chain`.
2. Restart or reload the game if possible.
3. Resolve it again.
4. Only document it as stable if it survives.

## Patch Set Flow

Use patch sets for code changes that must be reversible.

1. Disassemble and capture original bytes.
2. Define the patch:

```json
{
  "name": "stop_damage",
  "patches": [
    {
      "name": "damage_write",
      "address": "game.exe+10C81F9",
      "original_bytes": "44 89 44 88 04",
      "patched_bytes": "83 64 88 04 00"
    }
  ]
}
```

3. Call `workflow_patch_status`.
4. Call `workflow_patch_apply`.
5. Verify behavior in game.
6. Call `workflow_patch_restore` before trying a different patch.

Never apply a patch blindly if `workflow_patch_status` says `unknown`, unless the user explicitly accepts `force=true`.

## Updating Cheats After a Game Update

1. Attach to the updated game.
2. Test old pointer chains with `read_pointer_chain`.
3. Test old code offsets with `workflow_patch_status`.
4. If code bytes moved, use `generate_signature`/`aob_scan` around the old instruction.
5. If the data structure moved, use `workflow_value_hunt_*` and `workflow_pointer_chain_find`.
6. Update any trainer/app docs with:
   - game version
   - module offsets
   - pointer chains
   - original bytes
   - patched bytes
   - test notes

## Safety Habits

- Keep scan ranges/protection narrow when possible.
- Do not leave hardware breakpoints running.
- Prefer workflow patch sets over ad-hoc `write_memory`.
- Save original bytes before patching.
- Verify after writes with batch reads or patch status.
- If the target crashes, reattach before reading stale addresses.

## ETS2 Notes From Prior Work

These are examples of what a game-specific note should contain. They may become stale after updates.

- Damage chain example: `eurotrucks2.exe+33C0548 -> +2F98 -> +18 -> +1A8`
- Money chain example: `eurotrucks2.exe+2D4F118 -> +10 -> +10`
- Stable trainer logic should prefer module-relative roots and original-byte verified patches.
