# 🌿 Git Workflow & Branching Strategy Cheat Sheet

> **Purpose:** Standardized Git workflow, branching strategy, commit conventions, and release management for enterprise teams.
> **Stack context:** Trunk-based development / GitHub / CI/CD / Feature flags

---

## 📋 Strategy Selection

| Team Size | Release Cadence | Strategy |
|-----------|----------------|----------|
| 1-5 devs, continuous deploy | Daily/on-demand | **Trunk-based** ✅ Recommended |
| 5-15 devs, weekly releases | Weekly | Trunk-based + release branches |
| 15+ devs, scheduled releases | Bi-weekly/monthly | GitFlow (more ceremony) |

---

## 🌳 Pattern 1: Trunk-Based Development

```
main (trunk) ─────●────●────●────●────●────●────●──── (always deployable)
                  │         │              │
                  └─ feat ──┘              └─ fix ──┘
                   (short-lived)           (short-lived)

RULES:
  • main is ALWAYS deployable
  • Feature branches live < 2 days
  • Merge via squash PR (clean history)
  • Feature flags for incomplete work
  • CI runs on every commit to main
  • Deploy from main (automated)
```

### Branch Naming

```
Feature:     feat/PAY-1234-add-idempotency-check
Bug fix:     fix/PAY-1235-retry-offset-commit
Hotfix:      hotfix/PAY-1236-null-pointer-gateway
Refactor:    refactor/PAY-1237-extract-fee-calculator
Chore:       chore/upgrade-spring-boot-3.3
Release:     release/2.1.0
```

---

## 📝 Pattern 2: Conventional Commits

```
Format: <type>(<scope>): <subject>

Types:
  feat:     New feature                    → MINOR version bump
  fix:      Bug fix                        → PATCH version bump
  perf:     Performance improvement        → PATCH
  refactor: Code change (no feature/fix)   → no version bump
  test:     Adding/fixing tests            → no version bump
  docs:     Documentation only             → no version bump
  chore:    Build, tooling, deps           → no version bump
  ci:       CI/CD changes                  → no version bump
  revert:   Revert a previous commit       → varies

  BREAKING CHANGE: in footer               → MAJOR version bump

Examples:
  feat(payment): add idempotency key validation
  fix(kafka): correct offset commit on batch error
  perf(mongo): add compound index for customer lookup
  refactor(gateway): extract retry logic to decorator
  test(payment): add circuit breaker state transition tests
  chore(deps): upgrade resilience4j to 2.2.0

  feat(api)!: change payment response schema
  BREAKING CHANGE: removed deprecated 'fee' field, use 'fees.total'
```

---

## 🔄 Pattern 3: PR Workflow

```
1. Create branch from main
   git checkout -b feat/PAY-1234-idempotency-check

2. Develop (small, focused commits)
   git commit -m "feat(payment): add idempotency key to request"
   git commit -m "test(payment): verify duplicate rejection"

3. Push and create PR
   git push origin feat/PAY-1234-idempotency-check

4. PR must pass:
   [ ] CI pipeline green (build, unit tests, integration tests)
   [ ] Code review approved (1-2 reviewers)
   [ ] No merge conflicts with main
   [ ] Branch is up to date with main

5. Squash merge to main
   → Single clean commit: "feat(payment): add idempotency key validation (#1234)"

6. Delete branch
   → Automatically after merge
```

### PR Template

```markdown
## What
Brief description of the change.

## Why
Link to ticket: PAY-1234
Context on why this change is needed.

## How
Technical approach taken.

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manually tested locally

## Rollback
How to rollback if this causes issues.
Feature flag: `payment.idempotency.enabled`

## Checklist
- [ ] No secrets in code
- [ ] Error handling for all external calls
- [ ] Metrics/logging added for observability
- [ ] Documentation updated (if API change)
```

---

## 🚀 Pattern 4: Release Management

### Semantic Versioning

```
MAJOR.MINOR.PATCH

MAJOR: Breaking API changes (1.0.0 → 2.0.0)
  • Removed endpoints, changed response schemas
  • Backwards-incompatible changes

MINOR: New features, backwards compatible (1.0.0 → 1.1.0)
  • New endpoints, optional fields added
  • New functionality

PATCH: Bug fixes, backwards compatible (1.0.0 → 1.0.1)
  • Bug fixes, performance improvements
  • Security patches
```

