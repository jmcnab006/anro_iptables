# Changelog

## 2.0.0 candidate

- Add layered `filter`, `nat`, `mangle`, and `raw` rule inputs for IPv4 and IPv6.
- Preserve legacy IPv4/IPv6 filter rule variables for compatibility.
- Fix Docker detection variable mismatch and IPv6 Docker rule duplication.
- Standardize IPSet input on `anro_iptables_ipsets`.
- Avoid globally flushing unrelated IP sets.
- Make `netfilter-persistent` the managed persistence service.
- Add authoritative restore behavior with optional `--noflush` compatibility mode.
- Add explicit IPv4/IPv6 enable controls.
- Add semantic assertion tasks and complete role argument specification.
- Update examples and installation documentation.
