# Implementing Dual Accounting in Slurm Using Existing Capabilities

## Overview

This document outlines an implementation strategy for tracking two different types of service units simultaneously in Slurm (e.g., traditional core-hours for on-premises resources and dollar-based units for cloud resources) using existing Slurm capabilities without core modifications.

## Architecture

The proposed solution uses a combination of:
1. Slurm Federation
2. Cluster-specific accounting databases
3. Custom reporting infrastructure
4. Unified management interface

```
┌──────────────────────────┐     ┌──────────────────────────┐
│      On-Prem Cluster     │     │       Cloud Cluster      │
│                          │     │                          │
│ ┌──────────┐ ┌─────────┐ │     │ ┌──────────┐ ┌─────────┐ │
│ │Slurm Ctld│ │Slurm DB │ │     │ │Slurm Ctld│ │Slurm DB │ │
│ │(Master)  │ │(Primary)│ │     │ │(Peer)    │ │(Primary)│ │
│ └──────────┘ └─────────┘ │     │ └──────────┘ └─────────┘ │
│       │                  │     │       │                  │
│ ┌─────▼────┐             │     │ ┌─────▼────┐             │
│ │Compute   │             │     │ │Dynamic   │             │
│ │Nodes     │             │     │ │EC2 Nodes │             │
│ └──────────┘             │     │ └──────────┘             │
└──────────────────────────┘     └──────────────────────────┘
          │                                 │
          └─────────────┬─────────────────┘
                        │
               ┌────────▼─────────┐
               │ Integration Layer│
               │                  │
               │ ┌──────────────┐ │
               │ │Custom Reports│ │
               │ └──────────────┘ │
               │ ┌──────────────┐ │
               │ │Unified Admin │ │
               │ │Interface     │ │
               │ └──────────────┘ │
               └──────────────────┘
```

## Implementation Steps

### 1. Federation Setup

#### 1.1 Configure On-Premises Cluster

```
# slurm.conf for on-prem cluster
ClusterName=onprem
SlurmctldHost=onprem-master

# Federation configuration
FederationParameters=fed_display
Federation=hybrid_cluster

# Accounting configuration
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=onprem-db
AccountingStoreFlags=job_comment
```

#### 1.2 Configure Cloud Cluster

```
# slurm.conf for cloud cluster
ClusterName=cloud
SlurmctldHost=cloud-master

# Federation configuration 
FederationParameters=fed_display
Federation=hybrid_cluster

# Accounting configuration
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=cloud-db
AccountingStoreFlags=job_comment
```

#### 1.3 Configure Federation in slurmdbd.conf

For each cluster's slurmdbd.conf:

```
# slurmdbd.conf
DbdHost=localhost
StorageType=accounting_storage/mysql
StorageLoc=slurm_acct_db

# Federation authentication
FederationAuthType=auth/jwt
FederationAuthFile=/etc/slurm/federation.key
```

### 2. Implement Separate Accounting Structures

#### 2.1 On-Premises Accounting Configuration

Traditional core-hour accounting:

```bash
# Set up TRESBillingWeights for core-hours
sacctmgr modify cluster onprem set TRESBillingWeights="CPU=1.0,Mem=0.0,Node=0.0"

# Configure partitions with appropriate weights
sacctmgr modify partition onprem-partition set TRESBillingWeights="CPU=1.0,Mem=0.0"
```

#### 2.2 Cloud Accounting Configuration

Dollar-based accounting:

```bash
# Set up TRESBillingWeights based on dollar costs
sacctmgr modify cluster cloud set TRESBillingWeights="CPU=4.0,Mem=0.5G,Node=2.0"

# Configure specific instances with appropriate costs
sacctmgr modify partition cloud-c5-partition set TRESBillingWeights="CPU=4.0,Mem=0.5G"
sacctmgr modify partition cloud-m5-partition set TRESBillingWeights="CPU=3.0,Mem=0.6G"
```

#### 2.3 Account Structure

Create mirrored account structures on both clusters:

```bash
# Create parent account organization
sacctmgr add account CompanyName description="Parent Organization"

# Create department accounts under parent
sacctmgr add account Engineering parent=CompanyName
sacctmgr add account Research parent=CompanyName

# Create user associations with different limits per cluster
sacctmgr add user user1 account=Engineering cluster=onprem,cloud \
    MaxSubmitJobs=1000 MaxJobs=200 \
    GrpTRES=cpu=1000,mem=1000G \
    GrpTRESMins=cpu=40000  # Core-hours for on-prem

sacctmgr add user user1 account=Engineering cluster=cloud \
    GrpTRES=cpu=250,mem=250G \
    GrpTRESMins=cpu=10000  # Dollar-units for cloud
```

### 3. Implement Custom Reporting Infrastructure

#### 3.1 Create dual-accounting reporting script

