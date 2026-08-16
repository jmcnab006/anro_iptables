# ANsible ROle iptables

This role manages host firewalls on Debian and Ubuntu systems using `iptables`, `iptables-persistent`, and `ipset`.

## Requirements

None.

## Dependencies

Systemd for system management on the remote host.

## Use the role

Use the name of the role directory in the `roles:` list to use it in a playbook.

```yaml
- name: ROLES | Run iptables role on all hosts. 
  hosts: all
  roles:
    - anro_iptables
```

## Role variables

Refer to [argument specifications](meta/argument_specs.yaml) for detailed information on all role variables.

## Examples

This role supports flexible and hierarchical rulesets, custom chains, and IP sets. Here are a few examples to help you get the most out of this role.

### Rules

This role lets you build out complex iptables rules, using a combination of additional CHAINS and ipset sets. A hierarchy of variables concatenated to form a full iptables ruleset.

```yaml
iptables_ipv4_rules1: []
iptables_ipv4_rules2: []
iptables_ipv4_rules3: []
iptables_ipv4_rules4: []
iptables_ipv4_rules5: []
iptables_ipv4_rules6: []

iptables_ipv6_rules1: []
iptables_ipv6_rules2: []
iptables_ipv6_rules3: []
iptables_ipv6_rules4: []
iptables_ipv6_rules5: []
iptables_ipv6_rules6: []
```

Define common rules applicable to all hosts as the first set of rules. Add endcap rules, such as an explict "deny all" as the final set of rules.

```yaml
---
# Example group_vars/all.yaml
iptables_ipv4_rules1:
  - -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
  - -A INPUT -i lo -m state --state NEW -j ACCEPT
  - -A INPUT -p icmp -m limit --limit 100/sec -m state --state NEW -j ACCEP
iptables_ipv4_rules6:
  - -A INPUT -j DROP
```

Other group or host-specific rules can go in between.

```yaml
---
# Example group_vars/web.yaml
iptables_ipv4_rules2:
  - -A INPUT -d {{ ansible_host }} -p tcp -m tcp --dport 22 -m state --state NEW -j PRIVATE
  - -A INPUT -p tcp -m tcp --dport 80 -m state --state NEW -j ACCEPT
  - -A INPUT -p tcp -m tcp --dport 443 -m state --state NEW -j ACCEPT
```

For simpler rulesets, populate the `iptables_ipv6_rules` and `iptables_ipv4_rules` variables directly.

```yaml
iptables_ipv4_rules:
  - -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
  - -A INPUT -i lo -m state --state NEW -j ACCEPT
  - -A INPUT -p icmp -m limit --limit 100/sec -m state --state NEW -j ACCEPT
  - -A INPUT -p udp -m udp --dport 161 -j ACCEPT
  - -A INPUT -s 216.17.37.170/32 -p tcp -m tcp --dport 9102 -m state --state NEW -j ACCEPT
  - -A INPUT -p tcp -m tcp --dport 22 -m state --state NEW -j ACCEPT
  - -A INPUT -j DROP

iptables_ipv6_rules:
  - -A INPUT -i lo -j ACCEPT
  - -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
  - -A INPUT -m conntrack --ctstate INVALID -j DROP
  - -A INPUT -s ::1/128 ! -i lo -j DROP
  - -A INPUT -p udp -m udp --dport 161 -j ACCEPT
  - -A INPUT -p tcp -m tcp --dport 22 -m state --state NEW -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 1 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 2 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 3 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 4 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 128 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 129 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 133 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 134 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 135 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 136 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 137 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 141 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 142 -j ACCEPT
  - -A INPUT -s fe80::/10 -p ipv6-icmp -m icmp6 --icmpv6-type 130 -j ACCEPT
  - -A INPUT -s fe80::/10 -p ipv6-icmp -m icmp6 --icmpv6-type 131 -j ACCEPT
  - -A INPUT -s fe80::/10 -p ipv6-icmp -m icmp6 --icmpv6-type 132 -j ACCEPT
  - -A INPUT -s fe80::/10 -p ipv6-icmp -m icmp6 --icmpv6-type 143 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 148 -j ACCEPT
  - -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 149 -j ACCEPT
  - -A INPUT -s fe80::/10 -p ipv6-icmp -m icmp6 --icmpv6-type 151 -j ACCEPT
  - -A INPUT -s fe80::/10 -p ipv6-icmp -m icmp6 --icmpv6-type 152 -j ACCEPT
  - -A INPUT -s fe80::/10 -p ipv6-icmp -m icmp6 --icmpv6-type 153 -j ACCEPT
  - -A INPUT -j DROP
```

### Chains and host groups

Use a list of dictionaries to define custom chains and host groups.

```yaml
---
iptables_host_groups:
  - name: PRIVATE
    ipv4:
      - 172.16.31.0/24 
    ipv6:
      - fd00::/8
  - name: DMZ
    ipv4: 
      - 192.168.0.0/24
```

Use the custom chain in your rule sets as normal.

```yaml
iptables_ipv4_rules2:
  - -A INPUT -p udp -m udp --dport 161 -j PRIVATE
  - -A INPUT -p tcp -m tcp --dport 3306 -m state --state NEW -j DMZ
  - -A INPUT -p tcp -m tcp --dport 3306 -m state --state NEW -j PRIVATE

You can also include Ansible groups as chains. This will generate a chain for every group defined in your inventory. Alternatively, pass a static list to limit which groups are added.

```yaml
iptables_ansible_host_groups: "{{ groups | list }}"
```

Note, the role converts any group variables to all upper-case for naming consistency.

### IP sets

Set the `iptables_ipset_enabled` variable to true to enable IP sets. Defined sets are saved to a specific location on the managed host (`/etc/iptables/ipsets` by default) and restored to apply any changes.

> :warning: **If you remove an ipset**: Be sure to remove any reference to the removed ipset from the rules before running the role again. A nonexitent ipset in a rule can lead to unexpected behavior, including an inaccessible remote system.

```yaml
---
iptables_ipset_enabled: true

iptables_ipset_sets:
  - name: PRIVATE
    type: 'hash:net'
    family: inet
    bucketsize: 10
    set:
      - 172.16.31.0/24
      - 192.168.0.0/24
  - name: PRIVATE6
    type: 'hash:net'
    family: inet6
    maxelem: 65536
    set:
      - fd00::/8
```
### Docker

If both the `anro_iptables_docker_enabled` and Docker is installed on the host. Additional rules need to be added
to restrict access to the container. Since docker uses the FORWARD chain this bypasses the INPUT chain completely. 
The following arrays can then be used to configure the docker rules. `anro_iptables_ipv6_docker_rules` is also 
available but there is limited if any real ipv6 support in docker at the moment.

```yaml
anro_iptables_ipv4_docker_rules: 
  - -A DOCKER-USER -i ens18 -s 216.17.68.192/26 -p tcp -m tcp --dport 4200 -j ACCEPT

```

The anro_iptables docker ipv4 template has implicit rules to allow local docker traffic but DROP all other docker traffic
via the DOCKER-USER chain.


## Example Playbook

See the [example play](example/example.yaml) for real-world use.
