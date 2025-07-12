## Setting Up apt-cacher-ng on Proxmox for Faster VM Package Downloads


## Overview

Running multiple Ubuntu/Debian VMs on Proxmox? Tired of waiting for repeated package downloads during VM setups? apt-cacher-ng is your solution—a lightweight caching proxy that stores packages locally. The first VM downloads from the internet, while subsequent VMs pull from your fast local network, dramatically reducing setup times and bandwidth usage.

This guide maximizes use of the Proxmox web GUI with minimal command-line work, perfect for beginners.

## Prerequisites

- Proxmox VE 7.0+ installed and accessible
- Internet connectivity on your Proxmox host  
- Access to Proxmox web interface (`https://your-proxmox-ip:8006`)
- Basic familiarity with the Proxmox GUI

## Part 1: Preparing the LXC Template

Since template downloads aren't available through the GUI, we'll use a quick shell command:

### Step 1: Download Debian Template
1. **Access Proxmox Shell**: In the web GUI, select your node → **Shell**
2. **Update template list**:
   ```bash
   pveam update
   ```
3. **Download Debian 12 template**:
   ```bash
   pveam download local debian-12-standard_12.2-1_amd64.tar.zst
   ```
   > **Note**: If this version doesn't exist, run `pveam available | grep debian` to see available versions and adjust the filename accordingly.

4. **Close the shell** when download completes

## Part 2: Creating the apt-cacher-ng Container

### Step 2: Container Creation via Web GUI
1. **Navigate**: Select your Proxmox node → Click **Create CT** (blue button)

2. **General Configuration**:
   - **Hostname**: `apt-cache-server`
   - **Password**: Create a strong root password (save this!)
   - **VMID**: Use auto-assigned or choose (e.g., 200)
   - Click **Next**

3. **Template Selection**: 
   - Choose the `debian-12-standard` template you downloaded
   - Click **Next**

4. **Storage Configuration**:
   - **Storage**: `local-lvm` (default)
   - **Disk size**: `12 GiB` (increased for cache storage)
   - Click **Next**

5. **CPU Configuration**:
   - **Cores**: `2` (better performance for multiple VMs)
   - Click **Next**

6. **Memory Configuration**:
   - **Memory**: `1024 MiB` (1GB for better caching)
   - **Swap**: `1024 MiB`
   - Click **Next**

