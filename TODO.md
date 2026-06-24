# TODO

## Done

- Added agent workflow MCP tools:
  - `workflow_value_hunt_start/refine/results/destroy`
  - `workflow_write_watch_start/poll/stop`
  - `workflow_writer_report`
  - `workflow_pointer_chain_find`
  - `workflow_patch_define/status/apply/restore`
  - `workflow_read_typed_batch`
  - `workflow_write_typed_batch`
  - `workflow_manifest_export/import/list/get/delete/verify`
- Updated `README.md`, `AGENTS.md`, `agent.md`, and `AI_Context/MCP_Bridge_Command_Reference.md` for the new workflow tools.
- Updated `AI_Context/Recommended_Workflows.md` to prefer workflow tools over low-level scan/breakpoint commands.
- Added Unit-27 integration smoke tests for manifests, typed batch IO, value hunts, and safe skipped watch/report placeholders.

## Next Useful Additions

1. Add a small `workflow_signature_rebase` helper that validates old patch signatures after game updates and suggests new module offsets.
2. Add a safer patch dry-run mode that reports original-byte mismatches across an entire patch set instead of stopping at the first mismatch.
3. Add optional module/range limits to `workflow_pointer_chain_find` so deep searches can be faster on large games.

## Validation Still Needed

- Reload `ce_mcp_bridge.lua` inside Cheat Engine and call `ping`.
- Restart/reconnect the MCP client so the Python tool list refreshes.
- Run a live smoke test against a harmless process:
  - `workflow_value_hunt_start`
  - `workflow_value_hunt_results`
  - `workflow_value_hunt_destroy`
  - `workflow_read_typed_batch`
- Run live write-watch testing only when a target process can safely be paused/continued.
- Run a writer report live test after a watch captures at least one hit.



