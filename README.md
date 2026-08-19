
# ANsible ROle iptables

`anro_iptables` is an Ansible role for building and managing persistent Linux `iptables` firewall configurations on Debian and Ubuntu systems.

It provides a declarative firewall model that compiles structured Ansible variables into deterministic `iptables-restore` and `ip6tables-restore` configurations.

---

## Features

The role manages complete IPv4 and IPv6 firewall stacks with support for:

* `filter`, `nat`, `mangle`, and `raw` tables
* Stateful firewalling using connection tracking
* Layered rule definitions (inventory-friendly design)
* First-class custom chains per table
* Persistent IP sets (`ipset`)
* Gateway / router mode with forwarding support
* Kernel sysctl management for forwarding
* Docker-aware firewall reconciliation (service restart model)
* Pre-apply validation using `iptables-restore --test`
* Molecule-based integration testing (Docker + systemd containers)

---

## Requirements

Target systems must be:

* Debian or Ubuntu based
* Using `systemd`
* Accessible via privilege escalation (`become: true`)
* Running Ansible 2.12+

The role installs:

```text
iptables
iptables-persistent
netfilter-persistent
```

When IP sets are enabled:

```text
ipset
ipset-persistent
```

`firewalld` is removed to avoid conflicting firewall managers.

---

## Installation

```yaml
---
roles:
  - name: anro_iptables
    src: https://github.com/jmcnab006/anro_iptables.git
    scm: git
    version: main
```

Install:

```bash
ansible-galaxy role install -r requirements.yaml
```

> Production usage should pin to a release tag or commit SHA.

---

## Minimal Example

```yaml
---
- name: Configure firewall
  hosts: all
  become: true

  vars:
    anro_iptables_input_policy: DROP
    anro_iptables_forward_policy: DROP
    anro_iptables_output_policy: ACCEPT

    anro_iptables_ipv4_filter_rules1:
      - '-A INPUT -i lo -j ACCEPT'
      - '-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT'
      - '-A INPUT -m conntrack --ctstate INVALID -j DROP'
      - '-A INPUT -p icmp -j ACCEPT'

    anro_iptables_ipv6_filter_rules1:
      - '-A INPUT -i lo -j ACCEPT'
      - '-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT'
      - '-A INPUT -m conntrack --ctstate INVALID -j DROP'
      - '-A INPUT -p ipv6-icmp -j ACCEPT'

    anro_iptables_ipv4_filter_rules4:
      - '-A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT'

    anro_iptables_ipv6_filter_rules4:
      - '-A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT'

  roles:
    - anro_iptables
```

---

# Configuration Model

The role compiles firewall state through a deterministic pipeline:

```text
Ansible variables
      ↓
normalization / preparation layer
      ↓
rendered rules.v4 / rules.v6
      ↓
iptables-restore / ip6tables-restore
```

All transformation logic is handled internally before rendering.

---

# Rule Tables

Supported tables:

* `filter`
* `nat`
* `mangle`
* `raw`

The `filter` table is always rendered.

Other tables are rendered only when required.

---

# Layered Rule System

Rules can be defined in ordered layers:

```yaml
anro_iptables_ipv4_filter_rules0: []
anro_iptables_ipv4_filter_rules1: []
...
anro_iptables_ipv4_filter_rules9: []
```

Plus a direct list:

```yaml
anro_iptables_ipv4_filter_rules: []
```

Execution order:

```text
0 → 1 → 2 → ... → 9 → direct rules
```

This enables clean inventory layering:

```yaml
# group_vars/all/firewall.yml
anro_iptables_ipv4_filter_rules1:
  - '-A INPUT -i lo -j ACCEPT'
  - '-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT'
```

```yaml
# group_vars/web/firewall.yml
anro_iptables_ipv4_filter_rules4:
  - '-A INPUT -p tcp --dport 80 -j ACCEPT'
  - '-A INPUT -p tcp --dport 443 -j ACCEPT'
```

---

# Custom Chains

Custom chains are defined per table:

```yaml
anro_iptables_filter_chains: []
anro_iptables_nat_chains: []
anro_iptables_mangle_chains: []
anro_iptables_raw_chains: []
```

Example:

```yaml
anro_iptables_filter_chains:
  - name: SSH_SOURCES

    ipv4:
      - '-s 10.20.0.0/16 -j ACCEPT'
      - '-j DROP'

    ipv6:
      - '-s fd20:20::/64 -j ACCEPT'
      - '-j DROP'
```

Chains are **definitions only**.

They must be referenced explicitly in rules:

```yaml
anro_iptables_ipv4_filter_rules4:
  - '-A INPUT -p tcp --dport 22 -j SSH_SOURCES'
```

---

## NAT Example

```yaml
anro_iptables_gateway_enable: true

anro_iptables_nat_chains:
  - name: LAN_NAT
    ipv4:
      - '-s 10.20.0.0/16 -o eth0 -j MASQUERADE'
      - '-j RETURN'

anro_iptables_ipv4_nat_rules5:
  - '-A POSTROUTING -j LAN_NAT'
```