### Release Branch Strategy (When Needed)

```
main ─────●────●────●────●────●────●────●────●─────
               │                   │
               └─ release/2.0.0 ──●──● (only bugfixes)
                                     │
                                     └── tag: v2.0.0

WHEN to use release branches:
  • Need to stabilize before release
  • Need to maintain multiple versions
  • Regulatory/compliance review period

WHEN NOT to use:
  • Continuous deployment (just deploy main)
  • Single version in production
```

### Hotfix Flow

```
main ─────●────●────●────●──────────●─────
                              │      ↑
                              └─ hotfix/PAY-1236
                                 (cherry-pick to main)

1. Branch from latest release tag (or main)
2. Fix the issue
3. PR → merge to main
4. If release branch exists, cherry-pick to release branch
5. Deploy immediately
```

---

## 🏷️ Pattern 5: Feature Flags

```java
// Simple property-based flag
@ConfigurationProperties(prefix = "feature")
public record FeatureFlags(
    boolean idempotencyCheckEnabled,
    boolean newFraudEngineEnabled,
    int canaryPercentage
) {
    public FeatureFlags {
        if (canaryPercentage < 0 || canaryPercentage > 100) canaryPercentage = 0;
    }
}

// Usage: deploy incomplete code behind flag
@Service
public class PaymentService {

    private final FeatureFlags flags;

    public PaymentResult process(Payment payment) {
        if (flags.newFraudEngineEnabled()) {
            return newFraudEngine.evaluate(payment);  // New code, behind flag
        }
        return legacyFraudCheck.evaluate(payment);    // Existing code
    }
}

// Lifecycle:
// 1. Deploy with flag OFF → code in production but inactive
// 2. Enable flag in staging → test with real-ish traffic
// 3. Enable flag for 10% (canary) → monitor metrics
// 4. Enable flag for 100% → full rollout
// 5. Remove flag + old code → cleanup
```

---

## ⚡ Essential Git Commands

```bash
# ── Daily workflow ──
git checkout -b feat/PAY-1234-description     # New branch
git add -p                                     # Stage interactively (review each hunk)
git commit -m "feat(scope): description"       # Conventional commit
git push origin feat/PAY-1234-description      # Push for PR

# ── Keep branch updated ──
git fetch origin
git rebase origin/main                         # Rebase onto latest main
git push --force-with-lease                    # Safe force push (won't overwrite others)

# ── Undo mistakes ──
git reset --soft HEAD~1                        # Undo last commit, keep changes staged
git stash                                      # Temporarily shelve changes
git stash pop                                  # Restore stashed changes
git revert <sha>                               # Create new commit undoing a previous one

# ── Investigation ──
git log --oneline --graph -20                  # Visual recent history
git blame -L 50,60 PaymentService.java         # Who changed these lines
git bisect start                               # Binary search for bug introduction
```

---

## 🚫 Git Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Long-lived feature branches (weeks) | < 2 day branches + feature flags |
| Merge commits (messy history) | Squash merge PRs |
| "WIP" or "fix" commit messages | Conventional commits |
| Force push to main | Branch protection rules |
| Large PRs (1000+ lines) | Break into focused PRs (< 400 lines) |
| Committing secrets | Pre-commit hooks (git-secrets, gitleaks) |
| No branch protection | Require PR + CI + review |

---

## 💡 Golden Rules

```
1.  main is ALWAYS deployable — broken main = broken team.
2.  SMALL branches, FAST merges — < 2 days, < 400 lines.
3.  CONVENTIONAL COMMITS — automated changelogs, clear history.
4.  SQUASH MERGE — one commit per feature, clean main history.
5.  FEATURE FLAGS for incomplete work — deploy code before it's "done."
6.  REBASE before PR — clean history, no merge conflicts for reviewer.
7.  PR TEMPLATE — consistent quality, nothing forgotten.
8.  BRANCH PROTECTION on main — no direct pushes, require CI + review.
9.  AUTOMATE releases — conventional commits → semantic version → deploy.
10. DELETE branches after merge — branch list = active work only.
```

---

*Last updated: February 2026*
