# On-Premises to AWS Cloud Bursting Tutorial

## Tutorial Overview

This tutorial provides a practical, step-by-step guide for implementing a realistic on-premises to AWS cloud bursting solution using Slurm and the AWS Plugin for Slurm. Unlike the simplified cloud-to-cloud scenario in the main documentation, this tutorial addresses the real-world challenges of extending an existing on-premises Slurm cluster to AWS.

## Prerequisites

- Existing on-premises Slurm cluster (v20.02.3 or newer)
- AWS account with appropriate permissions
- Basic familiarity with AWS services (EC2, VPC, IAM)
- Admin access to on-premises infrastructure (network, DNS, storage)

## Architecture Overview

This tutorial implements the following architecture:

```
┌────────────────────────────┐              ┌─────────────────────────────┐
│     On-Premises Cluster    │              │         AWS Cloud           │
│                            │              │                             │
│ ┌──────────┐   ┌──────────┐│   WireGuard  │ ┌─────────┐    ┌─────────┐  │
│ │  Slurm   │   │On-Prem   ││   VPN Tunnel │ │ Compute │    │ Compute │  │
│ │Controller├───┤ Compute  ├┼─────────────►┼─┤ Node 1  ├────┤ Node N  │  │
│ │          │   │  Nodes   ││              │ │         │    │         │  │
│ └────┬─────┘   └──────────┘│              │ └─────────┘    └─────────┘  │
│      │                     │              │        │            │       │
│      │      ┌──────────┐   │              │        │            │       │
│      └──────┤Shared    │   │   Globus     │ ┌──────▼────────────▼─────┐ │
│             │Storage   ├───┼─────────────►┼─┤     S3 Staging Bucket   │ │
│             │          │   │              │ │                         │ │
│             └──────────┘   │              │ └─────────────────────────┘ │
└────────────────────────────┘              └─────────────────────────────┘
```

## Tutorial Steps

### Part 1: Prepare the Environment

#### 1.1. Assess Your On-Premises Cluster
- Document current Slurm configuration
- Identify workloads suitable for cloud bursting
- Analyze data requirements and movement patterns

#### 1.2. Set Up AWS Environment
- Create VPC with appropriate subnets
- Configure security groups
- Create IAM roles with necessary permissions
- Set up S3 bucket for data staging

#### 1.3. Prepare Network Connectivity
- Set up WireGuard VPN between on-premises and AWS
  - Configure WireGuard on on-premises gateway
  - Launch and configure WireGuard EC2 instance
  - Establish and test tunnel
- Configure routing between environments

#### 1.4. Set Up DNS Integration
- Configure dynamic DNS updates for cloud instances
- Set up split-horizon DNS if necessary
- Test name resolution across environments

### Part 2: Configure Slurm and AWS Plugin

#### 2.1. Install AWS Plugin Components
- Install prerequisites on Slurm controller
- Copy and configure plugin files
- Create initial configuration files

#### 2.2. Create Launch Template for Compute Nodes
- Select appropriate AMI with Slurm pre-installed
- Configure instance details
- Add bootstrap script with WireGuard setup
- Add DNS registration script

#### 2.3. Configure Slurm Partitions
- Configure on-premises partition
- Configure AWS cloud partition
- Set up TRESBillingWeights for cost accounting

#### 2.4. Configure Slurm Accounting
- Set up Slurm accounting database
- Configure fairshare settings
- Set up account and user limits

### Part 3: Data Management Solutions

#### 3.1. Set Up Globus for Data Transfer
- Install Globus endpoints
- Configure transfer automation
- Test file transfers

#### 3.2. Implement Job-Triggered Data Staging
- Create pre/post job scripts for data movement
- Configure S3 staging workflow
- Test end-to-end data flow

#### 3.3. Configure Burst Buffers (Optional)
- Configure Slurm burst buffer plugin
- Set up EFS auto-provisioning
- Test with sample workloads

### Part 4: Testing and Validation

#### 4.1. Basic Functionality Testing
- Verify node startup and registration
- Test job submission to cloud partition
- Validate accounting and service unit tracking

#### 4.2. Data Movement Testing
- Test small, medium, and large data transfers
- Measure performance and overhead
- Optimize based on findings

#### 4.3. Failure Scenario Testing
- Test VPN connection failure recovery
- Test instance launch failures
- Test data staging failure recovery

### Part 5: Monitoring and Management

#### 5.1. Set Up Cost Monitoring
- Configure AWS Budget alerts
- Set up Slurm usage reporting
- Create dashboard for cost tracking

#### 5.2. Performance Monitoring
- Set up CloudWatch metrics
- Configure cross-environment monitoring
- Create performance dashboards

