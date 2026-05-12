# VS Code Workspace Settings for Kubernetes Workshop

This directory contains the VS Code devcontainer configuration for the Kubernetes Workshop project.

## What's Included

The devcontainer provides a fully configured development environment with:

### Tools

- **kubectl** - Latest stable Kubernetes CLI
- **helm** - Latest Helm 3
- **kind** - Kubernetes in Docker
- **k9s** - Kubernetes TUI
- **flux** - GitOps toolkit CLI
- **marp-cli** - Markdown presentation tool
- **markdownlint** - Markdown linting
- **markdown-link-check** - Check for broken links
- **kubeval** - Kubernetes YAML validation
- **yq** - YAML processor
- **make** - Build automation
- **npm/node** - For Node.js based tools
- **git/gh** - Version control and GitHub CLI

### VS Code Extensions

- **Kubernetes Tools** - Kubernetes resource management
- **YAML Support** - YAML editing and validation
- **Helm IntelliSense** - Helm chart editing
- **Marp** - Markdown presentation preview and export
- **Markdown Tools** - Enhanced markdown editing, linting, and Mermaid preview
- **Docker** - Container management
- **GitLens** - Advanced Git integration
- **EditorConfig** - Consistent coding styles
- **Spell Checker** - Catch typos in documentation

## Getting Started

### Prerequisites

- Visual Studio Code with Remote-Containers extension
- Docker Desktop or Podman Desktop running

### Using the Devcontainer

1. **Open in Container**
   - Open this repository in VS Code
   - Press `F1` and select "Dev Containers: Reopen in Container" (For windows user, clone the repository already in a volume)
   - Or click the popup "Reopen in Container"

2. **Wait for Setup**
   - The container will build (first time takes 5-10 minutes)
   - Post-create scripts will run automatically
   - You'll see a welcome message when ready

3. **Start Working**

   ```bash
   # Validate workshop content
   make validate

   # Run tests
   make test

   # Preview slides
   marp part-1/slides.md --watch --server

   # Create a Kubernetes cluster
   kind create cluster --config setup/kind/simple.yaml
   ```

## Features

### Automatic Port Forwarding

The following ports are automatically forwarded:

- **8080** - HTTP services
- **8443** - HTTPS services
- **30000-30002** - NodePort services

### Bash Aliases

Convenient kubectl aliases are pre-configured:

```bash
k          # kubectl
kgp        # kubectl get pods
kgs        # kubectl get svc
kgd        # kubectl get deploy
kga        # kubectl get all
kdesc      # kubectl describe
klogs      # kubectl logs
```

### Marp Integration

Preview and export presentations:

```bash
# Live preview with hot reload
marp part-1/slides.md --watch --server

# Export to PDF
marp part-1/slides.md --pdf

# Export to HTML
marp part-1/slides.md --html

# Export all slides
marp part-1/slides.md part-2/slides.md --pdf
```

### Markdown Validation

```bash
# Lint all markdown files
markdownlint '**/*.md' --ignore node_modules

# Check for broken links
find . -name "*.md" -exec markdown-link-check {} \;

# Auto-fix markdown issues
markdownlint '**/*.md' --fix
```

### YAML Validation

```bash
# Validate Kubernetes manifests
kubectl apply --dry-run=client -f part-1/03-deployments/examples/

# Validate with kubeval
kubeval part-1/**/examples/*.yaml

# Use make targets
make validate
```

## Customization

### Adding Extensions

Edit `.devcontainer/devcontainer.json`:

```json
"customizations": {
  "vscode": {
    "extensions": [
      "your.extension-id"
    ]
  }
}
```

### Installing Additional Tools

Edit `.devcontainer/Dockerfile` or `.devcontainer/post-create.sh`

### Changing Settings

Modify settings in `.devcontainer/devcontainer.json` under:

```json
"customizations": {
  "vscode": {
    "settings": {
      // Your settings here
    }
  }
}
```

## Troubleshooting

### Container Won't Build

- Ensure Docker/Podman is running
- Try rebuilding without cache: `F1` → "Dev Containers: Rebuild Container"
- Check Docker/Podman has enough resources (8GB+ RAM recommended)

### Tools Not Found

- Reopen the container: `F1` → "Dev Containers: Rebuild Container"
- Check post-create script output in the terminal

### Permission Issues

- The container runs as user `vscode` (non-root)
- Use `sudo` for system-level operations if needed

### Kind Cluster Issues

- Ensure Docker-in-Docker is working: `docker ps`
- Try: `kind delete cluster && kind create cluster`

## Benefits

### For Contributors

- No local tool installation needed
- Consistent environment across team
- All dependencies pre-configured
- Immediate productivity

### For Workshop Development

- Test examples in isolated environment
- Preview slides while editing
- Validate YAML on save
- Lint markdown automatically

### For Workshop Participants

- Can use devcontainer for learning
- Same environment as workshop leaders
- All tools at correct versions

## Alternative: Codespaces

This devcontainer is also compatible with GitHub Codespaces:

1. Click "Code" button in GitHub
2. Select "Codespaces" tab
3. Click "Create codespace on main"

Everything will be set up automatically in the cloud!

## Resources

- [VS Code Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers)
- [Dev Container Specification](https://containers.dev/)
- [Marp Documentation](https://marp.app/)
- [kind Documentation](https://kind.sigs.k8s.io/)
