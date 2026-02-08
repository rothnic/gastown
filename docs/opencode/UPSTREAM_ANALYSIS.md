# Upstream Analysis: OpenCode Integration (2026-02-08)

This document analyzes the current state of upstream OpenCode integration efforts in the steveyegge/gastown repository, comparing with our branch (rothnic/gastown copilot/add-opencode-orchestration-layer-again). It provides an updated assessment since the previous PR_COMPARISON.md (2026-01-27) and offers strategic recommendations for contributing our work upstream.

## Current State of PR #775

PR #775 ("feat: Complete OpenCode Support") by arttttt remains **OPEN** as of 2026-02-08, with its last update on 2026-02-03. The PR introduces comprehensive OpenCode support but has not been merged despite being open for over 3 weeks.

### Related PR Ecosystem
- **PR #1103** ("fix(config): proper role_agents resolution for polecat sessions") by arttttt: OPEN, 2026-02-03, 11 files (+76/-42). Fixes polecat session stability issues when using role_agents configuration.
- **PR #1183** ("fix: resolve merge conflicts in OpenCode support branch") by andrewboldi: OPEN, 2026-02-06, 68 files (+2003/-1012). Resolves 17 merge conflicts between #775 and main branch.

### Key Issues
- **No maintainer reviews**: No visible responses from @steveyegge despite community activity (18+ comments).
- **Merge drift**: Upstream main has evolved significantly with Dolt migration, branch-per-polecat, formula trimming, and daemon fixes. This creates ongoing merge conflicts.
- **Community enthusiasm**: Comments show active interest (@zjpiazza: "yes plz lets get this merged ❤️", @sfncore: "thanks for this! trying now...") but no maintainer action.
- **Build verification**: PR #1183 author claims "GUYS THIS ACTUALLY WORKS!!!! Full opencode support!" with passing go build, go vet, and tests (except 1 pre-existing failure).

## Feature Comparison Update

Since PR_COMPARISON.md (2026-01-27), several developments have shifted the landscape:

### New Developments
- **PR #1183 emergence**: Resolves merge conflicts, claims working OpenCode support, represents the most current viable upstream implementation.
- **Main branch evolution**: Significant changes including VM integration tests, run-hardener updates, config patrol - increasing merge complexity.
- **No maintainer engagement**: Despite community activity, no official reviews or maintainer feedback on #775.
- **Consolidation efforts**: References to PR #794 consolidation and settings fixes extracted to PR #972.

### Core Architecture Differences
| Aspect | PR #775/#1183 | Our Branch |
|--------|---------------|------------|
| Plugin API | `client.modifyMessages()` (single API, experimental) | `injectPrompt()` with 3 fallback methods |
| Completion Detection | Session idle detection | Dual: idle + explicit `gt_done` tool |
| Testing | Unit tests only | 700+ line E2E harness + 12 shell scripts |
| Documentation | Minimal | 13+ files in docs/opencode/ |
| Environment Isolation | Basic | XDG, TMUX_TMPDIR, GT_ISOLATED_BEADS |

### Reliability Approaches
- **Upstream**: Provider-agnostic config refactor, GT_AUTO_INIT triggers, doctor checks.
- **Our branch**: Beads JSONL append workarounds, prompt passthrough chain, comprehensive isolation for testing.

> **Note**: Both plugin approaches remain unverified in production. Our fallbacks may be overengineered for test harness needs.

## Key Contributions FROM Our Branch

Our branch contains several unique contributions that upstream currently lacks:

### 1. E2E Test Infrastructure
- **`internal/e2e/runner.go`** (700+ lines): Real agent spawning and testing, not mocks
- **12 shell scripts** (`scripts/test-*.sh`): Comprehensive end-to-end validation
- **Environment isolation**: XDG_CONFIG_HOME, TMUX_TMPDIR, GT_ISOLATED_BEADS for clean testing

### 2. Explicit Completion Signaling
- **`gt_done` tool**: Immediate completion signaling to prevent indefinite waiting
- **Dual completion detection**: Combines session.idle with gt_done tool calls for reliability

### 3. Beads JSONL Workarounds
- **Direct append mechanism**: Bypasses CLI race conditions in beads writing
- **JSONL format handling**: Robust parsing and appending for session continuity

