# Part 2: The App Server

## Overview

- **What:** A Flask music app running on a private app server, accessed through a jump server.
- **Services:** EC2, EBS, snapshots, NAT Gateway, IAM role, and S3.
- **Result:** Songs play in the browser through an SSH tunnel, and the data disk has a snapshot backup.

## 1. Goal

Run the Flask music app on a server in the private subnet. The server should be reachable only through a jump server, with an extra data disk and a snapshot backup.

## 2. What I Built

### Jump server: `jump-server`

- **OS:** Amazon Linux 2023
- **Instance type:** `t3.micro`
- **Network:** Public subnet with a public IP
- **Security group:** `public-sg`
- **Purpose:** The entry point into the private network.

### App server: `app-server-1`

- **OS / instance type:** Same type as the jump server
- **Network:** Private subnet, with no public IP
- **Security group:** `private-sg`
- **IAM role:** `MiniSpotifyEC2Role`
- **Purpose:** Runs the app and stays hidden from the internet. The IAM role lets it read from S3 without storing access keys on the server.

### Port 5000 rule

- `private-sg` allows **Custom TCP port 5000** from `public-sg`.
- Flask listens on port 5000, and only the jump server's security group is allowed to reach it through this rule.

### Extra EBS disk

- **Size:** 8 GiB
- **Volume type:** `gp3`
- **Attached as:** `/dev/sdb` in the EC2 console
- **Mounted at:** `/data`
- **Purpose:** Keeps data separate from the operating system disk.

### Snapshot

- Created a snapshot of the data disk.
- **Purpose:** Provides a restore point if the disk is damaged or deleted.

### NAT Gateway

- Recreated the NAT Gateway.
- Created a new private route table pointing to it.
- **Purpose:** Allows the private app server to download Flask and other libraries.

## 3. SSH Configuration

Edit the SSH config file at `~/.ssh/config` and replace the placeholders with the actual IP addresses.

```sshconfig
Host jump-server
    HostName <JUMP-SERVER-PUBLIC-IP>
    User ec2-user
    IdentityFile ~/.ssh/mini-spotify-key.pem

Host app-server-1
    HostName <APP-SERVER-PRIVATE-IP>
    User ec2-user
    IdentityFile ~/.ssh/mini-spotify-key.pem
    ProxyJump jump-server
```

**Important:** The jump server's public IP can change after stopping and starting it. The private IP does not change just because the instance is stopped and started.

## 4. Setting Up the App Server

Run these commands after connecting to `app-server-1`.

### Step 1: Check internet access

```bash
ping -c 3 google.com
```

Sends three test packets. Replies indicate that the server can reach the internet through the NAT setup.

### Step 2: Install pip

```bash
sudo dnf install -y python3-pip
```

Installs pip, which is needed to install Python libraries. The `-y` option automatically answers yes to prompts.

### Step 3: Create the project directory

```bash
mkdir ~/music-app && cd ~/music-app
```

Creates the project folder and moves into it.

### Step 4: Create and activate a virtual environment

```bash
python3 -m venv venv
source venv/bin/activate
```

The virtual environment keeps this project's Python libraries separate from the system's libraries. Activate it before installing packages.

### Step 5: Install the app's libraries

```bash
pip install flask boto3 gunicorn
```

- **Flask:** Serves the app's pages.
- **boto3:** Lets Python communicate with AWS services such as S3.
- **Gunicorn:** Runs the Flask app.

### Step 6: Create the templates directory

```bash
mkdir templates
```

Flask looks in this directory for HTML templates.

## 5. Copying the App From My Laptop

Run these commands in PowerShell on your laptop, from the directory containing `app.py` and the `templates` folder.

```powershell
scp app.py app-server-1:/home/ec2-user/music-app/app.py
scp templates\index.html app-server-1:/home/ec2-user/music-app/templates/index.html
```

`scp` copies files over SSH. It uses the SSH config, so the connection to `app-server-1` goes through `jump-server` automatically because of `ProxyJump`.

