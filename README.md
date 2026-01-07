# Ubuntu Message of the Day (MOTD)

A collection of customizable 'Message of the Day' scripts for Ubuntu/Debian systems, designed for automated deployment with Ansible.

![motd](motd.png)

## Features

- **Colored Hostname Banner**: Customizable hostname display with optional prefix/suffix stripping and figlet/lolcat rendering
- **System Information**: Shows OS, kernel, uptime, CPU/memory usage, IP address, and optional network zone classification
- **Disk Space Monitoring**: Visual progress bars for disk usage with color-coded alerts
- **Service Status**: Multi-column display of systemd services with emoji icons and color-coded status
- **Additional Scripts**: Uptime, ZFS pool status, disk temperature, Fail2ban stats, Docker/LXD containers
- **Environment-Based Configuration**: Centralized configuration via `/etc/environment` variables

## Deployment with Ansible (Recommended)

The included `ansible-playbook` automates the complete setup. This is the recommended deployment method.

### What the Playbook Does

1. Installs dependencies (figlet, lolcat)
2. Clones the repository to `/opt/ubuntu-motd`
3. Deploys selected MOTD scripts to `/etc/update-motd.d/`
4. Disables default Ubuntu MOTD snippets (keeps files, removes executable bit)
5. Disables Ubuntu motd-news
6. Configures environment variables in `/etc/environment`

### Basic Usage

```bash
ansible-playbook -i inventory.ini ansible-playbook --limit your-server
```

## Customizing Your MOTD

### 1. Choose Which Scripts to Deploy

Edit the `ansible-playbook` file and modify the **"Deploy MOTD scripts"** task to select which scripts you want:

```yaml
- name: Deploy MOTD scripts
  copy:
    src: "/opt/ubuntu-motd/{{ item }}"
    dest: "/etc/update-motd.d/{{ item }}"
    owner: root
    group: root
    mode: "0755"
    remote_src: yes
  loop:
    - 10-hostname-color    # Colored hostname banner (requires figlet/lolcat)
    # - 10-hostname        # Simple hostname (alternative to above)
    - 20-sysinfo          # System info with network zone detection
    # - 20-uptime         # Basic uptime only (alternative to 20-sysinfo)
    - 35-diskspace        # Disk usage with bars
    # - 36-diskstatus     # Disk temperature (requires hddtemp)
    # - 30-zpool-bar      # ZFS pool status with bars (requires zpool)
    # - 30-zpool-simple   # ZFS pool status without bars
    - 40-services         # Service status monitor
    # - 50-fail2ban       # Fail2ban statistics
    # - 50-fail2ban-status # Fail2ban status summary
    # - 60-docker         # Docker containers
    # - 60-lxd            # LXD/LXC containers
  when: ansible_os_family == "Debian"
```

**Tip**: Uncomment the scripts you want, comment out those you don't need.

### 2. Configure Hostname Display (`10-hostname-color`)

Adapt the hostname stripping to match your naming convention:

```yaml
# Example 1: Strip "vm-" prefix from hostnames like "vm-webserver01"
- { key: "MOTD_HOST_PREFIX_STRIP", value: "vm-" }

# Example 2: Strip "-prod" suffix from hostnames like "webserver-prod"
- { key: "MOTD_HOST_SUFFIX_STRIP", value: "-prod" }

# Example 3: Override hostname completely (useful for aliases)
- { key: "MOTD_HOST_OVERRIDE", value: "Production Web Server" }
```

**Common scenarios**:
- **Cloud VMs with prefixes**: `vm-`, `aws-`, `azure-`
- **Environment suffixes**: `-prod`, `-staging`, `-dev`
- **Location codes**: `-nyc`, `-lon`, `-par`

### 3. Configure Network Zones (`20-sysinfo`)

Classify your IPs into zones (DMZ, LAN, etc.) based on subnet patterns:

