# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible playbook repository for automating Arch Linux development environment setup. It configures a local workstation with development tools, window managers, shells, and productivity applications. The playbooks are designed to run locally (not against remote hosts) using `localhost` as the target.

The project has evolved from monolithic playbooks to a modular, role-based architecture with feature flags for flexible deployments.

## Project Structure

```
arch_custom/
├── playbooks/
│   ├── local_setup.yml          # MAIN PLAYBOOK (production)
│   └── debug/                   # Archived legacy playbooks
│       ├── playbook.yml         # Original monolithic version
│       ├── playbook-2.yml       # Iteration attempt
│       └── local_setups.yml     # Incomplete test version
├── roles/
│   ├── custom/                  # Core system setup (modular)
│   │   ├── tasks/
│   │   │   ├── main.yml         # Base packages, yay, UV, Hyprland, productivity tools
│   │   │   ├── capps.yml        # Development apps (PyCharm, R/RStudio, databases)
│   │   │   └── terms.yml        # Terminal setup (Alacritty, TMUX with TPM)
│   │   └── files/
│   │       └── alacritty.yml    # Alacritty configuration file
│   ├── manage_zsh/              # Zsh with optional Powerlevel10k (modular)
│   │   ├── tasks/
│   │   │   ├── main.yml         # Install packages via pacman, copy .zshrc, fastfetch
│   │   │   ├── zsh-plugin.yml   # Git clone method for plugins (conditional)
│   │   │   └── zsh-p10k.yml     # Oh My Zsh + Powerlevel10k (conditional)
│   │   ├── files/
│   │   │   ├── .zshrc           # Zsh configuration for Powerlevel10k
│   │   │   └── .p10k.zsh        # Powerlevel10k theme configuration
│   │   ├── defaults/
│   │   ├── handlers/
│   │   └── vars/
│   ├── starship/                # Starship prompt (production-ready)
│   │   ├── tasks/
│   │   │   └── main.yml         # Install Starship + modern CLI tools
│   │   └── files/
│   │       ├── starship.toml    # Starship prompt configuration
│   │       └── zshrc            # Zsh config optimized for Starship
│   └── geerlingguy.docker/      # External Galaxy role for Docker
├── inventories/
│   └── hosts                    # Localhost inventory with groups
├── requirement.yml              # Ansible Galaxy role dependencies
├── ansible.cfg                  # Ansible configuration (local connection, logging, facts cache)
├── pyproject.toml               # Python dependencies (ansible, ansible-lint, yamllint)
├── README.md                    # User-facing documentation
└── CLAUDE.md                    # This file (developer documentation)
```

## Key Architecture Decisions

### Execution Model
- All playbooks target `localhost` with `connection: local`
- Uses privilege escalation (`become: yes`) for system package installation
- Switches to non-root (`become: false` or `become_user: "{{ user }}"`) for user-specific configurations
- Variables `username` and `user_home` are derived from the executing user's environment

### Modular Role Architecture
The project has evolved significantly:
- **Legacy**: Monolithic playbooks (now in `playbooks/debug/`)
- **Current**: Modular roles with sub-task files and feature flags

**Custom Role Modularity:**
- `main.yml`: Core system setup (packages, yay, UV, system tools)
- `capps.yml`: Development applications (PyCharm, R, RStudio, databases)
- `terms.yml`: Terminal emulators and multiplexers (Alacritty, TMUX with TPM)

**Manage_zsh Role Modularity:**
- `main.yml`: Package installation via pacman, .zshrc deployment, fastfetch setup
- `zsh-plugin.yml`: Git clone method for plugins (used when not using pacman)
- `zsh-p10k.yml`: Oh My Zsh + Powerlevel10k installation (conditional on `USE_P10K`)

**Shell Configuration Strategy:**
- Two mutually exclusive shell prompt options controlled by feature flags
- `USE_STARSHIP: true` → Modern, fast, Rust-based prompt (current default)
- `USE_P10K: true` → Feature-rich, customizable prompt with Oh My Zsh
- Only one should be enabled at a time to avoid conflicts

### Feature Flags
The playbook uses boolean variables for conditional component installation:

```yaml
USE_STARSHIP: true           # Enable Starship prompt (recommended)
USE_P10K: false              # Enable Powerlevel10k with Oh My Zsh
USE_ALACRITTY: false         # Set Alacritty as default terminal
INSTALL_CUSTOM_APPS: false   # Install development apps (PyCharm, R, databases)
install_tiling_wm: true      # Install tiling window manager
```

