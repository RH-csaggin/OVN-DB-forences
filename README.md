# OVN Database Forensics Tools

A collection of bash scripts for debugging and analyzing OVN (Open Virtual Network) databases from must-gather archives in OpenShift/OKD environments.

## Overview

These tools help network engineers and SREs troubleshoot OVN networking issues by mounting and analyzing OVN database snapshots locally using containers. They are particularly useful for post-mortem analysis and debugging network connectivity problems.

## Tools

### 1. `ovn-db-run-locally`
Runs a single OVN database (northbound or southbound) in a container for interactive debugging.

**Features:**
- Automatically converts clustered databases to standalone format
- Provides interactive shell access to query the database
- Works with both Docker and Podman

**Usage:**
```bash
./ovn-db-run-locally <raw_db_file> [docker|podman]

# Examples:
./ovn-db-run-locally ./must-gather/network_logs/leader_nbdb
./ovn-db-run-locally ./leader_sbdb podman
```

### 2. `ovn-dbs-run-locally`
Runs multiple OVN databases from a directory in separate containers and generates helper functions for batch operations.

**Features:**
- Processes all database files in a directory
- Generates interactive helper commands via a sourced environment file
- Supports filtering by database type (northbound/southbound)
- Enables parallel querying across multiple nodes

**Usage:**
```bash
./ovn-dbs-run-locally <db_directory> [n|s|all] [docker|podman]

# Start all databases
./ovn-dbs-run-locally ./network_logs/dbs all

# Source the generated helper file
source /tmp/net_tools_env

# View available containers
ntool_show

# Execute commands on specific databases
ntool_cmd_n ovn-nbctl show  # Run on all northbound DBs
ntool_cmd_s ovn-sbctl show  # Run on all southbound DBs
ntool_0                     # Shell into container 0

# Cleanup
ntool_clean
# or
./ovn-dbs-run-locally --clean
```

### 3. `ovn-flow-check`
Performs comprehensive flow analysis for a specific pod IP or pod name across OVN databases.

**Features:**
- Analyzes SNAT flows
- Checks EgressIP configurations
- Examines Service Load Balancer flows
- Traces node and cluster routing
- Validates external connectivity flows
- Reviews ACL and security policies
- Tests CoreDNS connectivity
- Supports pod name resolution
- Search mode for finding pods by partial name

**Usage:**
```bash
./ovn-flow-check <nbdb_file> <sbdb_file> <pod_ip_or_name> [docker|podman]

# Examples:
./ovn-flow-check ./leader_nbdb ./leader_sbdb 10.128.0.45
./ovn-flow-check ./leader_nbdb ./leader_sbdb openshift-ovn-kubernetes/ovnkube-node-abc123
./ovn-flow-check ./leader_nbdb ./leader_sbdb --search ovnkube-node
./ovn-flow-check ./leader_nbdb ./leader_sbdb --search my-app podman
```

**Analysis includes:**
- Node routing (LSP, port bindings, chassis)
- Cluster routing (router policies, static routes)
- SNAT flow configuration
- EgressIP NAT rules and reroute policies
- Service load balancer backends
- External gateway flows
- ACLs and port groups
- CoreDNS service configuration

### 4. `ovn-flow-check-all`
Processes all OVN databases from a directory one pair at a time and performs comprehensive flow checks.

**Features:**
- Resource-efficient processing (one NB+SB pair at a time)
- Suitable for large must-gather archives (100+ database files)
- Automatic cleanup between database pairs
- Verbose mode for debugging
- Supports both IP and pod name searches

**Usage:**
```bash
./ovn-flow-check-all [--verbose] <db_directory> <pod_ip_or_name> [docker|podman]

# Examples:
./ovn-flow-check-all ./network_logs/dbs 10.128.0.45
./ovn-flow-check-all ./network_logs/dbs openshift-ovn-kubernetes/ovnkube-node-abc123
./ovn-flow-check-all ./network_logs/dbs --search ovnkube-node
./ovn-flow-check-all --verbose ./network_logs/dbs --search my-app podman
```

## Prerequisites

- **Container Runtime**: Docker or Podman
- **jq**: JSON processor (required for `ovn-dbs-run-locally` and `ovn-flow-check-all`)
- **Bash**: Version 4.0 or later

### Installing jq

**macOS:**
```bash
brew install jq
```

**RHEL/CentOS/Fedora:**
```bash
sudo dnf install jq
```

**Ubuntu/Debian:**
```bash
sudo apt-get install jq
```

## Common Use Cases

### 1. Debug a pod connectivity issue
```bash
# Find the pod's IP
./ovn-flow-check-all ./dbs --search my-pod-name

# Analyze flows for that specific IP
./ovn-flow-check ./leader_nbdb ./leader_sbdb 10.128.0.45
```

### 2. Check SNAT configuration across all nodes
```bash
./ovn-dbs-run-locally ./network_logs/dbs all
source /tmp/net_tools_env
ntool_cmd_n ovn-nbctl find NAT type=snat
```

### 3. Verify EgressIP setup
```bash
./ovn-flow-check ./leader_nbdb ./leader_sbdb <pod-ip>
# Look for the "EGRESSIP FLOW ANALYSIS" section
```

### 4. Investigate CoreDNS issues
```bash
./ovn-flow-check ./leader_nbdb ./leader_sbdb <pod-ip>
# Review the "COREDNS CONNECTIVITY ANALYSIS" section
```

## Important Notes

- **Clustered Databases**: All scripts automatically convert clustered OVN databases to standalone format for local analysis. Database UUIDs are preserved during conversion.
- **Local Operation Only**: These are local debugging tools and cannot be used directly with must-gather without extracting the database files first.
- **Container Cleanup**: Always clean up containers when finished to free resources:
  - `ovn-dbs-run-locally --clean`
  - Or press `Ctrl+D` when in interactive shells

## Troubleshooting

### Database won't start
- Check that the database file is not corrupted
- Verify you have enough disk space
- Ensure the container engine is running

### "ovn-nbctl show" returns non-zero code
- Wait a few more seconds for the database to fully initialize
- Check container logs: `docker logs <container-name>`

### jq command not found
- Install jq using your package manager (see Prerequisites)

## Container Images

These scripts use the official OpenShift OVN-Kubernetes image:
```
quay.io/openshift/origin-ovn-kubernetes:latest
```

## License

These tools are provided as-is for debugging and troubleshooting purposes.

## Contributing

Feel free to submit issues or pull requests to improve these tools.
