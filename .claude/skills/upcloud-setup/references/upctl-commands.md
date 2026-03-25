# upctl CLI Quick Reference

Complete command reference for the UpCloud CLI tool.

## Authentication

```bash
upctl account login --with-token    # Interactive token login (saved to keyring)
export UPCLOUD_TOKEN="your-token"   # Environment variable
```

Config file: `~/.config/upctl.yaml`
```yaml
token: "your-token"
output: "human"                     # human, json, yaml
```

## Global Flags

| Flag | Short | Description |
| ---- | ----- | ----------- |
| `--output` | `-o` | Output format: `human`, `json`, `yaml` |
| `--config` | | Config file path |
| `--debug` | | Verbose debug output |
| `--client-timeout` | `-t` | API timeout (e.g., `30s`) |

## Server (`upctl server` / `upctl srv`)

```bash
upctl server list [--show-ip-addresses all]
upctl server plans
upctl server show <uuid|hostname>
upctl server create --hostname NAME --zone ZONE --plan PLAN --os OS [--ssh-keys PATH] [--enable-firewall] [--wait]
upctl server modify <uuid|hostname> [--plan PLAN] [--hostname NAME] [--cores N] [--memory MiB]
upctl server start <uuid|hostname>
upctl server stop <uuid|hostname> [--type soft|hard]
upctl server restart <uuid|hostname> [--stop-type soft|hard]
upctl server delete <uuid|hostname> [--delete-storages] [--stop]
```

### Server Network Interface
```bash
upctl server network-interface create <server> --network UUID --type private --family IPv4
upctl server network-interface modify <server> --index N
upctl server network-interface delete <server> --index N
```

### Server Firewall
```bash
upctl server firewall show <server>
upctl server firewall create <server> --direction in|out --action accept|drop --protocol tcp|udp [--destination-port-start N --destination-port-end N]
upctl server firewall delete <server> --position N
```

## Storage (`upctl storage` / `upctl st`)

```bash
upctl storage list [--private] [--public] [--all]
upctl storage show <uuid|title>
upctl storage create --title TITLE --zone ZONE [--size GiB] [--tier maxiops|standard|hdd] [--encrypt]
upctl storage modify <uuid|title> [--title TITLE] [--size GiB]
upctl storage clone <uuid|title> --title TITLE --zone ZONE
upctl storage delete <uuid|title> [--backups keep|keep_latest|delete]
upctl storage import --source-location PATH|URL [--storage UUID] [--title TITLE --zone ZONE]
```

### Storage Backups
```bash
upctl storage backup create <uuid|title>
upctl storage backup restore <uuid|title>
```

## Database (`upctl database` / `upctl db`)

```bash
upctl database list
upctl database show <uuid|title>
upctl database types
upctl database plans [--type pg|mysql|opensearch|valkey]
upctl database create --title TITLE --zone ZONE --hostname-prefix PREFIX --type TYPE [--plan PLAN] [--wait]
upctl database start <uuid|title>
upctl database stop <uuid|title>
upctl database delete <uuid|title>
```

### Database Properties
```bash
upctl database properties pg show       # PostgreSQL properties
upctl database properties mysql show    # MySQL properties
```

### Database Sessions
```bash
upctl database session list <uuid>
upctl database session cancel <uuid> --pid PID
```

## Network (`upctl network`)

```bash
upctl network list
upctl network show <uuid|name>
upctl network create --name NAME --zone ZONE --ip-network "address=10.0.1.0/24,dhcp=true,family=IPv4"
upctl network modify <uuid|name> [--name NAME]
upctl network delete <uuid|name>
```

## Router (`upctl router` / `upctl rt`)

```bash
upctl router list
upctl router show <uuid|name>
upctl router create --name NAME
upctl router modify <uuid|name> [--name NAME]
upctl router delete <uuid|name>
```

## IP Address (`upctl ip-address`)

```bash
upctl ip-address list
upctl ip-address show <address>
upctl ip-address assign --server UUID --family IPv4|IPv6
upctl ip-address modify <address> [--ptr-record FQDN]
upctl ip-address remove <address>
```

## Kubernetes (`upctl kubernetes` / `upctl k8s`)

```bash
upctl kubernetes list
upctl kubernetes show <uuid|name>
upctl kubernetes versions
upctl kubernetes plans
upctl kubernetes create --name NAME --network UUID --zone ZONE [--version VER] [--node-group "count=2,name=grp,plan=2xCPU-4GB"]
upctl kubernetes config <uuid|name>     # Get kubeconfig
upctl kubernetes modify <uuid|name>
upctl kubernetes delete <uuid|name>
```

### Node Groups
```bash
upctl kubernetes nodegroup create <cluster> --name NAME --count N --plan PLAN
upctl kubernetes nodegroup scale <cluster> --name NAME --count N
upctl kubernetes nodegroup show <cluster> --name NAME
upctl kubernetes nodegroup delete <cluster> --name NAME
```

## Object Storage (`upctl object-storage` / `upctl obs`)

```bash
upctl object-storage list
upctl object-storage show <uuid|name>
upctl object-storage regions
upctl object-storage create --name NAME --region REGION --network "type=public,name=public,family=IPv4" [--wait]
upctl object-storage delete <uuid|name>
```

### Users & Access Keys
```bash
upctl object-storage user create --service-uuid UUID --username NAME
upctl object-storage user list --service-uuid UUID
upctl object-storage user delete --service-uuid UUID --username NAME
upctl object-storage access-key create --service-uuid UUID --username NAME
upctl object-storage access-key list --service-uuid UUID
upctl object-storage access-key delete --service-uuid UUID --access-key-id ID
```

### Buckets
```bash
upctl object-storage bucket create --service-uuid UUID --name BUCKET
upctl object-storage bucket list --service-uuid UUID
upctl object-storage bucket delete --service-uuid UUID --name BUCKET
```

## Load Balancer (`upctl loadbalancer` / `upctl lb`)

```bash
upctl loadbalancer list
upctl loadbalancer show <uuid>
upctl loadbalancer plans
upctl loadbalancer delete <uuid>
```

## Account & Zones

```bash
upctl account show                 # Account details and balance
upctl account list                 # Sub-accounts
upctl zone list                    # Available zones
```

## Other

```bash
upctl server-group list|create|show|modify|delete
upctl version                      # CLI version
upctl completion bash|zsh|fish     # Shell completions
```

## Common Patterns

- **Multiple targets**: Most commands accept multiple UUIDs/names: `upctl server stop srv1 srv2`
- **Wait for ready**: `--wait` blocks until resource reaches target state
- **JSON output**: `-o json` for scripting, pipe to `jq`
- **Labels**: `--label key=value` (repeatable) on create/modify
- **Toggle features**: `--enable-X` / `--disable-X` pattern
