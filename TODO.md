# TODO: fix ACP `session/load` rejection for cross-project sessions

## Bug

A session stored under one project directory cannot be loaded through ACP from
another project, even though `AcpAgent.#findStoredSession` deliberately finds it.
The failure surfaces in the editor (e.g. Zed) as:

```
Failed to Launch — Internal error: {
  "details": "ACP session load was cancelled: <sessionId>"
}
```

Reproduced with omp 18.4.5 against the same session id:

- `session/load { cwd: /home/austin/ml/rust1 }` → `ACP session load was cancelled: …`
- `session/load { cwd: /home/austin/ml/ml1 }`   → loads fine

## Chain of events

1. Zed thread bound to worktree `rust1` references a session created earlier
   with `cwd = ml1` (recorded as `"cwd"` in the session `.jsonl`), so
   `session/load` arrives with `cwd = rust1`.
2. `packages/coding-agent/src/modes/acp/acp-agent.ts` — `#findStoredSession`:
   not found under the cwd-derived directory, so the global by-id fallback
   (added for #7779) returns the session stored under the *other* project dir.
   The comment there claims "`#openStoredSession` reopens the file with the
   request cwd" — but that is exactly what the next step forbids.
3. `#openStoredSession` → `AgentSession.switchSession(sessionPath)` with **no
   options** (no `onCwdChange`).
4. `packages/coding-agent/src/session/agent-session.ts` — `switchSession`:
   `recordedCwd` (ml1) !== `previousSessionState.cwd` (rust1) → throws
   `SESSION_CWD_CHANGE_REJECTED` (~line 10562) → caught, `return false`
   (~line 10811).
5. Back in `#openStoredSession` (acp-agent.ts ~line 1317): `success === false`
   → `throw new Error("ACP session load was cancelled: <sessionId>")`.

So the global by-id lookup and the cwd guard are mutually inconsistent, and
the "cancelled" message misleads — nothing cancelled the load; a cwd mismatch
rejected it.

## Fix directions (pick one, add tests)

- Pass an `onCwdChange` handler (or a dedicated opt-in flag) from the ACP
  `session/load`/`session/resume` paths in `#openStoredSession` so a session
  found via the global by-id fallback may be reopened under the request cwd —
  matching the stated intent of the #7779 fallback.
- Or restrict `#findStoredSessionById` to refuse (with a precise error such as
  "session belongs to cwd X, not Y") instead of returning a session that is
  then guaranteed to be rejected.
- Either way, stop reporting cwd-mismatch as "load was cancelled".

## Regression test sketch

Spawn `omp acp` over stdio, `initialize`, create a session under cwd A, close
it, then `session/load` with cwd B and assert a load success (opt-in) or a
clear cwd-mismatch error — not `ACP session load was cancelled`.
