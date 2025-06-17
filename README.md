# Network Interface Subnet Compatibility Checker

A comprehensive bash script for analyzing network interface subnet compatibility across RHEL8 cluster nodes, designed specifically for high-performance computing environments with complex point-to-point network topologies.

## Overview

This tool helps validate network configurations in HPC clusters by:
- **Analyzing subnet compatibility** across cluster nodes
- **Validating point-to-point (/31) network links** 
- **Detecting cross-interface connections** (e.g., node1[100g4] ↔ node2[100g1])
- **Identifying configuration issues** before physical cabling
- **Supporting both local and cluster-wide analysis**

## Features

✅ **Point-to-Point Link Validation**: Properly handles /31 subnets for direct node connections  
✅ **Cross-Interface Matching**: Detects valid connections between different interface names  
✅ **Comprehensive Analysis**: Shows network topology and connection patterns  
✅ **IBM GPFS Integration**: Works seamlessly with `mmdsh` for cluster-wide execution  
✅ **Subnet Compatibility Analysis**: Validates network configurations before deployment  

## Installation

```bash
# Download the script
wget https://example.com/checkP2PNet
chmod +x checkP2PNet

# Or copy to your cluster management node
scp checkP2PNet user@cluster-mgmt:/path/to/scripts/
```

## Usage Modes

### 1. Local Interface Analysis

Analyze network interfaces on the current node:

```bash
# Basic local analysis
./checkP2PNet

# Verbose local analysis  
./checkP2PNet -v

# Show what would be analyzed (dry run)
./checkP2PNet --dry-run
```

**Output Example:**
```
kalray-mgmt:     backplane       UP         10.11.26.185/24
kalray-mgmt:     eno8303.248     UP         10.11.48.205/24
kalray-mgmt:     docker0         UP         172.17.0.1/16
[SUCCESS] ✓ No subnet conflicts found on local node
```

### 2. Cluster-Wide Analysis (Recommended)

The most powerful mode for HPC cluster validation:

#### Step 1: Collect Cluster Interface Data

```bash
# Use mmdsh to collect interface data from all nodes
mmdsh -N all -f1 '/path/to/checkP2PNet -v' > cluster_interfaces.txt

# Or collect from specific node groups
mmdsh -N compute -f1 '/path/to/checkP2PNet -v' > compute_interfaces.txt
mmdsh -N storage -f1 '/path/to/checkP2PNet -v' > storage_interfaces.txt
```

#### Step 2: Analyze Cluster Compatibility

```bash
# Analyze collected cluster data
./checkP2PNet -f cluster_interfaces.txt -v

# Quick analysis without verbose output
./checkP2PNet -f cluster_interfaces.txt
```

## Example Workflow

### Complete HPC Cluster Validation

```bash
# 1. Collect interface data from all cluster nodes
echo "Collecting cluster network interface data..."
mmdsh -N all -f1 '/mmfs1/.arcapix/anelson/checkP2PNet -v' > ifaces.txt

# 2. Analyze cluster-wide subnet compatibility
echo "Analyzing cluster network topology..."
./checkP2PNet -f ifaces.txt -v

echo "=== Network validation complete ==="
```

## Understanding the Output

### Shared Subnet Interfaces

For management and switched networks:
```
[ANALYSIS] Analyzing interface group: backplane
[INFO]   kalray-gw1: 10.11.26.180/24 (subnet: 10.11.26.0/24)
[INFO]   kalray-gw2: 10.11.26.181/24 (subnet: 10.11.26.0/24)
[INFO]   kalray-mgmt: 10.11.26.185/24 (subnet: 10.11.26.0/24)
[SUCCESS]   ✓ Interface backplane: All 6 nodes in same subnet (10.11.26.0/24)
```

### Point-to-Point (/31) Links

For high-performance direct connections:
```
[ANALYSIS] Analyzing interface group: 100g1
[ANALYSIS]   Point-to-point (/31) interface detected
[SUCCESS]     ✓ Valid P2P link: kalray-nvme1[100g1] ↔ kalray-gw1[100g4] (10.100.0.6/31)
[WARN]       ⚠ Cross-interface connection: 100g1 ↔ 100g4
[INFO]       kalray-nvme1: 10.100.0.7/31 ↔ kalray-gw1: 10.100.0.6/31
[SUCCESS]   ✓ Interface 100g1: All 1 P2P links are paired (1 cross-interface)
```

### Configuration Issues

When problems are detected:
```
[ANALYSIS] Analyzing interface group: 100g2
[ERROR]   ✗ Interface 100g2: Multiple incompatible subnets!
[ERROR]     Found 3 different subnets: 10.100.0.8/31 10.100.0.10/31 10.100.0.12/31
[WARN]     Subnet 10.100.0.8/31: kalray-nvme1
[WARN]     Subnet 10.100.0.10/31: kalray-nvme2  
[WARN]     Subnet 10.100.0.12/31: kalray-gw1
[ERROR]   ✗ 1 interface group(s) have subnet configuration issues
```

