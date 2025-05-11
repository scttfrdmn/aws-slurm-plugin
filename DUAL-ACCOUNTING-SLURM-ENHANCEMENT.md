# Enhancing Slurm for Multi-Currency Accounting

## Overview

This document proposes an enhancement to Slurm's core accounting functionality to support multiple accounting currencies or domains simultaneously within a single Slurm cluster. This would enable organizations to track traditional resource usage (core-hours) for on-premises resources while simultaneously tracking dollar-based usage for cloud resources.

## Current Limitations

In the current Slurm architecture:

1. All tracked resources share a single billing dimension (TRESBillingWeights)
2. Account allocations and usage tracking use a single "currency"
3. The same resource type (e.g., CPU) can only have one billing weight across the cluster
4. Partitions can have different weights but still accumulate into the same accounting unit

## Proposed Enhancement: Multi-Domain Accounting

### Conceptual Model

We propose extending Slurm's accounting system to support multiple "accounting domains" that can coexist within a single Slurm cluster:

```
┌─────────────────────────────────────────────────────────┐
│                   Slurm Cluster                         │
│                                                         │
│  ┌──────────────────────┐    ┌──────────────────────┐   │
│  │   On-Prem Domain     │    │    Cloud Domain      │   │
│  │                      │    │                      │   │
│  │ TRESBillingWeights:  │    │ TRESBillingWeights:  │   │
│  │ CPU=1.0 (core-hour)  │    │ CPU=4.0 (dollars)    │   │
│  │ Mem=0.0 (ignored)    │    │ Mem=0.5G (dollars)   │   │
│  │ Node=0.0 (ignored)   │    │ Node=2.0 (dollars)   │   │
│  │                      │    │                      │   │
│  │ Partitions:          │    │ Partitions:          │   │
│  │ - compute            │    │ - aws-spot           │   │
│  │ - highmem            │    │ - aws-ondemand       │   │
│  └──────────────────────┘    └──────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   Account Structure                     │
│                                                         │
│  Engineering (Parent Account)                           │
│  │                                                      │
│  ├── On-Prem Domain:                                    │
│  │   - Allocation: 100,000 core-hours                   │
│  │   - Usage: 45,320 core-hours                         │
│  │                                                      │
│  └── Cloud Domain:                                      │
│      - Allocation: $5,000                               │
│      - Usage: $1,245                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Core Components to Modify

#### 1. Data Structures

Extend the following data structures to support domain-specific accounting:

##### 1.1 New Domain Definition Structure
```c
typedef struct acct_domain_info {
    char *    domain_name;      /* Name of the accounting domain */
    char *    description;      /* Human-readable description */
    uint32_t  domain_id;        /* Unique identifier */
    List      tres_weights;     /* List of TRES billing weights for this domain */
    List      partitions;       /* List of partitions in this domain */
} acct_domain_info_t;
```

##### 1.2 Extend Account Association Structure
```c
typedef struct slurmdb_assoc_rec {
    /* Existing fields... */
    
    /* New fields for multi-domain accounting */
    List acct_domains;         /* List of domain-specific usage/limits */
} slurmdb_assoc_rec_t;

typedef struct slurmdb_assoc_domain {
    uint32_t domain_id;        /* Reference to accounting domain */
    uint64_t grp_tres;         /* TRES limits for this domain */
    uint64_t grp_tres_mins;    /* TRES-minute limits for this domain */
    uint64_t grp_tres_run_mins; /* TRES-minute usage running in this domain */
    uint32_t shares_raw;       /* Shares allocated in this domain */
    double   usage_raw;        /* Raw usage in this domain */
    double   usage_norm;       /* Normalized usage in this domain */
    uint64_t *usage_tres_raw;  /* TRES usage in this domain */
} slurmdb_assoc_domain_t;
```

##### 1.3 Extend Partition Structure
```c
typedef struct part_record {
    /* Existing fields... */
    
    /* New field for domain association */
    uint32_t acct_domain_id;   /* Accounting domain for this partition */
} part_record_t;
```

##### 1.4 Extend Job Record for Domain Tracking
```c
typedef struct job_record {
    /* Existing fields... */
    
    /* New field for domain tracking */
    uint32_t acct_domain_id;   /* Accounting domain for this job */
} job_record_t;
```

#### 2. Database Schema Changes

Extend the Slurm database schema to support the new structures:

```sql
-- New table for accounting domains
CREATE TABLE acct_domain (
    id INT NOT NULL AUTO_INCREMENT,
    name VARCHAR(64) NOT NULL,
    description TEXT,
    PRIMARY KEY (id),
    UNIQUE KEY (name)
);