### 4. Prompt Passthrough Architecture
- **Sling→polecat chain**: Proper prompt forwarding through agent layers
- **Context preservation**: Maintains full conversation context across agent handoffs

### 5. Comprehensive Documentation
- **13+ files in docs/opencode/**: Complete integration guides, API references, troubleshooting
- **Architecture diagrams**: Visual representations of agent orchestration
- **Contributing guidelines**: Standards for OpenCode development

### 6. Production-Ready Patterns
- **Error handling**: Graceful degradation and recovery mechanisms
- **Logging integration**: Structured logging for debugging agent interactions

## Key Features FROM Upstream

PR #775/#1183 offers architectural improvements we should consider adopting:

### 1. Provider-Agnostic Architecture
- **Extensive internal/config refactor**: Flexible configuration system supporting multiple LLM providers
- **Modular design**: Clean separation between provider-specific and core logic

### 2. Per-Role Agent Configuration
- **`gt role agent` command**: Dynamic agent assignment based on roles
- **`role_agents` settings**: Configurable agent mappings for different contexts

### 3. GT_AUTO_INIT Trigger Pattern
- **`client.modifyMessages()` API**: Single API call for context injection
- **Automatic initialization**: Seamless setup without manual intervention

### 4. Validation and Health Checks
- **Doctor checks**: OpenCode configuration validation
- **Claude settings location fixes**: Proper path resolution for settings files

### 5. Session Management Improvements
- **Polecat session stability** (from #1103): Fixes for role_agents resolution
- **Hooks auto-fill**: Automatic prompt passing and context management

### 6. Multi-Runtime Support
- **AGENTS.md support**: Runtime-agnostic agent definitions
- **Flexible deployment**: Works across different execution environments

## Strategic Recommendation

**Adopt a HYBRID approach: Build on #775/#1183 as base, contribute our unique features as independent PRs.**

### Rationale
- **Architectural foundation**: #775's provider-agnostic refactor is more complete than our current approach
- **Community momentum**: Active discussion and PR #1183's working implementation
- **Merge drift reality**: Main branch evolution makes waiting for #775 merge risky
- **Value proposition**: Our E2E tests and completion signaling would benefit ANY plugin approach

### Recommended Strategy
1. **Use #1183 as integration base**: Their resolved merge conflicts and working implementation
2. **Port unique contributions as targeted PRs**:
   - **E2E test infrastructure** (highest upstream value - real testing vs mocks)
   - **gt_done tool + dual completion detection** (immediate reliability improvement)
   - **Documentation enhancements** (low-risk, high-value addition)
3. **Independent development**: Don't wait for #775 merge - prepare contributions now
4. **Monitor upstream**: Track #775 progress while developing independent PRs

### Risk Assessment
- **Low risk**: Our contributions are additive, not conflicting with upstream architecture
- **High value**: E2E testing infrastructure would be valuable to any OpenCode implementation
- **Flexible timing**: Can contribute regardless of #775 merge status

## Concrete Next Steps

### Immediate Actions (This Week)
1. **Set up upstream remote**: Add steveyegge/gastown as remote on VPS clone
2. **Fetch target branches**: Pull #775 and #1183 branches locally
3. **Local verification**: Build and test #1183 to confirm working OpenCode support

### Contribution Preparation (Next 2 Weeks)
4. **Extract E2E harness**: Create standalone PR for `internal/e2e/` infrastructure
5. **Extract completion tools**: Separate PR for `gt_done` tool and dual detection
6. **Documentation PR**: Submit improved docs/opencode/ content upstream

### Upstream Engagement (Ongoing)
7. **Open gap issues**: Document missing features from our analysis
8. **Monitor PR activity**: Track #775, #1103, #1183 progress and maintainer responses
9. **Community participation**: Engage in PR discussions, offer testing feedback

### Long-term Planning
10. **Architecture alignment**: Plan migration to #775's provider-agnostic design
11. **Integration testing**: Validate our contributions work with upstream base
12. **Documentation sync**: Keep docs/opencode/ updated with upstream changes

---

*This analysis is based on PR states as of 2026-02-08. Upstream development is active and may change rapidly.*