These allow users to customize their installation without modifying role code.

### Package Management Strategy
- **Official Arch packages**: `community.general.pacman` module
- **AUR packages**:
  - Method 1: `ansible.builtin.command` with `yay -S --noconfirm <package>`
  - Method 2: `community.general.pacman` with `executable: yay`
- **Yay installation**: Handled conditionally (checks if already installed before building)
- **Ansible Galaxy roles**: Installed via `ansible-galaxy install -r requirement.yml`

**Zsh Plugin Management:**
- **Default approach (manage_zsh)**: Install via pacman system packages
  - `zsh-autosuggestions`, `zsh-syntax-highlighting`, `zsh-completions`, `zsh-transient-prompt`
  - Source from `/usr/share/zsh/plugins/` paths
  - Benefits: Automatic updates with `pacman -Syu`, clean uninstall
- **Alternative approach (starship)**: Git clone to `~/.zsh/` directory
  - Used for cross-distro compatibility in dotfiles
  - Manual updates required

### User Context Switching Pattern
Tasks frequently switch between root and user contexts:
```yaml
# System package installation (as root)
- name: Install packages
  community.general.pacman:
    name: package-name
    state: present
  # Implicit: become: yes (from playbook level)

# User-specific configuration (as non-root)
- name: Copy config file
  become: false
  ansible.builtin.copy:
    src: config.yml
    dest: "{{ user_home }}/.config/app/config.yml"

# Explicit user context
- name: Install AUR package
  become: false
  ansible.builtin.command: yay -S --noconfirm package-name
```

### External Role Dependencies
The project uses `geerlingguy.docker` role from Ansible Galaxy:
- Professional, well-maintained role for Docker installation
- Handles Docker service management and user group setup
- Version pinned in `requirement.yml`: `7.3.0`
- Install before running playbook: `ansible-galaxy install -r requirement.yml`

## Configuration Variables

### Required Variables (set in playbooks/local_setup.yml)
```yaml
username: "{{ lookup('env', 'USER') }}"     # Auto-detected from environment
user_home: "/home/{{ username }}"           # User's home directory
user: "{{ username }}"                      # Alias for role compatibility
home: "{{ user_home }}"                     # Alias for role compatibility
```

### Feature Flags
```yaml
USE_STARSHIP: true           # Enable Starship shell prompt
USE_P10K: false              # Enable Powerlevel10k (don't enable with Starship)
USE_ALACRITTY: false         # Set Alacritty as default terminal emulator
INSTALL_CUSTOM_APPS: false   # Install PyCharm, R/RStudio, databases
install_tiling_wm: true      # Install tiling window manager
tiling_wm: "i3-gaps"         # Which tiling WM (i3-gaps, sway, awesome, qtile, bspwm)
git_server_port: 3000        # Port for local Git server (if enabled)
```

## Common Commands

### First-Time Setup
```bash
# Install Ansible on Arch Linux
sudo pacman -S ansible

# Set up Python environment for linting (optional)
uv sync
source .venv/bin/activate

# Install Galaxy role dependencies
ansible-galaxy install -r requirement.yml

# Run the main playbook
ansible-playbook playbooks/local_setup.yml
```

### Run with Specific Tags
```bash
# Only setup core packages and tools
ansible-playbook playbooks/local_setup.yml --tags custom

# Only setup Zsh environment
ansible-playbook playbooks/local_setup.yml --tags manage_zsh

# Only setup Starship prompt
ansible-playbook playbooks/local_setup.yml --tags starship

# Only setup Docker
ansible-playbook playbooks/local_setup.yml --tags docker

# Run all base setup tasks (custom + manage_zsh + starship + docker)
ansible-playbook playbooks/local_setup.yml --tags base_setup

# Skip Docker installation
ansible-playbook playbooks/local_setup.yml --skip-tags docker
```

### Validation and Testing
```bash
# Syntax check
ansible-playbook playbooks/local_setup.yml --syntax-check

# Dry run (check mode)
ansible-playbook playbooks/local_setup.yml --check

# Run with verbose output
ansible-playbook playbooks/local_setup.yml -vv

# Linting (requires uv environment)
uv sync && source .venv/bin/activate
ansible-lint playbooks/local_setup.yml
yamllint .
```

