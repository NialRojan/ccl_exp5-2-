# Lab 5 — Assignment 2: DynamoDB (NoSQL) CRUD Demo

## Overview
Assignment 2 requires implementing the **opposite database type** to Assignment 1. Since Assignment 1 used Amazon RDS (SQL), this assignment uses **Amazon DynamoDB (NoSQL)**. A new, lightweight Node.js/Express application was deployed alongside the existing Mealie application on the same EC2 instance, demonstrating full CRUD against DynamoDB with at least 5 distinct attribute types.

## Architecture

```
Client → Caddy (EC2, :80)
           ├── "/"        → Mealie container (127.0.0.1:9000) → Amazon RDS PostgreSQL
           └── "/dynamo*" → Node.js CRUD app (127.0.0.1:3000) → Amazon DynamoDB
```

Both applications run on the same EC2 instance (`mealie-lab4`, i-0074ba1ec324ef939) and share the same public IP and Caddy reverse proxy, routed by path rather than port, so no new firewall rules were required.

## Security: IAM Role (No Hardcoded Keys)

Per the mandatory security rule, DynamoDB access uses an IAM role attached to the EC2 instance rather than hardcoded AWS access keys anywhere in the code or configuration.

| Setting | Value |
|---|---|
| Role name | mealie-ec2-dynamodb-role |
| Attached policy | AmazonDynamoDBFullAccess |
| Attached to | EC2 instance mealie-lab4 |
| Credentials in code | None — AWS SDK v3 automatically retrieves temporary credentials via the EC2 Instance Metadata Service (IMDSv2) |

Verified via:
```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
# mealie-ec2-dynamodb-role
```

## DynamoDB Table

| Setting | Value |
|---|---|
| Table name | mealie-dynamo-demo |
| Partition key | id (String) |
| Sort key | none |
| Capacity mode | On-demand |

## Datatype Proof (Mandatory Rule: 5+ Datatypes)

Each item written to the table includes the following attributes, covering 5 distinct DynamoDB datatypes:

| Attribute | DynamoDB Type | Example |
|---|---|---|
| id | String (S) | "item1" |
| name | String (S) | "Test Widget" |
| quantity | Number (N) | 10 |
| inStock | Boolean (BOOL) | true |
| tags | List (L) | ["demo", "test"] |
| metadata | Map (M) | {"note": "first item"} |

## Application

A minimal Node.js/Express app (`/srv/dynamo-demo/app.js`) using the AWS SDK v3 (`@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`) exposes a simple HTML form and REST API:

| Method | Route | Operation |
|---|---|---|
| GET | /dynamo/ | HTML UI |
| POST | /dynamo/api/items | Create |
| GET | /dynamo/api/items/:id | Read (single) |
| GET | /dynamo/api/items | Read (all / scan) |
| PATCH | /dynamo/api/items/:id | Update (increments quantity) |
| DELETE | /dynamo/api/items/:id | Delete |

Runs as a systemd service (`dynamo-demo.service`), enabled for automatic startup on boot, consistent with the rest of the stack (Caddy, PostgreSQL, Docker/Mealie).

## CRUD Demonstration

All four operations were demonstrated live through the running application at `http://<public-ip>/dynamo`:

- **Create** — added item with id `item1`, all 5 datatypes populated
- **Read** — retrieved `item1` by ID, confirmed all attributes present
- **Update** — incremented `quantity` from 10 to 11
- **Delete** — removed `item1`, confirmed via subsequent scan

## Access

- **EC2 App URL**: http://\<current-public-ip\>/dynamo (IP auto-updates on instance restart via systemd automation)
- **DynamoDB Table**: mealie-dynamo-demo (private, accessed only via the EC2 instance's IAM role — no public endpoint)
