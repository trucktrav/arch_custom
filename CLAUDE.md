# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible playbook repository for automating Arch Linux development environment setup. It configures a local workstation with development tools, window managers, shells, and productivity applications. The playbooks are designed to run locally (not against remote hosts) using `localhost` as the target.

## Project Structure

- **`playbook.yml`**: Legacy monolithic playbook (kept for reference)
- **`playbooks/local_setup.yml`**: Current recommended playbook that uses roles
- **`roles/`**: Modular Ansible roles for specific configurations
  - `custom`: Core system packages, development tools, IDEs, Docker, databases, project directories
  - `manage_zsh`: Oh My Zsh setup with Powerlevel10k theme, plugins, and fastfetch integration
  - `starship`: Alternative shell prompt using Starship with zsh plugins (minimal setup)
- **`inventories/hosts`**: Inventory file defining localhost with groups
- **`ansible.cfg`**: Configuration with local connection settings, facts caching, and logging

## Key Architecture Decisions

### Execution Model
- All playbooks target `localhost` with `connection: local`
- Uses privilege escalation (`become: yes`) for package installation, but switches to non-root for user-specific configurations
- The `username` and `user_home` variables are derived from the executing user's environment

### Role-Based vs Monolithic
The repository has transitioned from a monolithic `playbook.yml` to role-based organization in `playbooks/local_setup.yml`. The `custom` role handles Oh My Zsh setup inline (commented out sections show it was migrated to the `manage_zsh` role), while the `manage_zsh` role provides a complete, idempotent Zsh configuration.

### Package Management Strategy
- Official packages: `community.general.pacman` module
- AUR packages: Mix of `yay` command-line calls and `community.general.pacman` with `executable: yay`
- Yay installation is handled conditionally (only if not already installed)

### User Context Switching
Tasks frequently switch between root and user contexts:
- System packages: as root (default with `become: yes`)
- User configurations: `become: false` or `become_user: "{{ user }}"`
- Git clones, dotfile management: always as the target user

## Common Commands

### Run the main setup playbook
```bash
ansible-playbook playbooks/local_setup.yml
```

### Run with specific tags
```bash
# Only setup base packages and tools
ansible-playbook playbooks/local_setup.yml --tags custom

# Only setup Zsh environment
ansible-playbook playbooks/local_setup.yml --tags manage_zsh

# Run all base setup tasks
ansible-playbook playbooks/local_setup.yml --tags base_setup
```

### Run the legacy monolithic playbook
```bash
ansible-playbook playbook.yml
```

### Validate playbook syntax
```bash
ansible-playbook playbooks/local_setup.yml --syntax-check
```

### Dry run (check mode)
```bash
ansible-playbook playbooks/local_setup.yml --check
```

### Linting and validation (using project dependencies)
```bash
# Activate the UV virtual environment first
uv sync
source .venv/bin/activate

# Run ansible-lint
ansible-lint playbooks/local_setup.yml

# Run yamllint
yamllint .
```

### View available hosts and groups
```bash
ansible-inventory --list
```

## Development Workflow

### Adding New Roles
1. Create role structure: `ansible-galaxy init roles/<role-name>`
2. Add tasks to `roles/<role-name>/tasks/main.yml`
3. Define defaults in `roles/<role-name>/defaults/main.yml` if needed
4. Include role in `playbooks/local_setup.yml` with appropriate tags
5. Store config files in `roles/<role-name>/files/` for the `copy` module

### Variable Precedence
The playbooks use these variable patterns:
- `username`: From environment variable `USER`
- `user_home`: Derived as `/home/{{ username }}`
- `user` and `home`: Aliases for the above (used by `manage_zsh` role)
- Roles can access these via `{{ user }}` or `{{ username }}`

### Managing Oh My Zsh Plugins
The `manage_zsh` role clones plugins into `.oh-my-zsh/custom/plugins/` and manages them in a loop. To add a new plugin:
1. Add to the loop in `roles/manage_zsh/tasks/main.yml`
2. Ensure the plugin is activated in the `.zshrc` file template

### Handling AUR Packages
Use this pattern for AUR packages with yay:
```yaml
- name: Install AUR package
  become: false
  ansible.builtin.command: yay -S --noconfirm <package-name>
  register: result
  changed_when: result.rc == 0
  failed_when: result.rc != 0 and 'error' in result.stderr
```

Or use the pacman module with yay executable:
```yaml
- name: Install AUR package
  become: false
  community.general.pacman:
    name: <package-name>
    state: present
    executable: yay
    extra_args: "--builddir /tmp/yay"
```

### Facts Caching
Facts are cached to `./facts_cache/` with a 2-hour timeout. Clear the cache if you need fresh facts:
```bash
rm -rf facts_cache/*
```

## Environment Variables Used

- `USER`: Determines the target user for configurations
- Ansible automatically populates `ansible_env` with environment variables
- Custom vars defined in playbooks: `username`, `user_home`, `git_server_port`, `install_tiling_wm`, `tiling_wm`

## Logs and Debugging

- Ansible logs are written to `./ansible.log` (configured in `ansible.cfg`)
- Use `-v`, `-vv`, `-vvv`, or `-vvvv` for increasing verbosity levels
- Check specific task output: `ansible-playbook playbooks/local_setup.yml -vv --start-at-task="Task Name"`