## Command Line Options

```bash
Usage: ./checkP2PNet [OPTIONS]

OPTIONS:
    -h, --help                Show this help message
    -v, --verbose             Enable verbose output and detailed logging
    -f, --cluster-file FILE   File containing cluster interface info for analysis
    --dry-run                 Show what would be analyzed without running tests

EXAMPLES:
    # Local interface analysis
    ./checkP2PNet -v
    
    # Collect cluster data via mmdsh
    mmdsh -N all -f1 './checkP2PNet -v' > cluster_nets.txt
    
    # Analyze cluster topology  
    ./checkP2PNet -f cluster_nets.txt -v
    
    # Test actual connectivity (after physical setup)
    ./checkP2PNet -m ping-all -v
```

## Network Topology Validation

### Point-to-Point Networks (/31)

The script properly understands /31 point-to-point links as defined in **RFC 3021**:
- Each /31 subnet should connect exactly **2 nodes**
- Cross-interface connections are **valid** (e.g., `node1[100g1] ↔ node2[100g4]`)
- Warns about interface name mismatches but recognizes valid topology

### Shared Networks (/24, /27, etc.)

For management and switched networks:
- All nodes on the same interface should be in the **same subnet**
- Detects IP conflicts and subnet misconfigurations
- Validates VLAN and management network consistency

## File Formats

### Cluster Interface File Format

The script expects mmdsh output in this format:
```
node1.domain: node1:          interface1      UP         10.100.0.4/31
node1.domain: node1:          interface2      UP         10.11.26.180/24
node2.domain: node2:          interface1      UP         10.100.0.5/31
```

### Generated by mmdsh

```bash
# This command generates the correct format:
mmdsh -N all -f1 '/path/to/checkP2PNet -v' > interfaces.txt
```

## Troubleshooting

### Common Issues

**"No interface data found"**
- Check that the cluster file was generated correctly with mmdsh
- Verify the script has execute permissions on all nodes
- Ensure NetworkManager is running on cluster nodes

**"Cross-interface connections detected"**
- This is often **normal** in HPC environments
- Review your network topology design
- Cross-interface P2P links are valid for performance optimization

**"Unpaired /31 nodes"**
- Check physical cabling connections
- Verify IP address assignments match your topology design
- Look for typos in network configuration files

### Debug Mode

Enable maximum verbosity for troubleshooting:
```bash
./checkP2PNet -f cluster_file.txt -v -l /tmp/debug.log
cat /tmp/debug.log  # Review detailed logs
```

## Integration with HPC Workflows

### Pre-Deployment Validation

```bash
#!/bin/bash
# pre_deploy_network_check.sh

echo "=== HPC Cluster Network Pre-Deployment Validation ==="

# 1. Collect all interface configurations
echo "Step 1: Collecting interface data..."
mmdsh -N all -f1 '/shared/scripts/checkP2PNet -v' > /tmp/cluster_interfaces.txt

# 2. Validate network topology
echo "Step 2: Validating network topology..."
if /shared/scripts/checkP2PNet -f /tmp/cluster_interfaces.txt -v; then
    echo "✅ Network topology validation PASSED"
else
    echo "❌ Network topology validation FAILED"
    exit 1
fi

# 3. Archive results
echo "Step 3: Archiving results..."
cp /tmp/cluster_interfaces.txt "/shared/logs/network_validation_$(date +%Y%m%d_%H%M%S).txt"

echo "=== Network validation complete ==="
```

## Performance Considerations

- **Large clusters**: Analysis scales linearly with cluster size
- **Network complexity**: More interfaces = longer analysis time  
- **Log file size**: Use log rotation for large deployments
- **Memory usage**: ~1MB per 1000 network interfaces analyzed

## Requirements

- **RHEL 8** or compatible Linux distribution
- **NetworkManager** for interface management
- **IBM GPFS/mmdsh** for cluster-wide execution (optional)
- **Bash 4.0+** with associative arrays

## License

This script is provided under the MIT License. See LICENSE file for details.

## Contributing

Contributions welcome! Please:
1. Test on your HPC environment
2. Submit pull requests with test cases
3. Report issues with cluster configuration details
4. Suggest enhancements for specific HPC scenarios

## Support

For technical support:
- Review the troubleshooting section above
- Check logs in `/tmp/network_connectivity_*.log`
- Provide cluster configuration details when reporting issues
- Include mmdsh output samples for debugging

---

**Note**: This tool is designed for HPC cluster administrators familiar with network topology design and IBM GPFS cluster management.