# Ansible role common
Ansible Role for common servers setup tasks

## Requirements
Ansible version => 2.9

## Role tags:
  - apt - apt-packages setup/update 
  - (apt|yum)repos - set common system repos
  - locales - set default system locales
  - yum - yum-packages setup/update
  - ulimit - set system limits (e.g. open files limit)
  - profile - copy bashrc file for root

# Vars
## Default role variables
```yaml
do_upgrade: False

locales:
    - en_US.UTF-8

system_limits:
  ulimits:
    - 'root soft nofile 65536'
    - 'root hard nofile 65536'
    - '* soft nofile 65536'
    - '* hard nofile 65536'

apt_packages:
    - bash-completion
    - curl
    - git
    - tig
    - htop
    - vim-nox
```
## Example Playbook
```yaml
---
- hosts: all
  roles:
    - common
  vars:
    apt_repos_list:
      - repo: "deb [arch=amd64] https://apt.releases.hashicorp.com {{ ansible_distribution_release }} main"
        key: "https://apt.releases.hashicorp.com/gpg"
      - repo: deb [signed-by=/etc/apt/trusted.gpg.d/kubernetes-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main
        key: https://packages.cloud.google.com/apt/doc/apt-key.gpg
        keyring: /etc/apt/trusted.gpg.d/kubernetes-keyring.gpg

    apt_packages:
      - nomad
      - consul
      - kubectl
```
# Playbook run
```
ansible-playbook playbooks/common.yml --skip-tags ulimit,profile
```
