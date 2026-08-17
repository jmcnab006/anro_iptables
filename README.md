# ANsible ROle iptables

`anro_iptables` builds and applies persistent IPv4 and IPv6 `iptables-restore` rulesets on Debian and Ubuntu. It supports the `filter`, `nat`, `mangle`, and `raw` tables, layered Ansible variables, static and inventory-derived filter chains, optional IP sets, Docker `DOCKER-USER` policy, and kernel forwarding for gateway hosts.

The role renders complete `/etc/iptables/rules.v4` and `/etc/iptables/rules.v6` files, validates them with `iptables-restore --test` / `ip6tables-restore --test`, applies them, and relies on `netfilter-persistent` at boot.

## Requirements

- Debian or Ubuntu
- systemd
- privilege escalation (`become: true`)
- Ansible 2.12 or newer

The role installs `iptables`, `iptables-persistent`, and `netfilter-persistent`. When IP sets are enabled it also installs `ipset` and `ipset-persistent`.

## Install with `requirements.yaml`

Add the role to your project requirements file:

```yaml
---
roles:
  - name: anro_iptables
    src: https://github.com/jmcnab006/anro_iptables.git
    scm: git
    version: main
```

For production, pin `version` to a tested release tag or commit rather than `main`.

Install it with:

```bash
ansible-galaxy role install -r requirements.yaml
```

A ready-to-copy file is included at `example/requirements.yaml`.

## Minimal use

```yaml
---
- name: Configure host firewall
  hosts: all
  become: true
  roles:
    - role: anro_iptables
```

The default filter policies are `ACCEPT`. Define a restrictive policy explicitly when that is what you intend:

```yaml
anro_iptables_input_policy: DROP
anro_iptables_forward_policy: DROP
anro_iptables_output_policy: ACCEPT
```

## Rule model

Rules are organized by address family, table, and layer:

```text
anro_iptables_<ipv4|ipv6>_<filter|nat|mangle|raw>_rules<0-9>
```

Layers are concatenated from `0` through `9`. A direct, non-numbered list is appended after the numbered lists for that table. This makes it possible to place shared baseline rules in `group_vars/all`, service rules in group variables, host exceptions in `host_vars`, and final rules in a later layer without copying the full ruleset.

Example:

```yaml
# group_vars/all/firewall.yaml
anro_iptables_ipv4_filter_rules1:
  - '-A INPUT -i lo -j ACCEPT'
  - '-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT'

# group_vars/web/firewall.yaml
anro_iptables_ipv4_filter_rules4:
  - '-A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT'
```

The `filter` table is always rendered. `raw`, `mangle`, and `nat` are rendered only when their effective rule list is non-empty. Tables are emitted in `raw`, `mangle`, `nat`, `filter` order.

### NAT

```yaml
anro_iptables_gateway_enable: true
anro_iptables_ipv4_nat_rules5:
  - '-A POSTROUTING -s 10.20.0.0/16 -o eth0 -j MASQUERADE'
```

### Mangle

```yaml
anro_iptables_ipv4_mangle_rules5:
  - '-A PREROUTING -p tcp --dport 443 -j MARK --set-mark 10'
```

### Raw

```yaml
anro_iptables_ipv4_raw_rules5:
  - '-A PREROUTING -p udp --dport 8125 -j NOTRACK'
  - '-A OUTPUT -p udp --sport 8125 -j NOTRACK'
```

Rules are inserted verbatim into the selected table. Custom chains for `nat`, `mangle`, or `raw` can therefore be declared directly in the rule list:

```yaml
anro_iptables_ipv4_nat_rules3:
  - ':WEB_DNAT - [0:0]'
  - '-A PREROUTING -p tcp --dport 8443 -j WEB_DNAT'
  - '-A WEB_DNAT -j DNAT --to-destination 10.20.30.40:443'
```

## Filter host groups

Static host groups create custom chains in the `filter` table. Matching source addresses are accepted by the generated chain; traffic that does not match returns to the calling chain.

