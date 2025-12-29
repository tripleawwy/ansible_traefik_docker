# Ansible Role: Traefik Docker

[![Molecule CI](https://github.com/tripleawwy/ansible_traefik_docker/actions/workflows/molecule.yml/badge.svg)](https://github.com/tripleawwy/ansible_traefik_docker/actions/workflows/molecule.yml)

An Ansible role for deploying [Traefik][traefik_url] reverse proxy as a Docker container with automatic HTTPS via Let's Encrypt.

## Table of Contents

- [About The Project](#about-the-project)
- [Features](#features)
- [Requirements](#requirements)
- [Role Variables](#role-variables)
- [Dependencies](#dependencies)
- [Installation](#installation)
- [Usage](#usage)
- [Testing](#testing)
- [Contributing](#contributing)
- [License](#license)
- [Author Information](#author-information)

## About The Project

This Ansible role provides automated deployment of Traefik v3.6.5 as a containerized reverse proxy with:
- Automatic HTTPS certificate management via Let's Encrypt
- Docker provider for automatic service discovery
- HTTP to HTTPS redirection
- Configurable dashboard and logging
- Production-ready defaults

### Built With

- [Ansible][ansible_url] - Automation platform
- [Traefik][traefik_url] - Modern reverse proxy
- [Docker][docker_url] - Container runtime
- [Molecule][molecule_url] - Testing framework

## Features

- [x] Traefik v3.6.5 deployment
- [x] Automatic Let's Encrypt HTTPS certificates
- [x] Docker service discovery
- [x] HTTP to HTTPS redirection
- [x] Customizable entrypoints and routing
- [x] Optional dashboard access
- [x] Production-ready security defaults
- [x] Fully tested with Molecule

## Requirements

### Target Host Requirements

- Docker installed and running
- Python 3 (for Ansible modules)
- Systemd (optional, for service management)

### Control Node Requirements

- Ansible core >= 2.13.5
- Python 3.8 or higher

## Role Variables

### Core Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `traefik_image_tag` | `"3.6.5"` | Traefik Docker image version |
| `traefik_service_name` | `"traefik"` | Container and network name |
| `traefik_source_base_path` | `"/etc/traefik"` | Configuration directory |
| `traefik_source_acme_path` | `"/etc/traefik/acme.json"` | Let's Encrypt certificate storage |

### Configuration Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `traefik_dashboard_enabled` | `"false"` | Enable Traefik dashboard |
| `traefik_log_level` | `"ERROR"` | Logging level (DEBUG, INFO, WARN, ERROR) |
| `traefik_checknewversion` | `"false"` | Check for Traefik updates |
| `traefik_sendanonymoususage` | `"false"` | Send anonymous usage stats |

### ACME / Let's Encrypt Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `traefik_acme_challenge_type` | `"http"` | Challenge type: `"http"` or `"dns"` |
| `traefik_acme_dns_provider` | `""` | DNS provider (e.g., `cloudflare`, `route53`, `digitalocean`, `ovh`) |
| `traefik_acme_dns_env` | `{}` | DNS provider environment variables (credentials) |
| `traefik_acme_email` | `""` | Email for Let's Encrypt notifications (optional but recommended) |
| `traefik_acme_dns_delay_before_check` | `0` | Delay before DNS record verification (seconds) |

### Network Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `traefik_ports` | `["80:80", "443:443"]` | Published ports |
| `traefik_networks` | `[{"name": "traefik"}]` | Docker networks |
| `traefik_api_rule` | `"Host(\`traefik.{{ inventory_hostname }}\`)"` | Dashboard routing rule |

See [defaults/main.yml](defaults/main.yml) for complete variable list.

### Supported DNS Providers

This role supports DNS challenge for Let's Encrypt certificate generation, which is required for wildcard certificates and useful when port 80 is not accessible.

**Common DNS Providers:**

| Provider | Value for `traefik_acme_dns_provider` | Required Environment Variables |
|----------|---------------------------------------|-------------------------------|
| **Cloudflare** | `cloudflare` | `CF_API_EMAIL`, `CF_DNS_API_TOKEN` or `CF_API_KEY` |
| **AWS Route53** | `route53` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` |
| **DigitalOcean** | `digitalocean` | `DO_AUTH_TOKEN` |
| **Google Cloud DNS** | `gcloud` | `GCE_PROJECT`, `GCE_SERVICE_ACCOUNT_FILE` |
| **Azure DNS** | `azure` | `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_RESOURCE_GROUP` |
| **OVH** | `ovh` | `OVH_ENDPOINT`, `OVH_APPLICATION_KEY`, `OVH_APPLICATION_SECRET`, `OVH_CONSUMER_KEY` |
| **Hetzner** | `hetzner` | `HETZNER_API_KEY` |
| **Linode** | `linode` | `LINODE_TOKEN` |
| **Namecheap** | `namecheap` | `NAMECHEAP_API_USER`, `NAMECHEAP_API_KEY` |
| **GoDaddy** | `godaddy` | `GODADDY_API_KEY`, `GODADDY_API_SECRET` |

**Full List:** See [Lego DNS Providers](https://go-acme.github.io/lego/dns/) for complete list of 100+ supported providers.

**Security Note:** Use Ansible Vault to encrypt sensitive DNS provider credentials:

```yaml
# vars/secrets.yml (encrypted with ansible-vault)
vault_cf_api_token: "your-secret-token"

# playbook.yml
vars:
  traefik_acme_dns_env:
    CF_API_EMAIL: "your@email.com"
    CF_DNS_API_TOKEN: "{{ vault_cf_api_token }}"
```

## Dependencies

### Ansible Collections

- `community.docker` >= 3.0.0

Install with:
```bash
ansible-galaxy collection install community.docker
```

## Installation

### From Ansible Galaxy

```bash
ansible-galaxy install tripleawwy.traefik_docker
```

### Using Requirements File

Create `requirements.yml`:
```yaml
---
roles:
  - name: tripleawwy.traefik_docker
    version: v1.0.0
```

Install:
```bash
ansible-galaxy install -r requirements.yml
```

## Usage

### Basic Deployment

Minimal playbook for Traefik with Let's Encrypt:

```yaml
---
- name: Deploy Traefik
  hosts: traefik_servers
  become: true

  roles:
    - tripleawwy.traefik_docker
```

### With Dashboard Enabled

```yaml
---
- name: Deploy Traefik with Dashboard
  hosts: traefik_servers
  become: true

  vars:
    traefik_dashboard_enabled: "true"
    traefik_log_level: "INFO"
    traefik_labels:
      traefik.enable: "true"
      traefik.http.routers.traefik_api.rule: "Host(`traefik.example.com`)"
      traefik.http.routers.traefik_api.entrypoints: "web_secure"
      traefik.http.routers.traefik_api.service: "api@internal"
      traefik.http.routers.traefik_api.tls: "true"

  roles:
    - tripleawwy.traefik_docker
```

### DNS Challenge with Cloudflare

```yaml
---
- name: Deploy Traefik with DNS Challenge
  hosts: traefik_servers
  become: true

  vars:
    traefik_acme_challenge_type: "dns"
    traefik_acme_dns_provider: "cloudflare"
    traefik_acme_email: "admin@example.com"
    traefik_acme_dns_env:
      CF_API_EMAIL: "your-cloudflare@email.com"
      CF_DNS_API_TOKEN: "your-cloudflare-api-token"

  roles:
    - tripleawwy.traefik_docker
```

### DNS Challenge with Route53

```yaml
---
- name: Deploy Traefik with AWS Route53 DNS Challenge
  hosts: traefik_servers
  become: true

  vars:
    traefik_acme_challenge_type: "dns"
    traefik_acme_dns_provider: "route53"
    traefik_acme_email: "admin@example.com"
    traefik_acme_dns_env:
      AWS_ACCESS_KEY_ID: "your-aws-access-key"
      AWS_SECRET_ACCESS_KEY: "your-aws-secret-key"
      AWS_REGION: "us-east-1"
    traefik_acme_dns_delay_before_check: 60  # AWS propagation delay

  roles:
    - tripleawwy.traefik_docker
```

### DNS Challenge with DigitalOcean

```yaml
---
- name: Deploy Traefik with DigitalOcean DNS Challenge
  hosts: traefik_servers
  become: true

  vars:
    traefik_acme_challenge_type: "dns"
    traefik_acme_dns_provider: "digitalocean"
    traefik_acme_email: "admin@example.com"
    traefik_acme_dns_env:
      DO_AUTH_TOKEN: "your-digitalocean-auth-token"

  roles:
    - tripleawwy.traefik_docker
```

### Custom Configuration

```yaml
---
- name: Deploy Traefik with Custom Settings
  hosts: traefik_servers
  become: true

  vars:
    traefik_image_tag: "3.6.5"
    traefik_dashboard_enabled: "true"
    traefik_log_level: "DEBUG"
    traefik_source_base_path: "/opt/traefik"

    # Additional CLI commands
    traefik_cli_commands_extension:
      - "--metrics.prometheus=true"
      - "--metrics.prometheus.entrypoint=metrics"

    # Custom ports
    traefik_ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Metrics endpoint

  roles:
    - tripleawwy.traefik_docker
```

## Testing

This role is tested using [Molecule][molecule_url] with Docker-in-Docker support.

### Quick Test

```bash
# Set up testing environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Install collections
ansible-galaxy collection install community.docker community.general

# Run tests
molecule test
```

For detailed testing instructions, see [molecule/default/README.md](molecule/default/README.md).

### CI/CD

Automated testing via GitHub Actions runs on every push and pull request. See [.github/workflows/molecule.yml](.github/workflows/molecule.yml).

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Guidelines

- Follow Ansible best practices
- Maintain ansible-lint production profile compliance
- Add tests for new features
- Update documentation

## License

[MIT License][mit_url]

See [LICENSE](LICENSE) for details.

## Author Information

**tripleawwy**
- GitHub: [@tripleawwy](https://github.com/tripleawwy)
- Role: [tripleawwy.traefik_docker](https://galaxy.ansible.com/tripleawwy/traefik_docker)

## Acknowledgements

- [Traefik](https://traefik.io/) - The Cloud Native Application Proxy
- [Molecule](https://molecule.readthedocs.io/) - Ansible testing framework
- [Ansible Community](https://ansible.com/) - Community collections
- [Docker](https://www.docker.com/) - Container platform

<!-- MARKDOWN LINKS -->
[traefik_url]: https://traefik.io/
[ansible_url]: https://www.ansible.com/
[docker_url]: https://www.docker.com/
[molecule_url]: https://molecule.readthedocs.io/
[mit_url]: https://spdx.org/licenses/MIT.html