### Inventory Management
```bash
# View inventory structure
ansible-inventory --list

# View inventory graph
ansible-inventory --graph
```

## Development Workflow

### Adding New Roles
1. Create role structure: `ansible-galaxy init roles/<role-name>`
2. Add tasks to `roles/<role-name>/tasks/main.yml`
3. Define defaults in `roles/<role-name>/defaults/main.yml` if needed
4. Store config files in `roles/<role-name>/files/` for the `copy` module
5. Include role in `playbooks/local_setup.yml`:
   ```yaml
   roles:
     - role: <role-name>
       tags:
         - <role-name>
         - base_setup  # Optional: include in base setup
   ```

### Modularizing Existing Roles
When a role's `main.yml` becomes too large, split into focused sub-task files:

**Example: Custom role structure**
```yaml
# roles/custom/tasks/main.yml
---
- name: Base system setup
  # Core tasks...

- name: Include development apps
  include_tasks: capps.yml
  when: INSTALL_CUSTOM_APPS

- name: Include terminal setup
  include_tasks: terms.yml
```

**Sub-task files:**
- `capps.yml`: Development applications (PyCharm, R, databases)
- `terms.yml`: Terminal emulators and configurations

This pattern keeps files focused (< 200 lines) and improves readability.

### Adding Configuration Files
Store configuration files in `roles/<role-name>/files/`:

```yaml
# Reference file in copy task
- name: Configure application
  ansible.builtin.copy:
    src: config.yml              # Looks in roles/<role-name>/files/config.yml
    dest: "{{ user_home }}/.config/app/config.yml"
    mode: '0644'
```

For inline content, use `content` parameter:
```yaml
- name: Create config file
  ansible.builtin.copy:
    dest: "{{ user_home }}/.config/app/config.yml"
    content: |
      key: value
      setting: true
    mode: '0644'
```

### Handling AUR Packages
Standard pattern for AUR packages:
```yaml
- name: Install AUR package
  become: false
  ansible.builtin.command: yay -S --noconfirm <package-name>
  register: result
  changed_when: result.rc == 0
  failed_when: result.rc != 0 and 'error' in result.stderr
```

Alternative using pacman module:
```yaml
- name: Install AUR package
  become: false
  community.general.pacman:
    name: <package-name>
    state: present
    executable: yay
    extra_args: "--builddir /tmp/yay"
```

### Managing Shell Plugins

**For Powerlevel10k (manage_zsh role):**
- Plugins installed via pacman: Edit `roles/manage_zsh/tasks/main.yml`
- Add package to the pacman task list
- Source in `.zshrc` from `/usr/share/zsh/plugins/<plugin-name>/`

**For Starship (starship role):**
- Plugins cloned via git: Edit `roles/starship/tasks/main.yml`
- Add git clone task for the plugin
- Source in `zshrc` from `~/.zsh/<plugin-name>/`

### Conditional Role Execution
Use `when` clauses with feature flags:
```yaml
roles:
  - role: starship
    when: USE_STARSHIP
    tags:
      - starship
      - base_setup

  - role: manage_zsh
    when: USE_P10K
    tags:
      - manage_zsh
      - base_setup
```

Internal conditionals in roles:
```yaml
# In manage_zsh/tasks/main.yml
roles:
  - role: zsh-plugin
    when: USE_P10K
  - role: zsh-p10k
    when: USE_P10K
```

### Facts Caching
Facts are cached to `./facts_cache/` with a 2-hour timeout:
```bash
# Clear cache if needed
rm -rf facts_cache/*

# Disable cache for testing
ansible-playbook playbooks/local_setup.yml --flush-cache
```

## Important Patterns and Conventions

### Variable Naming
- Feature flags: `USE_<FEATURE>` or `INSTALL_<FEATURE>` (boolean)
- User variables: `username`, `user_home`, `user`, `home`
- Role-specific: Prefix with role name (`manage_zsh_user`)

### Task Naming
- Prefix with role name: `Manage_zsh | Install required packages`
- Use descriptive, action-oriented names
- Follow pattern: `<Role> | <Action> <Target>`

### File Organization
- Keep `main.yml` under 200 lines
- Split into logical sub-task files when needed
- Use `include_tasks` for conditional sub-tasks
- Store static files in `files/` directory
- Use `templates/` for Jinja2 templates (if needed)

