# Molecule Testing - Default Scenario

This directory contains the default Molecule test scenario for the `tripleawwy.traefik_docker` Ansible role using Docker-in-Docker (DinD) testing.

## Overview

The test scenario validates that the Ansible role correctly:
- Deploys Traefik as a Docker container
- Creates required configuration files with proper permissions
- Sets up Docker networks for Traefik
- Ensures the Traefik container runs successfully

## Test Architecture

### Docker-in-Docker (DinD)

This scenario uses Docker-in-Docker to allow the Ansible role to deploy real Docker containers within the test environment.

**Why DinD?** The role deploys Traefik as a Docker container, so tests need Docker available inside the test container to validate actual deployment behavior.

### Test Components

```
molecule/default/
├── Dockerfile       # Custom test image with Docker-in-Docker support
├── molecule.yml     # Molecule configuration (Docker driver, platforms)
├── converge.yml     # Playbook that executes the role
├── prepare.yml      # Environment preparation (Docker daemon setup)
├── verify.yml       # Verification tests
└── README.md        # This file
```

## Files Explained

### Dockerfile

Builds a custom test image based on `geerlingguy/docker-debian12-ansible` with:
- **Base OS**: Debian 12 (Bookworm)
- **systemd**: Full systemd support for service management
- **Docker CE**: Docker Engine for running containers
- **Python Docker SDK**: Required for Ansible `community.docker` collection

**Key Features:**
- Proper GPG key management for Docker repository
- Docker Compose plugin installation
- Python Docker library for Ansible integration
- systemd as init system

### molecule.yml

Main Molecule configuration defining:
- **Driver**: Docker (molecule-plugins[docker])
- **Platform**: Custom Debian 12 image with DinD
- **Privileged Mode**: Required for nested Docker operations
- **Volumes**: `/sys/fs/cgroup` for systemd support
- **Capabilities**: `SYS_ADMIN` for container operations
- **Provisioner**: Ansible with debug output (`-D`)
- **Linting**: yamllint + ansible-lint

**Important Settings:**
```yaml
privileged: true              # Required for DinD
cgroupns_mode: host          # Host cgroup namespace
command: /lib/systemd/systemd # systemd as PID 1
pre_build_image: false       # Build image fresh each time
```

### prepare.yml

Prepares the test environment before running the role:

1. **Docker Daemon Configuration** (`/etc/docker/daemon.json`)
   - Sets `storage-driver: vfs` for DinD compatibility
   - Avoids nested overlay filesystem issues

2. **Docker Service Management**
   - Ensures Docker daemon is started and enabled
   - Restarts Docker if configuration changes
   - Uses proper Ansible handler pattern

3. **Docker Readiness Check**
   - Waits for Docker daemon to be fully operational
   - Retries up to 5 times with 2-second delays

### converge.yml

Simple playbook that executes the role:
```yaml
- name: Converge
  hosts: all
  become: true
  tasks:
    - name: Include tripleawwy.traefik_docker
      ansible.builtin.include_role:
        name: tripleawwy.traefik_docker
```

### verify.yml

Comprehensive verification tests:

**Container Validation:**
- [x] Traefik container exists
- [x] Container status is "running"
- [x] Container is actually running (not just created)

**Network Validation:**
- [x] Traefik Docker network exists

**File Validation:**
- [x] `acme.json` file exists at `/etc/traefik/acme.json`
- [x] File permissions are `0600` (owner read/write only)

## Running Tests

### Prerequisites

```bash
# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r ../../requirements.txt

# Install Ansible collections
ansible-galaxy collection install community.docker community.general
```

### Full Test Suite

```bash
cd ../..  # Navigate to role root
molecule test
```

**Test Stages:**
1. **Lint** - yamllint + ansible-lint
2. **Syntax** - Ansible syntax check
3. **Create** - Build Docker image and start container
4. **Prepare** - Configure Docker daemon for DinD
5. **Converge** - Run the Ansible role
6. **Idempotence** - Run role again, expect no changes
7. **Verify** - Run verification tests
8. **Destroy** - Clean up test container

### Development Workflow

For faster iteration during development:

```bash
# Create test environment once
molecule create

# Make changes to role...

# Test changes quickly (skip create/destroy)
molecule converge

# Run verification
molecule verify

# Destroy when done
molecule destroy
```

