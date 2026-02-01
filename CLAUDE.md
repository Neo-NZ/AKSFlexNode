# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AKS Flex Node is a Go service that extends Azure Kubernetes Service (AKS) to non-Azure VMs through Azure Arc integration. It transforms any Ubuntu VM into a fully managed AKS worker node by installing container runtime components, Kubernetes components, and connecting to Azure Arc for cloud management.

**Target Platform:** Ubuntu 22.04.5 LTS (x86_64)
**Language:** Go 1.24+
**Module Path:** `go.goms.io/aks/AKSFlexNode`

### Architecture Documentation

For comprehensive architecture details, refer to **[ARCHITECTURE.md](ARCHITECTURE.md)**:
- High-level system architecture with visual diagrams
- Component interactions and data flow (with numbered sequence)
- Azure API specifications (Arc, RBAC, AKS, Identity Service)
- Authentication and security model
- Detailed 11-step bootstrap process
- Network requirements and communication patterns

This document is essential for understanding how the system works before making code changes.

## Build and Test Commands

```bash
# Build for current platform
make build

# Build for specific platforms
make build-linux-amd64
make build-linux-arm64

# Build all platforms
make build-all

# Run tests
make test
go test ./...

# Run tests with coverage
make test-coverage

# Run tests with race detector
make test-race

# Run specific package tests
go test ./pkg/config/
go test ./pkg/logger/

# Code quality checks
make fmt          # Format code
make vet          # Run go vet
make lint         # Run golangci-lint
make check        # Run all checks (fmt + vet + lint + test)
make verify       # Verify and tidy dependencies

# Clean build artifacts
make clean

# View build metadata
make update-build-metadata
```

## Running the Application

The application has three main commands:

```bash
# Bootstrap: Transform VM into AKS node
sudo aks-flex-node bootstrap --config /etc/aks-flex-node/config.json

# Unbootstrap: Clean removal of all components
sudo aks-flex-node unbootstrap --config /etc/aks-flex-node/config.json

# Version: Show version information
aks-flex-node version
```

## Architecture Overview

### Command Flow

1. **Entry Point:** `main.go` sets up Cobra CLI with signal handling and context management
2. **Commands:** `commands.go` defines bootstrap/unbootstrap/version commands
3. **Bootstrapper:** `pkg/bootstrapper/` orchestrates the installation/removal steps
4. **Components:** `pkg/components/` contains individual installation modules

### Bootstrap Process (Sequential Execution)

The bootstrap process executes these steps in strict order (defined in `pkg/bootstrapper/bootstrapper.go:34-48`):

1. **Arc Registration** - Register VM with Azure Arc for managed identity
2. **Service Stop** - Stop kubelet if running
3. **Directories** - Create required filesystem directories
4. **System Configuration** - Configure kernel parameters, disable swap
5. **Runc** - Install container runtime (runc)
6. **Containerd** - Install container runtime (containerd)
7. **Kubernetes Components** - Install kubectl, kubelet, kubeadm
8. **CNI** - Setup Container Network Interface plugins
9. **Cluster Credentials** - Download kubeconfig using Arc managed identity
10. **Kubelet** - Configure kubelet service with Arc MSI authentication
11. **Services** - Start containerd and kubelet services

**Critical:** Bootstrap fails fast on first error. Each step must succeed before proceeding to the next.

### Unbootstrap Process (Sequential Cleanup)

Unbootstrap executes steps in reverse order (defined in `pkg/bootstrapper/bootstrapper.go:54-69`) and continues even if individual steps fail, to ensure maximum cleanup.

### Component Architecture

All components follow the `Executor` interface pattern (defined in `pkg/bootstrapper/executor.go:14-23`):

```go
type Executor interface {
    Execute(ctx context.Context) error
    IsCompleted(ctx context.Context) bool
    GetName() string
}
```

Bootstrap steps also implement `StepExecutor` which adds validation:
```go
type StepExecutor interface {
    Executor
    Validate(ctx context.Context) error
}
```

Each component has:
- **Installer:** `*_installer.go` - Installation logic
- **Uninstaller:** `*_uninstaller.go` - Cleanup logic
- **Base/Helpers:** Shared logic
- **Constants:** `consts.go` - Component-specific constants

Component locations:
- `pkg/components/arc/` - Azure Arc registration and RBAC
- `pkg/components/cluster_credentials/` - Kubeconfig download
- `pkg/components/cni/` - CNI plugin setup
- `pkg/components/containerd/` - Containerd installation
- `pkg/components/directories/` - Directory creation
- `pkg/components/kubelet/` - Kubelet configuration
- `pkg/components/kubernetes_components/` - kubectl/kubelet/kubeadm binaries
- `pkg/components/runc/` - Runc installation
- `pkg/components/services/` - Systemd service management
- `pkg/components/system_configuration/` - Kernel parameters, swap

