# Mini Spotify on AWS

A music streaming platform built stage by stage on AWS, to learn cloud and DevOps by building something real.
Users will sign up, log in, and play songs from a private S3 bucket, served by an app running on hidden servers.
The platform grows from one server into a secure, load-balanced, monitored system that uses most of the core AWS services.
At the end, the whole infrastructure is rebuilt with Terraform and deployed automatically from GitHub with CI/CD.

## Progress

| Stage | What it covers | Status | Notes |
|---|---|---|---|
| Part 1: Network | VPC, public and private subnets, gateways, route tables, security groups | Done | [part-1-network.md](part-1-network.md) |
| Part 2: App server | Jump server, private app server, extra EBS disk, snapshot, Flask app | Done | [part-2-app-server.md](part-2-app-server.md) |
| Part 3: S3 endpoint | Private road from the VPC to S3, with no internet or NAT needed | Done | [part-3-s3-endpoint.md](part-3-s3-endpoint.md) |
| Part 4: Database | RDS MySQL, Secrets Manager, KMS, sign up and login | In progress | |
| Part 5: Scaling | Load balancer, Auto Scaling | Planned | |
| Part 6: Serverless | SQS, Lambda, DynamoDB, SNS, EventBridge | Planned | |
| Part 7: Speed and domain | CloudFront, Route 53 | Planned | |
| Part 8: Monitoring | CloudWatch, CloudTrail | Planned | |
| Part 9: DevOps layer | Git, Terraform, GitHub Actions CI/CD | Planned | |

## Folders and files

| Name | What is inside |
|---|---|
| `part-1-network.md` | Notes for Part 1: VPC, subnets, gateways, route tables |
| `part-2-app-server.md` | Notes for Part 2: jump server, app server, disk and snapshot |
| `part-3-s3-endpoint.md` | Notes for Part 3: VPC endpoint for S3 |
| `screenshots/` | Proof screenshots for all parts |

Each part file explains what I built, why, the commands I used, and the problems I fixed.

## What I have learned so far

- **IAM:** users with MFA, least-privilege policies, roles for EC2 instead of access keys
- **S3:** private buckets, Block Public Access, encryption, presigned URLs
- **EC2:** instances, key pairs, security groups, EBS volumes and snapshots
- **VPC:** CIDR ranges, public and private subnets, internet gateway, NAT gateway, route tables, VPC endpoints
- **Linux:** SSH, jump hosts, SSH tunnels, mounting disks, running a Flask app with gunicorn

## Security notes

- No access keys, passwords, or `.pem` files are stored in this repository.
- Account IDs and server IPs are hidden or removed from screenshots.
- Servers, the NAT gateway, and the database are stopped or deleted after each session to avoid charges.

## Status

This is a learning project and it is not finished. I update this repository after each part.
