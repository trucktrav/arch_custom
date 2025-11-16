# Arch Linux Development Environment Setup

Ansible playbook for automated setup of a complete Arch Linux development workstation. Configures shell environments, development tools, window managers, and system utilities in a modular, reproducible way.

## Features

- **Shell Environments**: Choose between Starship (modern, fast) or Powerlevel10k (feature-rich)
- **Development Tools**: PyCharm Professional, R/RStudio, UV package manager
- **Terminal Emulators**: Alacritty with custom config, TMUX with TPM plugin manager
- **Containerization**: Docker and Docker Compose via geerlingguy.docker role
- **Databases**: PostgreSQL, MariaDB (MongoDB optional)
- **Desktop Environment**: Hyprland, i3-gaps, and tiling window managers
- **Productivity Tools**: flameshot, meld, btop, ranger, timeshift, syncthing, keepassxc
- **ML/AI Support**: Automatic NVIDIA CUDA/cuDNN installation if GPU detected
- **Shell Plugins**: zsh-autosuggestions, zsh-syntax-highlighting, fzf, fastfetch

## Prerequisites

```bash
# Install Ansible
sudo pacman -S ansible git

# Clone this repository
git clone <your-repo-url> ~/Projects/ansible/arch_custom
cd ~/Projects/ansible/arch_custom
```

## Quick Start

```bash
# 1. Set up Python environment (optional, for ansible-lint)
uv sync
source .venv/bin/activate

# 2. Install Galaxy role dependencies
ansible-galaxy install -r requirement.yml

# 3. Run the playbook
ansible-playbook playbooks/local_setup.yml

# Run specific components only
ansible-playbook playbooks/local_setup.yml --tags custom
ansible-playbook playbooks/local_setup.yml --tags starship
ansible-playbook playbooks/local_setup.yml --tags docker
ansible-playbook playbooks/local_setup.yml --tags claude_code
```

## Configuration Variables

Edit `playbooks/local_setup.yml` to customize your installation:

### Shell Configuration
```yaml
USE_STARSHIP: true          # Enable Starship prompt (modern, minimal)
USE_P10K: false             # Enable Powerlevel10k (feature-rich, Oh My Zsh)
INSTALL_CUSTOM_APPS: false  # installs a bunch of custom apps for dev
INSTALL_CLAUDE_CODE: true   # Install Claude Code CLI (requires Node.js)
```
**Note**: Only enable one shell prompt at a time to avoid conflicts.

### Terminal Configuration
```yaml
USE_ALACRITTY: false        # Set Alacritty as default terminal emulator
```

### Window Manager
```yaml
install_tiling_wm: true     # Install tiling window manager
tiling_wm: "i3-gaps"        # Options: i3-gaps, sway, awesome, qtile, bspwm
```

### User Settings
```yaml
username: "{{ lookup('env', 'USER') }}"     # Auto-detected from environment
user_home: "/home/{{ username }}"           # Home directory path
git_server_port: 3000                       # Local Git server port (if enabled)
```

## What Gets Installed

### Core System Packages
- Base development tools (base-devel, git, curl, wget, openssh)
- Python 3 and pip
- Node.js and npm (for Claude Code CLI)
- yay (AUR helper)
- UV (modern Python package manager)
- Hyprland compositor

### Development Applications
- **PyCharm Professional** (via AUR)
- **R and RStudio Desktop** (data science stack)
- **PostgreSQL and MariaDB** (databases)
- **Docker and Docker Compose** (containerization)
- **Claude Code CLI** (AI coding assistant, optional)

### Terminal & Shell
- **Alacritty** terminal with custom configuration
- **TMUX** with TPM (Tmux Plugin Manager)
- **Zsh** with choice of:
  - **Starship** prompt (Rust-based, fast, modern)
  - **Powerlevel10k** with Oh My Zsh (customizable, feature-rich)
- **Fish** shell (alternative)
- **fastfetch** (system info display)

### Shell Plugins & Tools
- zsh-autosuggestions
- zsh-syntax-highlighting
- zsh-completions
- zsh-transient-prompt
- fzf (fuzzy finder)
- eza (modern ls replacement)
- bat (modern cat replacement)
- zoxide (smart cd command)

### Desktop Environment
- Tiling window manager (i3-gaps/sway/etc.)
- rofi (application launcher)
- dunst (notifications)
- picom (compositor)
- feh (wallpaper manager)
- Additional fonts (Nerd Fonts, Noto, etc.)

### Productivity Tools
- flameshot (screenshots)
- meld (diff viewer)
- zeal (offline documentation)
- btop (system monitor)
- ranger (file manager)
- timeshift (system backups)
- syncthing (file sync)
- keepassxc (password manager)

### GPU/ML Support
If NVIDIA GPU is detected:
- CUDA toolkit
- cuDNN (deep learning library)

## Post-Installation Steps

After running the playbook, complete these manual steps:

### 1. Docker Group Permissions
```bash
# Log out and log back in for docker group changes to take effect
# Or run: newgrp docker
```

### 2. TMUX Plugin Installation
```bash
# Start tmux
tmux

# Inside tmux, install plugins
# Press: Ctrl+Space + I (capital I)

# Or manually run:
~/.config/tmux/plugins/tpm/bin/install_plugins
```

### 3. Python Virtual Environments (Optional)
If you need Python environments for development:
```bash
# For LLM development
cd ~/Projects/llm-dev
uv venv
source .venv/bin/activate
uv pip install torch transformers datasets huggingface_hub

# For general app development
cd ~/Projects/app-dev
uv venv
source .venv/bin/activate
uv pip install <your-packages>

# For data analysis
cd ~/Projects/data-analysis
uv venv
source .venv/bin/activate
uv pip install pandas numpy matplotlib jupyter
```

