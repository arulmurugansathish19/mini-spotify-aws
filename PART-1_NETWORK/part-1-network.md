# Part 1: The network

- **What:** A private network with public and private subnets.
- **Services:** VPC, internet gateway, NAT gateway, route tables, security groups.
- **Result:** A private server that is reachable only through a jump server, with out-only internet access through the NAT gateway.

## Goal
Build a private network for Mini Spotify in AWS, with a public side (reachable from the internet)
and a private side (hidden), then prove it works.

## What I built

| Component | Details | Why |
|---|---|---|
| VPC `mini-spotify-vpc` | `10.0.0.0/16` (65,536 addresses) | My own private area in AWS, so I control who can reach what |
| 4 subnets | `public-subnet-a/b` (`10.0.1.0/24`, `10.0.2.0/24`), `private-subnet-a/b` (`10.0.11.0/24`, `10.0.12.0/24`) | Public ones hold things the internet must reach. Private ones hold app servers and the database. Two zones keep the app running if one data center fails. |
| Internet gateway `mini-spotify-igw` | Created, then attached to the VPC | The main gate between the VPC and the internet. A gateway that is not attached does nothing. |
| `public-rt` | Route `0.0.0.0/0` goes to the internet gateway, linked to the 2 public subnets | A subnet is public only because its route table points to the internet gateway |
| Private route table | Linked to the 2 private subnets, with no internet route at first | With no route to the internet, the subnets stay private |
| NAT gateway | Created in a public subnet with an Elastic IP, then added to the private route table | Lets private servers go out to the internet (for downloads) while nothing can come in |
| `public-sg` | SSH (port 22) from my IP only | Only I can log in to the public server |
| `private-sg` | SSH from `public-sg` only | The private server accepts logins only from the public server |

## Key ideas
- **CIDR:** `/16` means 16 bits are fixed, so 2^(32-16) = 65,536 addresses. A `/24` gives 256.
- **Public vs private subnet:** the only difference is the route table. A route to an internet gateway makes it public.
- **Internet gateway vs NAT gateway:** the internet gateway is two-way. The NAT gateway is out-only.
- **Security group source:** a rule can allow another security group instead of an IP address, so it keeps working when addresses change.

## Setting up SSH from Windows (PowerShell)

```powershell
mkdir $env:USERPROFILE\.ssh -Force
Move-Item "$env:USERPROFILE\Downloads\mini-spotify-key.pem" "$env:USERPROFILE\.ssh\mini-spotify-key.pem"
icacls "$env:USERPROFILE\.ssh\mini-spotify-key.pem" /inheritance:r
icacls "$env:USERPROFILE\.ssh\mini-spotify-key.pem" /grant:r "$($env:USERNAME):(R)"
```

| Command | Meaning | Why |
|---|---|---|
| `mkdir ... .ssh -Force` | Make the `.ssh` folder | SSH looks there for keys |
| `Move-Item` | Move the key into it | One safe place, away from the project folder |
| `icacls ... /inheritance:r` | Remove permissions copied from the folder | Start with nobody having access |
| `icacls ... /grant:r "user:(R)"` | Give only my user read permission | SSH refuses a key that others could read |

SSH config file (`~/.ssh/config`), with placeholders for the addresses:

```
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

`ProxyJump` sends the login through the public server, because the private server has no public address.

## Tests I ran

| Test | Command | Result | What it proved |
|---|---|---|---|
| Log in to the public server | `ssh public-test` | Connected | Internet gateway, public route table, and `public-sg` work |
| Public server reaches the internet | `ping -c 3 google.com` | 0% packet loss | The route to the internet gateway works |
| Log in to the private server | `ssh private-test` | Connected through the public server | The private server is reachable only that way |
| Private server without NAT | `ping -c 3 -W 2 google.com` | Failed (expected) | Private means no route to the internet |
| Private server with NAT | `ping -c 3 google.com` | Replies | The NAT gateway gives out-only access |

## Problems I fixed
1. **`Could not resolve hostname`:** the config file still had placeholder text instead of a real IP.
2. **SSH timeout:** my home IP changed, so the firewall rule did not match. I found the IP AWS sees with `echo $SSH_CLIENT` and used it.

## Cleanup
Deleted the NAT gateway, released its Elastic IP, and terminated the test servers, because the NAT gateway and idle Elastic IPs charge by the hour.

## Screenshots
**VPC resource map**
![VPC resource map](screenshots/p1-01-vpc-resource-map.png)

**Public route table**
![Public route table](screenshots/p1-02-public-rt.png)

**Private route table**
![Private route table](screenshots/p1-03-private-rt.png)

**Private server reaching the internet through the NAT**
![Private ping](screenshots/p1-04-private-ping.png)
