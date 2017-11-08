# Ansible role common
Ansible Role for common servers setup tasks

## Requirements
Ansible version => 2.3

## Role tags:
  - apt - apt-packages setup/update 
  - (apt|yum)repos - set common system repos
  - locales - set default system locales
  - yum - yum-packages setup/update
  - ulimit - set system limits (e.g. open files limit)
# Vars
## Default role variables
```yaml
country: ru
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
```
# Playbook run
## For all hosts
```
ansible-playbook -i production playbooks/common.yml
```
## For single host
```
ansible-playbook -i develop playbooks/common.yml --limit backend-01
```
