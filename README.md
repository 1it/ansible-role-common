# Ansible role common
Ansible role for baseline server setup and high-load tuning on Debian (12 Bookworm, 13 Trixie), Ubuntu (24.04 Noble, 26.04 Resolute), and RHEL-family (RHEL, Rocky, AlmaLinux, CentOS EL 8/9/10) distributions.

## Requirements
- Ansible >= 2.15
- Collections:
  - `ansible.posix` (`sysctl`)
  - `community.general` (`locale_gen`, `timezone`, `modprobe`)

## Role tags
- `repos`, `apt_repos`, `yum_repos` - configure system and custom package repositories
- `pkg`, `apt`, `yum`, `dnf` - install base + extra packages (and optionally upgrade packages when `do_upgrade: true`)
- `time`, `timezone`, `ntp` - set system timezone (`UTC`) and enable `chrony` NTP sync
- `journald`, `logging` - cap `systemd-journald` disk usage and retention
- `ulimit`, `limits` - configure PAM limits (`/etc/security/limits.d/99-common.conf`) and systemd service defaults (`/etc/systemd/system.conf.d/99-common-limits.conf`)
- `sysctl` - configure kernel sysctl parameters in `/etc/sysctl.d/99-common.conf` (with automatic BBR detection)
- `locales` - generate and set default system locale (`LANG`) on Debian/Ubuntu and EL/RedHat
- `profile` - optionally manage `/root/.bashrc` when `manage_root_profile: true`

## Key defaults
See `defaults/main.yml` for the full list of variables and packages:
```yaml
do_upgrade: false
apt_upgrade_type: safe
manage_apt_sources: false

manage_timezone: true
timezone: "UTC"
manage_ntp: true

manage_journald: true
journald_config:
  SystemMaxUse: "1G"
  SystemKeepFree: "2G"
  MaxRetentionSec: "1month"
  Compress: "yes"

locales:
  - en_US.UTF-8
default_locale: "en_US.UTF-8"

system_limits:
  ulimits:
    - "root soft nofile 65536"
    - "root hard nofile 524288"
    - "* soft nofile 65536"
    - "* hard nofile 524288"

manage_systemd_limits: true
systemd_default_limit_nofile: "65536:524288"

sysctl_file: "/etc/sysctl.d/99-common.conf"
sysctl_enable_bbr: true
sysctl_extra_params: []

apt_packages_extra: []
yum_packages_extra: []
manage_root_profile: false
```

## Custom APT Repositories
Prefer `apt_repositories` (DEB822 `.sources` via `ansible.builtin.deb822_repository`), which scopes `signed_by` per repository and cleans up both `.sources` and keyring files on `state: absent`.

Legacy one-line entries in `apt_repos_list` remain supported and are written to `/etc/apt/sources.list.d/*.list` (set explicit `keyring` + `signed-by=` rather than omitting `keyring`, which drops the key into `/etc/apt/trusted.gpg.d/` for all sources; `ppa:` shortcuts are not expanded and `state: absent` removes only the `.list` file).

## Example Playbook
```yaml
---
- hosts: all
  become: true
  roles:
    - common
  vars:
    apt_repositories:
      - name: hashicorp
        uris: https://apt.releases.hashicorp.com
        suites: "{{ ansible_facts['distribution_release'] }}"
        components:
          - main
        architectures: amd64
        signed_by: https://apt.releases.hashicorp.com/gpg

    # Append or override individual sysctl parameters without replacing defaults
    sysctl_extra_params:
      - key: vm.swappiness
        value: 1

    # Add project-specific packages while keeping base server tools
    apt_packages_extra:
      - nginx
      - redis-tools
```

## Playbook run
```bash
ansible-playbook playbooks/common.yml --skip-tags limits,profile
```