-- Table for domain-specific TRES weights
CREATE TABLE acct_domain_tres_weights (
    domain_id INT NOT NULL,
    tres_id INT NOT NULL,
    weight DOUBLE NOT NULL DEFAULT 0.0,
    PRIMARY KEY (domain_id, tres_id),
    CONSTRAINT domain_tres_fk1 FOREIGN KEY (domain_id) 
        REFERENCES acct_domain (id),
    CONSTRAINT domain_tres_fk2 FOREIGN KEY (tres_id) 
        REFERENCES tres_table (id)
);

-- Table for domain-partition assignments
CREATE TABLE acct_domain_partition (
    domain_id INT NOT NULL,
    partition_id INT NOT NULL,
    PRIMARY KEY (domain_id, partition_id),
    CONSTRAINT domain_part_fk1 FOREIGN KEY (domain_id) 
        REFERENCES acct_domain (id),
    CONSTRAINT domain_part_fk2 FOREIGN KEY (partition_id) 
        REFERENCES partitions (id)
);

-- Table for domain-specific association data
CREATE TABLE assoc_domain_usage (
    assoc_id INT NOT NULL,
    domain_id INT NOT NULL,
    shares_raw INT DEFAULT 1,
    usage_raw DOUBLE DEFAULT 0.0,
    usage_norm DOUBLE DEFAULT 0.0,
    PRIMARY KEY (assoc_id, domain_id),
    CONSTRAINT assoc_domain_fk1 FOREIGN KEY (assoc_id) 
        REFERENCES assoc_table (id) ON DELETE CASCADE,
    CONSTRAINT assoc_domain_fk2 FOREIGN KEY (domain_id) 
        REFERENCES acct_domain (id)
);

-- Table for domain-specific TRES limits
CREATE TABLE assoc_domain_tres_limits (
    assoc_id INT NOT NULL,
    domain_id INT NOT NULL,
    tres_id INT NOT NULL,
    grp_tres BIGINT DEFAULT NULL,
    grp_tres_mins BIGINT DEFAULT NULL,
    grp_tres_run_mins BIGINT DEFAULT NULL,
    PRIMARY KEY (assoc_id, domain_id, tres_id),
    CONSTRAINT assoc_domain_tres_fk1 FOREIGN KEY (assoc_id, domain_id) 
        REFERENCES assoc_domain_usage (assoc_id, domain_id) ON DELETE CASCADE,
    CONSTRAINT assoc_domain_tres_fk2 FOREIGN KEY (tres_id) 
        REFERENCES tres_table (id)
);
```

#### 3. API Changes

Extend Slurm's API to support management of accounting domains:

##### 3.1 Create/Modify Domains
```c
/* Create a new accounting domain */
extern int slurmdb_create_acct_domain(slurmdb_conn_t *db_conn,
                                     acct_domain_info_t *domain_info);

/* Modify an existing accounting domain */
extern int slurmdb_modify_acct_domain(slurmdb_conn_t *db_conn,
                                     slurmdb_update_object_t *update);

/* Remove an accounting domain */
extern int slurmdb_remove_acct_domain(slurmdb_conn_t *db_conn,
                                     char *domain_name);
```

##### 3.2 Manage Domain Associations
```c
/* Set domain-specific limits for an association */
extern int slurmdb_set_assoc_domain_limits(slurmdb_conn_t *db_conn,
                                          slurmdb_update_object_t *update);

/* Get domain-specific usage for an association */
extern List slurmdb_get_assoc_domain_usage(slurmdb_conn_t *db_conn,
                                          slurmdb_assoc_cond_t *assoc_cond,
                                          char *domain_name);
```

#### 4. Configuration Changes

Extend Slurm configuration to support accounting domains:

##### 4.1 slurm.conf additions
```
# Define accounting domains
AccountingDomains=onprem,cloud
```

##### 4.2 New acct_domains.conf file
```
# On-premises domain configuration
DomainName=onprem
Description=Traditional HPC resources
Partitions=compute,highmem,gpu
TRESBillingWeights="CPU=1.0,Mem=0.0,Node=0.0,GRES/gpu=0.5"

