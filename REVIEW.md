# Review: Spine Exec Routing (footgun/spine-exec-routing)

**Reviewer:** FootGun (manual review — Claude Code OOM'd on this codebase)
**Date:** 2026-02-21
**Diff:** `git diff footgun/alpha-fast-track...HEAD` (3 files, +438/-5)

## Summary

Adds opt-in spine routing: when `BOTSTER_EXEC_VIA_SPINE=1`, exec/process/read/write/edit tools proxy through the broker (`POST /v1/command`) to a remote actuator. Clean wrapper pattern — no modifications to original tool implementations.

## Findings

### WARNING — `as unknown as AnyAgentTool` casts in pi-tools.ts

**Location:** `src/agents/pi-tools.ts` (multiple lines in the spine wrapping block)
**Problem:** The `createSpine*Tool` functions return `AgentTool<any, any>`, which gets cast through `unknown` to `AnyAgentTool`. This bypasses type checking on the tool shape.
**Fix:** Consider making the spine wrapper functions generic or accepting/returning `AnyAgentTool` directly. Low risk since the wrapper preserves all properties via spread, but the casts hide potential mismatches.

### WARNING — No retry on transient broker failures

**Location:** `src/seks/spine-client.ts:75-100`
**Problem:** A single network failure (DNS hiccup, TCP reset) immediately throws. The exec tool locally has retry/backoff logic via the shell. Spine routing has none.
**Fix:** Add 1-2 retries with short backoff for network errors (not HTTP 4xx). Not blocking for alpha but should be added before production.

### NIT — `timeout_ms` field name inconsistency

**Location:** `src/seks/spine-client.ts:88` vs `spine-exec-intercept.ts:67`
**Problem:** The payload interface uses `timeout_ms` (snake_case) while the function signature uses `timeoutMs` (camelCase). Both are correct for their context (wire protocol vs TS code) but worth documenting.
**Fix:** Add a comment noting the snake_case is intentional for broker wire protocol.

### NIT — `resolveExecTimeoutMs` caps at 65s

**Location:** `src/seks/spine-exec-intercept.ts:63`
**Problem:** Hard cap of 65,000ms may be too low for long-running commands (builds, large file operations).
**Fix:** Consider making this configurable or at least bumping to match broker's own timeout. Not blocking for alpha.

### NIT — `baseExecute` assigned but never used

**Location:** `src/seks/spine-exec-intercept.ts` lines 178, 199, 217, 234, 251
**Problem:** Each `createSpine*Tool` function assigns `const baseExecute = baseTool.execute` but never references it (the wrapper completely replaces execution). Dead code.
**Fix:** Remove the unused assignments, or use them as fallback when spine is unreachable.

## Passed Checks

1. **Breaking changes:** ✅ When `BOTSTER_EXEC_VIA_SPINE` is unset, `getSpineConfig()` returns null, `spineConfig` is falsy, all wrapping is skipped, `base` is used as-is. Zero behavioral change.
2. **Security:** ✅ Token only appears in Authorization header. Error messages don't leak tokens. Broker URL in error is acceptable (not a secret).
3. **Protocol match:** ✅ `capability` field names (exec, process, read, write, edit) match actuator's tool dispatch. Payload shapes passed through as-is.
4. **Type safety:** ✅ Using `any` instead of `unknown` for `AgentTool` generics is the correct fix — matches how the codebase uses `AgentTool<any>` elsewhere (see `AgentState.tools`).

## Verdict

**SHIP IT** — No critical issues. Warnings are real but acceptable for alpha. The code is clean, well-structured, and correctly isolated behind the env var flag.
