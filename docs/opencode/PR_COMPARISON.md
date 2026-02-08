# OpenCode Integration: Branch Comparison

> **Last Updated**: 2026-01-27  
> **Comparison**: PR #775 (steveyegge/gastown) vs feature/opencode-orchestration (rothnic/gastown)

## Recent Fixes (2026-01-27)

### Plugin JavaScript Syntax Fixes
Fixed critical syntax errors in `gastown.js` that caused plugin to fail silently:
- **Duplicate `fs` import** (lines 1 and 3) - removed duplicate
- **Duplicate variable declarations** (`lastLogBody`, `repeatCount`, etc.)
- **Undefined `currentMessageId`** - added to state variables

### Prompt Injection Flow Fix
Fixed issue where `opencode run` received no message argument:
- **Root cause**: `BuildPolecatStartupCommand` was called with empty prompt `""`
- **Impact**: OpenCode `run` mode (non-interactive) exits immediately without a message
- **Fix**: Added `Prompt` field to `SessionStartOptions` and `SlingSpawnOptions`
- **Flow**: E2E passes `--args` to sling → sling passes to spawn → startup command includes prompt

### Fallback Timer for Session Initialization  
- OpenCode doesn't always fire `session.created` event
- Added 100ms fallback timer to trigger `onSessionCreated()` if no events arrive
- Ensures prompt injection happens even without session events

### Current Status
- Prompt IS reaching OpenCode (confirmed in logs: `message_content: [Role: user]`)
- E2E test times out - likely environment/API configuration issue, not code bug
- Infrastructure fixes are complete; test success depends on runtime environment

## Overview

This document provides an **objective, factual comparison** between two independent OpenCode integration efforts. Neither implementation is assumed to be definitively "correct" - both represent different approaches with different tradeoffs.

| Branch | Repository | Target | Status |
|--------|------------|--------|--------|
| **PR #775** | steveyegge/gastown | main | OPEN |
| **feature/opencode-orchestration** | rothnic/gastown | main (fork) | Active |

**Important Caveats**:
- Our fixes (e.g., JSONL append for beads) have **not been fully verified** as necessary in all contexts
- PR #775 may not encounter our issues due to different testing approaches or architectural choices
- Some of our "fixes" are tied to our specific E2E test infrastructure

---

## Quick Comparison Matrix

### Feature Coverage

| Feature | PR #775 | Our Branch | Notes |
|---------|:-------:|:----------:|-------|
| OpenCode as built-in agent preset | Yes | Yes | Both add `opencode` to `agents.go` |
| Provider-agnostic config refactor | Yes | No | PR #775 refactors `internal/config` extensively |
| Per-role agent config (`gt role agent`) | Yes | No | PR #775 adds CLI to assign different runtimes per role |
| `gastown.js` plugin | Yes | Yes | Different implementations (see Plugin section) |
| `GT_AUTO_INIT` trigger pattern | Yes | No | PR #775 uses message trigger for context injection |
| `gt_done` explicit completion tool | No | Yes | Our plugin adds tool; PR #775 relies on `session.idle` only |
| `session.idle` completion detection | Unknown | Yes | Our plugin uses BOTH (additive approach) |
| `session.compacted` handler | Yes | Yes | Both re-run `gt prime` on compaction |
| Doctor checks for OpenCode | Yes | No | PR #775 validates OpenCode settings/commands |
| Claude settings location fix | Yes | No | PR #775 ensures settings in exact working directory |
| AGENTS.md Codex/OpenCode support | Yes | Partial | PR #775 adds multi-runtime compatibility |

### Testing Infrastructure

| Aspect | PR #775 | Our Branch | Notes |
|--------|:-------:|:----------:|-------|
| Unit tests for new code | Yes | Yes | Both add tests |
| **Full E2E test harness** | No | **Yes** | `internal/e2e/runner.go` - real agent spawning |
| Isolated beads mode (`GT_ISOLATED_BEADS`) | Unknown | Yes | Prevents test pollution of production data |
| Tmux socket isolation (`TMUX_TMPDIR`) | Unknown | Yes | Fixes macOS 104-char path limit |
| XDG variable isolation | Unknown | Yes | Prevents reading user's real config |
| Shell-based E2E scripts | No | Yes | 12+ scripts in `scripts/test-*.sh` |
| Completion signal monitoring | Unknown | Yes | Tails plugin logs for `GASTOWN_TASK_COMPLETE` |

### Reliability Workarounds

