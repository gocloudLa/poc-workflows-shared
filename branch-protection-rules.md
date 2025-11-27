# Branch Protection Rules Reference

This document describes the recommended branch protection rules for repositories using the centralized CI/CD workflows.

## develop Branch

### Protection Rules

1. **Require pull request reviews before merging**
   - Required number of reviewers: 1
   - Dismiss stale pull request approvals when new commits are pushed: Yes
   - Require review from Code Owners: Optional

2. **Require status checks to pass before merging**
   - Require branches to be up to date before merging: Yes
   - Required status checks:
     - `pull-request-events / workflow-summary`
     - Note: This single check validates that all PR validation jobs passed:
       - Conventional Commit Check
       - Linter
       - Security Scan
       - Unit Tests
       - Coverage Validation

3. **Require conversation resolution before merging**: Optional

4. **Allow specified actors to bypass required pull requests**
   - Add team leaders or administrators who can bypass rules

5. **Do not allow bypassing the above settings**: No (to allow leaders to bypass)

### Rules Configuration

```
Settings → Branches → develop → Edit

✓ Require a pull request before merging
  ✓ Require approvals: 1
  ✓ Dismiss stale pull request approvals when new commits are pushed
  ✓ Require review from Code Owners (optional)

✓ Require status checks to pass before merging
  ✓ Require branches to be up to date before merging
  ✓ Required status checks:
    - pull-request-events / workflow-summary

✓ Allow specified actors to bypass required pull requests
  [Add team leaders]

✓ Do not allow bypassing the above settings: No
```

## main Branch

### Protection Rules

1. **Require pull request reviews before merging**
   - Required number of reviewers: 2 (recommended for production)
   - Dismiss stale pull request approvals when new commits are pushed: Yes
   - Require review from Code Owners: Yes (recommended)

2. **Require status checks to pass before merging**
   - Require branches to be up to date before merging: Yes
   - Required status checks:
     - `pull-request-events / workflow-summary`
     - Note: This single check validates that all PR validation jobs passed:
       - Conventional Commit Check
       - Linter
       - Security Scan
       - Unit Tests
       - Coverage Validation
     - Important: Push workflows (`release-and-changelog`, `build-and-push`, `staging-validation`) run AFTER merge on push events, not during PR validation, so they don't appear as status checks

3. **Require conversation resolution before merging**: Yes

4. **Require linear history**: Yes (recommended)

5. **Include administrators**: No (recommended for PCI compliance)

6. **Allow specified actors to bypass required pull requests**
   - Add team leaders or administrators who can bypass rules

7. **Do not allow bypassing the above settings**: No (to allow leaders to bypass in emergencies)

### Rules Configuration

```
Settings → Branches → main → Edit

✓ Require a pull request before merging
  ✓ Require approvals: 2
  ✓ Dismiss stale pull request approvals when new commits are pushed
  ✓ Require review from Code Owners

✓ Require status checks to pass before merging
  ✓ Require branches to be up to date before merging
  ✓ Required status checks:
    - pull-request-events / workflow-summary

✓ Require conversation resolution before merging

✓ Require linear history

✓ Include administrators: No

✓ Allow specified actors to bypass required pull requests
  [Add team leaders]

✓ Do not allow bypassing the above settings: No
```

## PCI Compliance Considerations

For PCI-compliant services:

1. **Increase required reviewers** for main branch to 3-4
2. **Require review from security team** (Code Owners)
3. **Do not allow administrators to bypass** (set "Include administrators" to No)
4. **Require all status checks** (no optional checks)
5. **Enable branch protection for all branches** that can be merged to main

## Non-PCI Services

For non-PCI services, you can relax some rules:

1. **Required reviewers**: 1-2
2. **Allow administrators to bypass**: Optional
3. **Code Owners requirement**: Optional

## Emergency Bypass

Team leaders can bypass branch protection rules in emergencies. This should be:
- Documented in incident reports
- Reviewed in post-mortem meetings
- Used sparingly

## Implementation Steps

1. Go to repository Settings → Branches
2. Click "Add rule" or edit existing rule
3. Configure rules as described above
4. Add team leaders to bypass list (if applicable)
5. Save changes
6. Test with a test PR to ensure rules work correctly

## Notes

- Branch protection rules are repository-specific
- Rules apply to all branches matching the pattern
- Use wildcards for pattern matching (e.g., `feature/*`)
- Rules can be different for different branch patterns
- **Status Checks**: Only PR workflows appear as status checks. Push workflows run after merge and don't block PRs
- **workflow-summary**: This is the final job in the PR workflow that validates all previous jobs passed. It's the single source of truth for PR validation

