# Network & Security Reference

Reference tables for the 3-tier architecture (ALB → ECS Fargate → RDS PostgreSQL Multi-AZ) deployed in `us-east-1`, VPC `10.0.0.0/16`.

## 1. Subnets

| Subnet | CIDR | AZ | Tier | Resources |
|---|---|---|---|---|
| `PublicSubnetA` | `10.0.1.0/24` | `us-east-1a` | Public | ALB, NAT Gateway A |
| `PublicSubnetB` | `10.0.2.0/24` | `us-east-1b` | Public | ALB, NAT Gateway B |
| `PrivateAppSubnetA` | `10.0.11.0/24` | `us-east-1a` | Private (app) | ECS Fargate task |
| `PrivateAppSubnetB` | `10.0.12.0/24` | `us-east-1b` | Private (app) | ECS Fargate task |
| `PrivateDBSubnetA` | `10.0.21.0/24` | `us-east-1a` | Private (data) | RDS PostgreSQL Primary |
| `PrivateDBSubnetB` | `10.0.22.0/24` | `us-east-1b` | Private (data) | RDS PostgreSQL Standby |

## 2. Route tables

### RTB-Public

| **Destination** | **Target** | **Purpose** |
|----|----|----|
| `10.0.0.0/16` | `local` | VPC internal traffic |
| `0.0.0.0/0` | `Internet Gateway` | Public internet access (ALB, NAT Gateways) |

**Associated subnets:** `PublicSubnetA`, `PublicSubnetB`

### RTB-PrivateApp-A

| **Destination** | **Target** | **Purpose** |
|----|----|----|
| `10.0.0.0/16` | `local` | VPC internal traffic |
| `0.0.0.0/0` | `NAT Gateway A` | Egress: ECR image pull, CloudWatch Logs |

**Associated subnets:** `PrivateAppSubnetA`

### RTB-PrivateApp-B

| **Destination** | **Target** | **Purpose** |
|----|----|----|
| `10.0.0.0/16` | `local` | VPC internal traffic |
| `0.0.0.0/0` | `NAT Gateway B` | Egress: ECR image pull, CloudWatch Logs |

**Associated subnets:** `PrivateAppSubnetB`

### RTB-PrivateDB

| **Destination** | **Target** | **Purpose** |
|----|----|----|
| `10.0.0.0/16` | `local` | VPC internal traffic |

**Associated subnets:** `PrivateDBSubnetA`, `PrivateDBSubnetB`

### RTB-VPC-Default (main)

| **Destination** | **Target** | **Purpose** |
|----|----|----|
| `10.0.0.0/16` | `local` | VPC internal traffic |

**Associated subnets:** none. Every subnet is explicitly associated with one of the tables above, so a new subnet starts without internet access.

**Notes**
- Each private app subnet routes to the NAT Gateway in its **own AZ**, so an AZ failure does not take down egress for the other AZ.
- Data subnets do not have a default route for Internet traffic, which ensures that the database has no path to or from the internet.

## 3. Security groups

Traffic chain: `Internet → alb-sg → ecs-sg → rds-sg`

### `alb-sg` (Application Load Balancer)

| Direction | Protocol | Port | Source / Destination | Description |
|---|---|---|---|---|
| Inbound | TCP | 443 | `0.0.0.0/0` | HTTPS from the internet |
| Outbound | TCP | 80 | `ecs-sg` | Forward traffic and health checks to ECS tasks |

### `ecs-sg` (ECS Fargate tasks)

| Direction | Protocol | Port | Source / Destination | Description |
|---|---|---|---|---|
| Inbound | TCP | 80 | `alb-sg` | Traffic from the ALB only |
| Outbound | TCP | 5432 | `rds-sg` | PostgreSQL connection to the RDS endpoint |
| Outbound | TCP | 443 | `0.0.0.0/0` | Egress via NAT Gateway: ECR and CloudWatch Logs |

### `rds-sg` (RDS PostgreSQL)

| Direction | Protocol | Port | Source / Destination | Description |
|---|---|---|---|---|
| Inbound | TCP | 5432 | `ecs-sg` | PostgreSQL from ECS tasks only |
| Outbound | None | None | None | No outbound rules required |

**Notes**
- Source and destination between the 3-tier resources reference **security groups, not CIDR block**, so they keep working even if their IPs change.
  
- No security group exposes SSH or any admin port, and the RDS instance is not publicly accessible.

- Default NACLs are kept; security groups are the enforcement layer.
