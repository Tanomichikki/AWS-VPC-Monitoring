# AWS VPC Monitoring Project – Flow Logs, CloudWatch Insights & VPC Peering

This project demonstrates how to monitor, analyze, and troubleshoot network traffic in AWS using:
- VPC Flow Logs
- Amazon CloudWatch Logs
- CloudWatch Log Insights
- EC2 Traffic Testing
- VPC Peering

It recreates a real-world scenario where cloud engineers must track traffic, detect failures, monitor blocked connections, and troubleshoot networking issues across environments

### Project Goal

The goal of this project is to monitor network traffic inside a VPC and analyze accepted and rejected connections.

This helps in:
- Troubleshooting connectivity
- Identifying blocked traffic
- Monitoring workloads
- Understanding cross-VPC communication
- Ensuring secure and compliant networking


### Architecture of the Project
<p align="center"> <img src="https://github.com/Tanomichikki/AWS-VPC-Monitoring/blob/main/Architecture.drawio.png" width="60%" /> </p>


### ▶️ Watch the Project

[![VPC Monitoring Project](https://img.youtube.com/vi/SX-RKo2n3M8/0.jpg)](https://www.youtube.com/watch?v=SX-RKo2n3M8)


### Step 1 – Create Two VPCs

I created two separate VPCs to simulate a real-world multi-environment network setup. These VPCs act as two isolated networks that will later communicate through VPC Peering.


| VPC Name          | CIDR Block    | Public IPv4 | Purpose              |
| ----------------- | ------------- | ----------- | -------------------- |
| **Network-1-vpc** | `10.1.0.0/16` | Enabled     | Hosts EC2 Instance 1 |
| **Network-2-vpc** | `10.2.0.0/16` | Enabled     | Hosts EC2 Instance 2 |


Each VPC includes:
- One public subnet
- An Internet Gateway for outbound connectivity
- A Route Table associated with the subnet
- Public IPv4 addressing enabled
- Only one Availability Zone
- No S3 endpoints

**Why we need two seperate VPCs?**

Using two separate VPCs helps replicate how companies isolate workloads — for example, separating development and production environments or creating network boundaries between teams and applications.
This setup allows us to test cross-VPC communication, monitoring, and peering just like in real AWS environments.

**Why use different CIDR blocks?**

We must use unique, non-overlapping CIDR blocks so that AWS can route traffic correctly between the two VPCs.
If the IP ranges overlap:
- VPC peering cannot be created
- Routing tables cannot distinguish destination networks
- Traffic will be dropped because AWS cannot identify the correct path

Therefore, 10.1.0.0/16 and 10.2.0.0/16 ensure clean routing and allow successful VPC peering.

### Step 2 – Launch EC2 Instances in Each VPC

In this step, I launched one EC2 instance inside each VPC to generate network traffic and validate connectivity between the two isolated networks.

| EC2 Instance Name  | Launched In          | VPC Attached      | Purpose                  |
| ------------------ | -------------------- | ----------------- | ------------------------ |
| **instance-1-vpc** | Public Subnet (AZ-1) | **Network-1-vpc** | Test connectivity + logs |
| **instance-2-vpc** | Public Subnet (AZ-1) | **Network-2-vpc** | Cross-VPC communication  |


**Security Group Rules Applied (Same for Both Instances)**

Both instances use identical security group configurations:

| Rule Type    | Protocol/Port | Source      | Reason                                          |
| ------------ | ------------- | ----------- | ----------------------------------------------- |
| **SSH**      | TCP 22        | `0.0.0.0/0` | Allows SSH access for configuration and testing |
| **All ICMP** | ICMP (Ping)   | `0.0.0.0/0` | Allows ping tests to verify connectivity        |

These rules ensure:
- SSH is available for admin access
- ICMP packets can flow between instances for ping-based testing
- Logs are generated for analysis in Flow Logs + CloudWatch Insights

**Why Use These Security Group Rules?**

**ICMP (Ping)**

Ping helps verify whether instances can reach each other.

It is the simplest and most reliable test for:
- Routing validation
- VPC peering connectivity
- Security group/NACL behavior
- Flow Logs capture

**SSH (Port 22)**

We need SSH access to:
- Log into the EC2 instance
- Generate controlled network traffic
- Perform connectivity testing
- Troubleshoot routing and permissions

**Allowing 0.0.0.0/0 While Testing**

Using 0.0.0.0/0 is acceptable in a learning environment because it guarantees:
- No accidental security group blocks
- All traffic reaches the instance
- Flow Logs generate complete data

**Why Launch EC2s in Separate VPCs?**

To simulate isolated environments
- To test VPC Peering
- To generate cross-VPC traffic
- To capture ACCEPT/REJECT logs for analysis
- To understand real-world cloud networking
Each instance serves as the “endpoint” for our connectivity and monitoring tests later in the project.


### Step 3 – Enable VPC Flow Logs

VPC Flow Logs are enabled at the VPC level so that every network event inside the VPC is captured — including traffic going through EC2 instances, Network ACLs, and Security Groups. Flow Logs record essential metadata:
- ACCEPT / REJECT traffic decisions
- Source & destination IP addresses
- Ports, protocols
- Bytes sent and received

**Why do we enable Flow Logs?**

Flow Logs act as network CCTV for your VPC. They allow you to see exactly what traffic is flowing, what is being blocked, and whether your routing and security configurations are working correctly. Without Flow Logs, diagnosing networking failures is almost impossible in real-world AWS environments.


### Step 4 – Create CloudWatch Log Group

Before enabling Flow Logs, we must create a CloudWatch Log Group where the logs will be stored.

**Log Group Name:** ProjectVPCFlowLogGroups

**Retention Period:** 7 or 30 days (to avoid unnecessary cost)

**Why CloudWatch Log Groups?**

Flow Logs cannot store data by themselves — they need a destination.
A CloudWatch Log Group provides:
- A storage location for all Flow Logs
- The ability to set log retention so logs don’t grow endlessly
- Compatibility with CloudWatch Log Insights, enabling powerful query-based analysis
Without this Log Group, Flow Logs cannot publish any data, and the feature simply won’t work.


### Step 5 – IAM Role + Policy for Flow Logs

When trying to enable Flow Logs on the first VPC, the logs failed to deliver because no IAM permissions existed.
Flow Logs do not automatically have permission to write into CloudWatch, so we must create:
- A custom IAM policy (permissions)
- An IAM role (identity that Flow Logs assume)

**Create IAM Policy (Flow Logs → CloudWatch)**
***Policy Name:*** VPCFlowLogPolicy
```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

**Why did I create this custom IAM Policy?**

Flow Logs need explicit permission to write logs into CloudWatch.
AWS does not create this automatically — so without this policy:
- No log groups or streams can be created
- Flow Logs fail silently
- No logs appear in CloudWatch
This policy enforces least-privilege access, controlling exactly what Flow Logs can do.

**Create IAM Role**

**Role name:** ProjectVPCFlowLogRole
```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
**Why did I create this IAM Role (and why assume it)?**

Flow Logs must assume an IAM Role to publish logs into CloudWatch securely.
The trust relationship allows the Flow Logs service to “become” this role temporarily through STS AssumeRole.
This ensures:
- Secure communication between VPC Flow Logs and CloudWatch
- AWS limits permission strictly to what Flow Logs need
- Logs are delivered reliably
This IAM Role is essential — Flow Logs cannot function without it.


### Step 6 – Create VPC Peering

To allow communication between the two VPCs, a VPC peering connection is required.

**Peering Name:** VPC1<>VPC2

**Requester VPC:** Network-1-vpc
CIDR: 10.1.0.0/16

**Accepter VPC:** Network-2-vpc
CIDR: 10.2.0.0/16

Steps Performed
- From Network-1-vpc, I initiated the peering request to Network-2-vpc.
- In Network-2-vpc, I accepted the request.
- The peering connection became active and received a peering ID (example: pcx-xxxxxxx).

**Why do we use VPC Peering?**

VPC Peering creates a direct, private, low-latency connection between two VPCs.
This allows:
- EC2 in VPC 1 to communicate with EC2 in VPC 2 privately
- No need for internet gateways, NAT, or public IPs
- Fully secure, internal AWS backbone communication

### Step 7 – Update Route Tables for Both VPCs

After creating a peering connection, traffic still cannot flow unless route tables are updated.

**Network-1-rt (for Network-1-vpc)**

Added the following route:

Destination CIDR	Target
10.2.0.0/16	pcx-<your-peering-id> (VPC1 <> VPC2)

This allows EC2 in VPC 1 to reach resources inside VPC 2.

**Network-2-rt (for Network-2-vpc)**

Added the following route:

Destination CIDR	Target
10.1.0.0/16	pcx-<your-peering-id> (VPC1 <> VPC2)

This allows EC2 in VPC 2 to reach resources inside VPC 1.

**Why update route tables?**

Even though the peering connection exists, routing does not configure itself automatically.
Route tables tell the VPC:

- “If you want to reach 10.2.0.0/16, send the traffic to the peering connection.”
- “If you want to reach 10.1.0.0/16, use the same peering connection.”

Without these routes:
- ping will fail
- SSH will fail
- Any cross-VPC communication will be dropped
- Flow Logs will show REJECT packets

Updating route tables is the crucial step that enables successful private communication between the two EC2 instances.


### Step 8 – Test Connectivity between EC2 Instances

Once VPC Peering and route tables are configured, the next step is to verify whether both EC2 instances can communicate privately.

**Connectivity Test**

From Instance-1 (inside Network-1-vpc), I ran:
```bash
ping <Private-IP-of-Instance-2>
```

From Instance-2 (inside Network-2-vpc), I ran:
```bash
ping <Private-IP-of-Instance-1>
```

**Why run ping?**

Pinging helps confirm whether:
- The peering connection is active
- The route tables are correctly configured
- The security groups allow ICMP traffic

Both VPCs can communicate over the private AWS network
- If the connectivity is successful, VPC Flow Logs will show ACCEPT entries.
- If anything is misconfigured (SG, route table, or missing peering), Flow Logs will capture REJECT entries.

**What I did**

I connected to Instance-1 using SSH and executed the ping command using the private IPv4 address of Instance-2.
This successfully validated that cross-VPC communication was working through the VPC Peering connection.


### Step 9 – Analyze Flow Logs Using CloudWatch Log Insights

After generating traffic between the EC2 instances, the next step is to analyze the recorded network activity using CloudWatch Log Insights.
This helps validate that the VPC Flow Logs are working correctly and provides deep visibility into network patterns.

**Open Log Insights**
Go to:
```pgsql
CloudWatch → Logs → Log Groups → ProjectVPCFlowLogGroups → Log Insights
```

CloudWatch Log Insights automatically detects and structures the fields in flow logs, including:
- srcAddr – Source IP
- dstAddr – Destination IP
- action – ACCEPT / REJECT
- bytes – Data transferred
- protocol – e.g., ICMP, TCP
- pktSrcAddr / pktDstAddr – Packet-level details

**Example Queries to Analyze Traffic**

View All Flow Log Events
```sql
fields @timestamp, srcAddr, dstAddr, action, bytes, protocol
| sort @timestamp desc
```

See Only REJECTED Traffic
```sql
filter action = 'REJECT'
| fields @timestamp, srcAddr, dstAddr, protocol, bytes
| sort @timestamp desc
```

Check Accepted Traffic Between the Two EC2 Instances
```sql
filter srcAddr = '<Private-IP-of-Instance-1>' or srcAddr = '<Private-IP-of-Instance-2>'
| fields @timestamp, srcAddr, dstAddr, action, bytes
| sort @timestamp desc
```

Count Traffic by Protocol
```sql
stats count() by protocol
```

**Why Analyze Logs with Log Insights?**

Using Log Insights helps you:
- Verify whether VPC peering is actually working
- Troubleshoot network connectivity issues
- Identify blocked or unwanted traffic
- Understand the flow patterns across subnets and VPCs
- Perform audit-level monitoring of your networking layer
It transforms raw flow logs into meaningful, searchable, query-based insights so you can confirm the exact behavior of your VPC traffic.


### Final Outcome

- Created two separate VPCs, Network-1-vpc (10.1.0.0/16) and Network-2-vpc (10.2.0.0/16), to simulate a real multi-VPC environment used in organizations.
- Launched two EC2 instances (instance-1 and instance-2) inside each VPC and allowed SSH and ICMP to test connectivity and generate traffic.
- Enabled VPC Flow Logs on both VPCs to capture network traffic information such as ACCEPT/REJECT, ports, protocols, and bytes transferred.
- Created a CloudWatch Log Group named ProjectVPCFlowLogGroups to store all flow log data for monitoring and analysis.
- Set up an IAM Policy (VPCFlowLogPolicy) and an IAM Role (ProjectVPCFlowLogRole) to give VPC Flow Logs permission to publish logs into CloudWatch.
- Established a VPC Peering connection between the two VPCs (VPC1 <> VPC2) so instances in different networks could communicate privately.
- Updated the route tables in both VPCs so each network could reach the other’s CIDR through the peering connection.
- Verified communication by pinging the private IPs of both EC2 instances, and finally analyzed the captured traffic using CloudWatch Log Insights to confirm successful ACCEPT entries.
