# Ansible Role: Logrotate

![Molecule Tests](https://github.com/CoffeeSprout/ansible-role-logrotate/actions/workflows/molecule.yml/badge.svg)

Manage logrotate configuration with improved defaults to prevent disk space issues on managed VMs. This role addresses the common problem of log files consuming excessive disk space by enabling compression by default and implementing more aggressive rotation policies.

## The Problem

**Default logrotate configurations waste disk space:**
- Compression is **disabled** by default on most distributions
- Logs rotate only **weekly** (too infrequent for busy systems)
- Keep only **4 rotations** (insufficient history)
- No size-based rotation (logs can grow unbounded between rotations)

**Real-world impact:** A production AlmaLinux 9 system was found with:
- 13GB in `/var/log` (87% of which was uncompressed)
- 137MB active `sftp.log` file (not yet rotated)
- Multiple 100MB+ uncompressed rotated logs

## The Solution

This role provides **sane defaults** that save 50-70% disk space:

| Setting | Distribution Default | Our Default | Impact |
|---------|---------------------|-------------|---------|
| **Compression** | Disabled | Enabled (gzip) | 50-70% space savings |
| **Rotation** | Weekly | Daily | Prevents large files |
| **Rotations kept** | 4 | 12 | Better history (12 days) |
| **Date extension** | Mixed | Always enabled | Better file naming |
| **Size threshold** | None | 100MB | Catches runaway logs |
| **Delay compression** | No | Yes | Safe for active services |

## Requirements

- Ansible 2.12 or higher
- Systemd-based Linux distribution

## Supported Platforms

- **Debian 12** (Bookworm)
- **Debian 13** (Trixie)
- **AlmaLinux 8**
- **AlmaLinux 9**
- **Rocky Linux 8**
- **Rocky Linux 9**

## Role Variables

### Global Configuration

```yaml
# Manage /etc/logrotate.conf (set false to only manage custom configs)
logrotate_manage_global_config: true

# Rotation interval: daily, weekly, monthly
# Default changed from "weekly" (dist default) to "daily"
logrotate_interval: "daily"

# Number of rotations to keep (changed from 4 default)
logrotate_rotate_count: 12

# Enable date-based file naming (YYYYMMDD)
# Debian default: disabled, our default: enabled
logrotate_dateext: true

# Create new log files after rotation
logrotate_create: true
```

### Compression Settings

```yaml
# Enable compression (changed from disabled default)
# Saves 50-70% disk space!
logrotate_compress: true

# Compression command: "gzip" (universal) or "zstd" (better)
logrotate_compress_command: "gzip"

# Compression options (e.g., "-9" for maximum compression)
logrotate_compress_options: ""

# Delay compression by one cycle (recommended for active services)
logrotate_delaycompress: true

# Install zstd package (required if using zstd compression)
logrotate_install_zstd: false
```

### Size-Based Rotation

```yaml
# Rotate logs when they exceed this size (NEW FEATURE)
# Catches runaway logs before they fill the disk
logrotate_size: "100M"

# Hard limit - force rotation even if interval hasn't passed
logrotate_maxsize: ""
```

### Custom Configurations

```yaml
# List of custom logrotate configurations
# Each creates /etc/logrotate.d/managed-{name}.conf
logrotate_custom_configs: []
```

## Compression Options

### gzip (Default - Universal Compatibility)

**Pros:**
- Pre-installed on all systems
- Universal compatibility
- ~50-60% compression ratio

**Cons:**
- Slower than zstd
- Lower compression ratio

**Usage:**
```yaml
logrotate_compress_command: "gzip"
logrotate_compress_options: "-9"  # Maximum compression
```

### zstd (Better Compression & Speed)

**Pros:**
- Better compression (~60-75% space savings)
- Faster compression and decompression
- Lower CPU usage

**Cons:**
- Requires installation
- Newer (may not be on all systems)

**Usage:**
```yaml
logrotate_install_zstd: true
logrotate_compress_command: "zstd"
logrotate_compress_options: "-19"  # Maximum compression
```

**Comparison:**
```
100MB log file compressed:
- gzip:  ~45MB (55% savings, ~3 seconds)
- zstd:  ~35MB (65% savings, ~1 second)
```

## Custom Configuration Examples

### Large Web Server Logs

```yaml
logrotate_custom_configs:
  - name: virtualmin-large
    paths:
      - /var/log/virtualmin/*_access_log
      - /var/log/virtualmin/*_error_log
    rotate: 30
    interval: daily
    size: 100M
    compress: true
    delaycompress: true
    sharedscripts: true
    postrotate: "systemctl reload httpd"
```

### High-Volume SFTP Logs

```yaml
logrotate_custom_configs:
  - name: proftpd-sftp
    paths:
      - /var/log/proftpd/sftp.log
    rotate: 14
    interval: daily
    size: 50M
    compress: true
    delaycompress: true
    postrotate: "systemctl reload proftpd"
```

### Application Logs with Custom Permissions

```yaml
logrotate_custom_configs:
  - name: myapp
    paths:
      - /var/log/myapp/*.log
    rotate: 7
    interval: daily
    maxsize: 200M
    compress: true
    create: "0640 appuser appgroup"
    postrotate: "systemctl reload myapp"
```

### Multiple Paths, Different Settings

```yaml
logrotate_custom_configs:
  - name: nginx-access
    paths:
      - /var/log/nginx/access.log
    rotate: 90  # Keep 3 months
    interval: daily
    size: 500M
    compress: true

  - name: nginx-error
    paths:
      - /var/log/nginx/error.log
    rotate: 365  # Keep 1 year
    interval: daily
    compress: true
    # No size limit - errors are important
```

## Example Playbooks

### Basic Usage (Recommended for Most Systems)

```yaml
- hosts: all_servers
  become: true
  roles:
    - role: coffeesprout.logrotate
```

**Result:**
- Daily rotation
- Keep 12 days of logs
- gzip compression enabled
- Rotate if logs exceed 100MB

### Production Servers (More Retention)

```yaml
- hosts: production
  become: true
  roles:
    - role: coffeesprout.logrotate
      vars:
        logrotate_rotate_count: 30  # Keep 30 days
        logrotate_size: "200M"       # Larger size threshold
```

### With zstd Compression (Better Space Savings)

```yaml
- hosts: high_volume_logging
  become: true
  roles:
    - role: coffeesprout.logrotate
      vars:
        logrotate_install_zstd: true
        logrotate_compress_command: "zstd"
        logrotate_compress_options: "-10"
```

### With Custom Configurations

```yaml
- hosts: web_servers
  become: true
  roles:
    - role: coffeesprout.logrotate
      vars:
        logrotate_custom_configs:
          - name: apache-vhosts
            paths:
              - /var/log/httpd/*_access_log
              - /var/log/httpd/*_error_log
            rotate: 14
            interval: daily
            size: 100M
            compress: true
            sharedscripts: true
            postrotate: "systemctl reload httpd"
```

### Only Manage Custom Configs (Don't Touch Global Config)

```yaml
- hosts: special_servers
  become: true
  roles:
    - role: coffeesprout.logrotate
      vars:
        logrotate_manage_global_config: false  # Don't touch /etc/logrotate.conf
        logrotate_custom_configs:
          - name: application
            paths:
              - /var/log/application/*.log
            rotate: 7
            interval: daily
            compress: true
```

## Safe Custom Configuration Management

### The "managed-" Prefix System

This role uses a **prefix-based system** to safely manage custom configurations without interfering with application-managed configs:

**Managed by this role:**
- `/etc/logrotate.d/managed-myapp.conf` ✅
- `/etc/logrotate.d/managed-apache.conf` ✅

**NOT touched by this role:**
- `/etc/logrotate.d/apache2` ✅ (application-managed)
- `/etc/logrotate.d/mysql` ✅ (package-managed)
- `/etc/logrotate.d/rsyslog` ✅ (system-managed)

**Why this matters:**
- Many packages (Apache, MySQL, PHP-FPM) install their own logrotate configs
- This role will **never** modify files without the `managed-` prefix
- You can safely use this role alongside package-managed configurations

**Example directory after role execution:**
```
/etc/logrotate.d/
├── apache2                          # Package-managed (untouched)
├── mysql                            # Package-managed (untouched)
├── php-fpm                          # Package-managed (untouched)
├── managed-virtualmin.conf          # Managed by Ansible ✓
└── managed-custom-app.conf          # Managed by Ansible ✓
```

## Documentation Generation

This role automatically generates documentation for each server:

```yaml
# Enable documentation generation (default: true)
doc_generate: true
doc_output_dir: "{{ playbook_dir }}/documentation"
doc_format: "both"  # markdown, json, or both
```

**Generated files:**
- `documentation/{hostname}/logrotate.md` - Human-readable markdown
- `documentation/{hostname}/logrotate.json` - Machine-readable JSON

**Documentation includes:**
- Current rotation settings
- Compression configuration
- Custom configurations
- Management commands
- Troubleshooting guide

## Operational Details

### File Locations

**Configuration:**
- Global config: `/etc/logrotate.conf`
- Drop-in configs: `/etc/logrotate.d/`
- Managed configs: `/etc/logrotate.d/managed-*.conf`
- Original backup: `/etc/logrotate.conf.original`

**Runtime:**
- State file: `/var/lib/logrotate/logrotate.status`
- Cron job: `/etc/cron.daily/logrotate`

### How Logrotate Works

**Automatic Execution:**
- Runs daily via `/etc/cron.daily/logrotate`
- Typical time: 6:25 AM (varies by distribution)
- Reads state from `/var/lib/logrotate/logrotate.status`

**State Tracking:**
```bash
# View last rotation times
cat /var/lib/logrotate/logrotate.status
```

### Manual Operations

**Dry-Run (Show What Would Happen):**
```bash
sudo logrotate -d /etc/logrotate.conf
```

**Force Immediate Rotation:**
```bash
sudo logrotate -f /etc/logrotate.conf
```

**Verbose Output:**
```bash
sudo logrotate -v /etc/logrotate.conf
```

**Test Specific Config:**
```bash
sudo logrotate -d /etc/logrotate.d/managed-myapp.conf
```

### Monitoring and Validation

**Check Disk Usage:**
```bash
# Overall log directory size
du -sh /var/log

# Largest log files
du -sh /var/log/* | sort -hr | head -10

# Compressed vs uncompressed
du -sh /var/log/*.log /var/log/*.gz 2>/dev/null
```

**Verify Rotation is Working:**
```bash
# Check recent rotations
grep "$(date +%Y%m%d)" /var/lib/logrotate/logrotate.status

# Check for today's compressed logs
ls -lh /var/log/*-$(date +%Y%m%d).gz
```

**Check Configuration Syntax:**
```bash
# Validate all configs
sudo logrotate -d /etc/logrotate.conf

# Check for errors
echo $?  # Should be 0
```

## Troubleshooting

### Logs Not Rotating

**Symptom:** Old logs remain, no rotation happening

**Check cron service:**
```bash
# Debian
systemctl status cron

# RHEL/AlmaLinux
systemctl status crond

# Check cron logs
journalctl -u cron | grep logrotate
```

**Check for errors:**
```bash
# Test configuration
sudo logrotate -d /etc/logrotate.conf

# Force rotation with verbose output
sudo logrotate -vf /etc/logrotate.conf
```

**Common causes:**
- Cron service not running
- Syntax error in configuration
- Permissions on log files
- Disk full (can't write compressed files)

### Compression Not Working

**Symptom:** Rotated logs are not compressed (missing .gz or .zst extension)

**Check global configuration:**
```bash
grep compress /etc/logrotate.conf
```

**Expected:**
```
compress          # Should NOT have # in front
delaycompress     # Optional
```

**Check specific config:**
```bash
cat /etc/logrotate.d/managed-myapp.conf
```

**Verify compression tool is installed:**
```bash
which gzip   # For gzip
which zstd   # For zstd
```

**Force rotation to test:**
```bash
sudo logrotate -vf /etc/logrotate.conf 2>&1 | grep -i compress
```

### Disk Still Full After Rotation

**Find large uncompressed logs:**
```bash
find /var/log -type f -name "*.log" -size +100M -exec ls -lh {} \;
```

**Check for logs not covered by logrotate:**
```bash
# Find logs not mentioned in any config
comm -23 \
  <(find /var/log -name "*.log" | sort) \
  <(grep -h "^/" /etc/logrotate.d/* | tr -d '{' | sort)
```

**Check if compression is actually saving space:**
```bash
# Before: 100MB log
ls -lh /var/log/myapp.log

# After rotation with compression:
ls -lh /var/log/myapp.log-*.gz
# Should be ~30-45MB
```

**Solutions:**
- Add missing logs to `logrotate_custom_configs`
- Reduce `logrotate_rotate_count` (keep fewer rotations)
- Enable `logrotate_install_zstd` for better compression
- Set smaller `logrotate_size` threshold

### Configuration Changes Not Applied

**Symptom:** Ansible runs successfully but settings don't change

**Verify file was updated:**
```bash
# Check modification time
ls -l /etc/logrotate.conf

# Check contents
cat /etc/logrotate.conf | grep -E "(daily|compress|rotate)"
```

**Check for Ansible backup:**
```bash
# Look for .YYYY-MM-DD@HH:MM:SS~ backup
ls -la /etc/logrotate.conf*
```

**Re-run Ansible with verbose output:**
```bash
ansible-playbook playbook.yml -v --tags logrotate
```

**Force re-deploy:**
```bash
# Remove existing file
sudo mv /etc/logrotate.conf /etc/logrotate.conf.old

# Re-run Ansible
ansible-playbook playbook.yml --tags logrotate
```

### Custom Config Syntax Errors

**Symptom:** Logrotate fails with "error: file:line syntax error"

**Test specific config:**
```bash
sudo logrotate -d /etc/logrotate.d/managed-myapp.conf
```

**Common syntax errors:**
```
# WRONG - missing opening brace
/var/log/app.log
    rotate 7
}

# RIGHT
/var/log/app.log {
    rotate 7
}

# WRONG - quotes around log path
"/var/log/app.log" {
    rotate 7
}

# RIGHT - no quotes needed (unless spaces in path)
/var/log/app.log {
    rotate 7
}
```

**Validate before deploy:**
```yaml
# In your playbook, use check mode first
ansible-playbook playbook.yml --check --diff
```

## Best Practices

### For All Servers

1. **Enable compression** - Saves 50-70% disk space (enabled by default)
2. **Use size limits** - Prevents runaway logs filling disk
3. **Monitor disk usage** - Set up alerts for /var/log usage >70%
4. **Test configurations** - Use `logrotate -d` before deploying

### For High-Volume Logging

1. **Use zstd compression** - Better compression ratio and speed
2. **Rotate daily** - Prevents large files
3. **Set aggressive size limits** - Catch problems early
4. **Consider shorter retention** - Don't keep unnecessary history

### For Production Systems

1. **Keep longer history** - Increase `rotate_count` to 30+
2. **Use `delaycompress`** - Safer for active services
3. **Test in staging first** - Verify rotation works before production
4. **Document custom configs** - Add comments explaining why

### For Compliance/Audit Requirements

1. **Keep long retention** - Set `rotate_count` to 365+ if needed
2. **Disable size-based rotation** - Only rotate on time
3. **Backup rotated logs** - Copy to long-term storage before deletion
4. **Use `dateext`** - Easier to identify log file dates

## Differences from Distribution Defaults

This role intentionally changes distribution defaults to address common disk space issues:

### Debian 12 Changes

| Setting | Debian Default | Our Default | Why |
|---------|---------------|-------------|-----|
| Compression | Disabled | Enabled | Save 50-70% disk space |
| Interval | Weekly | Daily | Prevent large files |
| Rotate count | 4 | 12 | Better history |
| Date extension | Disabled | Enabled | Better file naming |
| Size limit | None | 100M | Catch runaway logs |

### AlmaLinux/Rocky Changes

| Setting | RHEL Default | Our Default | Why |
|---------|-------------|-------------|-----|
| Compression | Disabled | Enabled | Save 50-70% disk space |
| Interval | Weekly | Daily | Prevent large files |
| Rotate count | 4 | 12 | Better history |
| Size limit | None | 100M | Catch runaway logs |

### Preserved Platform Differences

- **wtmp/btmp rotation** - Only on RHEL-family (preserved as-is)
- **Cron timing** - Uses distribution-provided cron.daily
- **Package names** - Handled automatically per platform

## Dependencies

None. This role manages logrotate independently.

## Testing

**Molecule tests cover:**
- Package installation (including Debian where logrotate isn't pre-installed)
- Global configuration deployment
- Compression enablement
- Custom configuration management
- Non-managed config preservation
- Configuration validation
- Documentation generation
- Idempotency
- Multi-platform compatibility

**Run tests locally:**
```bash
cd coffeesprout.logrotate
source /Users/barry/projects/roles/molecule-venv/bin/activate
molecule test
```

**CI/CD:**
- GitHub Actions runs tests on every push
- Tests run on Debian 12/13, AlmaLinux 8/9
- Full test cycle: syntax → create → converge → idempotence → verify

## License

BSD-2-Clause

## Author Information

Created by Barry van Someren for CoffeeSprout.

Part of the CaffeineStacks managed VM service infrastructure.

## Contributing

Issues and pull requests welcome at: https://github.com/CoffeeSprout/ansible-role-logrotate

## Related Roles

- **coffeesprout.docker-prune** - Automated Docker cleanup
- **coffeesprout.sshd** - SSH security hardening
- **coffeesprout.users** - User account management

## Changelog

### Version 1.0.0 (2025-10-05)

**Initial release:**
- Global logrotate configuration management
- Custom configuration support with managed- prefix
- Improved defaults (compression, daily rotation, size limits)
- zstd compression support
- Multi-platform support (Debian, AlmaLinux, Rocky)
- Molecule testing
- GitHub Actions CI/CD
- Auto-generated documentation
