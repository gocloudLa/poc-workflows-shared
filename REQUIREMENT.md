# Development Lifecycle Summary

## 1. Feature Development

1. **Create a feature branch** from `develop`:
   ```bash
   git checkout -b feature/my-new-feature develop
   ```

2. **Commit changes** using **Conventional Commits** format:
   - `feat: add user authentication`
   - `fix: resolve timeout issue`
   - `chore: update dependencies`
   - `ci: improve workflow performance`

3. **Open a Pull Request** to `develop`:
   - **Merge Strategy**: Squash merge (enforced)
   - **Required Checks**:
     - ✅ Conventional Commit validation
     - ✅ Linter (PHP or Python)
     - ✅ Security Scan:
       - **Python**: CodeQL, Dependency Review, Trivy
       - **PHP**: Dependency Review, Trivy
     - ✅ Unit Tests
     - ✅ Coverage Validation (minimum 80%)
   - **Merge Requirements**: Requires approval and all checks must pass

## 2. Integration (Develop Branch)

Upon merge to `develop`, the following automated pipeline runs:

1. **Security Scan**:
   - **Python**: CodeQL analysis, dependency review, and container scanning
   - **PHP**: Dependency review and container scanning
2. **Build**: Docker image is built and pushed to ECR with tags:
   - Commit SHA (e.g., `abc123def456`)
   - `develop-latest`
3. **Deploy to Dev**: Automatically deploys to **Dev** environment
4. **Deploy Verification**: Validates deployment success
5. **Health Check**: Verifies service health after deployment

## 3. Release (Main Branch)

1. **Open a Pull Request** from `develop` to `main`
2. **Upon merge to `main`**, the following automated pipeline runs:

   **Phase 1: Tag Creation & Build**
   - **Release & Changelog**: Creates a new version tag (e.g., `v1.2.0`) and generates changelog
   - **Build**: Docker image is built and pushed to ECR with version tag (e.g., `v1.2.0`)

   **Phase 2: Staging Deployment & Validation**
   - **Deploy to Staging**: Deploys to **Staging** environment using the version tag
   - **Staging Verification**: Validates deployment success
   - **Staging Health Check**: Verifies service health
   - **Staging Validation**: Runs automated validation tests
   - **Rollback**: Automatic rollback if validation fails
   - **Ready for Production**: Version is marked as ready for production deployment

3. **Manual Release Creation** (after staging validation passes):
   - Go to **Actions** tab: `https://github.com/<org>/<repo>/actions`
   - Select **"Create Release"** workflow
   - Click **"Run workflow"** button
   - Enter the tag name from Phase 1 (e.g., `v1.2.0`)
   - Optionally add release notes
   - Click **"Run workflow"** to create the release

4. **Production Deployment** (triggered by release event):
   - **Automatic Trigger**: Creating the GitHub Release automatically triggers production deployment
   - **Deploy to Production**: Deploys to **Production** environment using the release tag
   - **Production Verification**: Validates deployment success
   - **Production Health Check**: Verifies service health
   - **Rollback**: Automatic rollback if health check fails

## 4. Hotfix Process

1. **Create a hotfix branch** from `main`:
   ```bash
   git checkout -b hotfix/critical-bug main
   ```

2. **Fix, commit, and open Pull Request** to `main`
3. **Follows the Release flow** (same as main branch merge)
4. **Important**: After merging to `main`, **backport the fix** to `develop`:
   ```bash
   git checkout develop
   git cherry-pick <commit-hash>
   git push origin develop
   ```

---

## Supported Languages

This workflow system currently supports:
- **PHP** (default: 8.2, configurable)
- **Python** (default: 3.11, configurable)

## Coverage Requirements

- **Minimum Coverage**: 80% (configurable via `coverage_threshold` input)
- **Coverage Reports**: Generated in `coverage.xml` format
