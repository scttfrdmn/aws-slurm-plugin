# On-Premises to AWS Cloud Bursting Considerations

## Overview

The current AWS Plugin for Slurm documentation primarily focuses on deploying and managing Slurm resources within AWS (cloud-to-cloud bursting). However, true "bursting" typically refers to extending an on-premises cluster to the cloud when local resources are insufficient.

This document outlines critical considerations and challenges that are not fully addressed in the current implementation when attempting to configure on-premises to cloud bursting.

## Critical Challenges

### 1. Network Connectivity

**Issues:**
- The plugin assumes relatively low-latency, high-bandwidth connectivity between the head node and compute nodes
- No detailed guidance on establishing secure, reliable connections between on-premises and AWS environments

**Requirements:**
- VPN or AWS Direct Connect setup between on-premises and AWS
- Network architecture that allows bidirectional communication
- Consideration of bandwidth limitations and latency impact on job performance
- Proper NAT/routing configuration to handle potentially overlapping IP address spaces

**Suggestion:**
- Add documentation section on network architecture options
- Include reference architectures for common connectivity patterns
- Document performance implications for different connection types

### 2. Shared Storage Access

**Issues:**
- Jobs typically require access to shared storage
- No guidance on how compute nodes in AWS access data that resides on-premises
- Data transfer bottlenecks can severely impact performance

**Requirements:**
- Accessible shared filesystem across both environments
- Efficient data transfer mechanisms
- Management of data placement and staging

**Suggestions:**
- Document common HPC storage patterns (AWS FSx for Lustre, NFS, parallel data staging)
- Provide guidance on pre/post job data staging
- Include sample configurations for different storage scenarios
- Discuss performance impacts and optimization strategies

### 3. Authentication and Security

**Issues:**
- AWS authentication from on-premises systems requires additional security considerations
- Protection of AWS credentials on on-premises systems
- Managing security groups and firewall rules across environments

**Requirements:**
- Secure method of managing AWS credentials from on-premises
- Network security design that maintains security boundaries
- Consistent security policies across environments

**Suggestions:**
- Document IAM best practices for cross-environment scenarios
- Include guidance on security group configuration
- Address credential rotation and management

### 4. Consistent Runtime Environment

**Issues:**
- Software environments must be consistent across all nodes
- No guidance on maintaining consistency between on-premises and cloud nodes

**Requirements:**
- Consistent OS, libraries, and application stack
- Management of software differences between environments
- Handling version skew

**Suggestions:**
- Document container-based or image-based approaches to environment consistency
- Include guidance on automated environment validation
- Add section on software deployment strategies

### 5. Job Scheduling Considerations

**Issues:**
- Current documentation doesn't address scheduling policies optimized for hybrid environments
- No discussion of when and how to dispatch jobs to cloud resources based on cost/performance

**Requirements:**
- Intelligent scheduling that considers cost, performance, and data locality
- Policies that optimize for hybrid environments

**Suggestions:**
- Document Slurm scheduling policies appropriate for hybrid deployments
- Include guidance on workload-appropriate partitioning
- Add section on cost-aware scheduling options

### 6. Cost Management and Optimization

**Issues:**
- No discussion of cost control mechanisms
- No guidance on preventing runaway cloud costs

**Requirements:**
- Budget controls and monitoring
- Usage tracking and attribution
- Cost optimization strategies

**Suggestions:**
- Add guidance on AWS budgets and cost alerts
- Document how to track costs by partition, user, or project
- Provide strategies for cost optimization (spot instances, right-sizing, etc.)

### 7. Monitoring and Troubleshooting

**Issues:**
- Monitoring and management across hybrid environments is more complex
- Current documentation has limited troubleshooting guidance

**Requirements:**
- Unified monitoring across on-premises and cloud resources
- Cross-environment logging and diagnostics
- Failure detection and recovery

**Suggestions:**
- Add section on unified monitoring approaches
- Document common failure modes and resolution strategies
- Include hybrid-specific troubleshooting scenarios

## Implementation Recommendations

To successfully implement on-premises to cloud bursting, consider the following high-level approach:

