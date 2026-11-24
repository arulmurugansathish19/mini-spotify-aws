# Part 3: Private Access to S3 with a VPC Endpoint

## Overview

- **What:** Create a private path from the VPC to Amazon S3, so the app server can read songs without internet access.
- **AWS services:** VPC gateway endpoint, route tables, IAM role, and S3.
- **Result:** The app server lists songs from S3 even after the NAT gateway is deleted.

## 1. Goal

Allow the private app server to read songs from S3 without using the internet or a NAT gateway.

## 2. The Problem

Before this part, the app server accessed S3 through the NAT gateway and the internet.

This costs money, and the traffic leaves the private network.

## 3. What I Built

### S3 VPC Gateway Endpoint

- **Service name:** `com.amazonaws.ap-south-1.s3`
- **Endpoint type:** Gateway
- **Attached to:** The private route table
- **Purpose:** Provides a private path from the VPC to S3 through AWS's network.

### Route in the Private Route Table

- **Destination:** `pl-...` — S3's prefix list, which represents S3 IP address ranges.
- **Target:** `vpce-...` — the VPC endpoint.
- **Purpose:** Sends S3 traffic to the endpoint instead of through the NAT gateway.

## 4. How It Works

1. Create an S3 gateway endpoint in the VPC.
2. AWS adds a route to the private route table so traffic for S3 goes to the endpoint.
3. When the app server requests the song list, the request follows that private route.
4. S3 checks the IAM role's permissions before returning the requested data.

**Remember:**
- The VPC endpoint is the **road** to S3.
- The IAM role provides the **permission** to access S3.
- The app needs both.

## 5. Tests I Ran

### Test 1: Confirm That Internet Access Is Unavailable

Run this command on the app server:

```bash
ping -c 3 -W 2 google.com
```

- **Result:** Failed, as expected.
- **What it proves:** The app server has no general internet access after the NAT gateway is removed.

### Test 2: Retrieve the Song List Without NAT

Run this command from the jump server:

```bash
curl http://<APP-PRIVATE-IP>:5000/api/songs
```

- **Result:** The song list printed in the terminal.
- **What it proves:** The app server can reach S3 through the VPC endpoint.

### Test 3: Play Songs in the Browser

1. Create the SSH tunnel to port `8080`.
2. Open `http://localhost:8080` in the browser.

- **Result:** Songs played.
- **What it proves:** The complete setup still works.

## 6. Which Network Path Does Each Request Use?

- **Browser to the app server:** An SSH tunnel through the jump server.
- **App server to S3 for the song list:** The S3 VPC endpoint.
- **Song playback in the browser:** S3 sends the song to the browser through a presigned URL.
- **NAT gateway:** Not used.

## 7. Why I Did Not Restrict the Bucket to the Endpoint Only

A bucket policy that allows access only through the VPC endpoint would also block my browser from playing songs.

The browser accesses S3 through my normal internet connection using the presigned URL. That request does not go through the VPC endpoint.

The bucket is still protected by:

- S3 Block Public Access.
- The IAM role.
- Presigned URLs.

## 8. Interview Answer

**Question: How can a private server access S3 without using the internet?**

**Answer:**

I use an S3 VPC gateway endpoint. It adds a route to the private route table so traffic to S3 stays within AWS's network. The private server does not need a public IP address or a NAT gateway to access S3 through this endpoint.

**Important:** The endpoint only provides access to S3. Downloading software from the public internet, such as running `pip install`, still requires a suitable internet path, such as a NAT gateway.

## 9. Cleanup

- Stopped Gunicorn and the SSH tunnel.
- Stopped both EC2 instances.
- Left the VPC endpoint in place.