```python
#!/usr/bin/env python3
# dual_accounting_report.py

import subprocess
import csv
import datetime
import argparse

def get_usage(cluster, account, start_time, end_time):
    cmd = [
        'sacct', '-n', '-X', 
        '-S', start_time, '-E', end_time,
        '-a', '-c', cluster, 
        '--accounts', account,
        '--format=Account,User,Partition,AllocTRES,CPUTime,ElapsedRaw'
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.stdout.strip()

def main():
    parser = argparse.ArgumentParser(description='Generate dual accounting report')
    parser.add_argument('--account', required=True, help='Account name')
    parser.add_argument('--start', required=True, help='Start time (YYYY-MM-DD)')
    parser.add_argument('--end', required=True, help='End time (YYYY-MM-DD)')
    args = parser.parse_args()
    
    # Get usage from both clusters
    onprem_usage = get_usage('onprem', args.account, args.start, args.end)
    cloud_usage = get_usage('cloud', args.account, args.start, args.end)
    
    # Process and format report
    print(f"Dual Accounting Report for {args.account}")
    print(f"Period: {args.start} to {args.end}")
    print("\nOn-Premises Usage (Core-Hours):")
    # Process and print on-prem usage
    
    print("\nCloud Usage (Dollar-Units):")
    # Process and print cloud usage
    
    print("\nTotal Resource Consumption:")
    # Print summary statistics
    
if __name__ == "__main__":
    main()
```

#### 3.2 Create allocation tracking and enforcement script

```python
#!/usr/bin/env python3
# track_allocations.py

import subprocess
import json
import datetime
import sys

def get_accounts_usage():
    # Get usage details from both clusters
    onprem_cmd = ['sshare', '-n', '-o', 'Account,User,RawShares,NormShares,RawUsage,NormUsage', '-M', 'onprem']
    cloud_cmd = ['sshare', '-n', '-o', 'Account,User,RawShares,NormShares,RawUsage,NormUsage', '-M', 'cloud']
    
    onprem_result = subprocess.run(onprem_cmd, capture_output=True, text=True)
    cloud_result = subprocess.run(cloud_cmd, capture_output=True, text=True)
    
    # Process results and return structured data
    # ...

def enforce_allocations():
    # Check usage against allocations and enforce limits
    # Modify account status if needed
    # ...

def main():
    # Get current usage
    usage = get_accounts_usage()
    
    # Enforce allocations
    enforce_allocations(usage)
    
    # Generate warnings for accounts approaching limits
    # ...

if __name__ == "__main__":
    main()
```

### 4. Create a Unified Management Interface

#### 4.1 Job submission wrapper

```bash
#!/bin/bash
# submit_job.sh - Unified job submission

# Determine target cluster based on resource needs
if [[ "$@" == *"--cloud"* ]]; then
    # Submit to cloud cluster
    CLUSTER="cloud"
    # Remove the --cloud flag
    ARGS=$(echo "$@" | sed 's/--cloud//')
else
    # Submit to on-prem cluster
    CLUSTER="onprem"
    ARGS="$@"
fi

# Check allocation status before submission
ACCOUNT=$(echo "$ARGS" | grep -o "\-A [a-zA-Z0-9]*" | awk '{print $2}')
if [ -z "$ACCOUNT" ]; then
    echo "Error: Account must be specified with -A flag"
    exit 1
fi

# Check if account has sufficient allocation
ALLOC_CHECK=$(check_allocation.py --account=$ACCOUNT --cluster=$CLUSTER)
if [ $? -ne 0 ]; then
    echo "Error: Insufficient allocation in $CLUSTER for account $ACCOUNT"
    echo "$ALLOC_CHECK"
    exit 1
fi

# Submit job to appropriate cluster
sbatch -M $CLUSTER $ARGS
```

#### 4.2 Administrative interface

Create a simple web dashboard or CLI tool that:
- Shows allocations and usage from both clusters
- Allows allocation adjustment
- Provides projection of resource consumption
- Alerts on approaching limits

## Maintenance and Operation

### Regular Account Synchronization

Run a daily synchronization script to ensure accounts are consistent across clusters:

```bash
#!/bin/bash
# sync_accounts.sh

# Export accounts from on-prem
sacctmgr -i dump cluster onprem account withassoc file=/tmp/onprem_accounts.cfg

# Import accounts to cloud cluster (skip existing)
sacctmgr -i load file=/tmp/onprem_accounts.cfg
```

### Usage Reports

Schedule weekly usage reports for all accounts:

```
0 0 * * 0 /path/to/dual_accounting_report.py --account=ALL --start=$(date -d "7 days ago" +\%Y-\%m-\%d) --end=$(date +\%Y-\%m-\%d) > /path/to/reports/weekly_$(date +\%Y\%m\%d).txt
```

## Limitations and Considerations

1. **Federation Complexity**: Setting up and maintaining a Slurm federation adds operational complexity.

2. **Manual Intervention**: Some aspects of dual accounting may require manual reconciliation.

3. **Account Synchronization**: Care must be taken to keep accounts synchronized across clusters.

4. **User Experience**: Users must be educated about the different accounting structures.

5. **Optimization Opportunities**: Periodically review and adjust billing weights to accurately reflect costs.

## Conclusion

This approach leverages existing Slurm capabilities to implement dual accounting without modifying Slurm's core. While it requires additional infrastructure and custom scripts, it provides a functional solution for organizations needing to track different types of service units for on-premises and cloud resources.

The federation approach also offers additional benefits beyond dual accounting, such as workload distribution and resource sharing across environments, making it a powerful solution for hybrid HPC deployments.