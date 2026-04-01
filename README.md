# vaultctl

**vaultctl** is a lightweight Bash CLI utility designed to simplify interactions with HashiCorp Vault - secretsmanager. It provides utility commands for managing secrets, authentication, and environment variables, whether used in scripts or directly from the shell.


- [vaultctl](#vaultctl)
  - [Features](#features)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
    - [Quick Install (Recommended)](#quick-install-recommended)
    - [Installation Options](#installation-options)
    - [Add to PATH](#add-to-path)
  - [Quick Start](#quick-start)
  - [Configuration](#configuration)
    - [Environment Variables](#environment-variables)
    - [Configuration Files](#configuration-files)
  - [Commands](#commands)
    - [Authentication](#authentication)
      - [`login`](#login)
      - [`logout`](#logout)
      - [`status`](#status)
      - [`export-vault`](#export-vault)
      - [`config`](#config)
    - [Operations](#operations)
      - [`read`](#read)
      - [`write`](#write)
      - [`mv`](#mv)
      - [`renew`](#renew)
      - [`rm` / `delete`](#rm--delete)
    - [Find](#find)
    - [Environment Variable Export](#environment-variable-export)
      - [`export-env`](#export-env)
      - [`export-env-all`](#export-env-all)
    - [User specific management](#user-specific-management)
      - [`export-users`](#export-users)
      - [`export-users-all`](#export-users-all)
      - [`export-users-path`](#export-users-path)
      - [`export-users-path-all`](#export-users-path-all)
      - [`export-users-cbi`](#export-users-cbi)
      - [`export-users-cbi-all`](#export-users-cbi-all)
  - [Usage Examples](#usage-examples)
    - [Complete Workflow](#complete-workflow)
    - [In script](#in-script)
    - [Batch Mode for Scripts](#batch-mode-for-scripts)
    - [Cache Management](#cache-management)
  - [Performance Tips](#performance-tips)
  - [Contributing](#contributing)
  - [AI-Assisted Development](#ai-assisted-development)
  - [License](#license)


## Features

- **LDAP Authentication** - Simple ldap login with username/password
- **Token Management** - Token storage and validation
- **Secret Operations** - Read, write, move, and delete secrets
- **Environment Export** - Export secrets directly as shell environment variables
- **Search** - Find secrets with glob patterns and caching
- **Parallel Scanning** - Multi-worker secret discovery for large vaults
- **Caching** - Configurable TTL-based caching for search performance
- **User secrets specific mount** - Helpers for user-specific paths (`users/<username>`)


## Prerequisites

- **HashiCorp Vault CLI** - [Download](https://developer.hashicorp.com/vault/downloads)
- **jq** - JSON processor for command-line

## Installation

### Quick Install (Recommended)

Install with a single command using curl:

```bash
curl -fsSL https://raw.githubusercontent.com/eclipse-cbi/vaultctl/main/install.sh | bash
```

### Installation Options

```bash
git clone https://github.com/eclipse-cbi/vaultctl.git
cd vaultctl
./install.sh
```

### Add to PATH

If the installation directory is not in your PATH:

```bash
# For bash users - add to ~/.bashrc
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.bashrc
source ~/.bashrc

# For zsh users - add to ~/.zshrc
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc
source ~/.zshrc
```

## Quick Start

```bash
# Login to Vault
vaultctl login

# Check authentication status
vaultctl status

# Read a secret
vaultctl read cbi technology.cbi

# Logout
vaultctl logout
```

## Configuration

### Configuration Priority

All configuration variables follow this priority order:

1. **Environment variable** - Values set in your shell (highest priority)
2. **Configuration file** - Values saved in `~/.vaultctl`
3. **Default value** - Built-in defaults (lowest priority)

This means you can override config file settings by exporting environment variables, and config file settings override the script defaults.

### Environment Variables

- **`VAULT_ADDR`** - Vault server address (default: `https://secretsmanager.eclipse.org`)
  - Can be set via: environment variable, config file, or uses default
- **`VAULT_TOKEN`** - Vault authentication token (managed automatically)
- **`VAULT_USERNAME`** - Your LDAP username (saved after first login)
- **`VAULT_MOUNT`** - Default mount point for read/write operations (optional)
  - Can be set via: environment variable, config file, or left unset
- **`VAULT_CACHE_TTL`** - Cache time-to-live in seconds (default: `86400` = 1 day)
  - Can be set via: environment variable, config file, or uses default
- **`VAULT_PARALLEL`** - Number of parallel scan workers (default: `5`)
  - Can be set via: environment variable, config file, or uses default

### Configuration Files

- **`~/.vault-token`** - Stores your authentication token
- **`~/.vaultctl`** - Stores configuration (username, VAULT_MOUNT, etc.)
- **`~/.vaultctl_cache/`** - Directory for cached secret indexes

## Commands

### Authentication

#### `login`

Authenticate with Vault using LDAP credentials.

```bash
vaultctl login
```

#### `logout`
Revoke token and clear local authentication.

```bash
vaultctl logout
```

#### `status`
Display current authentication status and token information.

```bash
vaultctl status
```

#### `export-vault`
Export Vault environment variables for use in current shell.

```bash
eval $(vaultctl export-vault)
```

#### `config`
Manage vaultctl configuration settings.

```bash
# Show current configuration
vaultctl config

# Set default mount point
vaultctl config VAULT_MOUNT=cbi

# Set custom Vault server address
vaultctl config VAULT_ADDR=https://vault.example.com

# Set cache TTL (in seconds)
vaultctl config VAULT_CACHE_TTL=3600

# Set number of parallel workers for scanning
vaultctl config VAULT_PARALLEL=10

# Now you can use commands without specifying mount
vaultctl read technology.cbi/github.com/api-token
vaultctl write myproject/secrets key=value
```

**Supported Configuration Keys:**
- `VAULT_MOUNT` - Default mount point for read/write operations
- `VAULT_ADDR` - Vault server address
- `VAULT_CACHE_TTL` - Cache time-to-live in seconds
- `VAULT_PARALLEL` - Number of parallel scan workers

---

### Operations

#### `read`
Read a secret or list keys in a secret path.

```bash
# With explicit mount
vaultctl read <mount> <path/to/secret/field>

# With default mount (set via config or VAULT_MOUNT env var)
vaultctl read <path/to/secret/field>

# List keys in a secret
vaultctl read <mount> <path/to/secret>

# Options:
#   -b, --batch    Silent mode (exit code only)
#   -c, --clip     Copy secret to clipboard instead of displaying it
#   -v, --verbose  Show vault commands being executed
```

**Examples:**
```bash
# Set default mount
vaultctl config VAULT_MOUNT=users

# Use without mount
vaultctl read myuser/cbi/JENKINS_USERNAME

# Copy secret to clipboard (Linux: requires xclip or xsel, macOS: uses pbcopy)
vaultctl read -c myuser/cbi/JENKINS_USERNAME
vaultctl read -c cbi technology.cbi/github.com/api-token

# Or specify mount explicitly
vaultctl read users myuser/cbi/JENKINS_USERNAME
vaultctl read cbi technology.cbi/github.com/api-token
vaultctl read users myuser  # List all keys
vaultctl read -v cbi technology.cbi/repo  # Verbose mode
```

#### `write`
Write or update a secret.

```bash
# With explicit mount
vaultctl write <mount> <path> <key>=<value> [<key2>=<value2> ...]

# With default mount (set via config or VAULT_MOUNT env var)
vaultctl write <path> <key>=<value> [<key2>=<value2> ...]

# Write from file
vaultctl write <mount> <path> <key>=@file.txt

# Write all keys from JSON file
vaultctl write <mount> <path> @secrets.json
```

**Examples:**
```bash
vaultctl write users myuser/cbi username=john password=secret123
vaultctl write users myuser/github token=@github_token.txt
vaultctl write cbi technology.cbi/new-secret @credentials.json
```


#### `mv`
Move (rename) a secret path.

```bash
vaultctl mv <mount> <source-path> <destination-path>
```

**Examples:**
```bash
vaultctl mv cbi technology.cbi/old-repo technology.cbi/new-repo
vaultctl mv users myuser/old-dir myuser/new-dir
```

#### `renew`
Renew the Vault token to extend its validity.

```bash
vaultctl renew [increment]
# increment: Optional duration (e.g., 2h, 30m, 1d). Default is 2h.
```

**Examples:**
```bash
# Renew token for 2 hours (default)
vaultctl renew
# Renew token for 4 hours
vaultctl renew 4h
```

#### `rm` / `delete`
Permanently delete a secret and all its versions.

```bash
vaultctl rm <mount> <path>
vaultctl rm -f <mount> <path>  # Skip confirmation

# Options:
#   -f, --force    Skip confirmation prompt
```

**Examples:**
```bash
vaultctl rm cbi technology.cbi/deprecated-secret
vaultctl rm -f users myuser/old-credentials  # No confirmation
```

---

### Find

Search for secret paths matching a glob pattern with intelligent caching.

```bash
vaultctl find <mount> [pattern] [options]

# Options:
#   --no-cache          Bypass cache (still updates it)
#   --no-cache-write    Scan without updating cache
#   -b, --bare          Output paths only (no logs)
#   --clear-cache       Clear cache for this mount
#   --clear-all-cache   Clear all caches
#   --cache-info        Show cache status
#   --cache-ttl <sec>   Override cache TTL
#   --parallel <n>      Number of scan workers
```

**Examples:**
```bash
vaultctl find users                      # List all paths in users mount
vaultctl find users 'myuser/*'           # All secrets for myuser
vaultctl find users '*/cbi/*'            # All cbi secrets across users
vaultctl find cbi 'technology.cbi/*'     # All technology.cbi secrets
vaultctl find users --cache-info         # Show cache status
vaultctl find users --clear-cache        # Invalidate cache
vaultctl find --clear-all-cache          # Clear all caches
vaultctl find users --parallel 10        # Use 10 workers
```

---

### Environment Variable Export

#### `export-env`
Export specific secrets as environment variables from any mount/path.

```bash
vaultctl export-env <mount> <path> <ENV_VAR:key> [<ENV_VAR2:key2> ...] [options]

# Options:
#   --prefix <PREFIX>   Add prefix to all variable names
#   --uppercase         Convert variable names to uppercase
```

**Examples:**
```bash
eval $(vaultctl export-env users myuser USER:username PASS:password)
eval $(vaultctl export-env cbi technology.cbi/github TOKEN:api-token)
eval $(vaultctl export-env users myuser user:username --prefix MY_ --uppercase)
# Result: MY_USER=...
```

#### `export-env-all`
Export ALL secrets from a mount/path as environment variables.

```bash
vaultctl export-env-all <mount> <path> [options]

# Options:
#   --prefix <PREFIX>   Add prefix to all variable names
#   --uppercase         Convert variable names to uppercase
```

**Examples:**
```bash
eval $(vaultctl export-env-all users myuser)
eval $(vaultctl export-env-all cbi technology.cbi/github --prefix GH_)
eval $(vaultctl export-env-all users myuser --prefix MY_ --uppercase)
```

---

### User specific management

These commands provide convenient access to secrets in your user directory (`users/<username>`).

#### `export-users`
Export specific secrets from your user path.

```bash
vaultctl export-users <ENV_VAR[:key]> [<ENV_VAR2[:key2]> ...] [options]

# Options:
#   --prefix <PREFIX>   Add prefix to variable names
#   --uppercase         Convert to uppercase
```

**Examples:**
```bash
eval $(vaultctl export-users JENKINS_USERNAME JENKINS_PASSWORD)
eval $(vaultctl export-users USER:JENKINS_USERNAME PASS:JENKINS_PASSWORD)
eval $(vaultctl export-users jenkins_username --prefix CI_ --uppercase)
# Result: CI_JENKINS_USERNAME=...
```

#### `export-users-all`
Export ALL secrets from your user path.

```bash
eval $(vaultctl export-users-all)
eval $(vaultctl export-users-all --prefix MY_)
eval $(vaultctl export-users-all --prefix my_ --uppercase)
```

#### `export-users-path`
Export secrets from a subpath of your user directory.

```bash
vaultctl export-users-path <subpath> <ENV_VAR[:key]> [...] [options]
```

**Examples:**
```bash
eval $(vaultctl export-users-path cbi JENKINS_USERNAME JENKINS_PASSWORD)
eval $(vaultctl export-users-path github TOKEN:api-token --prefix GH_)
```

#### `export-users-path-all`
Export ALL secrets from a subpath.

```bash
eval $(vaultctl export-users-path-all github)
eval $(vaultctl export-users-path-all cbi --prefix CBI_ --uppercase)
```

#### `export-users-cbi`
Shortcut for exporting from `users/<username>/cbi`.

```bash
eval $(vaultctl export-users-cbi JENKINS_USERNAME JENKINS_PASSWORD)
eval $(vaultctl export-users-cbi JENKINS_USER:JENKINS_USERNAME --prefix CBI_)
```

#### `export-users-cbi-all`
Export ALL secrets from your CBI subpath.

```bash
eval $(vaultctl export-users-cbi-all)
eval $(vaultctl export-users-cbi-all --prefix CBI_)
```

---

## Usage Examples

### Complete Workflow

```bash
# 1. Login
vaultctl login
# Enter your LDAP username: john.doe
# Enter your password: ****
# ✅ Vault login successful

# 2. Check status
vaultctl status
# ✅ Authenticated
#  VAULT_TOKEN: hvs.CAE...

# 3. Read a secret
vaultctl read cbi technology.cbi/repo.eclipse.org/token-username
# my-jenkins-user

# 4. Export environment variables
eval $(vaultctl export-users JENKINS_USERNAME JENKINS_PASSWORD)
echo $JENKINS_USERNAME
# my-jenkins-user

# 5. Find secrets
vaultctl find cbi '*repo.eclipse.org*'
#  Searching in mount 'cbi' with pattern: *repo.eclipse.org**
# modeling.acceleo/repo.eclipse.org
# modeling.teneo/repo.eclipse.org
# science.statet/repo.eclipse.org
# technology.cbi/repo.eclipse.org
# technology.dash/repo.eclipse.org
# technology.edc/repo.eclipse.org
# technology.sensinact/repo.eclipse.org
# ...

# 6. Write a new secret
vaultctl write cbi technology.cbi/repo.eclipse.org token-username=my-secret-token

# 7. Logout
vaultctl logout
# ✅ Token revoked successfully
# ✅ Logged out successfully
```

### In script

```bash
#!/bin/bash
set -e

# Login and export credentials
vaultctl login
eval $(vaultctl export-users-cbi JENKINS_USERNAME JENKINS_PASSWORD)

# Use the credentials
curl -u "$JENKINS_USERNAME:$JENKINS_PASSWORD" https://jenkins.example.com/job/my-job/build

# Cleanup
vaultctl logout
```

### Batch Mode for Scripts

```bash
#!/bin/bash

# Check if authenticated (silent mode)
if vaultctl read -b cbi technology.cbi/repo.eclipse.org/user-token; then
    echo "Secret exists"
else
    echo "Secret not found or not authenticated"
    exit 1
fi
```

### Cache Management

```bash
# View cache status for all mounts
vaultctl find --cache-info

# View cache for specific mount
vaultctl find users --cache-info

# Clear specific mount cache
vaultctl find users --clear-cache

# Clear all caches
vaultctl find --clear-all-cache

# Bypass cache for one search
vaultctl find users 'myuser/*' --no-cache

# Change cache TTL to 2 hours
VAULT_CACHE_TTL=7200 vaultctl find users

# Or per-command
vaultctl find users --cache-ttl 7200
```

## Performance Tips

1. **Use caching** - The first `find` scan is slow, but subsequent searches are using local cache
2. **Adjust parallelism** - Increase `VAULT_PARALLEL` for large vaults, default `5`
   ```bash
   export VAULT_PARALLEL=10
   vaultctl find users
   ```
3. **Use batch mode** - For scripting, use `-b` to suppress logs

## Contributing

Contributions are welcome! Please ensure:
- Scripts are compatible with Bash 4.0+
- Code follows existing style conventions
- All changes are tested
- Documentation is updated

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines.

## AI-Assisted Development

Parts of this codebase were developed with the assistance of AI tools including GitHub Copilot. These tools helped accelerate development while maintaining code quality and consistency with Eclipse Foundation services.

## License

Copyright (c) 2026 Eclipse Foundation and others.

This program and the accompanying materials are made available under the terms of the Eclipse Public License 2.0 which is available at http://www.eclipse.org/legal/epl-v20.html

SPDX-License-Identifier: EPL-2.0

See [LICENSE](LICENSE) file for full license text.