### Idempotency
Ensure tasks can be run multiple times safely:
```yaml
# Check before creating
- name: Check if exists
  ansible.builtin.stat:
    path: /path/to/resource
  register: resource_check

- name: Create resource
  when: not resource_check.stat.exists
  # ... creation task

# Use creates argument
- name: Build and install
  ansible.builtin.command: makepkg -si --noconfirm
  args:
    creates: /usr/bin/executable
```

### Error Handling
```yaml
# Allow task to fail without stopping playbook
- name: Optional task
  ansible.builtin.command: some-command
  failed_when: false

# Define specific failure conditions
- name: Install package
  ansible.builtin.command: yay -S --noconfirm package
  register: result
  failed_when: result.rc != 0 and 'error' in result.stderr

# Ignore specific return codes
- name: Check for resource
  ansible.builtin.command: which program
  register: check
  changed_when: false
  failed_when: check.rc not in [0, 1]
```

## Logs and Debugging

### Log Files
- Ansible logs: `./ansible.log` (configured in `ansible.cfg`)
- Location: Project root directory
- Persistent across runs (appends)

### Verbosity Levels
```bash
# Standard output
ansible-playbook playbooks/local_setup.yml

# Show task results (-v)
ansible-playbook playbooks/local_setup.yml -v

# Show task inputs and outputs (-vv)
ansible-playbook playbooks/local_setup.yml -vv

# Debug level (-vvv)
ansible-playbook playbooks/local_setup.yml -vvv

# Connection debug level (-vvvv)
ansible-playbook playbooks/local_setup.yml -vvvv
```

### Debugging Specific Tasks
```bash
# Start at specific task
ansible-playbook playbooks/local_setup.yml --start-at-task="Task Name"

# Run single task using tags
ansible-playbook playbooks/local_setup.yml --tags custom -vv

# Step through tasks interactively
ansible-playbook playbooks/local_setup.yml --step
```

### Debug Tasks
Add debug tasks to troubleshoot:
```yaml
- name: Debug user variables
  ansible.builtin.debug:
    msg:
      - "Username: {{ username }}"
      - "Home: {{ user_home }}"
      - "Variable: {{ my_var | default('NOT SET') }}"
```

## Post-Installation Manual Steps

Document these for users in README.md:

1. **Docker group**: Log out and back in for group changes
2. **TMUX plugins**: Run `tmux` then press `Ctrl+Space + I`
3. **Starship config**: Edit `~/.config/starship.toml` for customization
4. **Powerlevel10k**: Run `p10k configure` for initial setup
5. **Python venvs**: Set up per-project environments with `uv venv`

## Testing Recommendations

1. **Test in VM**: Use Arch Linux VM for testing before deploying to main system
2. **Use check mode**: Always run with `--check` first
3. **Tag-based testing**: Test individual roles before full run
4. **Incremental approach**: Add new features behind feature flags
5. **Version control**: Commit working states before major changes

## Common Issues and Solutions

### Yay build failures
- Ensure `base-devel` is installed
- Check disk space for build directory
- Clear yay cache: `yay -Sc`

### Plugin sourcing errors
- Verify plugin path exists: `ls -la ~/.zsh/` or `/usr/share/zsh/plugins/`
- Check .zshrc syntax: `zsh -n ~/.zshrc`
- Ensure plugin installed before sourcing

### Permission errors
- Verify `become: false` for user-specific tasks
- Check file ownership: `ls -la ~/.config/`
- Use `become_user` for explicit user context

### Role not found
- Install Galaxy dependencies: `ansible-galaxy install -r requirement.yml`
- Check role path in `ansible.cfg`: `roles_path = ./roles`
- Verify role directory structure exists

## Migration Notes

If working with older versions of this playbook:

1. **Legacy playbooks** are in `playbooks/debug/` (archived, not deleted)
2. **Old monolithic custom role** has been split into modular sub-tasks
3. **Plugin installation** migrated from git clone to pacman packages (where possible)
4. **Feature flags** introduced for conditional installations
5. **geerlingguy.docker** replaced inline Docker tasks

When updating from old versions:
- Review feature flags in `playbooks/local_setup.yml`
- Run `ansible-galaxy install -r requirement.yml` for new dependencies
- Check README.md for updated configuration variables
- to memorize