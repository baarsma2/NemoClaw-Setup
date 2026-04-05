# Phase 1: Security Hardening

## Overview
Transition the EC2 instance from default SSH on port 22 to a hardened configuration.
This is the highest-risk phase — an error here can lock you out of the instance permanently.

## Prerequisites
- Active SSH session on port 22
- AWS Management Console access to modify Security Groups
- Know your public IP address (use `curl ifconfig.me`)

## Steps

### 1.1 Install UFW (do NOT enable yet)
```bash
sudo apt update && sudo apt install -y ufw
sudo ufw allow 2222/tcp    # new SSH port
sudo ufw allow 80/tcp      # web gateway (if needed)
sudo ufw allow 443/tcp     # HTTPS (if needed)
```
**WARNING:** Do NOT run `sudo ufw enable` yet. Enabling before SSH is moved will lock you out.

### 1.2 Disable systemd ssh.socket
Modern Ubuntu uses systemd socket activation which overrides sshd_config port settings.
This is the #1 cause of "I changed the port but it's still on 22" issues.

```bash
sudo systemctl disable --now ssh.socket
sudo systemctl stop ssh.socket
```

Verify it's gone:
```bash
systemctl status ssh.socket   # should show "inactive (dead)"
```

### 1.3 Configure sshd_config
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

sudo sed -i 's/^#\?Port .*/Port 2222/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config
```

Verify the changes:
```bash
grep -E '^(Port|PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config
```
Expected output:
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
```

### 1.4 Restart SSH service
```bash
sudo systemctl restart ssh
```

### 1.5 Verify from a SECOND terminal (critical!)
Open a new terminal window and test:
```bash
ssh -i <key.pem> -p 2222 ubuntu@<public-ip>
```

**Only proceed if this succeeds.** If it fails, you still have your original session to fix things.

### 1.6 Enable UFW
Only after confirming SSH on 2222 works:
```bash
sudo ufw enable
sudo ufw status verbose
```

### 1.7 Update AWS Security Group
In the AWS Console:
1. Navigate to EC2 → Security Groups → your instance's SG
2. Remove the inbound rule for port 22
3. Add: Custom TCP, Port 2222, Source: `<your-ip>/32`
4. Ensure ports 80 and 443 are open if web gateway is needed

### 1.8 Verify old port is closed
```bash
sudo ss -tlnp | grep :22    # should show nothing
sudo ss -tlnp | grep :2222  # should show sshd
```

## VSCode SSH Config Update
If using VSCode Remote SSH, update `~/.ssh/config`:
```
Host nemoclaw-ec2
    HostName <public-ip-or-elastic-ip>
    User ubuntu
    Port 2222
    IdentityFile ~/.ssh/<key.pem>
```

## Rollback Plan
If locked out:
1. Use AWS EC2 Instance Connect (if enabled) or EC2 Serial Console
2. Or stop the instance, detach the root volume, attach to another instance,
   fix sshd_config, reattach, and restart
