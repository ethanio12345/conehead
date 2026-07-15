# OpenSpec Integration Guide

How OpenSpec integrates with the conehead project's verification and implementation workflows.

## When to Propose a Change

A proposal (`/opsx-propose`) should be created when:

1. **Verification finding rated Critical or Warning** — physics issues that need fixing
2. **New feature design** — before implementing any new calculation step
3. **Algorithm change** — modifying how an existing step works
4. **Parameter change** — updating beam model or kernel parameters

## Propose Workflow

```
/opsx-propose {feature-name}
```

**Required in the proposal:**
- Description of what needs to change
- Reference equations from papers (with equation numbers)
- Expected behavior vs. current behavior
- Affected modules and files
- Proposed implementation approach

**Before running any /opsx command, you MUST load two skills:**

1. `openspec-workflow` — the standard workflow framework
2. The stage-specific skill (e.g., for propose stage)

```python
skill("openspec-workflow")  # Load first — always required
# Then proceed with the /opsx command
```

## Apply Workflow

After a proposal is reviewed and accepted:

```
/opsx-apply {feature-name}
```

This creates the implementation plan with specific tasks:
- Task breakdown with file-level deliverables
- Dependencies between tasks
- Acceptance criteria for each task
- Reference files and context files to load

The apply stage reads the proposal and generates a structured task list in `openspec/changes/{name}/`.

## Verify Workflow

After implementation is complete:

```
/opsx-verify {feature-name}
```

Checks that:
- All tasks from the apply stage are completed
- Acceptance criteria are met
- Tests pass
- No regressions introduced

## Archive Workflow

After verification passes:

```
/opsx-archive {feature-name}
```

Closes the change and:
- Moves from active to archive
- Updates milestone tracker
- Records the change in project history

## State Tracking

### Verification Progress

File: `context/lookup/verification-dashboard.md`

| Column | Description |
|--------|-------------|
| Module | Which module was verified |
| Status | Pending → In Progress → Pass/Issues Found |
| Critical/Warning/Info | Count of findings by severity |
| OpenSpec Ref | Link to proposal if issues found |

### Milestone Progress

File: `context/lookup/milestone-tracker.md`

| Column | Description |
|--------|-------------|
| Milestone | Feature or calculation step |
| Status | ✅ Complete / ⬜ Remaining / 🔧 In Progress |
| OpenSpec Ref | Link to active change |

### OpenSpec Changes

Location: `openspec/changes/{name}/`

Each change directory contains:
- `PROPOSAL.md` — what and why
- `DESIGN.md` — how (if design stage was needed)
- `SPEC.md` — acceptance criteria and tasks
- `STATE.md` — current status of the change

## Reading State

The orchestrator determines current project state by reading:

1. **verification-dashboard.md** — which modules have been verified
2. **milestone-tracker.md** — which features are complete, in progress, remaining
3. **OpenSpec state** — active changes and their stage

This gives a complete picture of: what's done, what's being worked on, and what's pending.

## Command Flow Summary

```
1. skill("openspec-workflow")          # ALWAYS load first
2. /opsx-propose {name}                # Create proposal
3. Review proposal
4. /opsx-apply {name}                  # Create implementation plan
5. Implement tasks
6. /opsx-verify {name}                 # Verify implementation
7. /opsx-archive {name}                # Close and archive
```

## Integration with Verification Workflow

The verification workflow (`context/guides/verification-workflow.md`) produces findings. The OpenSpec workflow tracks fixes for those findings:

```
Verification → Critical/Warning Finding → /opsx-propose → /opsx-apply → Implement → /opsx-verify → /opsx-archive
```

Each step updates the verification dashboard and milestone tracker.
