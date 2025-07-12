
# Apt-Cacher NG on Proxmox: Speed Up VM Package Downloads

<img width="720" height="960" alt="image" src="https://github.com/user-attachments/assets/ef41a405-9b99-48e0-9e12-2e923945d6e0" />

## Overview

Apt-Cacher NG is a lightweight caching proxy that stores Debian/Ubuntu packages locally, speeding up package installations for multiple VMs on your Proxmox server. The first VM downloads packages from the internet, while subsequent VMs use the fast local cache, saving 80-90% of bandwidth and reducing setup times. This guide uses the Proxmox web GUI wherever possible, minimizing command-line work for beginners.

## Quick Start

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/bokiko/apt-cacher-ng.git
   cd apt-cacher-ng
   ```
2. **Download Debian Template** (in Proxmox Shell):
   ```bash
   pveam update
   pveam download local debian-12-standard_12.2-1_amd64.tar.zst
   ```
3. **Create LXC Container** (via Proxmox GUI):
   - Hostname: `apt-cache-server`
   - Template: `debian-12-standard`
   - Disk: `12 GiB`
   - CPU: `2 cores`
   - Memory: `1024 MiB`
   - Network: `vmbr0`, DHCP
4. **Install Apt-Cacher NG** (in container console):
   ```bash
   apt update && apt upgrade -y
   apt install apt-cacher-ng -y
   ```
5. **Configure Clients** (in each VM):
   ```bash
   echo 'Acquire::http::Proxy "http://<container_ip>:3142";' | sudo tee /etc/apt/apt.conf.d/01proxy
   apt update
   ```
6. **Verify**: Access `http://<container_ip>:3142/acng-report.html` in a browser.

## Prerequisites

- Proxmox VE 7.0+ with web GUI access (`https://<proxmox_ip>:8006`)
- Internet connectivity on the Proxmox host
- Basic familiarity with Proxmox GUI and shell
- (Optional) `nano` or another text editor installed in the container

## Setup Instructions

### Step 1: Prepare the LXC Template

1. **Open Proxmox Shell**: In the web GUI, select your node → **Shell**.
2. **Update template list**:
   ```bash
   pveam update
   ```
3. **Download Debian 12 template**:
   ```bash
   pveam download local debian-12-standard_12.2-1_amd64.tar.zst
   ```
   > **Note**: If the template version is unavailable, run `pveam available | grep debian` to list options and adjust the filename.
4. **Close the shell** once the download completes.

### Step 2: Create the Apt-Cacher NG Container

1. **Navigate**: In the Proxmox GUI, select your node → **Create CT**.
2. **General**:
   - **Hostname**: `apt-cache-server`
   - **Password**: Set a strong root password (save it)
   - **VMID**: Auto-assigned or choose (e.g., 200)
3. **Template**: Select `debian-12-standard`.
4. **Disks**:
   - **Storage**: `local-lvm`
   - **Size**: `12 GiB` (for cache storage)
5. **CPU**: `2 cores`.
6. **Memory**:
   - **Memory**: `1024 MiB`
   - **Swap**: `1024 MiB`
7. **Network**:
   - **Bridge**: `vmbr0`
   - **IPv4**: `DHCP` (note the assigned IP later)
8. **DNS**: Use host settings.
9. **Confirm**: Review and click **Finish**.
10. **Start**: Right-click the container → **Start**.

### Step 3: Install and Configure Apt-Cacher NG

1. **Access Console**: Select the container → **Console** → Login as `root` with your password.
2. **Update and install**:
   ```bash
   apt update && apt upgrade -y
   apt install apt-cacher-ng nano -y
   ```
3. **Edit configuration**:
   ```bash
   nano /etc/apt-cacher-ng/acng.conf
   ```
   Ensure these settings:
   ```bash
   Port: 3142
   BindAddress: 0.0.0.0
   PassThroughPattern: ^(.*):443$
   ```
   > **Note**: `PassThroughPattern` bypasses HTTPS repositories, which aren’t cached. See "Handling HTTPS Repositories" below.
4. **Save**: `Ctrl+O` → `Enter` → `Ctrl+X`.
5. **Restart service**:
   ```bash
   systemctl restart apt-cacher-ng
   systemctl enable apt-cacher-ng
   ```
6. **Get container IP**:
   ```bash
   ip addr show | grep inet
   ```
   Note the IP (e.g., `192.168.x.x`) for client configuration.
7. **Verify service**:
   ```bash
   systemctl status apt-cacher-ng
   ```
   Should show "active (running)".

### Step 4: Configure Client VMs

For each Debian/Ubuntu VM:

1. **Access VM Console**: Start the VM → **Console** → Login.
2. **Switch to root**:
   ```bash
   sudo -i
   ```