### Configuration System

Configuration is loaded from JSON files (default: `/etc/aks-flex-node/config.json`):

- **Structure:** Defined in `pkg/config/structs.go`
- **Loader:** `pkg/config/config.go` handles loading with defaults
- **Key sections:**
  - `azure` - Azure subscription, tenant, Arc settings, target cluster
  - `agent` - Log level and directory
  - `containerd` - Containerd version and pause image
  - `kubernetes` - K8s version and download URLs
  - `runc` - Runc version
  - `node` - Max pods, labels, kubelet settings
  - `paths` - Filesystem paths for K8s and CNI

### Authentication System (`pkg/auth/auth.go`)

Two authentication modes:

1. **Service Principal** - If configured in `azure.servicePrincipal` section
2. **Azure CLI** - Falls back to `az login` credentials
   - Automatically checks CLI auth status
   - Prompts for interactive login if needed

Arc operations use managed identity credential after Arc registration completes.

### Logging

Logger setup in `pkg/logger/logger.go`:
- Uses logrus for structured logging
- Writes to both file and stdout
- Logger is stored in context and retrieved via `logger.GetLoggerFromContext(ctx)`
- Log files stored in configured `agent.logDir` (default: `/var/log/aks-flex-node`)

## Important Development Considerations

### Error Handling Philosophy

- **Bootstrap:** Fail-fast on any error to prevent partial configurations
- **Unbootstrap:** Continue on errors to maximize cleanup
- **Validation:** All bootstrap steps validate preconditions before execution
- **Idempotency:** Steps check `IsCompleted()` before executing to support retries

### Adding New Components

When adding a new component step:

1. Create directory under `pkg/components/<component>/`
2. Implement installer with `StepExecutor` interface (Execute, Validate, IsCompleted, GetName)
3. Implement uninstaller with `Executor` interface (Execute, IsCompleted, GetName)
4. Add constants to `consts.go`
5. Add to bootstrap/unbootstrap step sequences in correct order
6. Consider dependencies on other steps (e.g., Arc must complete before cluster credentials)

### Testing Locally

The application requires root/sudo privileges because it:
- Installs system packages and binaries
- Creates system directories
- Modifies kernel parameters
- Manages systemd services

For development testing, use a VM environment, not your local machine.

### Version Injection

Version info is injected at build time via linker flags (see `Makefile:7` and `commands.go:16-19`):
- `Version` - Git tag or "dev"
- `GitCommit` - Short commit hash
- `BuildTime` - Build timestamp

### Testing and CI/CD

The project has automated testing via GitHub Actions (`.github/workflows/pr-checks.yml`):

**Automated Checks:**
- **Build** - Verifies builds on Go 1.24, tests all platforms
- **Test** - Runs tests with race detection, enforces 30% minimum coverage
- **Lint** - Runs golangci-lint with comprehensive checks (see `.golangci.yml`)
- **Security** - Scans with gosec, uploads results to GitHub Security
- **Code Quality** - Checks formatting (gofmt), correctness (go vet), static analysis (staticcheck)
- **Dependency Review** - Reviews dependencies for security vulnerabilities

**Before Submitting PRs:**
```bash
make check        # Run all quality checks
make build-all    # Verify all platforms build
make verify       # Check dependencies
```

See `TESTING.md` for comprehensive testing guide.

## Key Files

- `main.go` - Application entry point with Cobra setup
- `commands.go` - Command definitions and handlers
- `Makefile` - Build targets and quality check targets
- `pkg/bootstrapper/bootstrapper.go` - Bootstrap orchestration
- `pkg/bootstrapper/executor.go` - Executor interfaces and result types
- `pkg/config/structs.go` - Configuration structure
- `pkg/auth/auth.go` - Azure authentication factory
- `aks-flex-node@.service` - Systemd service template
- `aks-flex-node-sudoers` - Sudoers configuration for service user
- `scripts/install.sh` - Installation script
- `scripts/uninstall.sh` - Uninstallation script
- `.golangci.yml` - Linter configuration
- `TESTING.md` - Comprehensive testing and CI/CD guide

## Release Process

Releases are automated via GitHub Actions (`.github/workflows/release.yml`):
- Triggered on version tags (`v*`) or manual workflow dispatch
- Builds for linux/amd64 and linux/arm64
- Creates GitHub release with binaries and checksums
- Version info injected via LDFLAGS during build
