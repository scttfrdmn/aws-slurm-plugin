# AWS Plugin for Slurm - Version 2

A plugin that enables Slurm to dynamically create and terminate AWS EC2 instances based on workload demands.

> This project is forked from the [AWS Plugin for Slurm](https://github.com/aws-samples/aws-plugin-for-slurm/tree/plugin-v2) repository maintained by AWS Samples. This version includes enhanced documentation for on-premises to cloud bursting and dual accounting solutions.

## Table of Contents

- [Introduction](#introduction)
- [Quick Start Guide](#quick-start)
- [Architecture & Concepts](#architecture)
- [Installation Guide](#installation)
- [Configuration Reference](#configuration)
- [Usage Guide](#usage)
- [Advanced Configuration](#advanced)
- [Troubleshooting](#troubleshooting)
- [Additional Resources](#additional-resources)

<a name="introduction"/>

## Introduction

The AWS Plugin for Slurm enables a Slurm headnode to dynamically deploy and terminate compute resources in the cloud, regardless of where the headnode is executed. 

This plugin (Version 2) is a complete redevelopment of the initial release with major improvements:
- Support for EC2 Fleet capabilities including Spot instances and instance type diversification
- Decoupling of node names from instance hostnames or IP addresses
- Improved error handling when nodes fail to respond during launch

### Key Use Cases

- **Cloud Bursting**: Dynamically allocate additional resources from AWS when your on-premises cluster has insufficient capacity
- **Specialized Computing**: Access specialized instance types (GPUs, high-memory, etc.) for specific workloads
- **Cost Optimization**: Use Spot instances for significant cost savings
- **Self-contained Cloud HPC**: Deploy a complete, elastic HPC cluster in the cloud

<a name="quick-start"/>

## Quick Start Guide

The fastest way to get started is using the included CloudFormation template.

### Deploying with CloudFormation

1. Download the [`template.yaml`](template.yaml) file
2. Open the AWS CloudFormation console
3. Create a new stack using the template
4. Specify an existing VPC and two subnets in different availability zones
5. Complete the stack creation

The stack will create:
- A security group for Slurm traffic
- Required IAM roles
- Launch templates for compute nodes
- A fully-configured Slurm headnode

### Testing Your Deployment

1. Connect to the headnode via SSH
2. Submit a test job to the `aws` partition:
   ```bash
   srun -p aws hostname
   ```
3. Verify in the EC2 console that a new instance is launched
4. After completion, the instance will remain idle for the configured duration and then terminate

<a name="architecture"/>

## Architecture & Concepts

### How It Works

This plugin leverages Slurm's power save logic, which enables dynamic resource management:

1. Nodes are defined in the Slurm configuration but initially in power saving mode
2. When jobs are submitted, Slurm executes the `ResumeProgram` (provided by this plugin)
3. The plugin launches EC2 instances via EC2 Fleet and updates the node information in Slurm
4. When nodes are idle for a specified period, Slurm executes the `SuspendProgram` (provided by this plugin)
5. The plugin terminates the associated EC2 instances, returning nodes to power saving mode

This architecture enables fully elastic computing where resources exist only when needed.

### Key Components

- **resume.py**: Launches EC2 instances and configures Slurm nodes
- **suspend.py**: Terminates EC2 instances
- **change_state.py**: Handles nodes in transitional states
- **config.json**: Plugin and Slurm configuration
- **partitions.json**: Definition of node groups and EC2 instance configurations

<a name="installation"/>

## Installation Guide

### Prerequisites

Before installation, ensure you have:

- A functional Slurm headnode (tested with Slurm 20.02.3 or later)
- Subnets where EC2 compute nodes will be launched
- Connectivity between the headnode and compute node subnets
- Python 3 and boto3 installed on the headnode

**Important**: Compute nodes must be configured to specify their cluster name when launching `slurmd`. This can be accomplished by retrieving the node name from EC2 instance tags.

### AWS Resources Setup

#### 1. Configure AWS Authentication

If the headnode is on AWS:
- Create an IAM role for EC2 with the required permissions (see below)
- Attach the role to the headnode instance

If the headnode is NOT on AWS:
- Create an IAM user with the required permissions
- Configure AWS credentials on the headnode using the AWS CLI

#### 2. Required Permissions

The headnode needs these minimum permissions:
```
ec2:CreateFleet
ec2:RunInstances
ec2:TerminateInstances
ec2:CreateTags
ec2:DescribeInstances
iam:CreateServiceLinkedRole
iam:PassRole
```

#### 3. Create IAM Role for Compute Nodes

Create an IAM role that allows:
- `ec2:DescribeTags` permission
- Any other permissions required by your workloads

#### 4. Create Launch Templates

Create EC2 launch templates that specify:
- AMI ID
- Security group(s)
- EC2 IAM role
- Optional: key pair and bootstrap scripts (UserData)

### Manual Deployment Steps

1. **Install Dependencies**:
   ```bash
   sudo yum install python3 python3-pip -y
   sudo pip3 install boto3
   sudo pip3 install awscli
   ```

2. **Copy Plugin Files**:
   ```bash
   cd /path/to/slurm/etc/aws
   wget -q https://github.com/scttfrdmn/aws-slurm-plugin/raw/main/common.py
   wget -q https://github.com/scttfrdmn/aws-slurm-plugin/raw/main/resume.py
   wget -q https://github.com/scttfrdmn/aws-slurm-plugin/raw/main/suspend.py
   wget -q https://github.com/scttfrdmn/aws-slurm-plugin/raw/main/generate_conf.py
   wget -q https://github.com/scttfrdmn/aws-slurm-plugin/raw/main/change_state.py
   chmod +x *.py
   ```

3. **Create Configuration Files**:
   - Create `config.json` and `partitions.json` in the same directory
   - See the [Configuration Reference](#configuration) section for details

4. **Generate Slurm Configuration**:
   ```bash
   ./generate_conf.py
   ```
   - Append the content of the generated `slurm.conf.aws` to your `slurm.conf`
   - Reload the configuration with `scontrol reconfigure` or restart Slurmctld

5. **Configure cron**:
   ```bash
   sudo crontab -e
   ```
   Add: `* * * * * /path/to/slurm/etc/aws/change_state.py &>/dev/null`

6. **Configure Node Name Resolution on Compute Nodes**:

   Create a script to retrieve the node name from instance tags:
   ```bash
   cat > /path/to/get_nodename <<'EOF'
   instanceid=`/usr/bin/curl --fail -m 2 -s 169.254.169.254/latest/meta-data/instance-id`
   if [[ ! -z "$instanceid" ]]; then
      hostname=`/usr/bin/curl -s http://169.254.169.254/latest/meta-data/tags/instance/Name`
   fi
   if [ ! -z "$hostname" -a "$hostname" != "None" ]; then
      echo $hostname
   else
      echo `hostname`
   fi
   EOF
   chmod +x /path/to/get_nodename
   ```

   Modify the slurmd service configuration:
   ```
   ExecStartPre=/bin/bash -c "/bin/systemctl set-environment SLURM_NODENAME=$(/path/to/get_nodename)"
   ExecStart=/path/to/slurm/sbin/slurmd -N $SLURM_NODENAME $SLURMD_OPTIONS
   ```

<a name="configuration"/>

## Configuration Reference

### File: config.json

This file contains plugin and Slurm configuration parameters:

```json
{
   "LogLevel": "INFO",
   "LogFileName": "/var/log/slurm/aws.log",
   "SlurmBinPath": "/slurm/bin",
   "SlurmConf": {
      "PrivateData": "CLOUD",
      "ResumeProgram": "/slurm/etc/aws/resume.py",
      "SuspendProgram": "/slurm/etc/aws/suspend.py",
      "ResumeRate": 100,
      "SuspendRate": 100,
      "ResumeTimeout": 300,
      "SuspendTime": 350,
      "TreeWidth": 60000
   }
}
```

| Parameter | Description | Default/Example |
|-----------|-------------|-----------------|
| LogLevel | Logging level | "DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL" |
| LogFileName | Path to log file | "/var/log/slurm/aws.log" |
| SlurmBinPath | Path to Slurm binaries | "/slurm/bin" |
| SlurmConf.PrivateData | Must be "CLOUD" to show power-saved nodes | "CLOUD" |
| SlurmConf.ResumeProgram | Path to resume.py | "/slurm/etc/aws/resume.py" |
| SlurmConf.SuspendProgram | Path to suspend.py | "/slurm/etc/aws/suspend.py" |
| SlurmConf.ResumeRate | Max EC2 launches per minute | 100 |
| SlurmConf.SuspendRate | Max EC2 terminations per minute | 100 |
| SlurmConf.ResumeTimeout | Max seconds to wait for node availability | 300 |
| SlurmConf.SuspendTime | Seconds of idle time before powersave | 350 |
| SlurmConf.TreeWidth | Slurm communication width | 60000 |

### File: partitions.json

This file defines the partitions and node groups:

```json
{
   "Partitions": [
      {
         "PartitionName": "aws",
         "NodeGroups": [
            {
               "NodeGroupName": "node",
               "MaxNodes": 10,
               "Region": "us-east-1",
               "SlurmSpecifications": {
                  "CPUs": "4"
               },
               "PurchasingOption": "on-demand",
               "OnDemandOptions": {
                   "AllocationStrategy": "lowest-price"
               },
               "LaunchTemplateSpecification": {
                  "LaunchTemplateName": "template-name",
                  "Version": "$Latest"
               },
               "LaunchTemplateOverrides": [
                  {
                     "InstanceType": "c5.xlarge"
                  }
               ],
               "SubnetIds": [
                  "subnet-11111111",
                  "subnet-22222222"
               ],
               "Tags": [
                  {
                     "Key": "NodeGroup",
                     "Value": "aws-nodes"
                  }
               ]
            }
         ],
         "PartitionOptions": {
            "Default": "YES"
         }
      }
   ]
}
```

Key components:

- **Partitions**: List of Slurm partitions
  - **PartitionName**: Name of the partition
  - **NodeGroups**: List of node groups in this partition
    - **NodeGroupName**: Name of the node group
    - **MaxNodes**: Maximum number of nodes in this group
    - **Region**: AWS region for instances
    - **SlurmSpecifications**: Slurm node configuration
    - **PurchasingOption**: "spot" or "on-demand"
    - **OnDemandOptions/SpotOptions**: EC2 Fleet options
    - **LaunchTemplateSpecification**: EC2 launch template
    - **LaunchTemplateOverrides**: Instance types
    - **SubnetIds**: Subnets for instances
    - **Tags**: EC2 instance tags
  - **PartitionOptions**: Additional Slurm partition options

<a name="usage"/>

## Usage Guide

### Basic Commands

To submit jobs to AWS nodes:

```bash
# Run a command on AWS nodes
srun -p aws hostname

# Submit a batch job
sbatch -p aws myjob.sh

# Request specific resources
srun -p aws -N 2 -n 8 myparalleljob
```

### Monitoring

To check the status of AWS nodes:

```bash
# View all nodes in AWS partition
sinfo -p aws

# View details of a specific node
scontrol show node aws-node-0
```

Nodes will show status:
- `idle~` - Powered down, available to be started
- `allocated` - Running and executing jobs
- `idle` - Running but not executing jobs (will be powered down after SuspendTime)
- `down` - Failed to start properly

### Manual Testing

You can manually test the resume and suspend functionality:

```bash
# Manually resume a node
/path/to/resume.py aws-node-0

# Manually suspend a node
/path/to/suspend.py aws-node-0
```

<a name="advanced"/>

## Advanced Configuration

### Using Spot Instances

Modify your `partitions.json` to use Spot instances:

```json
"PurchasingOption": "spot",
"SpotOptions": {
   "AllocationStrategy": "lowest-price",
   "MaxTotalPrice": "1.0"
}
```

### Mixed On-Demand and Spot Configuration

Create two node groups with different weights:

```json
{
   "Partitions": [
      {
         "PartitionName": "aws",
         "NodeGroups": [
            {
               "NodeGroupName": "ondemand",
               "MaxNodes": 10,
               "SlurmSpecifications": {
                  "Weight": "1"
               },
               "PurchasingOption": "on-demand",
               ...
            },
            {
               "NodeGroupName": "spot",
               "MaxNodes": 100,
               "SlurmSpecifications": {
                  "Weight": "2"
               },
               "PurchasingOption": "spot",
               ...
            }
         ]
      }
   ]
}
```

The lower weight (`1`) for on-demand means they will be used first.

### Multiple Availability Zones

This configuration creates nodes in specific AZs that can be requested with features:

```json
"NodeGroups": [
   {
      "NodeGroupName": "spotAza",
      "MaxNodes": 100,
      "SlurmSpecifications": {
         "Features": "us-east-1a"
      },
      "SubnetIds": ["subnet-in-us-east-1a"]
   },
   {
      "NodeGroupName": "spotAzb",
      "MaxNodes": 100,
      "SlurmSpecifications": {
         "Features": "us-east-1b"
      },
      "SubnetIds": ["subnet-in-us-east-1b"]
   }
]
```

Users can request a specific AZ:
```bash
srun -p aws --constraint=us-east-1a myjob
```

### Multiple Instance Types

Include multiple instance types in descending order of preference:

```json
"LaunchTemplateOverrides": [
   {
      "InstanceType": "c5.xlarge"
   },
   {
      "InstanceType": "c4.xlarge"
   },
   {
      "InstanceType": "m5.xlarge"
   }
]
```

<a name="troubleshooting"/>

## Troubleshooting

### Common Issues

**Nodes fail to start**:
- Check AWS permissions
- Verify subnet configuration
- Examine AWS CloudTrail logs for API errors
- Check plugin logs for CreateFleet errors

**Nodes start but jobs don't run**:
- Verify connectivity between head node and compute nodes
- Check Slurm configuration is properly updated
- Ensure compute nodes can retrieve their correct node name

**Nodes don't terminate**:
- Verify the cron job for `change_state.py` is running
- Check SuspendTime configuration
- Examine plugin logs for termination API calls

### Logging

Logs are stored at the location specified in `config.json` (LogFileName).

Increase verbosity by setting LogLevel to "DEBUG" for troubleshooting.

### Debugging Steps

1. View plugin logs: `tail -f /var/log/slurm/aws.log`
2. Check Slurm logs: `tail -f /var/log/slurmd.log`
3. Verify AWS API calls are working: `aws ec2 describe-instances --region us-east-1`
4. Test node name resolution script on compute nodes
5. Manually run resume/suspend to test functionality

For more assistance, consult the Slurm documentation and AWS EC2 troubleshooting resources.

<a name="additional-resources"/>

## Additional Resources

This repository includes additional documentation on advanced topics:

- [On-Premises to Cloud Bursting Considerations](ON-PREM-TO-CLOUD-CONSIDERATIONS.md) - Detailed analysis of considerations for true on-premises to cloud bursting
- [On-Premises to Cloud Bursting Tutorial Outline](ON-PREM-TO-CLOUD-TUTORIAL-OUTLINE.md) - Step-by-step tutorial outline for implementing on-prem to AWS bursting
- [Dual Accounting with Existing Slurm](DUAL-ACCOUNTING-EXISTING-SLURM.md) - Implementation guide for tracking separate accounting units using existing Slurm capabilities
- [Dual Accounting Slurm Enhancement Proposal](DUAL-ACCOUNTING-SLURM-ENHANCEMENT.md) - Technical proposal for enhancing Slurm to support multiple accounting domains

These resources provide more in-depth guidance for implementing advanced configurations beyond the basic AWS-to-AWS setup.