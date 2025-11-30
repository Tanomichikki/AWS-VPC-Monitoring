# 🌐 VPC Monitoring Project — Flow Logs, CloudWatch Log Insights & VPC Peering

A complete hands-on AWS networking project demonstrating VPC Monitoring, Flow Logs, IAM Roles, CloudWatch Insights, and VPC Peering to analyze real traffic between two VPCs.


### 📌 📘 Project Overview

In this project, I built a complete monitoring and traffic analysis environment using:

- Two VPCs (VPC-A and VPC-B)

- EC2 instances to generate network traffic

- VPC Flow Logs to capture accepted/denied traffic

- CloudWatch Logs for storage

- CloudWatch Log Insights to analyze patterns

- VPC Peering to test cross-VPC traffic

- IAM Role & Policy to publish flow logs

This project reflects real-world troubleshooting scenarios used by AWS Cloud Support Engineers.



### 🏛️ Architecture Diagram
             +----------------------------------------------+
             |                  AWS Cloud                   |
             +----------------------------------------------+

      VPC-A (10.0.0.0/16)                         VPC-B (20.0.0.0/16)
      ---------------------                       ----------------------
      | EC2 Instance A     | ---- Peering -----> | EC2 Instance B     |
      | (Traffic Source)    |                     | (Traffic Target)   |
      ----------------------                      ----------------------

                    |                                   |
                    |                                   |
                    +----------- VPC Flow Logs ---------+
                                (All Traffic)

                              ↓ Sent to CloudWatch

                      CloudWatch Log Group
                       /vpc-flow-logs/project

                              ↓ Analysis

                       CloudWatch Log Insights
                      (Query, Filter & Analyze)



### 🚀 1. Creating Two VPCs

We create VPC-A and VPC-B, each with:

Unique IPv4 CIDR blocks

Public subnets

Internet Gateways

Route tables



✔ Why separate VPCs?

To simulate real multi-VPC environments used in enterprises

Allows demonstrating VPC peering and cross-VPC traffic

Essential for network-level traffic monitoring



### 🖥️ 2. Launch EC2 Instances

One EC2 instance is launched in each VPC.

✔ Security Group Rules
Inbound Rules:
- SSH (22) → My IP
- ICMP (Ping) → Anywhere or specific CIDR

✔ Why EC2 is required?

We need real traffic (ping, SSH attempts)

Flow Logs only generate entries when traffic occurs

EC2 acts as the "client" and "server" for testing connectivity



### 📡 3. Enable VPC Flow Logs

Flow Logs capture:

Accepted traffic

Rejected traffic

Source/Destination IP

Port, protocol, bytes transferred

Security group evaluations

✔ Why Flow Logs?

Helps troubleshoot network failures

Identifies blocked traffic

Helps analyze security group / NACL issues

Useful for auditing and compliance



### 🔐 4. IAM Role & Policy for Flow Logs

Flow Logs cannot write to CloudWatch without permissions, so we manually create:

IAM Policy

IAM Role

Trust Relationship


##### 🛡️ IAM Policy for VPC Flow Logs

Attach this custom policy:
```json
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

✔ Why this policy?

Flow Logs need permission to create log groups & streams

Then continuously send log events to CloudWatch

#### 🎭 IAM Role Trust Relationship

Add this to allow VPC Flow Logs to assume the role:

```json
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

✔ Why trust policy?

AWS services must be explicitly trusted

This allows Flow Logs to use the role to publish logs


#### 🔗 Attach IAM Role in Flow Log Creation

During Flow Log creation:

Destination: CloudWatch Logs

Log group: /vpc-flow-logs/project

IAM role: VPCFlowLogsToCloudWatchRole



### 📁 5. Create CloudWatch Log Group
/vpc-flow-logs/project

✔ Set retention

7 days / 30 days

Saves cost

✔ Why CloudWatch?

Acts as central monitoring storage

Allows advanced querying with Log Insights



### 🔍 6. Analyze Traffic Using CloudWatch Log Insights

Example queries:

🔸 Show all ACCEPTED traffic
fields @timestamp, srcAddr, dstAddr, action
| filter action = "ACCEPT"
| sort @timestamp desc

🔸 Show all REJECTED traffic
fields @timestamp, srcAddr, dstAddr, action
| filter action = "REJECT"
| sort @timestamp desc

🔸 Top Talkers (Who sends most traffic?)
stats sum(bytes) by srcAddr



### 🔗 7. VPC Peering Connection

Create a peering connection between the two VPCs.

✔ Why?

Enables private communication between VPCs

No public internet involved

Low latency


### 🛣️ 8. Update Route Tables

Add routes:

In VPC-A route table:

Destination: 20.0.0.0/16 → Target: pcx-xxxx


In VPC-B route table:

Destination: 10.0.0.0/16 → Target: pcx-xxxx

✔ Why routes?

Peering is useless until route tables allow traffic


### 🧪 9. Test Connectivity

From EC2 in VPC-A:

```bash
ping <EC2-B-Private-IP>
```

From EC2 in VPC-B:
```bash
ping <EC2-A-Private-IP>
```

✔ Traffic now appears in Flow Logs:

accepted/rejected

bytes transferred

security group decisions


### 🏆 Final Achievement

By the end of this project, you have:


✔ Built two VPCs from scratch

✔ Launched EC2 instances to generate network traffic

✔ Configured VPC Flow Logs with IAM roles

✔ Sent logs to CloudWatch for monitoring

✔ Queried logs using Log Insights

✔ Created a VPC Peering connection

✔ Tested cross-VPC connectivity

✔ Understood how networking traffic flows in AWS