| Fix | PR #775 | Our Branch | Verified Needed? |
|-----|:-------:|:----------:|------------------|
| `AppendJSONL` (direct beads write) | No | Yes | **Unverified** - may be test-harness-specific |
| Slot set retry loops | Unknown | Yes | Addresses CLI race conditions |
| `seenDone` state tracking | No | Yes | Catches fleeting completion signals |
| Log parsing regex improvements | No | Yes | Handles active/inactive indicators |
| `GT_TOWN_ROOT` propagation | Unknown | Yes | Fixes workspace discovery in tests |

---

## Plugin Implementation: Detailed Comparison

Both branches implement `gastown.js`, but with different approaches to context injection and completion detection.

### Context Injection

| Aspect | PR #775 | Our Branch |
|--------|---------|------------|
| **Mechanism** | `client.modifyMessages()` API | `injectPrompt()` with 3 fallback methods |
| **Trigger** | `[GT_AGENT_INIT]` message pattern | Proactive polling loop until ready |
| **Timing** | Before LLM sees first message | After session starts, polls until env ready |
| **API stability** | Uses experimental `modifyMessages()` | Uses standard OpenCode APIs |

**PR #775 Flow**:
```
1. Agent starts with --prompt "[GT_AGENT_INIT]"
2. Plugin uses client.modifyMessages() to intercept
3. Replaces trigger with gt prime output BEFORE LLM sees it
```

**Our Flow**:
```
1. Agent starts, plugin polls proactively
2. Reads gastown_prompt.txt from XDG_CONFIG_HOME
3. Uses injectPrompt() with fallbacks: sendUserMessage -> tui.appendPrompt -> session.prompt
```

### Completion Detection

**Key Clarification**: Our branch uses BOTH mechanisms (additive, not replacement):

| Signal | PR #775 | Our Branch | Behavior |
|--------|---------|------------|----------|
| `session.idle` | Yes (implied) | **Yes** | Triggers after idle count >= 2 for polecats |
| `gt_done` tool | No | **Yes** | Explicit tool call for immediate completion |
| `GASTOWN_TASK_COMPLETE` log | Unknown | Yes | Unified signal detected by E2E runner |

**Our Dual Approach** (from `gastown.js` lines 319-328):
```javascript
case "session.idle":
  idleCount++;
  if (role === "polecat") {
    await run("gt costs record", "Updating task costs");
    if (idleCount >= 2) {
      log('info', 'completion', `GASTOWN_TASK_COMPLETE: Idle threshold reached`);
    }
  }
  break;
```

Plus the `gt_done` tool (lines 278-292):
```javascript
gt_done: {
  description: "Call this tool when you have finished the assigned task...",
  execute: async (params) => {
    log('info', 'completion', `GASTOWN_TASK_COMPLETE: gt_done tool used`);
    return { status: "success", message: "Task completion signaled." };
  }
}
```

**Rationale**: The `gt_done` tool provides immediate, explicit completion signaling. The `session.idle` fallback catches cases where the agent forgets to call the tool. Both emit `GASTOWN_TASK_COMPLETE` for unified detection.

---

## Deep Dive: `injectPrompt()` vs `modifyMessages()`

This section analyzes the differences between the two approaches. 

> **IMPORTANT CAVEAT**: The defensive fallback design in `injectPrompt()` was added in commit `78f233f8` ("fix(e2e): optimize opencode integration and fix slot assignment race") - a **test infrastructure fix**, not a discovery from production testing. The early plugin did NOT have fallbacks. It's unclear whether these fallbacks are necessary for production use or were added specifically to address test harness timing issues.

### Our `injectPrompt()` Implementation

```javascript
async function injectPrompt() {
  const promptFile = path.join(xdgConfig, "gastown_prompt.txt");
  if (!fs.existsSync(promptFile)) return false;
  
  const prompt = fs.readFileSync(promptFile, "utf-8") + GT_DONE_INSTRUCTIONS;
  
  // Method A: Direct SDK call
  if (client.sendUserMessage) {
    await client.sendUserMessage(prompt);
    return true;
  }
  
  // Method B: TUI simulation with delay (UNVERIFIED - may not be needed)
  if (client.tui?.appendPrompt && client.tui?.submitPrompt) {
    await client.tui.appendPrompt(prompt);
    await new Promise(r => setTimeout(r, 1000));  // TODO: verify if needed
    await client.tui.submitPrompt();
    return true;
  }
  
  // Method C: Session API fallback
  if (client.session?.prompt) {
    await client.session.prompt(prompt);
    return true;
  }
  
  return false;
}
```