7. **Network Configuration**:
   - **Bridge**: `vmbr0` (default)
   - **IPv4**: `DHCP` (we'll note the assigned IP)
   - Click **Next**

8. **DNS Configuration**: 
   - Select **Use host settings**
   - Click **Next**

9. **Confirmation**: 
   - Review settings and click **Finish**

10. **Start Container**: Right-click the new container → **Start**

## Part 3: Installing and Configuring apt-cacher-ng

### Step 3: Install apt-cacher-ng
1. **Access Container Console**: Select your container → **Console**
2. **Login**: Username `root` + the password you set
3. **Update package list**:
   ```bash
   apt update && apt upgrade -y
   ```
4. **Install apt-cacher-ng**:
   ```bash
   apt install apt-cacher-ng nano -y
   ```

### Step 4: Configure apt-cacher-ng
1. **Edit configuration**:
   ```bash
   nano /etc/apt-cacher-ng/acng.conf
   ```
2. **Find and modify these lines** (uncomment by removing `#`):
   ```bash
   # Change this line:
   # Port:3142
   # To:
   Port: 3142
   
   # Ensure this line exists:
   BindAddress: 0.0.0.0
   ```
3. **Save**: `Ctrl+O` → `Enter` → `Ctrl+X`

4. **Restart service**:
   ```bash
   systemctl restart apt-cacher-ng
   systemctl enable apt-cacher-ng
   ```

5. **Get container IP address**:
   ```bash
   ip addr show | grep inet
   ```
   > **Important**: Note the IP address (usually starts with 192.168.x.x or 10.x.x.x) - you'll need this for VM configuration.

6. **Test service**:
   ```bash
   systemctl status apt-cacher-ng
   ```
   Should show "active (running)"

## Part 4: Configuring VMs to Use the Cache

### Step 5: Configure Each VM (Repeat for All VMs)

1. **Access VM Console**: Start your VM → **Console**
2. **Login** to your VM with your user account
3. **Switch to root**:
   ```bash
   sudo -i
   ```
4. **Create proxy configuration**:
   ```bash
   nano /etc/apt/apt.conf.d/01proxy
   ```
5. **Add configuration** (replace `YOUR_CONTAINER_IP` with actual IP from Step 4):
   ```bash
   Acquire::http::Proxy "http://YOUR_CONTAINER_IP:3142";
   Acquire::https::Proxy "http://YOUR_CONTAINER_IP:3142";
   ```
6. **Save**: `Ctrl+O` → `Enter` → `Ctrl+X`

7. **Test configuration**:
   ```bash
   apt update
   ```
   Should complete without errors

### Step 6: Verify Cache is Working
1. **Install a test package** on first VM:
   ```bash
   apt install htop -y
   ```
2. **Install same package** on second VM:
   ```bash
   apt install htop -y
   ```
   Should be significantly faster the second time!

## Management and Monitoring

### Accessing Cache Statistics
- **Web Interface**: Open browser to `http://YOUR_CONTAINER_IP:3142/acng-report.html`
- **View cached packages**: Browse cache contents and statistics
- **Monitor usage**: Check hit/miss ratios and bandwidth savings

### Useful Commands

#### In the apt-cacher-ng Container:
```bash
# Check service status
systemctl status apt-cacher-ng

# View logs
journalctl -u apt-cacher-ng -f

# Check disk usage
df -h /var/cache/apt-cacher-ng

# Restart service
systemctl restart apt-cacher-ng
```

#### In Client VMs:
```bash
# Force cache refresh
apt update

# Remove proxy configuration (if needed)
rm /etc/apt/apt.conf.d/01proxy

# Test without cache
apt update
```

## Troubleshooting

### Common Issues and Solutions

**Connection Refused Errors**:
```bash
# In container, check firewall
ufw status
# If active, allow port 3142
ufw allow 3142
```

**Slow Initial Downloads**:
- This is normal - first download goes through internet
- Subsequent downloads from cache will be much faster

**Cache Not Working**:
1. Verify container IP hasn't changed: `ip addr show`
2. Check proxy config in VMs: `cat /etc/apt/apt.conf.d/01proxy`
3. Ensure service is running: `systemctl status apt-cacher-ng`

**Storage Issues**:
```bash
# Clean old cached packages
/usr/lib/apt-cacher-ng/expire-caller.pl
```

### Performance Tips

- **Monitor disk space**: Cache grows over time in `/var/cache/apt-cacher-ng`
- **Regular cleanup**: Enable automatic cleanup in acng.conf
- **Network placement**: Ensure container is on same network segment as VMs for best performance

## Advanced Configuration

### Automatic Cache Cleanup
Edit `/etc/apt-cacher-ng/acng.conf`:
```bash
# Enable automatic cleanup (uncomment)
ExTreshold: 4
```

### Multiple Repository Support
apt-cacher-ng automatically handles:
- Ubuntu repositories
- Debian repositories  
- Security updates
- Universe/multiverse packages

## Conclusion

You now have a fully functional apt-cacher-ng setup that will significantly speed up package installations across your Proxmox VMs. The cache will grow over time, providing even better performance as more packages are cached locally.

**Next Steps**:
- Configure all your existing VMs to use the cache
- Monitor cache statistics via the web interface
- Consider automating VM configuration with cloud-init templates

**Bandwidth Savings**: Expect 80-90% reduction in internet bandwidth for repeated package installations!

---
*This guide focuses on GUI-first approach with minimal CLI commands. For advanced configurations, consult the official apt-cacher-ng documentation.*
