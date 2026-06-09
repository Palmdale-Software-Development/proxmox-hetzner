# Proxmox VE Post-Installation, Optimization, and Troubleshooting Guide

This guide covers system updates, removing the subscription nag banner, optimizing network connection tracking (conntrack), configuring ZFS ARC limits, and basic troubleshooting commands.

---

## Phase 1: System Updates & Essential Tools

Run these commands immediately after a fresh Proxmox installation to ensure the system is up-to-date and equipped with necessary utilities.

```bash
# Update package repositories, upgrade system packages, remove unused dependencies, and update Proxmox templates
apt update && apt -y upgrade && apt -y autoremove && pveupgrade && pveam update

# Install essential tools for administration, networking, and storage management
apt install -y curl libguestfs-tools unzip iptables-persistent net-tools conntrack

```

---

## Phase 2: Proxmox Subscription Nag Removal

This script modifies the Proxmox web UI JavaScript file to disable the "No valid subscription" dialog box that appears upon logging in.

```bash
# Disable the subscription warning popup and restart the web proxy service to apply changes
sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service

```

---

## Phase 3: Kernel Modules & System Optimization (Sysctl & ZFS)

Optimizing the connection tracking table prevents network drops under high loads. Additionally, setting ZFS ARC limits prevents ZFS from consuming all available host RAM.

```bash
# Enable the connection tracking kernel module on boot
echo "nf_conntrack" >> /etc/modules

# Increase the maximum number of tracked connections and optimize TCP established timeout
echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.d/99-proxmox.conf
echo "net.netfilter.nf_conntrack_tcp_timeout_established=28800" >> /etc/sysctl.d/99-proxmox.conf

# Remove legacy ZFS configuration if it exists
rm -f /etc/modprobe.d/zfs.conf

# Set ZFS ARC memory limits (Min: 6GB, Max: 12GB) to protect host RAM
echo "options zfs zfs_arc_min=$[6 * 1024*1024*1024]" >> /etc/modprobe.d/99-zfs.conf
echo "options zfs zfs_arc_max=$[12 * 1024*1024*1024]" >> /etc/modprobe.d/99-zfs.conf

# Update initramfs to apply the new kernel module and ZFS settings on the next boot
update-initramfs -u

```

---

## Phase 4: Security Hardening

Disable unnecessary legacy network services to reduce the attack surface and free up system resources.

```bash
# Check the status of RPC bind services
systemctl status rpcbind rpcbind.socket

# Disable and immediately stop rpcbind services (rarely used in modern PVE setups unless using NFS)
systemctl disable --now rpcbind rpcbind.socket

```

---

## Phase 5: Verification & Diagnostics

Use these commands to verify that your optimizations have been applied successfully.

```bash
# Verify connection tracking (conntrack) limits in sysctl
sysctl -a | grep net.netfilter.nf_conntrack_max
sysctl -a | grep nf_conntrack_tcp_timeout_established

# Check the active ZFS ARC limits currently in use by the kernel
cat /sys/module/zfs/parameters/zfs_arc_min
cat /sys/module/zfs/parameters/zfs_arc_max

```

---

## Phase 6: Network Troubleshooting & Conntrack Monitoring

If you experience network instability or suspect connection drops, use these commands to analyze the connection tracking table.

### 1. Monitor Active Connection Counts

```bash
# Method 1: Check active connection count via sysctl
sysctl net.netfilter.nf_conntrack_count

# Method 2: Read active connection count directly from the proc filesystem
cat /proc/sys/net/netfilter/nf_conntrack_count

# Method 3: Use the conntrack utility to get the current count
conntrack -C

```

### 2. Analyze Traffic and Debug Table Full Errors

```bash
# View the entire table of currently tracked active network connections
conntrack -L

# Find the top 10 Source IP addresses with the most open connections
conntrack -L 2>/dev/null | awk '{print $7}' | cut -d "=" -f 2 | sort | uniq -c | sort -nr | head -n 10

# Find the top 10 Destination Ports receiving the most connections
conntrack -L 2>/dev/null | awk '{print $8}' | cut -d "=" -f 2 | sort | uniq -c | sort -nr | head -n 10

# Check kernel ring buffer (dmesg) for conntrack table full errors
dmesg | grep -i "nf_conntrack: table full"

# Search systemd journal logs for conntrack table full errors
journalctl -k | grep -i "nf_conntrack: table full"

```
