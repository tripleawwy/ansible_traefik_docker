# Ansible role containerized Traefik

<!-- TABLE OF CONTENTS -->
## Table of Contents

- [Ansible role containerized Traefik](#ansible-role-containerized-traefik)
  - [Table of Contents](#table-of-contents)
  - [About The Project](#about-the-project)
    - [Built With](#built-with)
  - [Getting Started](#getting-started)
    - [Requirements](#requirements)
    - [Role Variables](#role-variables)
    - [Dependencies](#dependencies)
  - [Usage](#usage)
    - [Example Playbook](#example-playbook)
  - [Testing](#testing)
  - [Contributing](#contributing)
  - [Author Information](#author-information)
  - [License](#license)
  - [Acknowledgements](#acknowledgements)

<!-- ABOUT THE PROJECT -->
## About The Project

This project aims to easily provision the reverse proxy [Traefik][traefik_url] with Ansible and Docker.

### Built With

- [Ansible][ansible_url]
- [VS Code][code_url]

<!-- GETTING STARTED -->
## Getting Started

The following instructions will guide you trough requirements needed for this Ansible role to run.

### Requirements
The remote host needs to have Docker installed. No other requirements are necessary.
### Role Variables
### Dependencies
- `ansible-core >= v2.13.5`
- `community.docker >= v3.0.0`

<!-- USAGE -->
## Usage
You'll need to install the Ansible role.

- e.g. by using the requirements file.

  ```yaml
  # requirements.yml
    - src: tripleawwy.traefik_docker
      version: v1.0.0
  ```
  ```bash
  ansible-galaxy install -r requirements.yml
  ```
- or via Ansible Galaxy directly
  ```bash
  ansible-galaxy install tripleawwy.traefik_docker
  ```

### Example Playbook
The following Playbook provisions Traefik with a default Traefik certificate on localhost
```yaml
---
- name: Example
  hosts: localhost
  become: true
  vars:
    traefik_dashboard_enabled: "true"
    traefik_log_level: "DEBUG"
    traefik_labels:
      traefik.enable: "true"
      traefik.http.routers.traefik_api.rule: "{{ traefik_api_rule }}"
      traefik.http.routers.traefik_api.entrypoints: "web_secure"
      traefik.http.routers.traefik_api.service: "api@internal"
      traefik.http.routers.traefik_api.tls: "true"
  tasks:
    - name: "Include traefik_docker role"
      ansible.builtin.include_role:
        name: "tripleawwy.traefik_docker"
```

<!-- Testing -->
## Testing
Tests are done with [Molecule][molecule_url] with the help of [Vagrant][vagrant_url] and [Virtual Box][vb_url]. You can find more detailed requirements [here][molecule_deps].

If your test environment is set up:
```bash
molecule test
```

<!-- CONTRIBUTING -->
## Contributing
Feel free to open issues, pull requests or ask me anything directly!

<!-- AUTHOR INFORMATION -->
## Author Information
[tripleawwy](https://github.com/tripleawwy)

<!-- License -->
## License
[MIT][mit_url]

Detailed information: https://choosealicense.com/licenses/mit/

<!-- ACKNOWLEDGEMENTS -->
## Acknowledgements

- [markdown-guide](https://about.gitlab.com/handbook/product/technical-writing/markdown-guide/#colorful-sections)
- [readme-help](https://github.com/othneildrew/Best-README-Template)

<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[traefik_url]: https://traefik.io/
[ansible_url]: https://www.ansible.com/
[code_url]: https://code.visualstudio.com/
[molecule_url]: https://molecule.readthedocs.io/en/latest/ci.html
[vagrant_url]: https://www.vagrantup.com/
[vb_url]: https://www.virtualbox.org/
[molecule_deps]: molecule/default/INSTALL.rst
[mit_url]: https://spdx.org/licenses/MIT.html