# Cloud domain configuration
DomainName=cloud
Description=AWS cloud resources
Partitions=aws-spot,aws-ondemand
TRESBillingWeights="CPU=4.0,Mem=0.5G,Node=2.0,GRES/gpu=8.0"
```

### Command-line Tool Extensions

#### 1. sacctmgr Extensions

Extend sacctmgr to support multi-domain operations:

```
# Create a new accounting domain
sacctmgr add domain cloud Description="AWS Cloud Resources"

# Set TRES weights for a domain
sacctmgr modify domain cloud set TRESBillingWeights="CPU=4.0,Mem=0.5G,Node=2.0"

# Assign partitions to a domain
sacctmgr modify domain cloud set Partitions=aws-spot,aws-ondemand

# Set domain-specific allocation for an account
sacctmgr modify account engineering set GrpTRESMins=cpu=100000 domain=onprem
sacctmgr modify account engineering set GrpTRESMins=cpu=5000 domain=cloud

# View domain-specific usage
sacctmgr show account engineering withusage format=domain,account,user,rawusage,rawshares
```

#### 2. sshare Extensions

Extend sshare to show domain-specific fairshare information:

```
# Show fairshare information for a specific domain
sshare -A engineering --domain=cloud

# Show fairshare across all domains
sshare -A engineering --domains
```

#### 3. sreport Extensions

Extend sreport to support domain-specific reporting:

```
# Generate usage report for a specific domain
sreport cluster AccountUtilizationByUser start=2023-01-01 end=2023-02-01 --domain=cloud

# Generate comparative report across domains
sreport cluster DomainComparison start=2023-01-01 end=2023-02-01
```

### Job Submission Integration

Modify job submission to optionally specify or automatically determine the accounting domain:

```bash
# Explicit domain specification
sbatch --domain=cloud myjob.sh

# Automatic domain selection based on partition
sbatch -p aws-spot myjob.sh  # Automatically uses 'cloud' domain
```

### Implementation Approach

#### Phase 1: Core Data Structures
- Add domain definition structures
- Extend database schema
- Implement basic domain management

#### Phase 2: Accounting Integration
- Implement domain-specific accounting calculations
- Extend job completion accounting to track by domain
- Implement domain-specific usage reporting

#### Phase 3: User Tools
- Extend command-line tools
- Add configuration file support
- Implement user-facing documentation

#### Phase 4: Policy & Scheduling
- Implement domain-specific fairshare calculations
- Add domain-specific job prioritization
- Implement domain-aware scheduling decisions

### Compatibility and Migration

1. **Backward Compatibility**:
   - Default "system" domain for existing configurations
   - Automatic mapping of existing accounting to default domain
   - Compatibility layer for existing API calls

2. **Migration Path**:
   - Database upgrade scripts for existing installations
   - Tool to assist in defining initial domains
   - Gradual adoption path allowing partial implementation

3. **Graceful Degradation**:
   - System continues to function if domain features aren't used
   - Clear error messages for domain-specific operations on systems without the feature

## Benefits and Use Cases

1. **Hybrid Infrastructure Management**:
   - Accurate accounting across on-premises and cloud resources
   - Different billing models for different resource types

2. **Multi-Currency Resource Allocation**:
   - Track traditional core-hour allocations for on-premises
   - Track dollar-based allocations for cloud
   - Support for organizational charge-back models

3. **Enhanced Reporting**:
   - Domain-specific usage reports
   - Cross-domain comparison
   - Resource utilization by domain

4. **Policy Enforcement**:
   - Domain-specific usage limits
   - Differentiated fairshare policies
   - Cost control for expensive resources

## Conclusion

This enhancement to Slurm's accounting system would address a critical need in hybrid HPC environments by supporting multiple accounting domains or currencies within a single Slurm cluster. By extending the core data structures and APIs, this approach provides a clean, integrated solution that maintains compatibility with existing Slurm functionality while adding powerful new capabilities.

The implementation would involve significant changes to Slurm's core accounting system but would provide substantial benefits for organizations managing mixed resource environments, particularly those combining traditional on-premises HPC with cloud resources.