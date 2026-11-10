# Multi-Region Warm Standby Disaster Recovery on AWS

## 1. Overview

This project implements a **Warm Standby Disaster Recovery (DR)** architecture across two AWS regions:

- **Primary:** `us-east-1`
- **Secondary (DR):** `us-west-1`

The secondary region maintains a scaled-down but operational environment, ready to take over production traffic during a regional failure.

Infrastructure is provisioned and managed using AWS CloudFormation.

- [Diagram link: ](https://lucid.app/lucidchart/0ae83608-eddb-4dba-9743-c4293e531346/view)
  ![Multi-Region Warm Standby architecture](./architecture.png)

## 2. Architecture Components

| Component                 | Primary Region                  | DR Region            |
| ------------------------- | ------------------------------- | -------------------- |
| VPC                       | Active                          | Pre-provisioned      |
| Application Load Balancer | Active                          | Standby              |
| EC2 Auto Scaling Group    | Full capacity                   | Minimum capacity     |
| Aurora                    | Primary cluster + read replicas | Secondary cluster    |
| DynamoDB                  | Global Table replica            | Global Table replica |
| S3                        | Source bucket                   | Cross-region replica |
| Route 53                  | Production DNS                  | Failover target      |

Additional services:

- **CloudWatch:** Monitors infrastructure health and detects failures.
- **EventBridge:** Captures alarms and triggers the DR workflow.
- **Step Functions:** Orchestrates the failover process.
- **CloudFormation:** Deploys infrastructure consistently across regions.

## 3. Normal Operations

1. Users access the application through Route 53.
2. Route 53 directs traffic to the primary region.
3. The primary ALB distributes requests across EC2 instances managed by an Auto Scaling Group.
4. Aurora handles relational database operations, with read replicas supporting read workloads.
5. DynamoDB Global Tables replicate data across regions.
6. S3 Cross-Region Replication asynchronously copies objects to the DR region.
7. The DR environment remains operational with reduced EC2 capacity and a replicated database.

## 4. Disaster Recovery / Failover Workflow

When the primary region becomes unavailable:

1. CloudWatch detects the failure and triggers an alarm.
2. EventBridge receives the alarm event and starts the Step Functions workflow.
3. Step Functions initiates Aurora promotion in the DR region, making the secondary cluster writable.
4. The workflow triggers the DR Auto Scaling Group to increase EC2 capacity.
5. Event-driven readiness checks verify that:
   - Aurora promotion has completed successfully.
   - The Auto Scaling Group has reached its required capacity.
   - The application is healthy behind the DR ALB.
6. Once all required checks succeed, the workflow updates the Route 53 production DNS record to direct users to the DR region.
7. The DR region becomes the active production environment.

**Important:** DNS failover is performed only after database promotion and application capacity scaling have completed successfully. Step Functions coordinates these asynchronous operations.

## 5. Data Replication Strategy

- **Aurora:** Cross-region replication maintains a secondary database cluster. Promotion is required before accepting writes in the DR region.
- **DynamoDB:** Global Tables provide multi-region data replication.
- **S3:** Cross-Region Replication maintains a copy of objects in the DR region.
