# AGENTS.md - AI Agent Development Guide

**Project:** ansible_traefik_docker
**Type:** Ansible Galaxy Role
**Primary Language:** YAML (Ansible)
**Testing Framework:** Molecule with Docker-in-Docker
**Last Updated:** 2025-12-29

---

## Table of Contents

- [Project Overview](#project-overview)
- [Development Environment Setup](#development-environment-setup)
- [Quality Tools](#quality-tools)
- [Testing with Molecule](#testing-with-molecule)
- [Development Workflows](#development-workflows)
- [CI/CD Pipeline](#cicd-pipeline)
- [Common Tasks](#common-tasks)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Project Overview

### Purpose
Ansible role for deploying Traefik v3.6.5 reverse proxy as a Docker container with automatic HTTPS via Let's Encrypt.

### Project Structure
```
ansible_traefik_docker/
├── defaults/main.yml          # Role variables and configuration
├── tasks/
│   ├── main.yml              # Task orchestration
│   ├── prepare_host.yml      # Host preparation (directories, files)
│   └── provision_traefik.yml # Docker deployment
├── meta/main.yml             # Ansible Galaxy metadata
├── molecule/default/         # Testing framework
│   ├── Dockerfile            # Custom test image (Debian 12 + Docker)
│   ├── molecule.yml          # Test configuration
│   ├── prepare.yml           # Test environment setup
│   ├── converge.yml          # Role execution
│   └── verify.yml            # Post-deployment validation
├── .github/workflows/
│   └── molecule.yml          # GitHub Actions CI/CD
├── .yamllint                 # YAML linting configuration
├── requirements.txt          # Python dependencies
└── README.md                 # User documentation
```

### Key Technologies
- **Ansible Core:** >= 2.13.5
- **Collections:** community.docker, community.general
- **Testing:** Molecule 25.0.0+ with Docker driver
- **Linting:** ansible-lint 25.0.0+, yamllint 1.28.0+
- **Target:** Debian 12 (Bookworm) with Docker

---

## Development Environment Setup

### 1. Prerequisites

**System Requirements:**
- Python 3.8+ (recommended: 3.11 or higher)
- Docker Engine (for Molecule testing)
- Git
- GitHub CLI (gh) for PR workflows (optional but recommended)

**Host OS Compatibility:**
- Linux (native Docker support)
- macOS (Docker Desktop)
- Windows (WSL2 + Docker Desktop)

### 2. Initial Setup

```bash
# Clone repository
git clone https://github.com/tripleawwy/ansible_traefik_docker.git
cd ansible_traefik_docker

# Create Python virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install Python dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Install Ansible collections
ansible-galaxy collection install community.docker community.general

# Install GitHub CLI (optional, for PR workflows)
# Ubuntu/Debian
sudo apt install gh

# macOS
brew install gh

# Authenticate with GitHub
gh auth login
```

### 3. Verify Installation

**IMPORTANT:** Always activate the virtual environment before running any tools:

```bash
# Activate venv (required for every new shell session)
source .venv/bin/activate

# Check Ansible version
ansible --version

# Check Molecule installation
molecule --version

# Verify linting tools
ansible-lint --version
yamllint --version

# Test Docker connectivity
docker info

# Verify GitHub CLI (optional)
gh --version
```

---

## Quality Tools

### YAML Linting (yamllint)

**Configuration:** `.yamllint`

**Key Rules:**
- Line length: max 120 characters
- Braces/brackets: max 1 space inside
- Truthy values: `["true", "false", "yes", "no"]` only
- New lines: Unix format (LF)
- Octal values: forbidden (both implicit and explicit)

**Run Manually:**
```bash
# Activate venv first
source .venv/bin/activate

# Lint all YAML files
yamllint .

# Lint specific file
yamllint defaults/main.yml

# Lint with custom config
yamllint -c .yamllint tasks/
```

**Common Issues:**
```yaml
# ❌ Wrong: mixed quotes, long lines
traefik_api_rule: 'Host(`traefik.{{ inventory_hostname }}`) && PathPrefix(`/api`) || PathPrefix(`/dashboard`)'

# ✅ Right: consistent quotes, readable
traefik_api_rule: "Host(`traefik.{{ inventory_hostname }}`)"
```

### Ansible Linting (ansible-lint)

**Configuration:** Uses Molecule's built-in configuration

**Run Manually:**
```bash
# Activate venv first
source .venv/bin/activate

# Lint entire role
ansible-lint

# Lint specific playbook
ansible-lint molecule/default/converge.yml

# Auto-fix issues (use cautiously)
ansible-lint --write
```

**Key Checks:**
- Production profile compliance
- FQCN usage (e.g., `ansible.builtin.file`, not `file`)
- Task naming conventions
- Handler naming and usage
- Idempotency patterns

**Common Fixes:**
```yaml
# ❌ Wrong: short-form module name
- copy:
    src: config.yml
    dest: /etc/

# ✅ Right: FQCN (Fully Qualified Collection Name)
- name: Copy configuration file
  ansible.builtin.copy:
    src: config.yml
    dest: /etc/config.yml
    mode: '0644'
```

---

## Testing with Molecule

### Architecture: Docker-in-Docker (DinD)

**Why DinD?**
This role deploys Docker containers, so tests need Docker running *inside* the test container to validate real deployment behavior.

**Test Platform:**
- **Base Image:** `geerlingguy/docker-debian12-ansible:latest`
- **Docker Engine:** CE (Community Edition) via official repository
- **Init System:** systemd (PID 1)
- **Storage Driver:** VFS (for nested Docker compatibility)

### Molecule Test Phases

```
molecule test
├── 1. Dependency  → Install Galaxy collections
├── 2. Lint        → yamllint + ansible-lint
├── 3. Syntax      → Ansible syntax validation
├── 4. Create      → Build Docker image, start container
├── 5. Prepare     → Configure Docker daemon (VFS storage)
├── 6. Converge    → Execute role
├── 7. Idempotence → Re-run role (expect 0 changes)
├── 8. Verify      → Run verification tests
└── 9. Destroy     → Clean up container
```

### Running Tests

**Full Test Suite:**
```bash
molecule test
```

**Development Iteration (faster):**
```bash
# Create environment once
molecule create

# Make changes to role...

# Test changes (skip create/destroy)
molecule converge

# Run verification
molecule verify

# Debug inside container
molecule login

# Clean up when done
molecule destroy
```

**Selective Testing:**
```bash
# Linting only
molecule lint

# Syntax check only
molecule syntax

# Skip destroy on failure (for debugging)
molecule test --destroy=never

# Specific scenario
molecule test -s default
```

### Verification Tests (verify.yml)

**Validated Conditions:**
1. ✅ Traefik container exists
2. ✅ Container status is "running"
3. ✅ Container is actively running (not just created)
4. ✅ Traefik Docker network exists
5. ✅ `acme.json` file exists at `/etc/traefik/acme.json`
6. ✅ File permissions are `0600` (owner read/write only)

**Test Output Example:**
```
TASK [Assert Traefik container is running] ************************************
ok: [traefik-debian12] => {
    "changed": false,
    "msg": "Traefik container is running"
}
```

### Docker-in-Docker Configuration

**Storage Driver (VFS):**
```json
{
  "storage-driver": "vfs"
}
```

**Why VFS?**
- Nested overlayfs not supported (cannot run overlay2 inside overlay2)
- VFS works reliably in Docker-in-Docker scenarios
- Performance trade-off: slower but more stable for testing

**Privileged Mode Required:**
```yaml
# molecule.yml
platforms:
  - name: traefik-debian12
    privileged: true          # Required for DinD
    capabilities:
      - SYS_ADMIN             # Container operations
    security_opts:
      - seccomp=unconfined    # Allow Docker syscalls
```

---

## Development Workflows

### Workflow 1: Adding New Feature

```bash
# 1. Ensure venv is active
source .venv/bin/activate

# 2. Create feature branch
git checkout -b feature/add-prometheus-metrics

# 3. Modify role (e.g., defaults/main.yml)
# Add new variables and configuration

# 4. Update tasks (e.g., tasks/provision_traefik.yml)
# Implement feature logic

# 5. Run linting
yamllint .
ansible-lint

# 6. Test changes
molecule test

# 7. Verify idempotence
molecule converge  # Should show 0 changes on second run

# 8. Update documentation
# Edit README.md with new variables

# 9. Commit changes
git add .
git commit -m "feat: add Prometheus metrics support"

# 10. Push and create PR (using gh CLI)
git push origin feature/add-prometheus-metrics
gh pr create --title "feat: add Prometheus metrics support" \
  --body "## Summary
- Add traefik_metrics_enabled variable
- Configure Prometheus metrics endpoint
- Update documentation

## Testing
- Molecule tests passing
- Idempotency verified
- Lint checks passed

## Related Issues
Closes #123"

# Alternative: Push and create PR manually
# git push origin feature/add-prometheus-metrics
# Then create PR via GitHub web interface
```

### Workflow 2: Fixing Bug

```bash
# 1. Create bugfix branch
git checkout -b fix/acme-permissions

# 2. Reproduce issue locally
molecule create
molecule converge
molecule login
# Debug inside container

# 3. Implement fix

# 4. Test fix
molecule converge
molecule verify

# 5. Ensure idempotency
molecule converge  # Should be idempotent

# 6. Run full test suite
molecule destroy
molecule test

# 7. Commit fix
git commit -m "fix: ensure acme.json permissions are 0600"
```

### Workflow 3: Updating Dependencies

```bash
# 1. Update requirements.txt
# Change version: molecule>=25.0.0 → molecule>=26.0.0

# 2. Update virtual environment
pip install -r requirements.txt --upgrade

# 3. Test compatibility
molecule test

# 4. Update CI/CD if needed
# Edit .github/workflows/molecule.yml

# 5. Document changes in commit
git commit -m "chore: upgrade molecule to v26"
```

### Workflow 4: Refactoring Tasks

```bash
# 1. Ensure tests pass before refactoring
molecule test

# 2. Refactor code (maintain functionality)
# Split tasks, improve naming, etc.

# 3. Verify behavior unchanged
molecule converge
molecule verify

# 4. Check idempotency
molecule converge  # Must show 0 changes

# 5. Lint refactored code
ansible-lint

# 6. Full regression test
molecule test
```

---

## Pull Request Workflow

### Creating a Pull Request with gh CLI

**Prerequisites:**
```bash
# Install and authenticate gh CLI (one-time setup)
gh auth login
```

**Standard PR Creation:**
```bash
# 1. Ensure clean working tree and branch
git status

# 2. Push branch to origin
git push origin feature/your-feature-name

# 3. Create PR with gh CLI
gh pr create \
  --title "feat: descriptive title of your change" \
  --body "## Summary
Describe what this PR does

## Changes
- Change 1
- Change 2
- Change 3

## Testing
- [ ] Molecule tests passing
- [ ] Linting passed (yamllint + ansible-lint)
- [ ] Idempotency verified
- [ ] Documentation updated

## Related Issues
Closes #123"

# 4. View PR in browser
gh pr view --web
```

**Quick PR Creation (interactive):**
```bash
# Push and create PR interactively
git push origin feature/your-feature-name
gh pr create
# Follow interactive prompts for title and body
```

**Draft PR (for work in progress):**
```bash
gh pr create --draft \
  --title "WIP: feature description" \
  --body "Work in progress, not ready for review"
```

**PR with Reviewers:**
```bash
gh pr create \
  --title "feat: add DNS challenge support" \
  --body "..." \
  --reviewer tripleawwy \
  --assignee @me
```

### Viewing and Managing PRs

```bash
# List all PRs
gh pr list

# View specific PR
gh pr view 42

# View PR in browser
gh pr view 42 --web

# Check PR status
gh pr status

# View PR diff
gh pr diff 42

# View PR checks (CI status)
gh pr checks 42
```

### Updating an Existing PR

```bash
# Make additional changes
git add .
git commit -m "fix: address review feedback"
git push origin feature/your-feature-name
# PR automatically updates

# Force push (use with caution)
git push origin feature/your-feature-name --force-with-lease
```

### Merging PRs

```bash
# Merge PR (requires approval if configured)
gh pr merge 42

# Merge with squash
gh pr merge 42 --squash

# Merge and delete branch
gh pr merge 42 --squash --delete-branch

# Auto-merge when checks pass
gh pr merge 42 --auto --squash
```

### Example: Complete Feature PR Workflow

```bash
# 1. Activate venv
source .venv/bin/activate

# 2. Create and checkout feature branch
git checkout -b feature/dns-challenge-support

# 3. Make changes...
# Edit files...

# 4. Run quality checks
yamllint .
ansible-lint

# 5. Run tests
molecule test

# 6. Commit changes
git add .
git commit -m "feat: add DNS challenge support with environment variables"

# 7. Push and create PR
git push origin feature/dns-challenge-support
gh pr create \
  --title "feat: add DNS challenge support for Let's Encrypt" \
  --body "## Summary
Add support for DNS challenge authentication using provider-specific environment variables.

## Changes
- Add traefik_acme_challenge_type variable (http or dns)
- Add traefik_acme_dns_provider variable for provider selection
- Add traefik_acme_dns_env for environment variables
- Update provision_traefik.yml to pass env vars to container
- Add comprehensive documentation with examples

## Testing
- [x] Molecule tests passing
- [x] yamllint passed
- [x] ansible-lint passed (production profile)
- [x] Idempotency verified
- [x] Documentation updated

## DNS Providers Tested
- Cloudflare
- DigitalOcean
- AWS Route53

## Related Issues
Closes #45"

# 8. View PR in browser
gh pr view --web
```

### Reviewing Previous PRs

To learn from previous PR patterns:

```bash
# List merged PRs
gh pr list --state merged --limit 20

# View specific merged PR
gh pr view 2

# See commits from previous PR
gh pr view 2 --json commits -q '.commits[].commit.message'

# Example from previous PRs:
# PR #2: "refactor(testing): migrate from Vagrant to Docker-in-Docker testing"
# PR #1: "refactor"
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

**File:** `.github/workflows/molecule.yml`

**Triggers:**
- Push to `main` branch
- Pull requests targeting `main`

**Pipeline Steps:**
1. Checkout code
2. Set up Python 3.11 with pip cache
3. Install Python dependencies (`requirements.txt`)
4. Install Ansible Galaxy collections
5. Run `molecule test`
6. Upload logs on failure (artifact retention)

**Environment Variables:**
- `PY_COLORS='1'` - Colored output for readability
- `ANSIBLE_FORCE_COLOR='1'` - Colored Ansible output

**Artifact Upload (on failure):**
```yaml
- name: Upload Molecule logs on failure
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: molecule-logs
    path: molecule/
```

### Local CI Simulation

```bash
# Replicate CI environment locally
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
ansible-galaxy collection install community.docker community.general
molecule test

# Check what CI will do
cat .github/workflows/molecule.yml
```

---

## Common Tasks

### Task 1: Add New Variable

**Steps:**
1. Add to `defaults/main.yml`:
   ```yaml
   # New feature variable
   traefik_enable_metrics: "false"
   ```

2. Update CLI commands:
   ```yaml
   traefik_cli_base:
     - "--metrics.prometheus={{ traefik_enable_metrics }}"
   ```

3. Document in `README.md`:
   ```markdown
   | `traefik_enable_metrics` | `"false"` | Enable Prometheus metrics |
   ```

4. Test:
   ```bash
   molecule converge
   molecule verify
   ```

### Task 2: Modify Task Logic

**Example: Add retry logic**

```yaml
# Before
- name: Pull Traefik docker image
  community.docker.docker_image:
    name: "{{ traefik_image }}"
    source: pull

# After
- name: Pull Traefik docker image
  community.docker.docker_image:
    name: "{{ traefik_image }}"
    source: pull
  register: pull_result
  retries: 3
  delay: 5
  until: pull_result is not failed
```

**Validation:**
```bash
# Test with network simulation
molecule converge
# Check idempotency
molecule converge
```

### Task 3: Add Verification Test

**Edit:** `molecule/default/verify.yml`

```yaml
- name: Check Traefik responds on port 80
  ansible.builtin.uri:
    url: http://localhost:80
    status_code: [404, 301, 302]  # Expecting redirect or 404
  register: http_response

- name: Assert HTTP port accessible
  ansible.builtin.assert:
    that:
      - "http_response.status in [404, 301, 302]"
    success_msg: "Traefik HTTP port accessible"
    fail_msg: "Traefik HTTP port NOT accessible"
```

### Task 4: Update Traefik Version

**Steps:**
1. Update `defaults/main.yml`:
   ```yaml
   traefik_image_tag: "3.2.3"  # Was: "3.6.5"
   ```

2. Test deployment:
   ```bash
   molecule converge
   molecule login
   docker exec traefik traefik version
   ```

3. Run full test suite:
   ```bash
   molecule test
   ```

4. Update `README.md` documentation

5. Commit with semantic versioning:
   ```bash
   git commit -m "chore: bump Traefik to v3.2.3"
   ```

---

## Troubleshooting

### Issue 1: Molecule Test Failures

**Symptom:** Tests fail with Docker daemon errors

**Diagnosis:**
```bash
# Check if Docker is running on host
sudo systemctl status docker

# Verify privileged mode
grep privileged molecule/default/molecule.yml

# Check container logs
molecule create
docker logs traefik-debian12
```

**Solution:**
```bash
# Ensure Docker daemon running
sudo systemctl start docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify Docker access
docker info
```

### Issue 2: Storage Driver Errors

**Symptom:** `failed to mount: invalid argument` or overlayfs errors

**Diagnosis:**
```bash
molecule login
cat /etc/docker/daemon.json
docker info | grep "Storage Driver"
```

**Solution:** Ensure `prepare.yml` sets VFS storage driver:
```yaml
- name: Configure Docker daemon for DinD compatibility
  ansible.builtin.copy:
    content: |
      {
        "storage-driver": "vfs"
      }
    dest: /etc/docker/daemon.json
    mode: '0644'
  notify: Restart docker
```

### Issue 3: Idempotence Failures

**Symptom:** Second `molecule converge` shows changes

**Diagnosis:**
```bash
# Run twice and compare
molecule converge | tee first_run.log
molecule converge | tee second_run.log
diff first_run.log second_run.log
```

**Common Causes:**
- Tasks using `command` without `changed_when: false`
- File timestamps not preserved
- Conditional logic based on time/random values

**Solution:**
```yaml
# Mark command as never changing
- name: Check Docker status
  ansible.builtin.command: docker info
  changed_when: false

# Preserve file timestamps
- name: Create acme.json
  ansible.builtin.file:
    path: "{{ traefik_source_acme_path }}"
    state: touch
    modification_time: preserve  # Important!
    access_time: preserve
```

### Issue 4: Linting Errors

**Symptom:** `ansible-lint` or `yamllint` failures

**Common Errors:**

```yaml
# ❌ fqcn-builtins: Use FQCN for builtin actions
- file:
    path: /tmp/test

# ✅ Fix: Use fully qualified name
- name: Create test file
  ansible.builtin.file:
    path: /tmp/test
    state: touch
```

```yaml
# ❌ name[missing]: All tasks should be named
- copy:
    src: config.yml
    dest: /etc/

# ✅ Fix: Add descriptive name
- name: Copy configuration file
  ansible.builtin.copy:
    src: config.yml
    dest: /etc/config.yml
```

```yaml
# ❌ yaml[line-length]: Line too long (>120 chars)
- name: This is a very long task name that exceeds the maximum allowed line length for YAML files

# ✅ Fix: Break into multiple lines
- name: >
    This is a very long task name that exceeds
    the maximum allowed line length for YAML files
```

### Issue 5: Permission Denied in CI

**Symptom:** GitHub Actions fails with Docker socket permission errors

**Diagnosis:** Check CI logs for:
```
permission denied while trying to connect to the Docker daemon socket
```

**Solution:** GitHub Actions runners have Docker pre-configured, but ensure:

```yaml
# .github/workflows/molecule.yml
jobs:
  molecule-test:
    runs-on: ubuntu-latest  # Has Docker pre-installed

    steps:
      # No need for docker setup, pre-configured
      - name: Run Molecule tests
        run: molecule test
```

---

## Best Practices

### 1. Variable Naming

**Follow Pattern:**
```yaml
# ✅ Good: Prefixed, descriptive, typed
traefik_service_name: "traefik"
traefik_image_tag: "3.6.5"
traefik_dashboard_enabled: "false"

# ❌ Bad: No prefix, unclear, inconsistent
service: "traefik"
version: "3.6.5"
enable_dashboard: false  # Boolean vs string inconsistency
```

### 2. Task Organization

**Principle:** Separation of concerns

```yaml
# tasks/main.yml - Orchestration only
---
- name: Tasks to prepare the host
  ansible.builtin.import_tasks: prepare_host.yml

- name: Tasks to provision the Traefik proxy
  ansible.builtin.import_tasks: provision_traefik.yml
```

**Idempotency First:**
```yaml
# ✅ Idempotent: Creates only if missing
- name: Create acme.json file if not existing
  ansible.builtin.file:
    path: "{{ traefik_source_acme_path }}"
    state: touch
    mode: '0600'
    modification_time: preserve  # Critical for idempotency
    access_time: preserve

# ❌ Not idempotent: Always changes
- name: Create acme.json
  ansible.builtin.command: touch {{ traefik_source_acme_path }}
```

### 3. Testing Strategy

**Test Pyramid:**
1. **Lint** (fastest) - yamllint + ansible-lint
2. **Syntax** - Ansible syntax validation
3. **Converge** - Role execution
4. **Verify** - Post-deployment validation
5. **Idempotence** - Re-run verification

**Development Loop:**
```bash
# Fast iteration (keep container alive)
molecule create          # Once
# Edit code...
molecule converge        # Quick test
molecule verify          # Validation
# Repeat...
molecule destroy         # When done

# Full CI simulation before push
molecule test            # Complete pipeline
```

### 4. Documentation

**Keep In Sync:**
- Code changes → Update `README.md`
- New variables → Document in defaults + README
- New features → Add usage examples
- Breaking changes → Update version + changelog

**Example:**
```yaml
# defaults/main.yml
traefik_log_level: "ERROR"  # Logging level (DEBUG, INFO, WARN, ERROR)

# README.md
| Variable | Default | Description |
|----------|---------|-------------|
| `traefik_log_level` | `"ERROR"` | Logging level (DEBUG, INFO, WARN, ERROR) |
```

### 5. Git Workflow

**Branch Naming:**
```bash
feature/add-metrics      # New features
fix/acme-permissions     # Bug fixes
chore/update-deps        # Maintenance
docs/improve-readme      # Documentation
```

**Commit Messages (Semantic):**
```bash
feat: add Prometheus metrics support
fix: ensure acme.json permissions are 0600
chore: upgrade molecule to v26
docs: add troubleshooting section
refactor: split provision tasks
test: add HTTP port verification
```

### 6. Performance Optimization

**Molecule Speed:**
```bash
# Keep container running during development
molecule create
molecule converge  # Fast: no rebuild

# Use pre-built image (if stable)
# molecule.yml
platforms:
  - name: traefik-debian12
    pre_build_image: true  # Skip rebuild

# Skip destroy for debugging
molecule test --destroy=never
```

**CI Optimization:**
```yaml
# .github/workflows/molecule.yml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'  # Cache pip dependencies
```

---

## Agent-Specific Guidelines

### For AI Code Assistants

**When Modifying This Role:**

1. **Always Read First:**
   - Read existing files before editing
   - Understand variable dependencies
   - Check existing patterns

2. **Maintain Consistency:**
   - Follow existing naming conventions
   - Use FQCN for all modules
   - Match indentation and formatting

3. **Test Changes:**
   - Run `yamllint` before committing
   - Run `ansible-lint` to catch issues
   - Use `molecule converge` for quick testing
   - Full `molecule test` before PR

4. **Preserve Idempotency:**
   - Never use `command` without `changed_when`
   - Preserve file timestamps with `touch`
   - Use `state: present` not `state: touch`

5. **Document Changes:**
   - Update `README.md` for new variables
   - Add verification tests for new features
   - Update this AGENTS.md if workflow changes

### Recommended Tools

**Linting:**
```bash
yamllint .
ansible-lint
```

**Testing:**
```bash
molecule test              # Full suite
molecule converge          # Quick iteration
molecule verify            # Validation only
molecule login             # Debug access
```

**Debugging:**
```bash
molecule login             # SSH into test container
docker logs traefik        # Container logs
docker inspect traefik     # Container details
docker network ls          # Network inspection
```

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Setup environment | `python3 -m venv .venv && source .venv/bin/activate` |
| Install dependencies | `pip install -r requirements.txt` |
| Install collections | `ansible-galaxy collection install community.docker community.general` |
| Activate venv (required) | `source .venv/bin/activate` |
| Lint YAML | `source .venv/bin/activate && yamllint .` |
| Lint Ansible | `source .venv/bin/activate && ansible-lint` |
| Full test | `source .venv/bin/activate && molecule test` |
| Quick test | `source .venv/bin/activate && molecule converge` |
| Verify only | `source .venv/bin/activate && molecule verify` |
| Debug container | `source .venv/bin/activate && molecule login` |
| Clean up | `source .venv/bin/activate && molecule destroy` |
| Create PR (gh CLI) | `gh pr create --title "feat: ..." --body "..."` |
| View PR status | `gh pr status` |
| List PRs | `gh pr list` |

### File Locations

| Purpose | Path |
|---------|------|
| Role variables | `defaults/main.yml` |
| Main tasks | `tasks/main.yml` |
| Host preparation | `tasks/prepare_host.yml` |
| Docker provisioning | `tasks/provision_traefik.yml` |
| Test configuration | `molecule/default/molecule.yml` |
| Test preparation | `molecule/default/prepare.yml` |
| Test verification | `molecule/default/verify.yml` |
| CI/CD pipeline | `.github/workflows/molecule.yml` |
| YAML lint config | `.yamllint` |
| Python dependencies | `requirements.txt` |

### Key Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `traefik_image_tag` | `"3.6.5"` | Traefik version |
| `traefik_service_name` | `"traefik"` | Container name |
| `traefik_source_base_path` | `"/etc/traefik"` | Config directory |
| `traefik_dashboard_enabled` | `"false"` | Dashboard access |
| `traefik_log_level` | `"ERROR"` | Logging verbosity |
| `traefik_ports` | `["80:80", "443:443"]` | Published ports |

---

## Resources

### Official Documentation
- **Traefik:** https://doc.traefik.io/traefik/
- **Ansible:** https://docs.ansible.com/
- **Molecule:** https://molecule.readthedocs.io/
- **ansible-lint:** https://ansible-lint.readthedocs.io/
- **yamllint:** https://yamllint.readthedocs.io/

### Project Links
- **GitHub:** https://github.com/tripleawwy/ansible_traefik_docker
- **Ansible Galaxy:** https://galaxy.ansible.com/tripleawwy/traefik_docker
- **Issues:** https://github.com/tripleawwy/ansible_traefik_docker/issues

### Community Resources
- **Ansible Collections:** https://docs.ansible.com/ansible/latest/collections/community/docker/
- **Docker-in-Docker:** https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
- **Molecule Docker Plugin:** https://github.com/ansible-community/molecule-plugins

---

**Last Updated:** 2025-12-29
**Maintained By:** tripleawwy
**License:** MIT