### 4. Starship Configuration
If using Starship (`USE_STARSHIP: true`):
```bash
# Starship config is installed to: ~/.config/starship.toml
# ZSH config is installed to: ~/.zshrc
# Customize as needed by editing these files
```

### 5. Powerlevel10k Configuration
If using Powerlevel10k (`USE_P10K: true`):
```bash
# Run the configuration wizard
p10k configure

# Or manually edit: ~/.p10k.zsh
```

### 6. Switch Default Shell
```bash
# If you want to change your default shell (already done by playbook for zsh)
chsh -s /bin/zsh    # For Zsh
chsh -s /bin/fish   # For Fish
```

### 7. Claude Code Setup (Optional)
If you installed Claude Code (`INSTALL_CLAUDE_CODE: true`):
```bash
# Restart your terminal to load the updated PATH
# Or source your shell config:
source ~/.zshrc

# Set your Anthropic API key
export ANTHROPIC_API_KEY='your-api-key-here'

# Add to your shell config for persistence:
echo 'export ANTHROPIC_API_KEY="your-api-key-here"' >> ~/.zshrc

# Test Claude Code installation
claude --version

# Start using Claude Code
claude
```

For more information, visit: https://docs.claude.com/claude-code

## Project Structure

```
arch_custom/
├── playbooks/
│   ├── local_setup.yml          # Main playbook (PRODUCTION)
│   └── debug/                   # Archived legacy playbooks
├── roles/
│   ├── custom/                  # Core system setup
│   │   ├── tasks/
│   │   │   ├── main.yml         # Base packages, yay, UV, Hyprland
│   │   │   ├── capps.yml        # Development apps (PyCharm, R, DBs)
│   │   │   └── terms.yml        # Terminal setup (Alacritty, TMUX)
│   │   └── files/
│   │       └── alacritty.yml    # Alacritty configuration
│   ├── manage_zsh/              # Zsh with Powerlevel10k
│   │   ├── tasks/
│   │   │   ├── main.yml         # Main setup (packages, fastfetch)
│   │   │   ├── zsh-plugin.yml   # Plugin installation (git clone)
│   │   │   └── zsh-p10k.yml     # Oh My Zsh + Powerlevel10k
│   │   └── files/
│   │       ├── .zshrc           # Zsh configuration
│   │       └── .p10k.zsh        # Powerlevel10k theme
│   ├── starship/                # Starship prompt
│   │   ├── tasks/
│   │   │   └── main.yml         # Starship + modern CLI tools
│   │   └── files/
│   │       ├── starship.toml    # Starship configuration
│   │       └── zshrc            # Zsh config for Starship
│   ├── claude_code/             # Claude Code CLI
│   │   └── tasks/
│   │       └── main.yml         # Install Node.js, npm, Claude Code
│   └── geerlingguy.docker/      # Docker role (Galaxy)
├── inventories/
│   └── hosts                    # Localhost inventory
├── requirement.yml              # Ansible Galaxy dependencies
├── ansible.cfg                  # Ansible configuration
├── pyproject.toml               # Python dependencies
└── CLAUDE.md                    # Development guide
```

## Tags Reference

Use tags to run specific parts of the playbook:

- `custom` - Core system packages and tools
- `manage_zsh` - Zsh with Powerlevel10k setup
- `starship` - Starship prompt installation
- `docker` - Docker and Docker Compose
- `claude_code` - Claude Code CLI installation
- `dev_tools` - Development tools (includes Claude Code)
- `base_setup` - All base components (custom + manage_zsh + starship + docker + claude_code)

Example:
```bash
# Install only Docker
ansible-playbook playbooks/local_setup.yml --tags docker

# Install only Claude Code
ansible-playbook playbooks/local_setup.yml --tags claude_code

# Install everything except Docker
ansible-playbook playbooks/local_setup.yml --skip-tags docker
```

## Customization

### Adding New Packages
Edit `roles/custom/tasks/main.yml` and add to the appropriate pacman task:
```yaml
- name: Install development essentials
  community.general.pacman:
    name:
      - existing-package
      - your-new-package  # Add here
    state: present
```

### Adding AUR Packages
For AUR packages, add a task using yay:
```yaml
- name: Install your AUR package
  become: false
  ansible.builtin.command: yay -S --noconfirm your-package-name
  register: result
  changed_when: result.rc == 0
  failed_when: result.rc != 0 and 'error' in result.stderr
```

### Switching Shell Themes
1. Edit `playbooks/local_setup.yml`
2. Set `USE_STARSHIP: true` OR `USE_P10K: true` (not both)
3. Re-run playbook

## Troubleshooting

### Yay installation fails
```bash
# Manually install yay
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

### Docker permission denied
```bash
# Add yourself to docker group and reload
sudo usermod -aG docker $USER
newgrp docker
```

### Starship not showing
```bash
# Check if Starship is initialized in .zshrc
grep "starship init" ~/.zshrc

# Manually add if missing
echo 'eval "$(starship init zsh)"' >> ~/.zshrc
```

## Contributing

1. Test changes in a VM or on a non-production system
2. Use `--check` mode for dry runs: `ansible-playbook playbooks/local_setup.yml --check`
3. Run linting: `ansible-lint playbooks/local_setup.yml`
4. Update CLAUDE.md if adding new architectural patterns

## License

This is a personal configuration repository. Use at your own risk and customize to your needs.