---

## Mangle Example

```yaml
anro_iptables_mangle_chains:
  - name: MARK_WEB
    ipv4:
      - '-p tcp --dport 443 -j MARK --set-mark 0x10'
      - '-j RETURN'

anro_iptables_ipv4_mangle_rules4:
  - '-A PREROUTING -i eth1 -j MARK_WEB'
```

---

## Raw Example

```yaml
anro_iptables_raw_chains:
  - name: NOTRACK_SYSLOG
    ipv4:
      - '-p udp --dport 514 -j CT --notrack'
      - '-j RETURN'

anro_iptables_ipv4_raw_rules3:
  - '-A PREROUTING -i eth1 -p udp --dport 514 -j NOTRACK_SYSLOG'
```

---

# IP Sets

Enable IP sets:

```yaml
anro_iptables_ipset_enabled: true
```

Define sets:

```yaml
anro_iptables_ipsets:
  - name: MGMT_V4
    type: hash:net
    family: inet
    set:
      - 10.10.0.0/16
      - 192.168.50.0/24

  - name: MGMT_V6
    type: hash:net
    family: inet6
    set:
      - fd10:10::/48
```

Defaults:

```yaml
anro_iptables_ipset_hashsize: 1024
anro_iptables_ipset_maxelem: 65536
anro_iptables_ipset_family: inet
anro_iptables_ipset_type: hash:net
```

Use in rules:

```yaml
anro_iptables_filter_chains:
  - name: SSH_SOURCES
    ipv4:
      - '-m set --match-set MGMT_V4 src -j ACCEPT'
      - '-j DROP'
```

IP sets are restored before firewall rules are applied.

---

# IPv4 / IPv6 Control

```yaml
anro_iptables_ipv4_enabled: true
anro_iptables_ipv6_enabled: true
```

Disabling a family removes its persistent rules file.

---

# Filter Policies

```yaml
anro_iptables_input_policy: ACCEPT
anro_iptables_forward_policy: ACCEPT
anro_iptables_output_policy: ACCEPT
```

Valid values:

```text
ACCEPT
DROP
QUEUE
```

Typical hardened setup:

```yaml
anro_iptables_input_policy: DROP
anro_iptables_forward_policy: DROP
anro_iptables_output_policy: ACCEPT
```

---

# Gateway Mode

```yaml
anro_iptables_gateway_enable: true
```

Enables kernel forwarding via:

```text
/etc/sysctl.d/99-anro-iptables-forwarding.conf
```

---

# Docker Behavior

Docker is not directly managed by firewall rules.

Instead, when enabled:

```yaml
anro_iptables_docker_enabled: true
```

Docker is restarted after firewall changes to allow it to rebuild its internal rules.

```yaml
anro_iptables_docker_restart_on_change: true
```

---

# Restore Behavior

```yaml
anro_iptables_restore_noflush: false
```

When enabled:

```yaml
--noflush
```

is passed to restore commands.

---

# Persistent Files

Default paths:

```text
/etc/iptables/rules.v4
/etc/iptables/rules.v6
/etc/iptables/ipsets
```

Configurable via:

```yaml
anro_iptables_config_dir
anro_iptables_ipv4_save_file
anro_iptables_ipv6_save_file
anro_iptables_ipset_save_file
```

---

# Validation

Before applying changes:

* `iptables-restore --test`
* `ip6tables-restore --test`
* internal schema validation
* chain and rule normalization checks

---

# Full Example

See:

```text
example/example.yaml
```

---

# Variable Reference

Authoritative schema:

```text
meta/argument_specs.yaml
```

Defaults:

```text
defaults/main.yaml
```

---

# Molecule Testing

The role includes a Docker-based Molecule scenario.

## Prerequisites

* Docker
* Python 3
* virtualenv support

---

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install tooling:

```bash
pip install ansible-core ansible-lint molecule docker
```

Install collections:

```bash
ansible-galaxy collection install -r molecule/default/requirements.yml
```

---

## Run Tests

Full test:

```bash
molecule test -s default
```

Individual steps:

```bash
molecule create -s default
molecule converge -s default
molecule verify -s default
molecule destroy -s default
```

Idempotence:

```bash
molecule idempotence -s default
```

---

## Test Coverage

* Ubuntu 22.04
* Ubuntu 24.04
* IPv4/IPv6 rules
* custom chains
* IP sets
* NAT / mangle / raw tables
* systemd-based persistence
* iptables-restore validation
* idempotence

---

# Development Workflow

```bash
ansible-lint .

molecule converge -s default
molecule idempotence -s default
molecule verify -s default
```

Full run:

```bash
molecule test -s default
```

---

# License

See `LICENSE`.

---

If you want next step, I can also:

* tighten the README into a **Galaxy-ready documentation style**
* or generate a **docs/ site (MkDocs or Sphinx) from this structure**
* or align `argument_specs.yaml` perfectly with this README so they are 1:1 consistent