3. **Set proxy**:
   ```bash
   nano /etc/apt/apt.conf.d/01proxy
   ```
   Add (replace `<container_ip>` with the IP from Step 3):
   ```bash
   Acquire::http::Proxy "http://<container_ip>:3142";
   ```
   > **Note**: HTTPS repositories bypass the cache. See "Handling HTTPS Repositories" for workarounds.
4. **Save**: `Ctrl+O` → `Enter` → `Ctrl+X`.
5. **Test**:
   ```bash
   apt update
   ```
   Should complete without errors.

### Step 5: Verify Caching

1. **Install a package** on one VM:
   ```bash
   apt install htop -y
   ```
2. **Repeat on another VM**:
   ```bash
   apt install htop -y
   ```
   The second installation should be faster, indicating the cache is working.
3. **Check cache stats**: Open `http://<container_ip>:3142/acng-report.html` in a browser to view hit/miss ratios and cached packages.

## Handling HTTPS Repositories

Apt-Cacher NG does not cache HTTPS traffic (e.g., Docker or modern Ubuntu repositories). Options:

- **Convert to HTTP**: Edit `/etc/apt/sources.list` or `/etc/apt/sources.list.d/*.list` to use `http` (e.g., `deb http://archive.ubuntu.com/ubuntu ...`). Ensure GPG keys are installed for security:
  ```bash
  apt-key adv --keyserver keyserver.ubuntu.com --recv-keys <key_id>
  ```
  > **Warning**: HTTP is less secure; use only in trusted networks.
- **Use a Reverse Proxy**: Set up Nginx/Apache to proxy HTTPS to HTTP. See [this guide](https://jgillula.github.io) for details.
- **Disable HTTPS Proxy**: If HTTPS is required, skip proxying for HTTPS in `/etc/apt/apt.conf.d/01proxy`:
  ```bash
  Acquire::https::Proxy "false";
  ```

## Security Recommendations

- **Change Default Credentials**: If enabling the web interface admin, edit `/etc/apt-cacher-ng/security.conf`:
  ```ini
  AdminAuth: your_username:your_secure_password
  ```
- **Restrict Access**: Limit port `3142` to your local network:
  ```bash
  ufw allow from 192.168.0.0/16 to any port 3142
  ufw deny 3142
  ```
- **Network Isolation**: Place the container on a private Proxmox network segment.

## Cache Management

- **Monitor Usage**: Check cache stats at `http://<container_ip>:3142/acng-report.html`.
- **Clean Old Packages**:
  ```bash
  /usr/lib/apt-cacher-ng/expire-caller.pl
  ```
  Or enable auto-cleanup in `/etc/apt-cacher-ng/acng.conf`:
  ```bash
  ExThreshold: 7
  ```
- **Check Disk Space**:
  ```bash
  df -h /var/cache/apt-cacher-ng
  ```

## Troubleshooting

- **Connection Refused**:
  - Check firewall: `ufw status` → Allow port `3142` if needed: `ufw allow 3142`.
  - Verify service: `systemctl status apt-cacher-ng`.
- **Slow Downloads**: Initial downloads hit the internet; subsequent ones use the cache.
- **Cache Not Working**:
  - Confirm IP: `ip addr show` in container.
  - Check VM config: `cat /etc/apt/apt.conf.d/01proxy`.
  - View logs: `journalctl -u apt-cacher-ng -f`.
- **Storage Full**:
  - Clean cache: `/usr/lib/apt-cacher-ng/expire-caller.pl`.
  - Expand disk via Proxmox GUI.

## Advanced Configuration

- **Support Other Distros**:
  - For CentOS/Fedora, edit `/etc/yum.conf` or `/etc/dnf/dnf.conf`:
    ```ini
    proxy=http://<container_ip>:3142
    ```
- **Dynamic Proxy Discovery**: Use Avahi for zero-config proxy detection:
  ```bash
  apt install avahi-daemon -y
  echo "apt-cacher-ng.<domain>" >> /etc/avahi/aliases
  systemctl restart avahi-daemon
  ```
- **Cloud-Init Integration**: Automate VM proxy setup with cloud-init:
  ```yaml
  write_files:
    - path: /etc/apt/apt.conf.d/01proxy
      content: |
        Acquire::http::Proxy "http://<container_ip>:3142";
  ```

## References

- [Official Apt-Cacher NG Documentation](https://www.unix-ag.uni-kl.de/~bloch/acng/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [Apt-Cacher NG Source](https://salsa.debian.org/blade/apt-cacher-ng)

## Conclusion

Your Apt-Cacher NG setup now accelerates package installations across Proxmox VMs, saving bandwidth and time. Configure additional VMs, monitor cache stats, and explore advanced options like cloud-init for automation. For further details, consult the official documentation or open an issue on this repository.

*Last updated: July 2025*