```yaml
# Example 1: DMZ on 10.10.0.0/16, LAN on 10.20.0.0/16
- { key: "MOTD_ZONE_DMZ_REGEX", value: "^10\\.10\\.[0-9]{1,3}\\." }
- { key: "MOTD_ZONE_LAN_REGEX", value: "^10\\.20\\.[0-9]{1,3}\\." }

# Example 2: DMZ on 172.16.0.0/12, LAN on 192.168.0.0/16
- { key: "MOTD_ZONE_DMZ_REGEX", value: "^172\\.(1[6-9]|2[0-9]|3[01])\\.[0-9]{1,3}\\." }
- { key: "MOTD_ZONE_LAN_REGEX", value: "^192\\.168\\.[0-9]{1,3}\\." }

# Example 3: Add a third zone for management network
- { key: "MOTD_ZONE_DMZ_REGEX", value: "^10\\.10\\.[0-9]{1,3}\\." }
- { key: "MOTD_ZONE_LAN_REGEX", value: "^10\\.20\\.[0-9]{1,3}\\." }
- { key: "MOTD_ZONE_OTHER_LABEL", value: "MGMT" }  # Will match 10.30.x.x as MGMT
```

**Tip**: Use [regex101.com](https://regex101.com/) to test your patterns against your IP addresses.

### 4. Configure Service Monitoring (`40-services`)

Specify which systemd services to monitor, their display labels, and emoji icons:

```yaml
# Format: "service_name:display_label:emoji"
# Multiple services separated by spaces

# Example 1: Basic server services
- { key: "MOTD_SERVICES", value: "ssh:ssh:🔐 ufw:firewall:🚧 fail2ban:fail2ban:🛡️" }

# Example 2: Web server stack
- { key: "MOTD_SERVICES", value: "ssh:ssh:🔐 nginx:nginx:🌐 php8.1-fpm:php-fpm:⚡ mysql:mysql:🗄️ redis:redis:💾" }

# Example 3: Docker host
- { key: "MOTD_SERVICES", value: "ssh:ssh:🔐 docker:docker:🐳 containerd:containerd:📦 ufw:firewall:🚧" }

# Example 4: Complete monitoring setup
- { key: "MOTD_SERVICES", value: "ssh:ssh:🔐 fail2ban:fail2ban:🛡️ ufw:ufw:🚧 docker:docker:🐳 nginx:nginx:🌐 apache2:apache2:📦 postgresql:postgres:🐘 prometheus:prometheus:📊" }
```

**Common service names**:
- Web: `nginx`, `apache2`, `lighttpd`, `caddy`
- Databases: `mysql`, `mariadb`, `postgresql`, `mongodb`, `redis`
- PHP: `php7.4-fpm`, `php8.1-fpm`, `php8.2-fpm`
- Containers: `docker`, `containerd`, `lxd`, `lxcfs`
- Security: `ssh`, `fail2ban`, `ufw`, `iptables`
- Monitoring: `prometheus`, `grafana`, `node_exporter`, `telegraf`

**Emoji reference**: 🔐🛡️🚧🐳🌐📦🗄️💾⚡🐘📊🔥💻⚙️🚀📡🔧

### 5. Disable Unwanted Default Ubuntu MOTD Snippets

The playbook already disables common Ubuntu MOTD scripts. To add more, edit the **"Stat default Ubuntu MOTD snippets"** task:

```yaml
- name: Stat default Ubuntu MOTD snippets
  stat:
    path: "{{ item }}"
  register: ubuntu_motd_snippets
  loop:
    - /etc/update-motd.d/10-help-text
    - /etc/update-motd.d/50-motd-news
    - /etc/update-motd.d/50-landscape-sysinfo
    - /etc/update-motd.d/80-livepatch
    - /etc/update-motd.d/80-esm
    - /etc/update-motd.d/90-updates-available
    - /etc/update-motd.d/91-release-upgrade
    - /etc/update-motd.d/92-unattended-upgrades
    - /etc/update-motd.d/95-hwe-eol
    - /etc/update-motd.d/98-fsck-at-reboot
    - /etc/update-motd.d/98-reboot-required
    # Add any other scripts you want to disable
```

**Tip**: These files are not deleted, only made non-executable. You can re-enable them with `chmod +x`.

## Complete Configuration Example

Here's a typical configuration for a production web server in a DMZ:

```yaml
# In ansible-playbook, modify the environment variables section:
- name: Configure MOTD env vars in /etc/environment
  lineinfile:
    path: /etc/environment
    regexp: "^{{ item.key }}="
    line: '{{ item.key }}="{{ item.value }}"'
    create: yes
  loop:
    # Strip "prod-" prefix from hostnames
    - { key: "MOTD_HOST_PREFIX_STRIP", value: "prod-" }
    
    # Network zones: DMZ = 10.10.x.x, LAN = 10.20.x.x
    - { key: "MOTD_ZONE_DMZ_REGEX", value: "^10\\.10\\.[0-9]{1,3}\\." }
    - { key: "MOTD_ZONE_LAN_REGEX", value: "^10\\.20\\.[0-9]{1,3}\\." }
    
    # Monitor web stack services
    - { key: "MOTD_SERVICES", value: "ssh:ssh:🔐 fail2ban:fail2ban:🛡️ ufw:firewall:🚧 nginx:nginx:🌐 php8.1-fpm:php:⚡ mysql:mysql:🗄️ redis:redis:💾" }
```

## Manual Setup (Alternative)

If you prefer manual installation without Ansible:

1. Install dependencies: `sudo apt update && sudo apt install figlet lolcat`
2. Copy desired scripts to `/etc/update-motd.d/`
3. Make them executable: `sudo chmod +x /etc/update-motd.d/<script>`
4. Configure environment variables in `/etc/environment`
5. Disable unwanted default scripts: `sudo chmod -x /etc/update-motd.d/<script>`
6. Ensure `PrintMotd yes` in `/etc/ssh/sshd_config`

## Available Scripts

### Essential Scripts
- **`10-hostname-color`**: Displays hostname with figlet/lolcat (or fallback)
- **`10-hostname`**: Simple hostname display without color/figlet
- **`20-sysinfo`**: Comprehensive system information summary
- **`20-uptime`**: Basic uptime display
- **`35-diskspace`**: Disk usage with visual progress bars
- **`40-services`**: Systemd service status monitor

### Storage Scripts
- **`30-zpool-bar`**: ZFS pool status with usage bars
- **`30-zpool-simple`**: ZFS pool status (no bars)
- **`36-diskstatus`**: Disk temperature via hddtemp
- **`36-smartd`**: SMART self-test results (requires smartd)

### Security & Container Scripts
- **`50-fail2ban`**: Fail2ban jail statistics
- **`50-fail2ban-status`**: Fail2ban status summary
- **`60-docker`**: Docker container listing
- **`60-lxd`**: LXD/LXC container listing

## Special Notes

### Disk Status (`36-smartd`)
This script parses syslog for smartd entries. You must:
- Enable smartd monitoring
- Configure regular self-tests in `/etc/smartd.conf`

### Fail2ban Scripts (`50-fail2ban*`)
Comment out the `compress` option in `/etc/logrotate.d/fail2ban` to prevent log compression, allowing the scripts to grep uncompressed logs:
```bash
# compress
```

### ZFS Scripts (`30-zpool-*`)
Choose one variant:
- **`30-zpool-bar`**: Full details with colored usage bars
- **`30-zpool-simple`**: Compact display without bars

## Troubleshooting

### Scripts Not Executing
- Ensure scripts have executable permissions: `sudo chmod +x /etc/update-motd.d/<script>`
- Check for errors: `run-parts --test /etc/update-motd.d/`

### Environment Variables Not Loaded
- PAM should load `/etc/environment` automatically
- For non-PAM sessions, scripts source `/etc/environment` directly
- Logout and login after modifying `/etc/environment`

### Missing Dependencies
- Install missing packages: `sudo apt install figlet lolcat hddtemp`
- Scripts gracefully degrade if optional dependencies are unavailable

## Contributing

Contributions are welcome! Please ensure:
- Scripts are POSIX-compliant shell (sh) or bash when necessary
- Include error handling and graceful degradation
- Document any new environment variables
- Test on Ubuntu 20.04+ and Debian 11+

## License

This project is open source. Check individual script headers for specific licensing information.
 Reference

| Script | Description | Dependencies | Config Variables |
|--------|-------------|--------------|------------------|
| **`10-hostname-color`** | Colored hostname banner with figlet/lolcat | figlet, lolcat | `MOTD_HOST_PREFIX_STRIP`<br>`MOTD_HOST_SUFFIX_STRIP`<br>`MOTD_HOST_OVERRIDE` |
| **`10-hostname`** | Simple hostname display | none | Same as above |
| **`20-sysinfo`** | Full system info + network zone | none | `MOTD_ZONE_DMZ_REGEX`<br>`MOTD_ZONE_LAN_REGEX`<br>`MOTD_ZONE_OTHER_LABEL` |
| **`20-uptime`** | Basic uptime only | none | none |
| **`35-diskspace`** | Disk usage with colored bars | none | none |
| **`36-diskstatus`** | Disk temperatures | hddtemp | none |
| **`36-smartd`** | SMART test results | smartd | none |
| **`30-zpool-bar`** | ZFS pools with usage bars | zpool | none |
| **`30-zpool-simple`** | ZFS pools (compact) | zpool | none |
| **`40-services`** | Systemd service status | systemctl | `MOTD_SERVICES` |
| **`50-fail2ban`** | Fail2ban jail stats | fail2ban | none |
| **`50-fail2ban-status`** | Fail2ban status | fail2ban | none |
| **`60-docker`** | Docker containers | docker | none |
| **`60-lxd`** | LXD/LXC containers | lxc | none |Additional Configuration Tips

### For `36-smartd` (SMART monitoring)
1. Enable smartd: `sudo systemctl enable --now smartd`
2. Configure monitoring in `/etc/smartd.conf`:
   ```
   /dev/sda -a -o on -S on -s (S/../.././02|L/../../6/03)
   ```
3. The script parses syslog for smartd temperature and test results

### For `50-fail2ban` scripts
Disable log compression to allow grep access:
```bash
# Edit /etc/logrotate.d/fail2ban and comment out:
# compress
```

### For ZFS scripts (`30-zpool-*`)
- Use `30-zpool-bar` for detailed view with colored usage bars
- Use `30-zpool-simple` for compact output
- Deploy only one variant to avoid duplicationesting Your Configuration

After deploying with Ansible, test the MOTD without logging out:

```bash
# Test all scripts
run-parts /etc/update-motd.d/

# Test specific script
/etc/update-motd.d/20-sysinfo

# Check for errors
run-parts --test /etc/update-motd.d/
```

## Troubleshooting

**Environment variables not appearing**:
- Variables are loaded on login, logout and reconnect after playbook run
- Verify variables exist: `grep MOTD /etc/environment`
- Manually source for testing: `set -a; . /etc/environment; set +a`

**Scripts not executing**:
- Check permissions: `ls -la /etc/update-motd.d/`
- Verify executable bit: `sudo chmod +x /etc/update-motd.d/<script>`
- Test manually: `run-parts --test /etc/update-motd.d/`

**Service list not showing**:
- Verify `MOTD_SERVICES` format: `service_name:label:emoji`
- Check systemd service names: `systemctl list-units --type=service`
- Test with simple services first: `ssh:ssh:🔐`

**Network zone not detected**:
- Test your regex: `echo "10.10.5.23" | grep -E "^10\.10\.[0-9]{1,3}\."`
- Check actual IP: `hostname -I`
- Escape dots in regex: `\.` not `.`

## Contributing

When contributing new scripts:
- Use POSIX-compliant shell (`#!/bin/sh`) when possible
- Add environment variable documentation
- Include graceful fallbacks for missing dependencies
- Update the ansible-playbook with new configuration options