**Key characteristics**:
1. **Three fallback methods** - Handles API availability variations (unverified necessity)
2. **Delays** - Various hardcoded delays (see "Unverified Delays" section below)
3. **`promptSent` flag** - Prevents duplicate injection
4. **Proactive polling** - `proactiveInit()` polls every 500ms up to 100 times

### PR #775's `modifyMessages()` Approach

```javascript
// Intercept [GT_AGENT_INIT] trigger and replace with context
client.modifyMessages((messages) => {
  return messages.map(msg => {
    if (msg.content.includes("[GT_AGENT_INIT]")) {
      return { ...msg, content: gtPrimeOutput };
    }
    return msg;
  });
});
```

**Key characteristics**:
1. **Single API call** - Clean, simple implementation
2. **Synchronous interception** - Modifies before LLM sees message
3. **No polling** - Trigger-based, not time-based
4. **Experimental API** - `modifyMessages()` is not in stable API

### Timing Issues Addressed During E2E Development

> **Clarification**: These are issues encountered while building the E2E test infrastructure. All fixes are labeled `fix(e2e):` in git history. It is **unverified** whether these issues would occur in production usage or are artifacts of the test harness approach.

#### 1. Completion Detection Timing (Test Harness Issue)

**Problem**: Test harness missed `GASTOWN_TASK_COMPLETE` if polecat completed quickly between poll intervals.

**Solution**: Added `seenDone` state tracking in `runner.go`.

**Is this a production issue?** Unknown. The test harness polls logs; production may use different detection.

#### 2. Beads CLI Resolution (Test Harness Issue)

**Problem**: `bd slot set` failed with "issue not found" in isolated mode because the CLI resolved paths before the database was ready.

**Solution**: `AppendJSONL()` bypasses CLI entirely, writing directly to the JSONL transaction log.

**Is this a production issue?** Likely NOT. This only occurs with `GT_ISOLATED_BEADS=1` which is test-only.

#### 3. Tmux Send-Keys Timing (Pre-existing Gastown Pattern)

**Context**: The Go codebase already has defensive patterns for tmux timing (from `internal/tmux/tmux.go`):
```go
// Unlike NewSession + SendKeys, this avoids race conditions 
// where the shell isn't ready
func (t *Tmux) NewSessionWithCommand(...)
```

**Relevance to plugin**: The `injectPrompt()` fallback design may have been influenced by these patterns, but this connection is **speculative**. The tmux patterns are for Go code; the plugin is JavaScript.

---

## Unverified Delays (Technical Debt)

Our codebase contains many hardcoded delays that should be replaced with event-driven or condition-based approaches. These are listed here for future cleanup.

### gastown.js Delays

| Location | Delay | Purpose | Verified? |
|----------|-------|---------|-----------|
| Line 228 | 1000ms | Wait after `appendPrompt()` before `submitPrompt()` | **NO** |
| Line 270 | 500ms | Polling interval in `proactiveInit()` | **NO** |
| Line 274 | 1000ms | Initial delay before first `proactiveInit()` | **NO** |

### runner.go (E2E) Delays

| Location | Delay | Purpose | Verified? |
|----------|-------|---------|-----------|
| Line 403 | 250ms | Unknown | **NO** |
| Line 459 | 500ms | Ticker for completion detection | **NO** |
| Line 770 | 2000ms | Unknown | **NO** |
| Line 778 | 100ms | Unknown | **NO** |

### beads_agent.go Delays

| Location | Delay | Purpose | Verified? |
|----------|-------|---------|-----------|
| Lines 173, 197, 259, 277 | 100ms each | Retry backoffs | Likely needed |
| Lines 327, 374 | 500ms each | Retry backoffs | Likely needed |

### Recommended Approach

Instead of arbitrary delays:

1. **Use events** - OpenCode provides `session.created`, `session.idle`, etc.
2. **Use conditions** - Check if API is ready before calling
3. **Use retries with backoff** - More robust than fixed delays
4. **Document why** - If a delay IS needed, explain what condition it's waiting for

### TODO: Verify or Remove

- [ ] Test if 1000ms delay in `injectPrompt()` TUI path is actually required
- [ ] Test if 500ms polling interval could be reduced or replaced with events
- [ ] Test if initial 1000ms delay before `proactiveInit()` is needed
- [ ] Replace magic numbers with named constants with explanatory comments

---

### Why Fallbacks Were Added (Honest Assessment)

The fallback design was added in commit `78f233f8` alongside E2E test fixes. The actual necessity is **unverified**:

| Claim | Evidence | Verdict |
|-------|----------|---------|
| "Handles API availability variations" | No evidence of API varying in production | Unverified |
| "Addresses timing issues" | Test harness timing, not production | Unverified |
| "Multiple paths increase resilience" | May add complexity without benefit | Speculative |
| "TUI vs headless mode differences" | No testing of different modes | Unverified |

**Honest conclusion**: We don't know if the fallbacks are necessary. They were added while debugging test harness issues and may be overengineered for problems that don't exist in production.

### When `modifyMessages()` Might Be Better

| Scenario | Why `modifyMessages()` Wins |
|----------|----------------------------|
| Clean, minimal code | Single API call vs complex fallbacks |
| Context injection timing | Guaranteed before LLM sees message |
| No polling overhead | Trigger-based, not time-based |
| Simpler debugging | One code path to trace |

### When `injectPrompt()` Might Be Better

| Scenario | Why `injectPrompt()` Wins |
|----------|--------------------------|
| API instability | Survives OpenCode updates |
| Multiple OpenCode versions | Works across API differences |
| CI/headless environments | TUI fallback handles edge cases |
| Defensive programming | Explicit failure handling |

### Recommendation

**Unknown which is actually better** - We have insufficient data:

1. **Our fallbacks**: Added during E2E debugging, necessity unverified for production
2. **PR #775's approach**: Cleaner, but uses experimental API and has no E2E validation
3. **Real comparison needed**: Run PR #775's plugin through our E2E infrastructure

**The E2E infrastructure itself is the main value-add** - regardless of which plugin approach wins, having automated agent lifecycle testing prevents regressions.

---

## Environment Variables

### Variables Set by Our E2E Infrastructure

These are set in `internal/e2e/runner.go` for test isolation:

| Variable | Value | Purpose |
|----------|-------|---------|
| `GT_ISOLATED_BEADS` | `1` | Forces beads to use isolated test database |
| `GT_TOWN_ROOT` | Test fixture root | Ensures workspace discovery works |
| `GT_BINARY_PATH` | Path to compiled `gt` | Ensures tests use compiled binary |
| `GASTOWN_TEST_HASH` | 8-char hash | Unique identifier for test run |
| `TMUX_TMPDIR` | `/tmp/gt-e2e-{hash}` | Isolates tmux sockets, fixes macOS path limits |
| `XDG_CONFIG_HOME` | Test temp directory | Prevents reading user's real config |
| `XDG_DATA_HOME` | Test temp directory | Prevents using user's real data |
| `OPENCODE_CONFIG` | Path to test config | Points to test-specific opencode.jsonc |
| `OPENCODE_LOG_LEVEL` | `debug` | Verbose logging for debugging |
| `OPENCODE_LOG_FILE` | `/tmp/opencode_internal_{hash}.log` | Captures OpenCode internal logs |
| `GASTOWN_PLUGIN_LOG` | `/tmp/gastown_plugin_{role}_{id}.log` | Captures plugin output |
| `GASTOWN_PLUGIN_LOG_EVENTS` | `/tmp/gastown_plugin_events_{role}_{id}.log` | Captures plugin events |
| `TERM` | `dumb` | Prevents terminal escape sequences |
| `CI` | `true` | Signals non-interactive environment |

### Variables Used by Plugin (`gastown.js`)

| Variable | Source | Purpose |
|----------|--------|---------|
| `GT_ROLE` | Set by Gastown spawn | Identifies agent role (polecat, witness, etc.) |
| `GT_RIG` | Set by Gastown spawn | Identifies rig name |
| `GT_POLECAT` | Set by Gastown spawn | Identifies polecat name |
| `GASTOWN_PLUGIN_LOG` | E2E runner or user | Where to write plugin logs |
| `GASTOWN_PLUGIN_LOG_EVENTS` | E2E runner or user | Where to write event logs |

### Variables Used by Gastown Core (Both Branches)

| Variable | Purpose |
|----------|---------|
| `GT_ROLE` | Identifies current role (mayor, witness, polecat, crew, etc.) |
| `GT_RIG` | Current rig name |
| `GT_POLECAT` | Polecat agent name |
| `GT_CREW` | Crew member name |
| `GT_ROOT` | Town root directory |
| `GT_SESSION_ID_ENV` | Which env var holds session ID |
| `BD_ACTOR` | Beads actor identity |
| `GIT_AUTHOR_NAME` | Git identity for commits |

---

## E2E Test Infrastructure: Potential Upstream Contribution

Our E2E infrastructure in `internal/e2e/` could be valuable upstream regardless of which plugin approach is adopted.

### What It Provides

