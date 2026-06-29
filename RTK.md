# RTK - Rust Token Killer

**Usage**: Token-optimized CLI proxy (60-90% savings on dev operations)

## Meta Commands (always use rtk directly)

```bash
rtk gain              # Show token savings analytics
rtk gain --history    # Show command usage history with savings
rtk discover          # Analyze history for missed optimization opportunities
rtk proxy <cmd>       # Execute raw command without filtering (for debugging)
```

## Installation Verification

```bash
rtk --version         # Should show: rtk X.Y.Z
rtk gain              # Should work (not "command not found")
which rtk             # Verify correct binary
```

⚠️ **Name collision**: If `rtk gain` fails, you may have reachingforthejack/rtk (Rust Type Kit) installed instead.

## Hook-Based Usage

All Bash commands are automatically rewritten by the ZCode rtk-bridge hook
(PreToolUse). You write commands normally; supported commands are transparently
routed through rtk at execution time.

Example: `git status` → `rtk git status` (transparent, 0 tokens overhead)

Commands rtk does not optimize (e.g. `mkdir`, `echo`) run unchanged. You do not
need to prefix commands with `rtk` yourself — the hook handles it.
