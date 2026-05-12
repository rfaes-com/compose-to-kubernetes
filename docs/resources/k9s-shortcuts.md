# k9s Keyboard Shortcuts

Quick reference for k9s TUI (Terminal UI) keyboard shortcuts.

## Getting Started

```bash
# Start k9s
k9s

# Start in specific namespace
k9s -n development

# Start with specific view
k9s -c pods
k9s -c deployments
```

## Global Navigation

| Key             | Action                                    |
| --------------- | ----------------------------------------- |
| `?`             | Show help/keyboard shortcuts              |
| `:`             | Enter command mode                        |
| `/`             | Filter current view                       |
| `esc`           | Back/exit/clear filter                    |
| `ctrl-a`        | Show all available resources              |
| `ctrl-c` or `q` | Quit k9s                                  |
| `0-9`           | Switch to namespace (from namespace list) |
| `a`             | Show all namespaces                       |

## Resource Views

Type `:` followed by resource name to switch views:

| Command                             | Resource                 |
| ----------------------------------- | ------------------------ |
| `:pods` or `:po`                    | Pods                     |
| `:deployments` or `:deploy`         | Deployments              |
| `:services` or `:svc`               | Services                 |
| `:nodes` or `:no`                   | Nodes                    |
| `:namespaces` or `:ns`              | Namespaces               |
| `:configmaps` or `:cm`              | ConfigMaps               |
| `:secrets` or `:sec`                | Secrets                  |
| `:events` or `:ev`                  | Events                   |
| `:persistentvolumes` or `:pv`       | Persistent Volumes       |
| `:persistentvolumeclaims` or `:pvc` | Persistent Volume Claims |
| `:ingress` or `:ing`                | Ingresses                |
| `:replicasets` or `:rs`             | ReplicaSets              |
| `:statefulsets` or `:sts`           | StatefulSets             |
| `:daemonsets` or `:ds`              | DaemonSets               |
| `:jobs`                             | Jobs                     |
| `:cronjobs` or `:cj`                | CronJobs                 |

## Common Actions

| Key      | Action                       |
| -------- | ---------------------------- |
| `enter`  | View details/drill down      |
| `d`      | Describe selected resource   |
| `y`      | View YAML                    |
| `e`      | Edit resource                |
| `l`      | View logs                    |
| `s`      | Shell into container         |
| `ctrl-d` | Delete resource              |
| `ctrl-k` | Kill resource (force delete) |

## Pod Operations

In pod view:

| Key       | Action                 |
| --------- | ---------------------- |
| `enter`   | View containers in pod |
| `l`       | View logs              |
| `shift-f` | Port-forward           |
| `s`       | Shell into container   |
| `ctrl-d`  | Delete pod             |
| `ctrl-k`  | Kill pod               |
| `t`       | View pod's events      |
| `shift-r` | Sort by restart count  |
| `shift-s` | Sort by status         |
| `shift-c` | Sort by CPU            |
| `shift-m` | Sort by memory         |

## Container Operations

When inside a pod (after pressing `enter` on a pod):

| Key | Action                        |
| --- | ----------------------------- |
| `l` | View container logs           |
| `p` | View previous container logs  |
| `s` | Shell into container          |
| `c` | View container's CPU usage    |
| `m` | View container's memory usage |

## Log Viewing

In log view:

| Key      | Action                       |
| -------- | ---------------------------- |
| `0`      | Show fullscreen logs         |
| `1`      | Show 100 lines               |
| `2`      | Show 200 lines               |
| `3`      | Show 300 lines               |
| `4`      | Show 400 lines               |
| `5`      | Show 500 lines               |
| `f`      | Toggle log follow            |
| `s`      | Toggle autoscroll            |
| `w`      | Toggle wrap                  |
| `t`      | Toggle timestamps            |
| `/`      | Filter logs                  |
| `ctrl-s` | Save logs to file            |
| `c`      | Clear logs                   |
| `p`      | Show previous container logs |

## Filtering

| Key     | Action       |
| ------- | ------------ |
| `/`     | Start filter |
| `enter` | Apply filter |
| `esc`   | Clear filter |

Filter examples:

- `/nginx` - Show resources containing "nginx"
- `/Running` - Show only running pods
- `/-label app=web` - Filter by label

## Sorting

In any resource view:

| Key       | Action                       |
| --------- | ---------------------------- |
| `shift-a` | Sort alphabetically (A-Z)    |
| `shift-n` | Sort by name                 |
| `shift-t` | Sort by type                 |
| `shift-s` | Sort by status               |
| `shift-r` | Sort by restart count (pods) |
| `shift-c` | Sort by CPU                  |
| `shift-m` | Sort by memory               |

## Context and Namespace

| Key    | Action                                |
| ------ | ------------------------------------- |
| `:ctx` | View/switch contexts                  |
| `:ns`  | View/switch namespaces                |
| `0`    | Default namespace (in namespace view) |
| `a`    | All namespaces                        |

## Editing and Managing