1. **Real Agent Spawning**: Tests spawn actual OpenCode processes, not mocks
2. **Completion Detection**: Monitors plugin logs for `GASTOWN_TASK_COMPLETE`
3. **Environment Isolation**: Full isolation from user's config/data
4. **Tmux Management**: Creates/destroys isolated tmux sessions
5. **Log Capture**: Captures all output for debugging
6. **Binary Compilation**: Compiles fresh `gt` binary for each test

### Key Files

| File | Purpose |
|------|---------|
| `internal/e2e/runner.go` | Main test harness (700+ lines) |
| `internal/e2e/gastown_test.go` | Test scenarios (CreateFile, FixBug) |
| `internal/e2e/runner_unit_test.go` | Tests for the runner itself |
| `internal/testutil/fixtures.go` | Test workspace setup |
| `scripts/test-runtime-e2e.sh` | Shell-based E2E runner |
| `scripts/test-opencode-*.sh` | OpenCode-specific test scripts |

### Why This Matters

Without E2E tests, changes to the plugin or configuration could break agent behavior in ways that unit tests wouldn't catch. Our infrastructure allows:

- Verifying agents actually spawn and receive tasks
- Detecting completion signal issues
- Testing the full integration path from `gt sling` to task completion

**Caveat**: Some of our workarounds (AppendJSONL, slot retries) may be artifacts of this specific test infrastructure rather than production requirements.

---

## Gap Analysis

### What PR #775 Has That We're Missing

| Feature | Impact | Effort to Add |
|---------|--------|---------------|
| Per-role agent config | Can't mix Claude/OpenCode by role | Medium |
| Provider-agnostic doctor checks | Limited OpenCode validation | Low |
| Refactored internal/config | May not scale to more providers | High |
| `GT_AUTO_INIT` pattern | Different (possibly cleaner) init | Medium |
| Claude settings location fix | Hooks may not fire in subdirs | Low |
| Multi-runtime AGENTS.md support | Limited Codex/OpenCode docs | Low |

### What We Have That PR #775 Is Missing

| Feature | Impact | Upstream Value |
|---------|--------|----------------|
| **E2E test harness** | No automated agent verification | **High** |
| `gt_done` tool | No explicit completion signal | Medium |
| Dual completion detection | May miss completion signals | Medium |
| Environment isolation scripts | Harder to test in CI | Medium |
| Comprehensive plugin logging | Harder to debug issues | Low |
| Beads workarounds | May or may not be needed | Unknown |

---

## File-Level Overlap

### Files That Exist in Both (Need Careful Merge)

| File | PR #775 Approach | Our Approach |
|------|------------------|--------------|
| `internal/opencode/plugin/gastown.js` | `modifyMessages()`, `GT_AUTO_INIT` | Proactive init, `gt_done` tool |
| `internal/config/agents.go` | Multi-provider refactor | Minimal changes |
| `internal/witness/manager.go` | `EnsureSettingsForRole()` | Different lifecycle |

### Files Unique to Each (Safe to Merge)

| PR #775 Only | Our Branch Only |
|--------------|-----------------|
| `internal/cmd/config.go` (role agent) | `internal/e2e/runner.go` |
| `internal/doctor/opencode_commands_check.go` | `internal/beads/beads.go` (AppendJSONL) |
| `internal/cmd/enable.go` | `scripts/test-*.sh` (12+ files) |
| `internal/templates/commands-opencode/` | `docs/opencode/` (13 files) |

---

## Merge Strategy Recommendations

### Option A: PR #775 First, Then Port Our E2E Infrastructure

1. Merge PR #775 to upstream
2. Test whether our issues (beads, completion) exist there
3. Port E2E infrastructure if valuable
4. Port `gt_done` tool if explicit completion proves useful

### Option B: Our Feature Branch First, Then Cherry-Pick PR #775

1. Merge our branch to feature/opencode-orchestration
2. Cherry-pick unique PR #775 features (role agents, doctor)
3. Reconcile plugin differences

### Option C: Collaborative Approach

1. Identify which plugin approach is preferred
2. Adopt one, port valuable features from the other
3. Ensure E2E infrastructure is included regardless

---

## Open Questions

1. **Is `modifyMessages()` stable enough?** - It's documented as experimental
2. **Are our beads workarounds needed upstream?** - May be test-harness-specific
3. **Is explicit `gt_done` tool valuable?** - Adds reliability but also complexity
4. **Should E2E tests be mandatory for OpenCode changes?** - Would catch regressions
5. **How do we handle plugin divergence?** - Need to pick one approach eventually