1. **Start with a Limited Proof-of-Concept:**
   - Select non-critical workloads
   - Validate basic connectivity and job execution
   - Measure performance impacts

2. **Address Infrastructure First:**
   - Establish reliable, secure networking
   - Solve data access challenges
   - Set up consistent environments

3. **Implement Progressive Policies:**
   - Begin with manual cloud allocation
   - Progress to automated bursting for specific partitions
   - Finally implement fully dynamic bursting

4. **Continuously Monitor and Optimize:**
   - Regularly review usage patterns
   - Identify optimization opportunities
   - Refine cost controls

## Additional Critical Considerations

### 1. Cost Management with Slurm Service Units

**Issues:**
- No guidance on how to implement budgeting and accounting for hybrid environments
- Cloud instances have different costs than on-premises resources

**Requirements:**
- Tracking and limiting usage by department, project, or user
- Differential accounting for on-premises vs. cloud resources

**Suggestions:**
- Implement Slurm's service unit capability (TRESBillingWeights) to assign appropriate costs to cloud partitions
- Configure fairshare settings that account for the higher cost of cloud resources
- Example configuration:
  ```
  PartitionName=aws TRESBillingWeights="CPU=4.0,Mem=0.25G,Node=2.0"
  PartitionName=on-prem TRESBillingWeights="CPU=1.0,Mem=0.1G,Node=1.0"
  ```
- Document how to integrate with cost reporting and budget enforcement tools

### 2. Network Addressing & DNS Integration

**Issues:**
- Burst nodes need to be addressable by both the Slurm controller and other on-premises systems
- No guidance on DNS integration across environments

**Requirements:**
- Either public or private addressing strategies
- Name resolution across environments

**Suggestions:**
- **Public addressing option:**
  - Document security group and network ACL configurations
  - Address securing public-facing instances
  - Custom security hardening for Internet-accessible HPC resources

- **Private addressing option:**
  - Detailed tutorial on setting up WireGuard or similar VPN solution
  - Instructions for Slurm configuration with VPN-tunneled connections
  - Performance considerations for VPN overhead

- **DNS integration:**
  - Guidance on dynamic DNS updates from cloud instances
  - Split-horizon DNS configuration options
  - Example scripts for maintaining consistent naming across environments

### 3. Data Staging Strategies

**Issues:**
- Current documentation doesn't address options for efficient data movement
- No guidance on selecting appropriate data staging methods for different workloads

**Requirements:**
- Efficient movement of job input/output data
- Support for various data sizes and access patterns

**Suggestions:**
- **Slurm Burst Buffers:**
  - Configuration examples for managing temporary storage in AWS
  - Integration with job scripts

- **Direct File Transfer:**
  - Rsync-based solutions for smaller datasets
  - Globus-based solutions for large-scale, reliable transfers
  - Performance optimization guidelines

- **Cloud Storage Integration:**
  - Using S3 as intermediate staging area
  - Example job wrappers that handle S3 data retrieval/storage
  - Integration with AWS DataSync for scheduled transfers

- **Dynamic Storage Provisioning:**
  - Creating per-job ephemeral EFS instances
  - Automating storage lifecycle with job lifecycle
  - Cost implications and optimization strategies

## Common Architectural Patterns

### 1. Data-Centric Approach
Prioritize moving compute to where data resides, minimizing data movement across environments.

### 2. Job Type Segregation
Categorize jobs by their suitability for cloud bursting (considering latency sensitivity, data requirements, etc.)

### 3. Scheduled Expansion
Rather than real-time bursting, schedule expansion of cloud resources during periods of high demand.

### 4. Hybrid Storage Architecture
Implement tiered storage that spans on-premises and cloud, with automated data placement and migration.

## Conclusion

While the AWS Plugin for Slurm provides a solid foundation for AWS resource integration, implementing true on-premises to cloud bursting requires careful consideration of the challenges outlined above. Each organization will need to adapt these considerations to their specific workloads, security requirements, and performance expectations.

Future enhancements to this plugin should consider addressing these challenges more comprehensively to enable seamless hybrid cloud operations for Slurm environments.