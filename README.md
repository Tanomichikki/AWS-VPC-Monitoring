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
<p align="center"> <img src="https://github.com/Tanomichikki/AWS-VPC-Monitoring/blob/main/Architecture.drawio.png" width="80%" /> </p>

### Step 1 – Create Two VPCs From Scratch

We require two different isolated networks to simulate real-world multi-VPC communication. Each VPC needs a unique CIDR block to avoid IP conflicts and ensure smooth routing later during peering.

- A subnet is required to host EC2 instances.
- The Internet Gateway is needed for outbound traffic (software updates, testing).
- Route tables ensure the subnet can reach the internet and later the other VPC.

Create:
- VPC A
```bash
10.1.0.0/16
```

- VPC B
```bash
10.2.0.0/16
```

Each with:
- One public subnet
- One Internet Gateway
- Route table

**Why two VPCs?**
To simulate real-world multi-network AWS setups.
Engineers often connect different VPCs used by different teams, apps, or environments.

**Why different CIDR blocks?**
If CIDR ranges overlap, VPC peering cannot work.
Routing breaks.

### Step 2 – Launch EC2 Instances in Each VPC

Launch:
- EC2-A inside VPC-A
- EC2-B inside VPC-B

Security group inbound rules:
- Allow SSH (port 22)
- Allow ICMP (Ping)
- Allow traffic from the peer VPC CIDR

Security Groups determine what traffic is allowed.
Ping and SSH help us test connectivity and produce logs that we later analyze using CloudWatch Insights.

### Step 3 – Enable VPC Flow Logs

You enable Flow Logs at the VPC level so every EC2/NACL/SG traffic is captured.

Flow Logs capture:
- ACCEPT / REJECT
- Source/Destination IP
- Ports
- Protocols
- Bytes in/out

**Why enable Flow Logs?**
They act as a CCTV that records all network activity inside your VPC.


### Step 4 – Create CloudWatch Log Group

Example:
/vpc-flow-logs/project
Set retention to 7 or 30 days.

**Why CloudWatch?**
Flow Logs need somewhere to store captured traffic.
CloudWatch Log Group allows retention settings, log classes, and integration with Log Insights for deeper analytics.

### Step 5 – IAM Role + Policy for Flow Logs

Flow Logs need permission to publish logs into CloudWatch.

✔ Create IAM Policy (Flow Logs → CloudWatch)

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

✔ Create IAM Role
Role name:  VPCFlowLogsRole
Attach the above policy.

✔ Add Trust Relationship
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

**Why is IAM needed?**

VPC Flow Logs do NOT automatically have permission to publish data.

We create:
- IAM Policy → defines what logs Flow Logs can push
- IAM Role → attached to Flow Logs so AWS can assume it
This ensures secure, least-privilege access.

### Step 6 – Create VPC Peering

Steps:
1.	From VPC-A → Request peering to VPC-B
2.	In VPC-B → Accept the request
3.	Update route tables in both VPCs

**Why VPC Peering?**
- VPC Peering creates a private link between two VPCs.
- It enables the EC2 instances to communicate without using the internet.
- This replicates real-world multi-VPC architectures used by companies.

### Step 7 – Update Route Tables

**VPC-A Route Table**
```bash
Destination: 20.0.0.0/16
Target: pcx-abcd1234
```

**VPC-B Route Table**
```bash
Destination: 10.0.0.0/16
Target: pcx-abcd1234
```

**Why routes?**
- Peering alone does NOT enable communication.
- Route tables define how to reach the other VPC’s CIDR.
- Without route updates, packets would be dropped.

### Step 8 – Test Connectivity between EC2 Instances
From EC2-A:
```bash
ping <PrivateIP-of-EC2-B>
````
From EC2-B:
```bash
ping <PrivateIP-of-EC2-A>
```
If working → Flow Logs generate ACCEPT entries
If blocked → Flow Logs generate REJECT entries
Pinging confirms whether the route tables, security groups, and peering connection are correctly configured.
If traffic fails, Flow Logs help identify which layer is blocking it.

# Step 9 – Analyze Flow Logs via CloudWatch Log Insights

✔ View ACCEPTED traffic
```bash
fields @timestamp, srcAddr, dstAddr, action
| filter action = "ACCEPT"
| sort @timestamp desc
```
✔ View REJECTED traffic
```bash 
fields @timestamp, srcAddr, dstAddr, action
| filter action = "REJECT"
| sort @timestamp desc
```
✔ Top Ips sending data
```bash
stats sum(bytes) by srcAddr
```
✔ Traffic between the two VPCs
```bash
fields srcAddr, dstAddr, action
| filter (srcAddr like /^10\./ and dstAddr like /^20\./)
```
Or
```bash
(srcAddr like /^20\./ and dstAddr like /^10\./)
```

### Final Outcome

By finishing this project, you now understand:

✔ How VPCs send/receive traffic

✔ How to diagnose blocked traffic

✔ How CloudWatch Logs store and display network logs

✔ How Flow Logs act like traffic CCTV

✔ How VPC Peering connects two private networks

✔ How Log Insights helps analyze network patterns

✔ IAM Role + Policy required for Flow Logs

✔ EC2-level networking troubleshooting
