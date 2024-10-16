# iptables

## Description

Ansible role for iptables setup.

## Settings

### Enable ICMP

``` yaml
accept_icmp: { v4: true, v6: true }
```

Creates typical ICMP rules for IPv4 and IPv6.

### Supported chains

- `input`. INPUT chain for input traffic
- `enemy_input`. enemy_input chain for input traffic
- `output`. OUTPUT chain for outgoing traffic
- `docker_forward`. docker_forward chain for forward traffic
- `dnat`. nat table. DNAT rules
- `snat`. nat table. SNAT rules
- `masq`. nat table. MASQUERADE rules
- `postrouting`. postrouting chain, nat table.
- `forward`. filter, mangle tables.

Any custom chains for filter table can be described via `custom_filter_chains` list. For example:

``` yaml
custom_filter_chains:
  some_chain:
    v4:
      - { dport: 80, comment: "custom chain rule" }
    v6:
      - { dport: 80, comment: "custom chain rule" }
````

### Changing policies

Performed via `ipv4_policies` and `ipv6_policies`, for example:

``` yaml
ipv4_policies:
  filter:
    forward: drop
ipv6_policies:
  nat:
    prerouting: drop
```

By default everything is ACCEPT.

### Common settings for rules

| Setting | Default | iptables | Description |
| --- | --- | --- | --- |
| `i: eth1` | - | `-i eth1` | input interface |
| `ni: eth2` | - | `! -i eth2` | **not** input interface |
| `dport: 1234, p: tcp` | `p:tcp` if dport is defined | `-p tcp -m tcp --dport 1234` | port and protocol |
| `p: udp` | - | `-p udp` | protocol only |
| `dport: "80,443"` | `p:tcp` if dport is defined | `-p tcp -m multiport --dports 80,443` | multiport. doesn't work for snat/masq rules |
| `s: 123.123.123.123` | - | `-s 123.123.123.123` | source address |
| `ns: 123.123.123.123` | - | `! -s 123.123.123.123` | **not** source address |
| `d: 234.234.234.234` | - | `-d 234.234.234.234` | destination address |
| `nd: 234.234.234.234` | - | `! -d 234.234.234.234` | **not** destination address |
| `state: NEW` | - | `-m state --state NEW` | connection state |
| `list: v4` | - | `-m set --match-set v4 src` | ipset list |
| `nfacct: http-v4` | - | `-m nfacct --nfacct-name http-v4` | nfacct counter |
| `comment: "test rule"` | - | `-m comment --comment "test rule"` | rule comment |

These settings work for any supported chain.

### Settings for input and docker_forward

| Setting | Default | iptables | Description |
| --- | --- | --- | --- |
| `action: DROP` | `ACCEPT` | `-j DROP` | action for matched packet |

enemy_input examples:

| role rule |iptables rule |
| --- | --- |
| `{ s: 234.234.234.234, action: DROP }` | `-A enemy_input -s 234.234.234.234 -j DROP` |
| `{ i: eth0, ni: eth1, dport: 80, s: 123.123.123.0/24, comment: 'test rule' }` | `-A enemy_input -i eth0 ! -i eth1 -p tcp -m tcp --dport 80 -s 123.123.123.0/24 -m comment --comment "test rule"` |

docker_forward examples:

| role rule | iptables rule |
| --- | --- |
| `{ p: tcp, drport 443 }` | `-A docker_forward -p tcp -m tcp --dport 443 -j ACCEPT` |
| `{ action: DROP' }` | `-A docker_forward -j DROP` |

### Settings for dnat

| Setting | Default | iptables | Description |
| --- | --- | --- | --- |
| `o: eth1` | - | `-o eth1` | output interface |
| `dport: 61000-62000` | `p:tcp` if dport is defined | `-p tcp -m tcp --dport 61000:62000` | same as common but with port range support |
| `dstaddr: 10.10.10.10` | required | `-j DNAT --to-destination 10.10.10.10` | destination address |
| `dstport: 61000-62000` | - | `-j DNAT --to-destination 10.10.10.10:61000-62000` | destination port or port range |

### Settings for snat

| Setting | Default | iptables | Description |
| --- | --- | --- | --- |
| `o: eth1` | - | `-o eth1` | output interface |
| `srcaddr: 123.123.123.123` | required | `-j SNAT --to-source 123.123.123.123` | source address |

### Settings for masq

| Setting|Default|iptables|Description |
| ---|---|---|---|
| `o: eth1` | - | `-o eth1` | output interface |

example:

| role rule | iptables rule |
| --- | --- |
| `{ s: 10.10.10.0/24, nd: 10.10.10.0/24 }` | `-A PREROUTING -s 10.10.10.0/24 ! -d 10.10.10.0/24 -j MASQUERADE` |

### Disable interface check

``` yaml
disable_interface_check: true
```

### Settings for modules policy, ipv6header, tcpmss

| Setting | Default | iptables | Description |
| --- | --- | --- | --- |
| `m: policy` | - | `-m policy --dir in --pol ipsec` | policy module, you can override default `--dir` with `dir:` and `--pol` with `pol:` |
| `reqid: 1` | - | `--reqid 1` | matches the reqid of the policy rule |
| `m: ipv6header` | - | `-m ipv6header --header esp` | ipv6header module, you can override `--header` with `header:` |
| `mss: 123` | - | `-m tcpmss --mss 123` | tcpmss module |
| `action: TCPMSS` | - | `-j TCPMSS --clamp-mss-to-pmtu` | TCPMSS action, instead `--clamp-mss-to-pmtu` you can set `--set-mss` with `setmss:` |

## Usage example

``` yaml
- role: iptables
  ext_ifaces: [eno1, wlp1s0]
  accept_icmp: { v4: true, v6: true }
  input:
    v4:
      - { dport: 80, nfacct: "http-v4" }
    v6:
      - { dport: 80, nfacct: "http-v6" }
  enemy_input:
    v4:
      - { list: v4 }
      - { dport: 22, comment: "ssh host for clients access" }
    v6:
      - { list: v6 }
  docker_forward:
    v4:
      - { state: 'RELATED,ESTABLISHED' }
      - { dport: 80, comment: "nginx container" }
      - { dport: 443, comment: "nginx container" }
      - { p: tcp, dport: 5432, m: policy, dir: in, pol: ipsec, reqid: 1 }
      - { action: DROP }
    v6:
      - { state: 'RELATED,ESTABLISHED' }
      - { action: DROP }
  dnat:
    v4:
      - { dport: 8888, dstaddr: 10.10.10.10, comment: "dnat for smth", dstport: 8888 }
  snat:
    v4:
      - { s: 10.10.10.20, nd: 10.10.10.0/24, comment: "snat for smth", srcaddr: 123.123.123.123 }
  masq:
    v4:
      - { s: 10.10.10.0/24, nd: 10.10.10.0/24, comment: "default masq rule" }
  postrouting:
    v6:
      - { o: eth0, m: policy, dir: out }
  mangle_forward:
    v4:
      - { o: eth0, p: tcp, flags: 'SYN,RST', mss: '1361:1536', setmss: 1360, action: TCPMSS }
```
