# Part 1 — AWS Network

## 1. Overview

**Project:** Mini Spotify  
**Purpose:** Build an AWS network with public and private subnets, then test that it works.

### What I built
- A VPC with public and private subnets.
- An internet gateway for internet access to the public side.
- A NAT gateway so private servers can make outbound internet connections.
- Route tables and security groups to control traffic.
- A private server that I can access through a public jump server.

### Final result
The private server can be reached through the public jump server. It can also access the internet outbound through the NAT gateway.

---

## 2. Network resources

### VPC

- **Name:** `mini-spotify-vpc`
- **CIDR:** `10.0.0.0/16`
- **Address count:** 65,536

A VPC is my own private network area in AWS. I control how resources inside it communicate.

### Subnets

I created four subnets:

**Public subnets**
- `public-subnet-a` — `10.0.1.0/24`
- `public-subnet-b` — `10.0.2.0/24`

These are for resources that need to be reachable from the internet.

**Private subnets**
- `private-subnet-a` — `10.0.11.0/24`
- `private-subnet-b` — `10.0.12.0/24`

These are for resources such as application servers and databases that should not be directly reachable from the internet.

The subnets are spread across two Availability Zones so the application can be more resilient if one zone has a problem.

### Internet gateway

- **Name:** `mini-spotify-igw`
- Created and attached to the VPC.

The internet gateway connects the VPC to the internet. Creating one is not enough; it must also be attached to the VPC.

### Public route table

- **Name:** `public-rt`
- **Route:** `0.0.0.0/0` → Internet gateway
- **Associated with:** Both public subnets

The route sends traffic for destinations outside the VPC to the internet gateway. This route is what allows the public subnets to have internet routing.

### Private route table

- Associated with both private subnets.
- Initially, it had no route to the internet.
- Later, I added a route through the NAT gateway.

Without an internet route, the private subnets do not have a path to the internet.

### NAT gateway

- Created in a public subnet.
- Assigned an Elastic IP.
- Added as the target for the private route table's internet route.

The NAT gateway allows instances in private subnets to start outbound connections, such as downloading updates. It does not allow unsolicited inbound connections from the internet.

### Security groups

**`public-sg`**
- Allows SSH on port `22` from my IP address only.
- Controls who can log in to the public server.

**`private-sg`**
- Allows SSH from `public-sg`.
- Allows the private server to accept SSH connections through the public server rather than directly from the internet.

---

## 3. Key concepts to remember

### CIDR ranges

- A `/16` range contains `2^(32-16) = 65,536` IPv4 addresses.
- A `/24` range contains `256` IPv4 addresses.

### Public subnet vs. private subnet

A subnet is considered public when its route table has a route to an internet gateway. A private subnet does not have a direct route to the internet gateway.

### Internet gateway vs. NAT gateway

- **Internet gateway:** Provides the VPC's connection to the internet for appropriately configured resources.
- **NAT gateway:** Lets private-subnet resources initiate outbound internet connections without allowing unsolicited inbound connections.

### Security group as a source

A security group rule can refer to another security group instead of a fixed IP address. For example, `private-sg` can allow SSH from `public-sg`. This avoids depending on the public server's changing private IP address.

---

## 4. Set up the SSH key on Windows

Run these commands in **PowerShell**.

### Step 1 — Create the SSH folder

```powershell
mkdir $env:USERPROFILE\.ssh -Force
```

This creates the `.ssh` folder in my Windows user profile.

### Step 2 — Move the key into the folder

```powershell
Move-Item "$env:USERPROFILE\Downloads\mini-spotify-key.pem" "$env:USERPROFILE\.ssh\mini-spotify-key.pem"
```

This moves the key from Downloads into the `.ssh` folder, keeping it in one place outside the project folder.

### Step 3 — Remove inherited permissions

```powershell
icacls "$env:USERPROFILE\.ssh\mini-spotify-key.pem" /inheritance:r
```

This stops the key from inheriting permissions from its parent folder.

### Step 4 — Give my Windows user read permission

```powershell
icacls "$env:USERPROFILE\.ssh\mini-spotify-key.pem" /grant:r "$($env:USERNAME):(R)"
```

This grants my Windows user read permission for the key.

---

## 5. Configure SSH

Open this file in VS Code:

`~/.ssh/config`

Add the following configuration. Replace the placeholders with the actual server IP addresses.

```sshconfig
Host public-test
  HostName <PUBLIC-IP>
  User ec2-user
  IdentityFile ~/.ssh/mini-spotify-key.pem

Host private-test
  HostName <PRIVATE-IP>
  User ec2-user
  IdentityFile ~/.ssh/mini-spotify-key.pem
  ProxyJump public-test
```

### What this configuration does

- `public-test` is the shortcut for connecting to the public server.
- `private-test` is the shortcut for connecting to the private server.
- `IdentityFile` tells SSH which private key to use.
- `ProxyJump public-test` tells SSH to connect through the public server before reaching the private server.

The private server has no public IP address, so I connect to it through the public jump server.

---

## 6. Tests I ran

### Test 1 — Connect to the public server

**Command**

```bash
ssh public-test
```

**Result:** Connected.

**What it proved:** The public server's SSH access, public route table, internet gateway path, and security-group rule were configured sufficiently for the connection.

### Test 2 — Check internet access from the public server

**Command**

```bash
ping -c 3 google.com
```

**Result:** `0% packet loss`.

**What it proved:** The public server could reach the destination used for the test.

### Test 3 — Connect to the private server

**Command**

```bash
ssh private-test
```

**Result:** Connected through the public server.

**What it proved:** SSH through the jump server worked.

### Test 4 — Check private-server internet access before NAT

**Command**

```bash
ping -c 3 -W 2 google.com
```

**Result:** Failed, as expected.

**What it proved:** Before the NAT route was added, the private server did not have a working internet path.

### Test 5 — Check private-server internet access after NAT

**Command**

```bash
ping -c 3 google.com
```

**Result:** Replies received.

**What it proved:** The private server could reach the internet through the NAT gateway.

---

## 7. Problems I fixed

### Problem 1 — `Could not resolve hostname`

**Cause:** The SSH config still contained placeholder text instead of a real IP address.

**Fix:** Replace `<PUBLIC-IP>` and `<PRIVATE-IP>` with the correct IP addresses.

### Problem 2 — SSH timeout

**Cause:** My home IP address had changed, so the `public-sg` SSH rule no longer matched it.

**Fix:** I checked the client address visible to AWS using:

```bash
echo $SSH_CLIENT
```

Then I updated the security-group rule to use the appropriate current IP address.

---

## 8. Cleanup

After testing, I:
- Deleted the NAT gateway.
- Released its Elastic IP.
- Terminated the test servers.

I did this because NAT gateways and allocated Elastic IP addresses can incur charges.