#### 5.3. User Guidelines
- Document submission workflow
- Create examples for common scenarios
- Document best practices for efficient cloud usage

## Sample Configurations and Scripts

### WireGuard Configuration

```ini
# On-premises WireGuard configuration
[Interface]
PrivateKey = <on-prem-private-key>
Address = 10.200.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <aws-public-key>
AllowedIPs = 10.200.0.2/32, 172.31.0.0/16
Endpoint = <aws-wireguard-instance-public-ip>:51820
```

```ini
# AWS WireGuard configuration
[Interface]
PrivateKey = <aws-private-key>
Address = 10.200.0.2/24
ListenPort = 51820

[Peer]
PublicKey = <on-prem-public-key>
AllowedIPs = 10.200.0.1/32, 192.168.0.0/16
Endpoint = <on-prem-public-ip>:51820
```

### Dynamic DNS Registration Script

```bash
#!/bin/bash
# Script to register EC2 instance with on-premises DNS

# Get instance details
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
SLURM_NODENAME=$(curl -s http://169.254.169.254/latest/meta-data/tags/instance/Name)

# Update DNS via nsupdate
cat > /tmp/nsupdate.txt << EOF
server dns.on-premises.example
zone example.com
update delete $SLURM_NODENAME.example.com A
update add $SLURM_NODENAME.example.com 60 A $PRIVATE_IP
send
EOF

nsupdate -k /etc/dns-key.conf /tmp/nsupdate.txt
```

### Data Staging Job Wrapper Script

```bash
#!/bin/bash
# Wrapper script for staging data to/from S3

# Extract job information
JOB_ID=$SLURM_JOB_ID
INPUT_DIR=/shared/data/input/$SLURM_JOB_ID
OUTPUT_DIR=/shared/data/output/$SLURM_JOB_ID
S3_BUCKET=s3://slurm-burst-data/jobs/$SLURM_JOB_ID

# Stage input data to S3
echo "Staging input data to S3..."
aws s3 sync $INPUT_DIR $S3_BUCKET/input/

# Run the actual job command with all passed arguments
echo "Running job..."
"$@"
EXIT_CODE=$?

# Stage output data from S3
echo "Retrieving output data from S3..."
aws s3 sync $S3_BUCKET/output/ $OUTPUT_DIR/

exit $EXIT_CODE
```

### partitions.json with TRESBillingWeights

```json
{
   "Partitions": [
      {
         "PartitionName": "cloud",
         "NodeGroups": [
            {
               "NodeGroupName": "c5",
               "MaxNodes": 20,
               "Region": "us-east-1",
               "SlurmSpecifications": {
                  "CPUs": "4",
                  "RealMemory": "32000"
               },
               "PurchasingOption": "on-demand",
               "OnDemandOptions": {
                   "AllocationStrategy": "lowest-price"
               },
               "LaunchTemplateSpecification": {
                  "LaunchTemplateName": "slurm-bursting-compute",
                  "Version": "$Latest"
               },
               "LaunchTemplateOverrides": [
                  {
                     "InstanceType": "c5.xlarge"
                  }
               ],
               "SubnetIds": [
                  "subnet-012345abcdef"
               ]
            }
         ],
         "PartitionOptions": {
            "MaxTime": "12:00:00",
            "TRESBillingWeights": "CPU=4.0,Mem=0.25G,Node=2.0",
            "DefMemPerCPU": "8000"
         }
      }
   ]
}
```

## Best Practices and Recommendations

1. **Start Small**: Begin with a limited set of workloads and users
2. **Monitor Closely**: Watch costs, performance, and user behavior
3. **Educate Users**: Ensure users understand the cost implications
4. **Test Thoroughly**: Validate all components, especially failure scenarios
5. **Optimize Incrementally**: Refine based on actual usage patterns

## Troubleshooting Common Issues

| Issue | Possible Causes | Solutions |
|-------|----------------|-----------|
| Nodes fail to start | IAM permissions, network connectivity | Check IAM roles, verify VPN connectivity |
| Jobs fail to start on cloud nodes | SSH connectivity, Slurm configuration | Verify SSH keys, check Slurm configuration |
| Slow data transfer | Network bandwidth, transfer method | Consider using Globus, optimize transfer parameters |
| High latency affects jobs | VPN overhead, geographic distance | Filter suitable jobs, consider regional AWS deployment |
| Cost overruns | Jobs running too long, too many instances | Set stricter time limits, implement budget controls |

## Conclusion

While setting up on-premises to AWS bursting requires addressing various challenges, the result can provide significant flexibility and cost savings for your HPC environment. This tutorial has walked through a practical implementation that balances performance, security, and cost considerations.

As you implement this solution, remember that each environment is unique. Adapt these recommendations to your specific requirements, and iterate based on your findings and user feedback.