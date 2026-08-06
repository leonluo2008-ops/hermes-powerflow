# Debug Toolkits — When Systematic Debugging Needs a Real Debugger

Source skills archived 2026-06-23 during skill-curator consolidation pass. The three "debug-toolkit" skills each cover a specific debugger for a specific runtime. They are not phases of the superpowers pipeline — they are tools invoked during Phase 4 (Systematic Debugging) when `print()` / log-scraping isn't enough.

## When to load this reference

Phase 4 (Systematic Debugging) found the bug. You have a hypothesis. Before the fix, you want to inspect live state at a breakpoint, walk the call stack, or dump local variables. **That's when one of these tools beats console.log.**

Pick the toolkit that matches your runtime:

| Runtime | Toolkit | Reference |
|---|---|---|
| Python | pdb / debugpy | [§1](#1-python--pdb--debugpy) |
| Node.js (TypeScript / Ink TUI) | `node inspect` / CDP via `chrome-remote-interface` | [§2](#2-nodejs--node-inspect--cdp) |
| Hermes TUI slash commands (Python + tui_gateway JSON-RPC + Ink/TS frontend) | Three-layer debugging playbook | [§3](#3-hermes-tui-slash-commands) |

---

## 1. Python — pdb / debugpy

**Source skill (archived):** `superpowers-debug-toolkit-python-debugpy/`

Three tools, picked by situation:

| Tool | When |
|---|---|
| **`breakpoint()` + pdb** | Local, interactive, simplest. Add `breakpoint()` in the source, run normally, get a REPL at that line. |
| **`python -m pdb`** | Launch an existing script under pdb with no source edits. Useful for quick poking. |
| **`debugpy`** | Remote / headless / "attach to already-running process." Talks DAP, scriptable from terminal, works for long-lived processes (gateway, daemon, PTY children). |

**Start with `breakpoint()`.** It's the cheapest thing that works.

### Common use cases

- A test fails and the traceback doesn't reveal why a value is wrong
- A function returns wrong shape and you want to inspect mid-flight
- Need to attach to a long-lived Hermes gateway process and pause on a flag

### Recipe — local breakpoint

```python
def some_function(x):
    breakpoint()      # ← pdb REPL opens here
    return x * 2
```

```bash
python3 script.py
# → pdb REPL at the breakpoint
# > p x          # print x
# > n            # next line
# > c            # continue
```

### Recipe — remote attach (debugpy)

```python
# in your script
import debugpy
debugpy.listen(5678)                          # or .listen(("0.0.0.0", 5678)) for non-localhost
print("Waiting for debugger attach...")
debugpy.wait_for_client()                     # blocks until VS Code / curl attaches
```

```bash
# from another terminal / VS Code
python3 -m debugpy --client 127.0.0.1:5678
```

### Common pitfalls

- **`breakpoint()` no-op in pytest by default** — set `PYTHONBREAKPOINT=ipdb.set_trace` or pass `-s` to pytest to keep breakpoints.
- **`pdb` post-mortem** — `python3 -m pdb -c continue script.py` runs the script but drops you into pdb on exception.
- **debugpy port already in use** — `lsof -i :5678` and `kill -9`.

---

## 2. Node.js — `node inspect` / CDP

**Source skill (archived):** `superpowers-debug-toolkit-node-inspect-debugger/`

When `console.log` isn't enough, drive Node's built-in V8 inspector programmatically from the terminal. You get real breakpoints, step in/over/out, call-stack walking, local/closure scope dumps, and arbitrary expression evaluation in the paused frame.

Two tools, pick one:

- **`node inspect`** — built-in, zero install, CLI REPL. Best for quick poking.
- **`ndb` / CDP via `chrome-remote-interface`** — scriptable from Node/Python; best when you want to automate many breakpoints, collect state across runs, or debug non-interactively from an agent loop.

**Prefer `node inspect` first.** It's always available and the REPL is fast.

### Recipe — `node inspect`

```bash
node --inspect-brk script.js    # opens ws://127.0.0.1:9229 with breakpoint on first line
# Then attach from another terminal:
node inspect 127.0.0.1:9229
# Or use Chrome DevTools: chrome://inspect → click target
```

### Recipe — CDP via `chrome-remote-interface` (Node script)

```bash
npm install chrome-remote-interface
```

```javascript
const CDP = require('chrome-remote-interface');
(async () => {
  const client = await CDP();
  const { Debugger, Runtime } = client;
  await Debugger.enable();
  const { scriptId } = await Debugger.getScriptSource(...).then(/* resolve URL */);
  await Debugger.setBreakpoint({ location: { scriptId, lineNumber: 42 } });
  // ... event-driven from here
  await client.close();
})();
```

### Common pitfalls

- **`--inspect` vs `--inspect-brk`** — `--inspect` starts paused only if a debugger attaches within ~5s; `--inspect-brk` always pauses on first line. Use `--inspect-brk` when scripting.
- **Port collision** — default 9229; `lsof -i :9229` first.
- **Chrome DevTools URL changes per launch** — `http://localhost:9229/json` lists current targets.

---

## 3. Hermes TUI Slash Commands

**Source skill (archived):** `superpowers-debug-toolkit-debugging-hermes-tui-commands/`

Hermes slash commands span three layers — Python command registry, tui_gateway JSON-RPC bridge, and the Ink/TypeScript frontend. When a command misbehaves (missing from autocomplete, works in CLI but not TUI, config persists but UI doesn't update), the bug is almost always one layer being out of sync with another.

### When to use

- A slash command exists in one part of the codebase but doesn't work fully
- A command needs to be added to both backend and frontend
- Command autocomplete isn't working for specific commands
- Command behavior is inconsistent between CLI and TUI
- A command persists config but doesn't apply live in the TUI

### Three-layer architecture

```
┌─────────────────────────────────────┐
│  Ink/TypeScript Frontend (TUI)      │ ← autocomplete, /help text, key bindings
└─────────────────────────────────────┘
              ↕ JSON-RPC
┌─────────────────────────────────────┐
│  tui_gateway (Python bridge)        │ ← routes slash commands, translates to/from
└─────────────────────────────────────┘
              ↕ in-process
┌─────────────────────────────────────┐
│  Python command registry            │ ← actual implementation of each command
└─────────────────────────────────────┘
```

### Typical debugging flow

1. **Reproduce**: open the TUI, run `/help`. Which commands are missing or broken?
2. **Inspect each layer**:
   - Python registry: `grep -rn "command-name" hermes_cli/`
   - tui_gateway bridge: `grep -rn "command-name" tui_gateway/`
   - Ink frontend: `grep -rn "command-name" tui/components/`
3. **Pick a debugger**: Python layer → use `§1 Python`; TypeScript layer → use `§2 Node.js`
4. **Cross-layer debug**: set breakpoint in Python registry; attach debugpy; trigger from TUI; inspect IPC payload at the bridge boundary

### Common pitfalls

- **Frontend cache**: Ink may cache the `/help` output. Restart the TUI after backend changes.
- **JSON-RPC version mismatch**: if the bridge and registry disagree on schema, the command silently fails. Look for `protocol_version` constants.
- **Config persistence**: a command may write to `~/.hermes/config.yaml` correctly but the TUI never re-reads it on update — that's a frontend bug, not a backend bug.

---

## See also

- **Phase 4 (Systematic Debugging)** in `SKILL.md` — the four-phase methodology these tools support.
- **`agent-debugging-methodology`** — cross-runtime debugging iron rules and pitfall catalog. Load alongside this when Phase 1 root-cause investigation needs deeper-than-toolkit guidance.

## Provenance

Three archived skills merged here 2026-06-23:
- `superpowers-debug-toolkit-python-debugpy` (375 lines)
- `superpowers-debug-toolkit-node-inspect-debugger` (319 lines)
- `superpowers-debug-toolkit-debugging-hermes-tui-commands` (152 lines)

Full original SKILL.md bodies preserved under `~/.hermes/skills/.archive/superpowers-debug-toolkit-*/`.
