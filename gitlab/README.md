# Runway GitLab CI Template

This template provides complete Runway CLI setup functionality for GitLab CI/CD pipelines, equivalent to the GitHub Action version.

## Quick Start

1. Include the template in your `.gitlab-ci.yml`:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/your-org/setup-runway/main/gitlab/runway-complete.yml'

deploy:
  extends: .runway-setup
  variables:
    RUNWAY_USERNAME: $RUNWAY_USER
    RUNWAY_PASSWORD: $RUNWAY_PASS
  script:
    - runway app deploy
```

2. Set required CI/CD variables in your GitLab project:
   - `RUNWAY_USER`: Your runway username
   - `RUNWAY_PASS`: Your runway password

## Configuration Variables

### Required Variables

| Variable | Description |
|----------|-------------|
| `RUNWAY_USERNAME` | Runway account username |
| `RUNWAY_PASSWORD` | Runway account password |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `RUNWAY_APPLICATION` | Runway application name | `""` |
| `RUNWAY_VERSION` | Runway CLI version | `"latest"` |
| `RUNWAY_LOG_LEVEL` | Log level: debug, info, warn, error | `"error"` |
| `RUNWAY_CONTROLLER_URL` | Custom controller URL | `""` |
| `RUNWAY_ADD_KEY` | Import SSH key: "true" or "false" | `"false"` |
| `RUNWAY_SETUP_SSH` | Setup SSH agent: "true" or "false" | `"false"` |
| `RUNWAY_PRIVATE_KEY` | SSH private key content | `""` |
| `RUNWAY_PUBLIC_KEY` | SSH public key content | `""` |
| `RUNWAY_PUBLIC_KEY_LOCATION` | SSH public key file path | `"~/.ssh/id_rsa.pub"` |
| `RUNWAY_PRIVATE_KEY_LOCATION` | SSH private key file path | `"~/.ssh/id_rsa"` |

## Usage Examples

Please note that any deployment requires your SSH key. But you can set them up yourself if you don't want us to paste them into place and set permissions.

### Basic Example

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/your-org/setup-runway/main/gitlab/runway-complete.yml'

deploy:
  extends: .runway-setup
  variables:
    RUNWAY_USERNAME: $RUNWAY_USER
    RUNWAY_PASSWORD: $RUNWAY_PASS
  script:
    - runway app ls
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### Deployment (including SSH Setup)

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/your-org/setup-runway/main/gitlab/runway-complete.yml'

deploy:
  extends: .runway-setup
  variables:
    RUNWAY_USERNAME: $RUNWAY_USER
    RUNWAY_PASSWORD: $RUNWAY_PASS
    RUNWAY_APPLICATION: "my-app"
    RUNWAY_SETUP_SSH: "true"
    RUNWAY_ADD_KEY: "true"
    RUNWAY_PRIVATE_KEY: $SSH_PRIVATE_KEY
    RUNWAY_PUBLIC_KEY: $SSH_PUBLIC_KEY
  script:
    - runway app deploy -a $RUNWAY_APPLICATION
```

### Multi-Stage Pipeline

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/your-org/setup-runway/main/gitlab/runway-complete.yml'

stages:
  - deploy-staging
  - deploy-production

deploy-staging:
  stage: deploy-staging
  extends: .runway-setup
  variables:
    RUNWAY_USERNAME: $RUNWAY_USER
    RUNWAY_PASSWORD: $RUNWAY_PASS
    RUNWAY_APPLICATION: "my-app-staging"
  script:
    - runway app deploy -a my-app-staging
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy-production:
  stage: deploy-production
  extends: .runway-setup
  variables:
    RUNWAY_USERNAME: $RUNWAY_USER
    RUNWAY_PASSWORD: $RUNWAY_PASS
    RUNWAY_APPLICATION: "my-app-prod"
  script:
    - runway app deploy -a my-app-prod
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  when: manual
```

## Setting Up CI/CD Variables

1. Go to your GitLab project
2. Navigate to Settings → CI/CD → Variables
3. Add the required variables:
   - `RUNWAY_USER` (Type: Variable, Masked: Yes)
   - `RUNWAY_PASS` (Type: Variable, Masked: Yes)
   - `SSH_PRIVATE_KEY` (Type: File, Masked: Yes)
   - `SSH_PUBLIC_KEY` (Type: Variable)

## Features

This template provides the same functionality as the GitHub Action:

✅ **CLI Installation**: Downloads and installs runway CLI  
✅ **Authentication**: Logs into runway with credentials  
✅ **Architecture Detection**: Supports x86_64/amd64 and ARM64 runners  
✅ **SSH Key Management**: Optionally sets up SSH keys  
✅ **Application Management**: Creates or connects to runway applications  
✅ **SSH Agent Setup**: Configures SSH agent for deployments  
✅ **Custom Controllers**: Support for enterprise/testing environments  
✅ **Version Control**: Specify exact runway CLI versions  

## Architecture Support

The template automatically detects runner architecture:

- `x86_64` → `amd64` (Intel/AMD 64-bit)
- `aarch64` → `arm64` (Linux ARM64)  
- `arm64` → `arm64` (macOS ARM64)

## Troubleshooting

### Binary Installation Issues

If the runway binary can't be moved to `/usr/local/bin`, the template will try `~/bin` or place it in the current directory. Ensure your GitLab runner has appropriate permissions or add the current directory to PATH:

```yaml
script:
  - export PATH="$PWD:$PATH"
  - runway app deploy
```

### SSH Issues

Ensure your SSH keys are properly formatted in GitLab CI/CD variables:

 - Private key should include `-----BEGIN` and `-----END` lines
 - Public key should be a single line

### Permission Errors

The template sets proper permissions (0600) on SSH key files automatically.

## Contributing

This template mirrors the functionality of the GitHub Action in `action.yml`. When updating, ensure both remain in sync.