| Key      | Action                              |
| -------- | ----------------------------------- |
| `e`      | Edit resource (opens in $EDITOR)    |
| `y`      | View YAML                           |
| `ctrl-d` | Delete resource (with confirmation) |
| `ctrl-k` | Kill resource (force delete)        |

## Special Commands

Type `:` followed by command:

| Command            | Action                        |
| ------------------ | ----------------------------- |
| `:q` or `:quit`    | Quit k9s                      |
| `:help`            | Show help                     |
| `:alias`           | Show aliases                  |
| `:ctx`             | List/switch contexts          |
| `:ns`              | List/switch namespaces        |
| `:xray <resource>` | Show tree view                |
| `:popeye`          | Run cluster linter            |
| `:pulse`           | Show resource pulse (updates) |

## XRay View

Show resource relationships:

```text
:xray deployments
:xray services
:xray pods
```

Navigate with arrow keys, `esc` to exit.

## Port Forwarding

1. Navigate to pod
2. Press `shift-f`
3. Select container port
4. Enter local port
5. Access via `localhost:<port>`

Stop with `ctrl-c`

## Benchmarking

Select a pod and press `ctrl-b` to run a benchmark.

## Resource Usage

| Key       | Action                     |
| --------- | -------------------------- |
| `:node`   | View nodes                 |
| `shift-c` | Sort by CPU                |
| `shift-m` | Sort by memory             |
| `enter`   | View node details and pods |

## Useful Tips

### Copy to Clipboard

Many views support copying:

- `y` to view YAML
- Terminal-specific copy commands (Ctrl+Shift+C on Linux)

### Watching Resources

k9s automatically refreshes views every 2 seconds. Resources are color-coded:

- Green: Running/Ready
- Yellow: Warning/Pending
- Red: Error/Failed

### Resource Indicators

In pod view:

- `!` - Warning/Error
- `*` - Restarting
- `↺` - Restart count

### Shortcuts File

Customize shortcuts in `$HOME/.config/k9s/hotkey.yml`

### Skins

Change appearance with skins in `$HOME/.config/k9s/skin.yml`

## Command Mode Examples

Type `:` to enter command mode:

```text
:pods                    # Show pods
:svc                     # Show services
:deploy                  # Show deployments
:ns production           # Switch to production namespace
:ctx kind-kind          # Switch to kind-kind context
:xray deployments        # Show deployment tree
:pulse                   # Show resource pulse
:popeye                  # Run cluster linter
```

## Filter Examples

Press `/` in any view:

```text
/nginx                   # Filter for "nginx"
/Running                 # Show only Running pods
/!Running               # Show non-Running pods
/Error                  # Show resources with errors
```

## Common Workflows

### Debugging a Pod

1. `:pods` - Show pods
2. `/my-app` - Filter for your app
3. `enter` - View containers
4. `l` - View logs
5. `s` - Shell into container

### Checking Deployment

1. `:deploy` - Show deployments
2. Select deployment
3. `d` - Describe
4. `enter` - View pods
5. Check status

### Viewing Resource Usage

1. `:nodes` - Show nodes
2. `shift-c` - Sort by CPU
3. `enter` - View pods on node
4. `shift-m` - Sort by memory

### Port Forward to Service

1. `:svc` - Show services
2. Select service
3. `shift-f` - Port forward
4. Access via localhost

### Quick Pod Restart

1. `:pods` - Show pods
2. Select pod
3. `ctrl-d` - Delete pod
4. Kubernetes recreates it

## Advanced Features

### Popeye Integration

Popeye scans cluster for potential issues:

```text
:popeye
```

Review findings:

- Over-allocated resources
- Resource issues
- Security concerns

### Pulse View

Monitor resource changes in real-time:

```text
:pulse
```

### XRay Tree View

Visualize resource relationships:

```text
:xray deployments
:xray services
```

Navigate tree with arrows.

## Configuration

k9s configuration files:

```text
$HOME/.config/k9s/
├── config.yml          # Main configuration
├── hotkey.yml          # Custom hotkeys
├── plugin.yml          # Custom plugins
├── skin.yml            # UI skin/theme
└── views.yml           # Column customization
```

### Example Custom Hotkey

`~/.config/k9s/hotkey.yml`:

```yaml
hotKeys:
  shift-0:
    shortCut:    Shift-0
    description: View pods in default namespace
    command:     pods --context kind-kind --namespace default
```

## Color Codes

Default status colors:

- **Green**: Healthy/Running
- **Yellow**: Warning/Pending/Terminating
- **Red**: Error/Failed/CrashLoopBackOff
- **Blue**: Completed/Succeeded

## Performance Tips

- Use filters (`/`) to reduce resource lists
- Switch to specific namespace instead of "all namespaces"
- Close log views when not needed
- Use `:pulse` sparingly on large clusters

## Getting Help

- `?` - Show context-sensitive shortcuts
- `:help` - General help
- `:alias` - View available aliases
- [k9s GitHub](https://github.com/derailed/k9s)
- [k9s Documentation](https://k9scli.io/)