## 6. Running and Testing the App

### Step 1: Set the S3 bucket name

```bash
export BUCKET_NAME=<bucket>
```

Sets the bucket name as an environment variable so it does not need to be written directly into the app's code. Replace `<bucket>` with the actual bucket name.

### Step 2: Set the AWS Region

```bash
export AWS_REGION=ap-south-1
```

Sets the AWS Region used by the app. S3 buckets are associated with a Region.

### Step 3: Start Gunicorn

```bash
gunicorn -b 0.0.0.0:5000 app:app
```

- `-b 0.0.0.0:5000`: Binds the app to port 5000 on all network interfaces.
- `app:app`: Uses the Flask object named `app` in `app.py`.

### Step 4: Test the songs API from the jump server

Run this command on `jump-server`, replacing the placeholder with the app server's private IP.

```bash
curl http://<APP-PRIVATE-IP>:5000/api/songs
```

Requests the song list as text. If the songs are printed, the app can return its song list; with the configured IAM role and S3 bucket, this also helps test the app's S3 integration.

### Step 5: Open an SSH tunnel from the laptop

Run this command in a terminal on your laptop. Replace the placeholder with the app server's private IP.

```bash
ssh -L 8080:<APP-PRIVATE-IP>:5000 jump-server
```

This forwards your laptop's port `8080` to port `5000` on the private app server through the jump server. Your browser cannot connect directly to the private server, so the tunnel provides a path to the app.

### Step 6: Open the app in the browser

Visit:

```text
http://localhost:8080
```

The browser connects to the local end of the SSH tunnel, allowing you to use the music player and play songs.

## 7. The Extra EBS Disk

> **Warning:** Formatting a disk erases its existing data. Confirm the device name carefully and never format the root disk.

### Step 1: Find the new disk

```bash
lsblk
```

Lists disks and partitions. In this setup, the new 8 GiB disk appeared as `nvme1n1` and had nothing mounted under it.

### Step 2: Format the disk

```bash
sudo mkfs -t xfs /dev/nvme1n1
```

Creates an XFS filesystem on the new disk.

### Step 3: Create the mount point

```bash
sudo mkdir /data
```

Creates the directory where the disk will be mounted.

### Step 4: Mount the disk

```bash
sudo mount /dev/nvme1n1 /data
```

Makes the disk accessible through `/data`. Linux accesses mounted disks through directories rather than drive letters.

### Step 5: Give the user ownership

```bash
sudo chown ec2-user:ec2-user /data
```

Gives `ec2-user` ownership of the mount directory so the user can write to it.

### Step 6: Verify the disk space

```bash
df -h
```

Displays filesystem space. The `/data` mount should show approximately 8 GiB.

### Step 7: Create a snapshot

In the AWS Console:

1. Open **EC2**.
2. Select **Volumes**.
3. Select the data volume attached as `/dev/sdb` in the console.
4. Choose **Actions → Create snapshot**.

The volume did not have a Snapshot ID before the snapshot was created. A snapshot is a point-in-time backup that can be used to restore the volume's data.

## 8. Problems I Fixed

1. **`no such host`** — The SSH config still used the old server nicknames.
2. **SSH timeout** — The security-group rule had an outdated IP. For the learning phase, SSH was opened to anywhere. In a real setup, restrict SSH to a fixed office IP or use a VPN.
3. **`route already exists`** — An old private route table contained a stale route. Creating a new private route table fixed it.
4. **`venv/bin/activate: No such file`** — The virtual environment had not been created on that server.
5. **`Address already in use`** — An old Gunicorn process was still running. `pkill gunicorn` stopped it.
6. **`mount point does not exist`** — The `/data` directory had not been created yet.

## 9. Cleanup

- Stopped Gunicorn and the SSH tunnel.
- Deleted the NAT Gateway.
- Released its Elastic IP.
- Stopped both EC2 instances.