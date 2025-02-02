# Wireguard Ansible role

This Ansible role will deploy [Wireguard](https://www.wireguard.com/) VPN tunnel and setup peers.

[<img src="https://lablabs.io/static/ll-logo.png" width=350px>](https://lablabs.io/)

## Requirements

Wireguard Ansible Role requires root access rights with global `become: true`.

> Please note that this role will not setup additional routes or MASQUERADE and IPv4 Forwarding. This has to be done in a separate task(s) (see the [example playbook](#playbook-example)).

## Role Variables

This is a copy of `defaults/main.yml`.

```yaml
---
# Directory to store WireGuard configuration on the remote hosts
wireguard_dir: /etc/wireguard
wireguard_clients_dir: "{{ wireguard_dir }}/clients"

# Download client configs
wireguard_clients_download_dir: clients/
wireguard_download_clients: false

# Download private, public and preshared keys
wireguard_serverkeys_download_dir: server/
wireguard_download_serverkeys: false

# Path to Wireguard keys
wireguard_privatekey_path: "{{ wireguard_dir }}/privatekey"
wireguard_publickey_path: "{{ wireguard_dir }}/publickey"
wireguard_presharedkey_path: "{{ wireguard_dir }}/presharedkey"

# When defined, Ansible will restore wireguard keys (private key, public key, preshared key) from this directory.
# NOTE: The directory path must end with "/"
wireguard_restore_serverkeys_dir: ""

# Configure all servers with same wireguard keys.
wireguard_same_keys: false

wireguard_systemd_path: /etc/systemd/network

# Wireguard packages
wireguard_repo_url: "{{ _repo_url }}"
wireguard_distro_packages: "{{ _distro_packages }}"

wireguard_packages:
  - wireguard-dkms
  - wireguard-tools

# The default port WireGuard will listen if not specified otherwise.
wireguard_port: 51820

# Client destination Hostname
wireguard_hostname: "{{ inventory_hostname }}"

# The default interface name that wireguard should use if not specified otherwise.
wireguard_interface: wg0

# Base wireguard subnet
wireguard_address: 10.213.213.0/24

wireguard_server_ip: "{{ wireguard_address | ansible.utils.ipaddr('network') | ansible.utils.ipmath(1) }}"
wireguard_subnetmask: "{{ wireguard_address | ansible.utils.ipaddr('prefix') }}"
wireguard_peers_allowed_ips: "{{ ([(_wireguard_interface_addr | ansible.utils.ipaddr('network/prefix'))] + (wireguard_additional_routes | default([]))) | join(\", \") }}"

# This role works only with PrivateKeyFile/PresharedKeyFile.
wireguard_systemd_netdev:
  - NetDev:
      - Name: "{{ wireguard_interface }}"
      - Kind: wireguard
      - Description: "wireguard server: {{ wireguard_interface }} server on {{ wireguard_address }}"
  - WireGuard:
      - PrivateKey: "{{ _privkey_value['content'] | b64decode }}"
      - ListenPort: "{{ wireguard_port }}"

wireguard_systemd_network:
  - Match:
      - Name: "{{ wireguard_interface }}"
  - Network:
      - Address: "{{ wireguard_server_ip }}/{{ wireguard_subnetmask }}"
  - Route:
      - Destination: "{{ wireguard_address }}"
      - Gateway: "{{ wireguard_server_ip }}"

wireguard_keepalive: 25

# Additional IPs allowed to Wireguard
# (Following list of IPs will be added to Wireguard AllowedIPs)
wireguard_additional_routes: []

wireguard_peers: []
# - name: user1
#   allowed_ip: "10.213.213.2"
#   publickey: "asdasdasdadsasdasd"
# - name: user2
#   allowed_ip: "10.213.213.3/32"
#   publickey: "000000000000000000"
#   keepalive: 30
# - name: user3
#   allowed_ip: "10.213.213.50/30"
#   publickey: "111111111111111111"

# Test Wirequard public keys and find possible errors
# This will check the lenght of the key (44 characters) and test if it's a valid base64 string.
run_publickey_pre_check: true
```

## Playbook example

This playbook will run Wireguard Ansible Role to deploy Wireguard server and configure 3 users/peers.
It will also download the client configs into the specified directory. Remaining 4 tasks will set Masquarade and IPv4 Forwarding.

```yaml
---
- name: Wireguard deployment
  hosts: wireguard
  user: ubuntu
  become: true
  gather_facts: true
  vars:
    wireguard_clients_download_dir: /my_clients/
    wireguard_download_clients: true
    wireguard_peers:
      - name: user1
        allowed_ip: "10.213.213.2"
        publickey: "asdasdasdadsasdasd"
      - name: user2
        allowed_ip: "10.213.213.3/32"
        publickey: "000000000000000000"
        keepalive: 30
      - name: user3
        allowed_ip: "10.213.213.4/30"
        publickey: "111111111111111111"

  tasks:
  - import_role:
      name: lablabs.wireguard.wireguard

  - name: Install iptables-persistent
    ansible.builtin.package:
      name:
        - iptables
        - iptables-persistent
      state: present

  - name: Setup MASQUERADE for server access through vpn server
    community.general.iptables:
      chain: POSTROUTING
      jump: MASQUERADE
      table: nat
      source: "{{ wireguard_address }}"
      out_interface: "{{ wireguard_out_interface }}"

  - name: Save current state of the firewall in system file
    community.general.iptables_state:
      state: saved
      path: /etc/iptables/rules.v4

  - name: Setup ipv4 IP forward
    ansible.posix.sysctl:
      name: net.ipv4.ip_forward
      value: 1
      sysctl_set: true
      state: present
      reload: true

```

## License

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

See [LICENSE](LICENSE) for full details.

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.

## Author Information

Created in 2021 by [Labyrinth Labs](https://www.lablabs.io/)