### Individual Commands

```bash
molecule lint          # Run linting only
molecule syntax        # Check syntax only
molecule create        # Create test container
molecule converge      # Run the role
molecule verify        # Run verification tests
molecule login         # SSH into test container
molecule destroy       # Clean up
```

## Test Platform Details

### Container Specifications

| Setting | Value | Purpose |
|---------|-------|---------|
| **Base Image** | `geerlingguy/docker-debian12-ansible` | Debian 12 with Ansible |
| **Privileged** | `true` | Required for nested Docker |
| **Init** | `/lib/systemd/systemd` | systemd as PID 1 |
| **Capabilities** | `SYS_ADMIN` | Container management |
| **Security** | `seccomp=unconfined` | Allows Docker operations |
| **Volumes** | `/sys/fs/cgroup:rw` | cgroup support |

### Storage Driver: VFS

The test environment uses VFS (Virtual File System) storage driver instead of overlay2 because:
- **Nested overlayfs not supported**: Cannot run overlay2 inside overlay2
- **DinD compatibility**: VFS works reliably in nested Docker scenarios
- **Performance trade-off**: Slower but more reliable for testing

## Troubleshooting

### Docker Daemon Fails to Start

**Symptom:** `Cannot connect to Docker daemon`

**Solutions:**
```bash
# Check if Docker is running on host
sudo systemctl status docker

# Verify privileged mode in molecule.yml
grep privileged molecule.yml
```

### Storage Driver Errors

**Symptom:** `failed to mount: invalid argument` or overlayfs errors

**Solution:** Ensure `prepare.yml` correctly sets VFS storage driver:
```yaml
- name: Configure Docker daemon for DinD compatibility
  ansible.builtin.copy:
    content: |
      {
        "storage-driver": "vfs"
      }
    dest: /etc/docker/daemon.json
```

### Container Build Failures

**Symptom:** Dockerfile build errors

**Solutions:**
```bash
# Build manually to debug
cd molecule/default
docker build -t traefik-molecule-debian12:latest .

# Check Docker hub rate limits
docker pull geerlingguy/docker-debian12-ansible:latest
```

### Idempotence Failures

**Symptom:** Second converge shows changes

**Analysis:**
- Check for tasks using `command` without `changed_when: false`
- Review tasks that might have side effects
- Verify file timestamps preserved (use `modification_time: preserve`)

### Permission Denied Errors

**Symptom:** `permission denied` when accessing Docker socket

**Solution:**
```bash
# Ensure user is in docker group
sudo usermod -aG docker $USER

# Log out and back in, or:
newgrp docker
```

## Performance Tips

### Speed Up Tests

1. **Keep container running during development**
   ```bash
   molecule create
   # Make changes...
   molecule converge  # Fast, no rebuild
   ```

2. **Use cached images**
   - Set `pre_build_image: true` if image doesn't change
   - Pull base images before testing

3. **Skip destroy for debugging**
   ```bash
   molecule test --destroy=never
   ```

### Debug Test Failures

```bash
# Login to test container
molecule login

# Check Docker status inside container
docker info
docker ps -a

# Check Traefik logs
docker logs traefik

# Inspect files
ls -la /etc/traefik/
cat /etc/docker/daemon.json
```

## CI/CD Integration

This scenario runs automatically in GitHub Actions on:
- Push to `main` branch
- Pull requests to `main` branch

Workflow file: `../../.github/workflows/molecule.yml`

## Additional Resources

- [Molecule Documentation](https://molecule.readthedocs.io/)
- [Molecule Docker Plugin](https://github.com/ansible-community/molecule-plugins)
- [Docker-in-Docker Considerations](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Ansible community.docker Collection](https://docs.ansible.com/ansible/latest/collections/community/docker/)

## Test Maintenance

### Updating Test Platform

To update to a newer Debian version:
1. Update `Dockerfile` base image
2. Update platform image name in `molecule.yml`
3. Test locally before committing

### Adding New Verification Tests

Add tests to `verify.yml`:
```yaml
- name: Check new feature
  ansible.builtin.assert:
    that:
      - condition_here
    success_msg: "Feature works"
    fail_msg: "Feature broken"
```

### Updating Dependencies

Update `../../requirements.txt` when upgrading:
- Molecule version
- ansible-core version
- ansible-lint version
- Docker SDK version
