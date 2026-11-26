# AGENTS.md

Instructions for coding agents working on this standalone Ansible role (`common`). See `README.md` and `defaults/main.yml` for user-facing docs and full defaults.

## Targets & Requirements
- **Ansible:** `>= 2.15` (tested on `ansible-core` 2.22). Collections: `ansible.posix`, `community.general`.
- **Platforms:** Debian 12/13, Ubuntu 24.04/26.04 (`amd64` + `arm64`), EL 8/9/10 (Rocky, AlmaLinux, RHEL, CentOS Stream).

## Code Style & Conventions
- **FQCNs only:** Always use `ansible.builtin.*`, `ansible.posix.*`, `community.general.*`. Stick to modules available in Ansible 2.15+ (e.g., use `ansible.builtin.systemd`, not `systemd_service`; use `ansible.builtin.deb822_repository`, never deprecated `apt_repository`).
- **Facts access:** Always use `ansible_facts['...']` (never top-level `ansible_*` variables).
- **File permissions:** Always specify `owner: root`, `group: root`, and quoted `mode: "0644"` / `"0755"` on `template`, `copy`, `file`, `lineinfile`, and `get_url`.
- **Jinja2 native types:** Never mix `{% set %}` statements with `{{ ... }}` inside `set_fact` when producing lists/dicts (Ansible >= 2.19 returns a string). Use single `{{ ... }}` expressions with task-level `vars:`.
- **Task naming & tags:** Prefix task names with section (`Apt | ...`, `Yum | ...`, `Sysctl | ...`, `Limits | ...`, `Time | ...`, `Journald | ...`, `Locales | ...`, `Profile | ...`). Import task files statically in `tasks/main.yml` with explicit `tags:`.
- **Idempotency:** Every second run must report `changed=0`. Notify handlers in `handlers/main.yml` (with `meta: flush_handlers` when downstream tasks depend on repo changes) instead of unconditional `update_cache: true`.
- **Additive overrides:** Keep defaults overridable without wiping base lists (`apt_packages_extra`, `yum_packages_extra`, `sysctl_extra_params`). Deduplicate overridden `sysctl` keys so repeated runs stay idempotent.
- **Keep metadata in sync:** Whenever adding, renaming, or changing a variable in `defaults/main.yml`, update `meta/argument_specs.yml` and `README.md` in the same edit.

## Platform Guardrails (Do Not Break)
- **Opt-in defaults:** `manage_apt_sources`, `manage_root_profile`, and `do_upgrade` must stay `false` by default; `apt_upgrade_type` stays `safe` (not `dist`).
- **Debian 13+ / Ubuntu 26.04+ `sysctl`:** Never write to `/etc/sysctl.conf` (ignored by `systemd-sysctl`). Always target `/etc/sysctl.d/*.conf` (`sysctl_file`).
- **Systemd limits:** `systemd` ignores `/etc/security/limits.d/*`. Keep both `/etc/security/limits.d/99-common.conf` (and cleanup of legacy `limits.conf`, which sorts after `99-*`) and `/etc/systemd/system.conf.d/99-common-limits.conf` in sync.
- **Ubuntu ARM64 mirrors:** Both main and `<release>-security` suites are served from `http://ports.ubuntu.com/ubuntu-ports` (`ubuntu-ports-security` does **not** exist — returns 404).
- **Debian APT bootstrap:** Keep Debian security repos inside `templates/Debian.sources.list.j2` and never use deprecated `ansible.builtin.apt_repository`. Use `ansible.builtin.deb822_repository` for `apt_repositories` (installing `ca-certificates` and `python3-debian`, guarding `--check` mode when prereqs change, and skipping default `<distro>.sources` removal when `apt_repositories` manages an active repo with that slug). For legacy `apt_repos_list`, keep omitted `filename` values matching `apt_repository`'s URI slug convention, group entries sharing a `.list` file into a single `copy` write so lines do not clobber each other, and guard `get_url` in `--check` mode (note: `ppa:` expansion is not supported, `state: absent` deletes the computed `.list` file without removing downloaded keys, and omitting `keyring` drops keys into global `/etc/apt/trusted.gpg.d/`). Use real package names (`bind9-dnsutils`, not virtual `dnsutils`).
- **EL minimal images:** Keep `allowerasing: true` in `tasks/yum.yml` so `curl` cleanly replaces preinstalled `curl-minimal` on minimal EL 9 images. Install `epel-release` from built-in `extras` on EL clones (`distribution != 'RedHat'`) and via GPG key + RPM URL only on `RedHat`.
- **BBR safety:** Always guard `net.core.default_qdisc=fq` and `net.ipv4.tcp_congestion_control=bbr` behind `/proc/sys/net/ipv4/tcp_available_congestion_control` detection so containers/restricted kernels do not fail.

## Validation & Testing

### 1. Syntax & Argument Spec Check
Run after any YAML/task/variable change:
```bash
TMPDIR=$(mktemp -d) && trap 'rm -rf "$TMPDIR"' EXIT
ln -s "$PWD" "$TMPDIR/common"
cat > "$TMPDIR/check.yml" <<'EOF'
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - common
EOF
ANSIBLE_ROLES_PATH="$TMPDIR" ansible-playbook --syntax-check "$TMPDIR/check.yml"
```

### 2. Template Rendering Check
When editing anything under `templates/`, verify rendering with `jinja2.StrictUndefined` across:
- `Debian.sources.list.j2` (Debian 12 & 13, `apt_include_deb_src` false/true)
- `Ubuntu.sources.list.j2` (Ubuntu 24.04 & 26.04, `x86_64` & `aarch64`)
- `bashrc.j2` (`Debian` & `RedHat` `os_family`)
- `ulimits.conf.j2`, `systemd-limits.conf.j2`, `journald.conf.j2`

### 3. Docker Integration & Idempotency Tests
Use `community.docker.docker` against official images: `debian:12`, `debian:13`, `ubuntu:24.04`, `ubuntu:26.04`, `rockylinux/rockylinux:9`, `rockylinux/rockylinux:10`.
- **NEVER use `--privileged` with `sysctl` tasks:** `vm.*` and `fs.*` sysctls are not namespaced in Linux containers and will mutate the host kernel. Always pass `--skip-tags sysctl` in unprivileged containers.
- **Container hygiene:** Prefix test containers with `ansible-common-test-*` and always clean them up via `trap ... EXIT` so unrelated running containers are never touched.
- **Required checks:**
  1. Converge cleanly on all target images (including a fresh-host test with `manage_apt_sources: true` on Debian 13 & Ubuntu 26.04).
  2. Re-run the exact same playbook and assert `changed=0` on every host.
  3. For systemd tasks (`journald`, `limits`, `time`), test in a non-privileged systemd-booted container (`--cgroupns=host --tmpfs /tmp --tmpfs /run --tmpfs /run/lock -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cap-add=SYS_ADMIN --security-opt seccomp=unconfined --security-opt apparmor=unconfined`) and verify `systemctl show --property=DefaultLimitNOFILE,DefaultLimitNOFILESoft` and `systemctl is-active systemd-journald` (note: `chrony`/`chronyd` cannot stay active inside a container without `CAP_SYS_TIME`).