```yaml
anro_iptables_host_groups:
  - name: PRIVATE
    ipv4:
      - 10.0.0.0/8
      - 192.168.0.0/16
    ipv6:
      - fd00::/8

anro_iptables_ipv4_filter_rules4:
  - '-A INPUT -p tcp --dport 22 -j PRIVATE'
```

Either address family may be omitted from a host group.

Inventory groups can also become filter chains:

```yaml
anro_iptables_ansible_host_groups:
  - monitoring
  - database
```

By default the role reads `ipv4` and `ipv6` host variables from each member. Override these names with `anro_iptables_ansible_host_groups_ipv4_hostvar` and `anro_iptables_ansible_host_groups_ipv6_hostvar`.

## IP sets

Enable IP sets and define them with `anro_iptables_ipsets`:

```yaml
anro_iptables_ipset_enabled: true
anro_iptables_ipsets:
  - name: ADMIN_NETS
    type: hash:net
    family: inet
    set:
      - 192.0.2.0/24
      - 198.51.100.32/27

anro_iptables_ipv4_filter_rules3:
  - '-A INPUT -p tcp --dport 9100 -m set --match-set ADMIN_NETS src -j ACCEPT'
```

The role creates or updates only the declared sets. It does not globally flush unrelated IP sets. Each declared set is flushed and repopulated before firewall rules are applied.

## Docker

Set `anro_iptables_docker_enabled: true` to manage a restrictive `DOCKER-USER` chain when Docker is installed. The role adds local/Docker subnet and established-connection returns, then your custom rules, then a final `DROP`.

```yaml
anro_iptables_docker_enabled: true
anro_iptables_ipv4_docker_rules:
  - '-A DOCKER-USER -i eth0 -s 198.51.100.0/24 -p tcp --dport 8443 -j ACCEPT'
```

With the default authoritative restore behavior, a running Docker daemon is restarted after firewall changes so Docker can recreate its own dynamically managed chains. Disable that with `anro_iptables_docker_restart_on_change: false` only when you have another reconciliation mechanism.

## Gateway forwarding

```yaml
anro_iptables_gateway_enable: true
```

This manages IPv4 and IPv6 forwarding in `/etc/sysctl.d/99-anro-iptables-forwarding.conf`.

## Restore behavior

`anro_iptables_restore_noflush` defaults to `false`. The tables present in the generated rules file are therefore authoritative when restored. This avoids stale Ansible-managed rules surviving after they are removed from inventory.

Set the following only when this role must deliberately merge with another iptables manager:

```yaml
anro_iptables_restore_noflush: true
```

Be aware that `--noflush` changes ownership semantics and can leave pre-existing rules in place.

## IPv4 / IPv6 control

Both families are managed by default:

```yaml
anro_iptables_ipv4_enabled: true
anro_iptables_ipv6_enabled: true
```

If a family is disabled, its persistent rules file is removed so `netfilter-persistent` cannot reload a stale role-managed configuration at boot.

## Compatibility with the original role

The original filter variables remain supported:

```text
anro_iptables_ipv4_rules
anro_iptables_ipv4_rules0 .. anro_iptables_ipv4_rules9
anro_iptables_ipv6_rules
anro_iptables_ipv6_rules0 .. anro_iptables_ipv6_rules9
```

They are treated as legacy `filter` inputs and are assembled before the new `*_filter_rules*` variables. New configurations should use the table-specific names.

The inconsistent `anro_iptables_ipset_sets` spelling is not retained. The supported variable is `anro_iptables_ipsets`, matching the defaults and argument specification.

## Validation and safety

Before changing firewall files, the role validates platform assumptions and key variable structures. Rendered IPv4 and IPv6 files are validated with the corresponding restore command using `--test` before installation.

Firewall changes can disconnect remote systems even when syntax is valid. Test restrictive policy, NAT, raw-table `NOTRACK`, Docker policy, and forwarding behavior in a recoverable environment before broad deployment.

## Full example

See `example/example.yaml` for filter, NAT, mangle, raw, IPSet, Docker, and gateway examples.

## Variable reference

`meta/argument_specs.yaml` is the authoritative list of supported public variables and their types.
