# rtk-zcode-bridge

> Bring [rtk](https://github.com/monosans/rtk) (Rust Token Killer) token-optimization to [ZCode](https://z.ai) — transparently, with zero prompt overhead.

ZCode's hook system is Claude-Code-compatible. This plugin ports rtk's **Claude
Code integration pattern** to ZCode: every Bash command you (or the agent) run
is transparently routed through `rtk` when rtk has a token-optimized equivalent,
and left untouched otherwise. The agent never has to learn to prefix commands —
it just happens.

```
You write:   git status
Agent runs:  rtk git status     ← 60-90% fewer output tokens, automatically
```

## How it works

Two hooks, one plugin:

| Hook | Event | What it does |
|------|-------|--------------|
| `hooks/pre-tool-use` | `PreToolUse: Bash` | Forwards each Bash command to `rtk hook claude`. If rtk can optimize it, the command is rewritten in place (`git status` → `rtk git status`) and allowed. If not, the original runs unchanged. |
| `hooks/session-start` | `SessionStart` | Reads `RTK.md` and injects it as session context, so the model knows rtk exists and which meta commands to call directly (`rtk gain`, `rtk discover`, `rtk proxy`). |

The agent's original command **always** runs on any hook error — the shim is
strictly fail-open (silently exits 0, no output).

### Why not just put rtk instructions in AGENTS.md?

ZCode loads `~/.zcode/AGENTS.md` as a static instruction file but does **not**
expand `@import` references (verified in the ZCode source). So a static
`@RTK.md` reference would not work. Instead, the awareness layer is injected
dynamically via the SessionStart hook, keeping your `AGENTS.md` completely free
of rtk content.

## Requirements

- **ZCode** ≥ 3.1.8 (hook support for the native agent landed here; on earlier
  versions `PreToolUse` hooks may not fire — see [issue #32](https://github.com/zai-org/feedback/issues/32)).
- **rtk** installed and on `PATH` — `which rtk` should resolve. Install via
  [rtk's instructions](https://github.com/monosans/rtk).
- **jq** (preferred) or **python3** — used by the hooks for JSON handling. macOS
  ships both by default.
- **bash** — the hook scripts are bash. On Windows, the bundled `run-hook.cmd`
  polyglot wrapper locates Git Bash automatically.

## Install

The easiest way is to let ZCode's agent install it for you. Open a ZCode
session and paste:

```
Install the rtk-zcode-bridge plugin for zcode from https://github.com/macgaf/rtk-zcode-bridge
```

The agent reads the repo, clones it, sets permissions, registers it in
`~/.zcode/cli/config.json`, and tells you to restart. That's the whole install.

<details>
<summary>Prefer to do it manually?</summary>

```bash
# 1. Clone anywhere you like
git clone https://github.com/macgaf/rtk-zcode-bridge.git <path>/rtk-zcode-bridge

# 2. Make the hook scripts executable
chmod +x <path>/rtk-zcode-bridge/hooks/run-hook.cmd \
         <path>/rtk-zcode-bridge/hooks/pre-tool-use \
         <path>/rtk-zcode-bridge/hooks/session-start
```

Then add the cloned directory to the `plugins.dirs` array in
`~/.zcode/cli/config.json`:

```jsonc
{
  "plugins": {
    "enabledPlugins": {},
    "dirs": [
      "<path>/rtk-zcode-bridge"
    ]
  }
}
```

Inline plugins (those listed in `dirs`) are **enabled by default** — no entry in
`enabledPlugins` is needed.

</details>

After installing (either way), **restart ZCode** so the hooks load.

## Verify

After restarting, in a ZCode session:

```bash
# The agent runs this — it should be transparently rewritten to `rtk git status`
git status
```

Then check the debug log to confirm the rewrite happened:

```bash
tail ~/.zcode/cli/logs/rtk-bridge.log
# expect: "rtk exit=0 out={...updatedInput: rtk git status...}"
```

And confirm rtk recorded the execution:

```bash
rtk gain
# Total commands should have incremented
```

Commands rtk does **not** optimize (e.g. `mkdir`, `echo`) run unchanged — the
log will show `out=<empty>` for those, which is correct.

## What gets optimized

rtk transparently rewrites commands it has optimized proxies for, including
(among others): `git`, `gh`, `cargo`, `docker`, `kubectl`, `ls`, `tree`,
`find`, `grep`, `rg` (→ `rtk grep`), `diff`, `test`/`jest`/`vitest`/`pytest`,
`tsc`, `lint`, `pnpm`, `psql`, `aws`, `json`/`jq`, `wc`, and more. See
`rtk --help` for the full list. Anything not in that list passes through
untouched.

## Configuration

| Env var | Default | Purpose |
|---------|---------|---------|
| `RTK_BIN` | `rtk` | Path to the rtk binary (override if not on `PATH`). |
| `RTK_MD` | *(plugin's `RTK.md`)* | Path to the awareness file injected at session start. Defaults to the bundled `RTK.md`; set this to use a custom file. |
| `RTK_BRIDGE_LOG` | `~/.zcode/cli/logs/rtk-bridge.log` | Debug log path. Set to `/dev/null` to silence. |

To customize the awareness instructions, either edit the bundled `RTK.md` or
point `RTK_MD` at your own file.

## Uninstall

Tell the ZCode agent:

```
Uninstall the rtk-zcode-bridge plugin for zcode
```

The agent removes its path from `plugins.dirs` in `~/.zcode/cli/config.json` and
deletes the cloned directory. The plugin touches nothing else — your
`~/.zcode/AGENTS.md` is never modified.

## Troubleshooting

**Commands aren't being rewritten.**
- Confirm `which rtk` resolves in the shell ZCode uses.
- Check `~/.zcode/cli/logs/rtk-bridge.log` — if it's empty, the hook isn't
  firing. Ensure you're on ZCode ≥ 3.1.8 and that the plugin path in
  `plugins.dirs` is correct and absolute.
- Verify the hook scripts are executable (`chmod +x`).

**Log shows `rtk exit=127`.**
The rtk binary wasn't found. Set `RTK_BIN` to its absolute path, or ensure the
directory rtk lives in is on the PATH ZCode inherits.

**`Hook stdout was not valid JSON` in ZCode.**
This is a recoverable error — the session continues, just without that hook's
output. Check the log for what the hook emitted. The shim only emits JSON when
rtk rewrites; otherwise it's silent.

**Hook makes session start slow.**
`SessionStart` runs synchronously (`async: false`) but only reads a ~1 KB file.
If it's slow, check whether `RTK.md` is huge or on a slow filesystem.

## How this compares to rtk's other integrations

| | Claude Code | Codex | **ZCode (this plugin)** |
|---|---|---|---|
| Command rewrite | `PreToolUse` hook | none (manual prefix) | `PreToolUse` hook |
| Awareness layer | `CLAUDE.md` + `@RTK.md` | `AGENTS.md` + `@RTK.md` | `SessionStart` hook injects `RTK.md` |
| Why different | — | Codex has no hooks | ZCode doesn't expand `@import` in AGENTS.md |
| Force level | automatic | relies on model compliance | automatic |

## License

MIT — see [LICENSE](LICENSE).
